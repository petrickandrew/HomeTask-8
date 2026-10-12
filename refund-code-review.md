# Code Review — `refund-with-bugs.ts`

> **Контекст.** Модуль виконує повернення коштів (refund) через Stripe і оновлює статус транзакції в БД. Помилка тут = гроші мерчанта/клієнта розходяться з реальністю, а виявляється це зазвичай під час звірки (reconciliation) або зі скарги клієнта.
>
> **Примітка щодо формату.** Шаблон завдання описує `velocity-buggy.ts`, але аналізується `refund-with-bugs.ts`, тому таблиці та приклади адаптовано під refund-логіку. Мова тестів — **TypeScript/Jest** (дозволено інструкцією; код і тести в одній мові, тести реально запускаються проти файлу).

---

## 0. Розбір коду рядок за рядком (крок 01)

| Функція | Що робить | Чого бракує |
|---|---|---|
| `new Stripe("sk_test_HARDCODED_KEY")` | Створює клієнт Stripe з ключем у коді | Ключ береться не зі змінних середовища/секрет-менеджера |
| `processRefund` | `SELECT` → перевірка `status === "captured"` → `stripe.refunds.create` → `UPDATE status='refunded'` | Валідації `amount`, перевірки `tx` на `null`, блокування, idempotency key, аналізу відповіді Stripe, обробки часткових повернень |
| `batchRefund` | `Promise.all` по списку з `amount = 0` | Сума повного повернення, ліміт паралелізму, per-item результат |
| `refundWithRetry` | 3 спроби з паузою 1 с | Класифікації помилок, backoff, `throw` після останньої спроби, idempotency |
| `refundAndVerify` | `processRefund` → `sleep(5000)` → читання БД | Осмисленої перевірки (sleep нічого не гарантує) |

**Що код вважає «правдою» за замовчуванням:** що `tx` існує; що `amount` коректний; що між `SELECT` і `UPDATE` ніхто не втрутиться; що Stripe-відповідь завжди «успіх»; що `UPDATE` завжди виконається після успішного Stripe-виклику.

**Що буде, якщо БД/Stripe недоступні:** `tx === null` → `TypeError`; помилка на `UPDATE` після успішного Stripe → гроші повернуті, у БД статус `captured`, а повторний виклик повертає гроші ще раз.

---

## Зведена таблиця знайдених проблем

| # | Проблема | Серйозність |
|---|---|---|
| 1 | Hardcoded Stripe secret key | 🔴 Critical |
| 2 | Немає валідації `amount` / `tx`; після часткового повернення статус `refunded` | 🔴 High |
| 3 | Race condition (TOCTOU) → подвійний refund | 🔴 High |
| 4 | Немає idempotency key, retry після часткового успіху → подвійний refund | 🔴 High |
| 5 | `refundWithRetry` ковтає помилку після 3 спроб + ретраїть нересайтабельні помилки | 🟠 High |
| 6 | `batchRefund`: `amount = 0` і `Promise.all` | 🟠 Medium/High |
| 7 | Ігнорується відповідь Stripe (`refund.status`) | 🟠 Medium/High |
| 8 | `refundAndVerify`: `sleep(5000)` замість реальної перевірки | 🟡 Medium |

---

## Проблема 1 · Hardcoded Stripe secret key

| ПОЛЕ | ВІДПОВІДЬ |
|---|---|
| **Що не так** | Секретний ключ `sk_...` лежить у вихідному коді (`new Stripe("sk_test_HARDCODED_KEY")`). Він потрапляє в git-історію, CI-логи, образи, до всіх, хто має доступ до репозиторію. |
| **Ризик для бізнесу** | Хто отримав ключ, може створювати refund-и на довільні суми й читати дані платежів. Це фінансова втрата + порушення PCI DSS, обов'язкова ротація ключа, інцидент-репорт. Помітить зазвичай моніторинг/звірка зі Stripe Dashboard, а не код-рев'ю. |
| **Тест-кейс** | Вхід: текст файлу `refund-with-bugs.ts` → дія: пошук патерну `sk_(test\|live)_...` → очікування: збігів немає. **Падає на buggy**, бо ключ у коді. |
| **Виправлений код** | Ключ із `process.env.STRIPE_SECRET_KEY` (або secret manager), ледача ініціалізація клієнта, падіння з зрозумілою помилкою, якщо ключ відсутній. Скомпрометований ключ — відкликати й випустити новий. |

**Тест:**

```ts
describe("P1: секрети", () => {
  it("у коді немає захардкодженого Stripe-ключа", () => {
    const src = fs.readFileSync(path.join(__dirname, "refund-with-bugs.ts"), "utf8");
    expect(src).not.toMatch(/sk_(test|live)_\w+/);
  });
});
```

**Фікс:**

```ts
let stripeClient: Stripe | undefined;
function getStripe(): Stripe {
  if (!stripeClient) {
    const key = process.env.STRIPE_SECRET_KEY;
    if (!key) throw new Error("STRIPE_SECRET_KEY is not set");
    stripeClient = new Stripe(key);
  }
  return stripeClient;
}
```

---

## Проблема 2 · Немає валідації `amount` і `tx`; неправильний статус після часткового refund

| ПОЛЕ | ВІДПОВІДЬ |
|---|---|
| **Що не так** | (а) `amount` не перевіряється: `0`, від'ємне, дробове (Stripe приймає цілі в мінімальних одиницях — центи/копійки), більше за суму платежу або за залишок. (б) `tx` не перевіряється на `null` → `TypeError: Cannot read properties of null` замість зрозумілої помилки. (в) Після будь-якого refund статус ставиться `refunded`, а `refund_amount` **перезаписується**, а не накопичується: після часткового повернення $3 з $10 транзакція виглядає як повністю повернена, повторний refund неможливий, а облік «скільки повернуто» втрачено. |
| **Ризик для бізнесу** | Помилки в обліку refund-ів, розбіжність БД зі Stripe, ручна звірка. Через `null`-tx оператор бачить «500» без причини. Якщо ліміти Stripe ослаблені/змінені, є ризик повернути більше, ніж отримано. |
| **Тест-кейс** | (1) `tx.amount=1000`, `processRefund(tx, 5000 / 0 / -100 / 10.5)` → **очікується reject** і `stripe.refunds.create` не викликано. На buggy `create` викликається і функція завершується успіхом. (2) `db.query` повертає `null` → очікується помилка `/not found/i`. На buggy — `TypeError` без цього тексту. (3) Refund 300 із 1000 → статус `partially_refunded`, `refund_amount=300`; ще 700 → `refunded`, `refund_amount=1000`. На buggy після першого refund статус `refunded`. |
| **Виправлений код** | Перевірка існування, дозволених статусів (`captured`, `partially_refunded`), `Number.isInteger(amount) && amount > 0 && amount <= remaining`, накопичення `refund_amount`, статус залежно від того, чи повернуто все. |

**Тест:**

```ts
describe("P2: валідація і часткові повернення", () => {
  it.each([0, -100, 1001, 10.5])("відхиляє некоректну суму %p", async (bad) => {
    fakeDb([{ id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" }]);
    await expect(processRefund("tx1", bad)).rejects.toThrow();
    expect(mockCreate).not.toHaveBeenCalled();
  });

  it("зрозуміла помилка, якщо транзакції немає", async () => {
    jest.spyOn(db, "query").mockResolvedValue(null);
    await expect(processRefund("nope", 100)).rejects.toThrow(/not found/i);
  });

  it("часткові повернення накопичуються", async () => {
    const table = fakeDb([{ id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" }]);
    mockCreate.mockResolvedValue({ id: "re_1", status: "succeeded" });

    await processRefund("tx1", 300);
    expect(table.get("tx1")).toMatchObject({ status: "partially_refunded", refund_amount: 300 });

    await processRefund("tx1", 700);
    expect(table.get("tx1")).toMatchObject({ status: "refunded", refund_amount: 1000 });
  });
});
```

**Фікс (фрагмент):**

```ts
const tx: Transaction | null = await db.query("SELECT * FROM transactions WHERE id = $1", [txId]);
if (!tx) throw new NonRetryableRefundError(`Transaction ${txId} not found`);
if (tx.status !== "captured" && tx.status !== "partially_refunded")
  throw new NonRetryableRefundError(`Refund not allowed in state ${tx.status}`);

const alreadyRefunded = tx.refund_amount ?? 0;
const remaining = tx.amount - alreadyRefunded;
if (!Number.isInteger(amount) || amount <= 0 || amount > remaining)
  throw new NonRetryableRefundError(`Invalid refund amount ${amount}, remaining ${remaining}`);

// ... після успіху:
const total = alreadyRefunded + amount;
await db.query(
  "UPDATE transactions SET status = $1, refund_amount = $2 WHERE id = $3",
  [total === tx.amount ? "refunded" : "partially_refunded", total, txId]
);
```

---

## Проблема 3 · Race condition (check-then-act) → подвійний refund

| ПОЛЕ | ВІДПОВІДЬ |
|---|---|
| **Що не так** | Між `SELECT` + перевіркою `status === "captured"` і `UPDATE` немає блокування. Два паралельні запити (подвійний клік оператора, retry клієнта API, два воркери) обидва бачать `captured` і обидва викликають Stripe. |
| **Ризик для бізнесу** | Клієнту повертають гроші двічі (доти, доки Stripe не відхилить через вичерпання суми; для часткових refund-ів — взагалі без обмеження). Прямий збиток мерчанта; помічає фінансовий відділ під час звірки. |
| **Тест-кейс** | Транзакція `captured`, `amount=1000`. Запустити два `processRefund("tx1", 1000)` одночасно (`Promise.allSettled`) → очікується рівно **один** виклик `stripe.refunds.create`. На buggy — два. |
| **Виправлений код** | Атомарний «захват» рядка (compare-and-swap): `UPDATE ... SET status='refund_pending' WHERE id=$1 AND status=$2 AND COALESCE(refund_amount,0)=$3 RETURNING *`. Якщо рядок не повернувся — хтось інший уже обробляє refund. Альтернатива: `SELECT ... FOR UPDATE` у транзакції БД. |

**Тест:**

```ts
describe("P3: конкурентні refund-и", () => {
  it("два паралельні виклики → один виклик Stripe", async () => {
    fakeDb([{ id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" }]);
    mockCreate.mockResolvedValue({ id: "re_1", status: "succeeded" });

    await Promise.allSettled([processRefund("tx1", 1000), processRefund("tx1", 1000)]);

    expect(mockCreate).toHaveBeenCalledTimes(1);
  });
});
```

**Фікс (фрагмент):**

```ts
const claimed = await db.query(
  "UPDATE transactions SET status = 'refund_pending' " +
  "WHERE id = $1 AND status = $2 AND COALESCE(refund_amount, 0) = $3 RETURNING *",
  [txId, tx.status, alreadyRefunded]
);
if (!claimed) throw new NonRetryableRefundError("Refund already in progress");
```

---

## Проблема 4 · Немає idempotency key; retry після часткового успіху → подвійний refund

| ПОЛЕ | ВІДПОВІДЬ |
|---|---|
| **Що не так** | `stripe.refunds.create` викликається без `idempotencyKey`. Сценарій: Stripe виконав refund, але відповідь загубилась (timeout) або `UPDATE` у БД упав → `refundWithRetry` повторює виклик; у БД усе ще `captured`, перевірка проходить, Stripe створює **другий** refund. Виклик Stripe і запис у БД не атомарні, і код цього не враховує. |
| **Ризик для бізнесу** | Подвійні виплати клієнтам, особливо при мережевих збоях (саме тоді retry і спрацьовує). Розбіжність «у Stripe повернуто, у БД ні». |
| **Тест-кейс** | Перший `create` падає `StripeConnectionError`, другий успішний → `refundWithRetry`. Очікується: обидва виклики мають **однаковий** `idempotencyKey`. На buggy ключа немає (`undefined`). |
| **Виправлений код** | Передавати стабільний ключ: `refund:{txId}:{alreadyRefunded}:{amount}`. Ключ однаковий для повторів одного й того самого refund-у і змінюється після успішного (бо змінюється `alreadyRefunded`), тож навмисний другий refund тієї ж суми теж можливий. Для випадку «Stripe успіх, БД упала» — рядок залишається `refund_pending`, його добиває webhook `refund.updated` / reconciliation-job. |

**Тест:**

```ts
describe("P4: idempotency при retry", () => {
  it("повтор використовує той самий idempotencyKey", async () => {
    fakeDb([{ id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" }]);
    mockCreate
      .mockRejectedValueOnce(Object.assign(new Error("network"), { type: "StripeConnectionError" }))
      .mockResolvedValueOnce({ id: "re_1", status: "succeeded" });

    await refundWithRetry("tx1", 500);

    expect(mockCreate).toHaveBeenCalledTimes(2);
    const keys = mockCreate.mock.calls.map((c) => c[1]?.idempotencyKey);
    expect(keys[0]).toBeDefined();
    expect(keys[0]).toBe(keys[1]);
  });
});
```

**Фікс (фрагмент):**

```ts
refund = await getStripe().refunds.create(
  { payment_intent: tx.stripe_pi_id, amount },
  { idempotencyKey: `refund:${txId}:${alreadyRefunded}:${amount}` }
);
```

> Ключі ідемпотентності Stripe зберігаються ~24 години — для довших відкладених ретраїв потрібен reconciliation.

---

## Проблема 5 · `refundWithRetry` ковтає помилку і ретраїть усе підряд

| ПОЛЕ | ВІДПОВІДЬ |
|---|---|
| **Що не так** | (а) Після 3-ї невдалої спроби функція **завершується без помилки** (`catch` нічого не кидає) → виклик вважається успішним, хоча refund не виконано. (б) Ретраїться все: «транзакція не captured», невалідна сума, відхилена картка — повторювати це безглуздо. (в) Фіксована пауза 1 с, без backoff; зайвий `sleep` навіть після останньої спроби. (г) У лог не потрапляє причина помилки. |
| **Ризик для бізнесу** | Клієнт думає, що отримав refund, система/саппорт — теж; насправді гроші не повернуті → скарги, chargeback-и, штрафи. Тихий фейл виявляється, лише коли клієнт напише в підтримку. |
| **Тест-кейс** | (1) Stripe завжди падає `StripeConnectionError` → `refundWithRetry` має **reject**. На buggy — resolve. (2) Сума більша за залишок (помилка валідації) → reject одразу, `SELECT` виконано 1 раз (не 3). На buggy — 3 спроби й тихий успіх. |
| **Виправлений код** | Ретраїти лише тимчасові помилки (`StripeConnectionError`, `StripeAPIError`, `StripeRateLimitError`), експоненційний backoff, після вичерпання спроб `throw`, логувати причину. |

**Тест:**

```ts
describe("P5: refundWithRetry", () => {
  it("кидає помилку після вичерпання спроб", async () => {
    fakeDb([{ id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" }]);
    mockCreate.mockRejectedValue(Object.assign(new Error("down"), { type: "StripeConnectionError" }));
    await expect(refundWithRetry("tx1", 500)).rejects.toThrow();
  });

  it("не ретраїть помилки валідації", async () => {
    fakeDb([{ id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" }]);
    await expect(refundWithRetry("tx1", 5000)).rejects.toThrow();
    const selects = (db.query as jest.Mock).mock.calls.filter(([sql]) => sql.startsWith("SELECT"));
    expect(selects).toHaveLength(1);
  });
});
```

**Фікс (фрагмент):**

```ts
const RETRYABLE = new Set(["StripeConnectionError", "StripeAPIError", "StripeRateLimitError"]);

export async function refundWithRetry(txId: string, amount: number, maxAttempts = 3) {
  for (let attempt = 1; ; attempt++) {
    try {
      return await processRefund(txId, amount);
    } catch (error: any) {
      const retryable = !(error instanceof NonRetryableRefundError) && RETRYABLE.has(error?.type);
      if (!retryable || attempt >= maxAttempts) throw error;
      console.warn(`Refund ${txId}: attempt ${attempt} failed: ${error.message}`);
      await new Promise((r) => setTimeout(r, 500 * 2 ** (attempt - 1)));
    }
  }
}
```

---

## Проблема 6 · `batchRefund`: `amount = 0` і `Promise.all`

| ПОЛЕ | ВІДПОВІДЬ |
|---|---|
| **Що не так** | (а) Для кожної транзакції передається `amount = 0` (є навіть коментар `// amount=0 ???`). Stripe відхиляє `amount: 0` — пакетний refund не повертає нічого, а якщо «виправити» простим видаленням суми, повернеться все без перевірок. (б) `Promise.all` — fail-fast: перша помилка відхиляє весь результат, а решта refund-ів уже виконуються у фоні, і викликач не знає, які з них пройшли. (в) Необмежений паралелізм на великому списку впирається в rate-limit Stripe. |
| **Ризик для бізнесу** | Масові повернення (наприклад, скасування події) «наполовину» виконані без звіту: частина клієнтів отримала гроші, частина ні, а оператор не знає хто. Ручне розслідування сотень транзакцій. |
| **Тест-кейс** | (1) Дві `captured` транзакції по 1000 → `batchRefund(["tx1","tx2"])` → очікується `create` із `amount: 1000` для кожної. На buggy — `amount: 0`. (2) Stripe падає для `tx2` → очікується звіт `tx1: ok`, `tx2: failed`. На buggy `batchRefund` кидає виняток без розбивки по транзакціях. |
| **Виправлений код** | Рахувати суму повного повернення (`amount − refund_amount`), `Promise.allSettled`, обмеження паралелізму (чанки), повертати результат по кожній транзакції. |

**Тест:**

```ts
describe("P6: batchRefund", () => {
  const rows = [
    { id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" },
    { id: "tx2", stripe_pi_id: "pi_2", amount: 1000, status: "captured" },
  ];

  it("повертає повну суму кожної транзакції, а не 0", async () => {
    fakeDb(rows);
    mockCreate.mockResolvedValue({ id: "re_x", status: "succeeded" });

    await batchRefund(["tx1", "tx2"]);

    expect(mockCreate).toHaveBeenCalledWith(
      expect.objectContaining({ payment_intent: "pi_1", amount: 1000 }), expect.anything());
    expect(mockCreate).toHaveBeenCalledWith(
      expect.objectContaining({ payment_intent: "pi_2", amount: 1000 }), expect.anything());
  });

  it("повертає звіт по кожній транзакції, а не падає на першій помилці", async () => {
    fakeDb(rows);
    mockCreate.mockImplementation(async (params: any) => {
      if (params.payment_intent === "pi_2")
        throw Object.assign(new Error("card_declined"), { type: "StripeInvalidRequestError" });
      return { id: "re_ok", status: "succeeded" };
    });

    const res = await batchRefund(["tx1", "tx2"]);

    expect(res.find((r) => r.txId === "tx1")?.ok).toBe(true);
    expect(res.find((r) => r.txId === "tx2")?.ok).toBe(false);
  });
});
```

**Фікс (фрагмент):**

```ts
export interface BatchResult { txId: string; ok: boolean; error?: string }

export async function batchRefund(txIds: string[], concurrency = 5): Promise<BatchResult[]> {
  const results: BatchResult[] = [];
  for (let i = 0; i < txIds.length; i += concurrency) {
    const chunk = txIds.slice(i, i + concurrency);
    const settled = await Promise.allSettled(
      chunk.map(async (id) => {
        const tx: Transaction | null = await db.query("SELECT * FROM transactions WHERE id = $1", [id]);
        if (!tx) throw new NonRetryableRefundError(`Transaction ${id} not found`);
        return refundWithRetry(id, tx.amount - (tx.refund_amount ?? 0));
      })
    );
    settled.forEach((r, idx) =>
      results.push({
        txId: chunk[idx],
        ok: r.status === "fulfilled",
        error: r.status === "rejected" ? String(r.reason?.message ?? r.reason) : undefined,
      })
    );
  }
  return results;
}
```

---

## Проблема 7 · Ігнорується відповідь Stripe (`refund.status`)

| ПОЛЕ | ВІДПОВІДЬ |
|---|---|
| **Що не так** | Змінна `refund` ніде не використовується. Refund у Stripe може бути `pending`, `requires_action`, `failed`, `canceled`; код у будь-якому випадку пише в БД `refunded`. |
| **Ризик для бізнесу** | БД каже «повернуто», хоча гроші клієнту не пішли (наприклад, закрита картка → `failed`). Клієнт скаржиться, підтримка бачить «refunded» і не може допомогти; потрібна ручна звірка з виписками Stripe. |
| **Тест-кейс** | `stripe.refunds.create` повертає `{ status: "failed" }` → `processRefund` має reject, а статус у БД лишається `captured`. На buggy — успіх і статус `refunded`. |
| **Виправлений код** | Аналізувати `refund.status`: `failed/canceled` → відкотити стан і кинути помилку; `pending/requires_action` → залишити `refund_pending` до webhook; `succeeded` → фіналізувати. |

**Тест:**

```ts
describe("P7: статус refund зі Stripe", () => {
  it("failed refund не помічається як refunded", async () => {
    const table = fakeDb([{ id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" }]);
    mockCreate.mockResolvedValue({ id: "re_1", status: "failed" });

    await expect(processRefund("tx1", 1000)).rejects.toThrow();
    expect(table.get("tx1")?.status).toBe("captured");
  });
});
```

**Фікс (фрагмент):**

```ts
if (refund.status === "failed" || refund.status === "canceled") {
  await rollback();
  throw new Error(`Stripe refund ${refund.id} ${refund.status}`);
}
if (refund.status === "pending" || refund.status === "requires_action") {
  return { refundId: refund.id, status: refund.status }; // лишається refund_pending, фіналізує webhook
}
```

---

## Проблема 8 · `refundAndVerify`: `sleep(5000)` замість реальної перевірки

| ПОЛЕ | ВІДПОВІДЬ |
|---|---|
| **Що не так** | `processRefund` уже виконано (`await`), тож 5 секунд очікування нічого не гарантує і лише гальмує виклик (на пакеті з 100 refund-ів — зайві хвилини). «Перевірка» дивиться тільки в нашу БД, а не на реальний стан у Stripe; якщо читання йде з репліки з лагом більше 5 с, функція дає хибну помилку «not reflected», хоча refund у Stripe вже виконано, і оператор повторює його. Повідомлення про помилку не містить ні `refund id`, ні поточного статусу. |
| **Ризик для бізнесу** | Хибні фейли → ручні повтори → подвійні refund-и; повільні операції та тайм-аути API. |
| **Тест-кейс** | Виклик `refundAndVerify("tx1", 1000)` при успішному refund → очікується, що жодного таймера на 5000 мс не створено. На buggy `setTimeout(..., 5000)` викликається. |
| **Виправлений код** | Прибрати sleep; використовувати результат `processRefund` (`refundId`, `status`) і одразу звіряти стан БД; у помилці вказувати `refundId` і поточний статус. Для затримок реплікації — retry читання з короткими інтервалами, а не фіксований sleep. |

**Тест:**

```ts
describe("P8: refundAndVerify", () => {
  it("не чекає фіксовані 5 секунд", async () => {
    fakeDb([{ id: "tx1", stripe_pi_id: "pi_1", amount: 1000, status: "captured" }]);
    mockCreate.mockResolvedValue({ id: "re_1", status: "succeeded" });

    await refundAndVerify("tx1", 1000);

    expect(timerSpy).not.toHaveBeenCalledWith(expect.any(Function), 5000);
  });
});
```

**Фікс (фрагмент):**

```ts
export async function refundAndVerify(txId: string, amount: number): Promise<void> {
  const result = await processRefund(txId, amount);
  const tx: Transaction | null = await db.query("SELECT * FROM transactions WHERE id = $1", [txId]);
  if (!tx) throw new Error(`Transaction ${txId} not found after refund`);

  const ok = result.status === "pending"
    ? tx.status === "refund_pending"
    : tx.status === "refunded" || tx.status === "partially_refunded";
  if (!ok) throw new Error(`Refund ${result.refundId} not reflected in DB (status=${tx.status})`);
}
```

---

## Додаткові зауваження (дрібніші)

- `SELECT *` — краще явний список колонок.
- `any` у `db.query` — типізувати результат (`Promise<Transaction | null>`).
- `console.log` → структурований логер з `txId`, `refundId`, `attempt`.
- **Припущення про `db.query`:** у коді його результат використовується як один рядок (`const tx: Transaction = await db.query(...)`). Якщо реальний драйвер повертає масив рядків, треба брати `rows[0]`. У фіксах збережено оригінальний контракт «один рядок або `null`».
- Для рядків, що залишилися в `refund_pending` (збій між Stripe і БД), потрібна фонова звірка зі Stripe (`refunds.list` / webhook `refund.updated`).
- Ключ із коду вважається скомпрометованим → **ротація**.

---

## Додаток A · Тестовий каркас (спільний для всіх тестів)

> Щоб тести могли імпортувати модуль, у `refund-with-bugs.ts` додається `export` до функцій та `db` (мінімальна зміна для тестованості, без зміни логіки).
> Файл: `refund.test.ts` поруч з `refund-with-bugs.ts`. Запуск: `npx jest`.

```ts
import * as fs from "fs";
import * as path from "path";

const mockCreate = jest.fn();
jest.mock("stripe", () => ({
  __esModule: true,
  default: function () {
    return { refunds: { create: (...args: unknown[]) => mockCreate(...args) } };
  },
}));

import { processRefund, batchRefund, refundWithRetry, refundAndVerify, db } from "./refund-with-bugs";

type Row = { id: string; stripe_pi_id: string; amount: number; status: string; refund_amount?: number };

// Мінімальна in-memory "БД": SELECT, атомарний захват (refund_pending) і звичайний UPDATE
function fakeDb(rows: Row[]) {
  const table = new Map(rows.map((r) => [r.id, { ...r }]));
  jest.spyOn(db, "query").mockImplementation(async (sql: string, params: any[]) => {
    await Promise.resolve(); // віддає керування, щоб конкурентні виклики перемежовувались
    if (sql.trim().startsWith("SELECT")) {
      const r = table.get(params[0]);
      return r ? { ...r } : null;
    }
    if (sql.includes("'refund_pending'")) {
      const r = table.get(params[0]);
      if (!r || r.status !== params[1] || (r.refund_amount ?? 0) !== params[2]) return null;
      r.status = "refund_pending";
      return { ...r };
    }
    if (sql.trim().startsWith("UPDATE")) {
      const r = table.get(params[2])!;
      r.status = params[0];
      r.refund_amount = params[1];
      return null;
    }
    throw new Error("unexpected SQL: " + sql);
  });
  return table;
}

let timerSpy: jest.SpyInstance;

beforeAll(() => {
  process.env.STRIPE_SECRET_KEY = "sk_test_dummy_for_tests";
});

beforeEach(() => {
  mockCreate.mockReset();
  // прибирає реальні паузи (retry/backoff) у тестах
  timerSpy = jest.spyOn(global, "setTimeout").mockImplementation(((fn: () => void) => {
    fn();
    return 0 as any;
  }) as any);
});

afterEach(() => jest.restoreAllMocks());

// ... далі describe-блоки P1–P8 з розділів вище
```

---

## Додаток B · Повний виправлений код

```ts
// refund-fixed.ts
import Stripe from "stripe";

export interface Transaction {
  id: string;
  stripe_pi_id: string;
  amount: number;          // мінімальні одиниці (центи/копійки)
  status: string;          // captured | refund_pending | partially_refunded | refunded
  refund_amount?: number;
}

export interface RefundResult { refundId: string; status: string }
export interface BatchResult { txId: string; ok: boolean; error?: string }

export class NonRetryableRefundError extends Error {}

const RETRYABLE = new Set(["StripeConnectionError", "StripeAPIError", "StripeRateLimitError"]);

let stripeClient: Stripe | undefined;
function getStripe(): Stripe {
  if (!stripeClient) {
    const key = process.env.STRIPE_SECRET_KEY;
    if (!key) throw new Error("STRIPE_SECRET_KEY is not set");
    stripeClient = new Stripe(key);
  }
  return stripeClient;
}

export async function processRefund(txId: string, amount: number): Promise<RefundResult> {
  const tx: Transaction | null = await db.query(
    "SELECT * FROM transactions WHERE id = $1", [txId]
  );
  if (!tx) throw new NonRetryableRefundError(`Transaction ${txId} not found`);
  if (tx.status !== "captured" && tx.status !== "partially_refunded")
    throw new NonRetryableRefundError(`Refund not allowed in state ${tx.status}`);

  const alreadyRefunded = tx.refund_amount ?? 0;
  const remaining = tx.amount - alreadyRefunded;
  if (!Number.isInteger(amount) || amount <= 0 || amount > remaining)
    throw new NonRetryableRefundError(`Invalid refund amount ${amount}, remaining ${remaining}`);

  // Атомарний захват (compare-and-swap) — захист від race condition
  const claimed = await db.query(
    "UPDATE transactions SET status = 'refund_pending' " +
    "WHERE id = $1 AND status = $2 AND COALESCE(refund_amount, 0) = $3 RETURNING *",
    [txId, tx.status, alreadyRefunded]
  );
  if (!claimed) throw new NonRetryableRefundError("Refund already in progress");

  const rollback = () =>
    db.query(
      "UPDATE transactions SET status = $1, refund_amount = $2 WHERE id = $3",
      [tx.status, alreadyRefunded, txId]
    );

  let refund: Stripe.Refund;
  try {
    refund = await getStripe().refunds.create(
      { payment_intent: tx.stripe_pi_id, amount },
      { idempotencyKey: `refund:${txId}:${alreadyRefunded}:${amount}` }
    );
  } catch (e) {
    await rollback();
    throw e;
  }

  if (refund.status === "failed" || refund.status === "canceled") {
    await rollback();
    throw new Error(`Stripe refund ${refund.id} ${refund.status}`);
  }
  if (refund.status === "pending" || refund.status === "requires_action") {
    return { refundId: refund.id, status: refund.status }; // фіналізує webhook
  }

  const total = alreadyRefunded + amount;
  await db.query(
    "UPDATE transactions SET status = $1, refund_amount = $2 WHERE id = $3",
    [total === tx.amount ? "refunded" : "partially_refunded", total, txId]
  );
  return { refundId: refund.id, status: refund.status ?? "succeeded" };
}

export async function refundWithRetry(
  txId: string, amount: number, maxAttempts = 3
): Promise<RefundResult> {
  for (let attempt = 1; ; attempt++) {
    try {
      return await processRefund(txId, amount);
    } catch (error: any) {
      const retryable = !(error instanceof NonRetryableRefundError) && RETRYABLE.has(error?.type);
      if (!retryable || attempt >= maxAttempts) throw error;
      console.warn(`Refund ${txId}: attempt ${attempt} failed: ${error.message}`);
      await new Promise((r) => setTimeout(r, 500 * 2 ** (attempt - 1)));
    }
  }
}

export async function batchRefund(txIds: string[], concurrency = 5): Promise<BatchResult[]> {
  const results: BatchResult[] = [];
  for (let i = 0; i < txIds.length; i += concurrency) {
    const chunk = txIds.slice(i, i + concurrency);
    const settled = await Promise.allSettled(
      chunk.map(async (id) => {
        const tx: Transaction | null = await db.query(
          "SELECT * FROM transactions WHERE id = $1", [id]
        );
        if (!tx) throw new NonRetryableRefundError(`Transaction ${id} not found`);
        return refundWithRetry(id, tx.amount - (tx.refund_amount ?? 0));
      })
    );
    settled.forEach((r, idx) =>
      results.push({
        txId: chunk[idx],
        ok: r.status === "fulfilled",
        error: r.status === "rejected" ? String(r.reason?.message ?? r.reason) : undefined,
      })
    );
  }
  return results;
}

export async function refundAndVerify(txId: string, amount: number): Promise<void> {
  const result = await processRefund(txId, amount);
  const tx: Transaction | null = await db.query(
    "SELECT * FROM transactions WHERE id = $1", [txId]
  );
  if (!tx) throw new Error(`Transaction ${txId} not found after refund`);

  const ok = result.status === "pending"
    ? tx.status === "refund_pending"
    : tx.status === "refunded" || tx.status === "partially_refunded";
  if (!ok) throw new Error(`Refund ${result.refundId} not reflected in DB (status=${tx.status})`);
}

export const db = {
  query: async (sql: string, params: any[]): Promise<any> => {
    throw new Error("Not implemented — use real DB");
  },
};
```

---

## Підсумок

| # | Тест падає на buggy-коді | Проходить на виправленому |
|---|---|---|
| 1 | ключ знайдено у файлі | ✅ |
| 2 | `create` викликано з 0/-100/1001/10.5; `TypeError` замість «not found»; статус `refunded` після часткового refund | ✅ |
| 3 | `create` викликано двічі | ✅ |
| 4 | `idempotencyKey` відсутній | ✅ |
| 5 | помилка проковтнута; 3 ретраї на валідаційній помилці | ✅ |
| 6 | `amount: 0`; `Promise.all` без звіту | ✅ |
| 7 | `failed` refund позначено `refunded` | ✅ |
| 8 | створено таймер 5000 мс | ✅ |
