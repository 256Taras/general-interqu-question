# MongoDB Transactions — Питання для інтерв'ю

## Single Document Atomicity

MongoDB **гарантує атомарність** на рівні одного документа:
```javascript
// Це атомарна операція — або все, або нічого
db.orders.updateOne(
  { _id: orderId },
  {
    $set: { status: "paid" },
    $push: { history: { action: "payment", date: new Date() } },
    $inc: { totalPaid: 100 }
  }
);
```

**Тому в MongoDB часто вбудовують (embed) дані замість окремих колекцій** — щоб уникнути потреби в транзакціях.

---

## Multi-Document Transactions (MongoDB 4.0+)

Коли потрібно змінити кілька документів/колекцій атомарно:

```javascript
const session = client.startSession();

try {
  session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" },
    readPreference: "primary"
  });

  // Операція 1: списати з балансу
  await db.accounts.updateOne(
    { _id: fromId },
    { $inc: { balance: -amount } },
    { session }
  );

  // Операція 2: зарахувати на баланс
  await db.accounts.updateOne(
    { _id: toId },
    { $inc: { balance: amount } },
    { session }
  );

  // Операція 3: записати транзакцію
  await db.transactions.insertOne(
    { from: fromId, to: toId, amount, date: new Date() },
    { session }
  );

  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  session.endSession();
}
```

**Важливо:** `{ session }` — передається в кожну операцію!

---

## Read Concern Levels

```javascript
// local — читає локальні дані (може бути rolled back)
{ readConcern: { level: "local" } }

// majority — читає дані підтверджені majority (не буде rolled back)
{ readConcern: { level: "majority" } }

// snapshot — point-in-time snapshot (для транзакцій)
{ readConcern: { level: "snapshot" } }

// linearizable — найсуворіший, чекає підтвердження
{ readConcern: { level: "linearizable" } }
```

---

## Обмеження транзакцій

| Обмеження | Деталі |
|-----------|--------|
| Час | Транзакція має завершитись за 60 сек (за замовчуванням) |
| Розмір | Oplog entry < 16MB |
| Колекції | Не можна створювати колекції в транзакції (MongoDB < 4.4) |
| Sharded | Підтримка з MongoDB 4.2+ |
| Performance | Транзакції повільніші за single-document операції |
| DDL | Не можна створювати індекси в транзакції |

---

## Retry Logic

MongoDB рекомендує **retryable writes** та **retry logic** для транзакцій:

```javascript
async function runTransactionWithRetry(session, txnFunc) {
  while (true) {
    try {
      await txnFunc(session);
      break;
    } catch (error) {
      // TransientTransactionError — можна повторити
      if (error.hasErrorLabel("TransientTransactionError")) {
        console.log("TransientTransactionError, retrying...");
        continue;
      }
      throw error;
    }
  }
}

async function commitWithRetry(session) {
  while (true) {
    try {
      await session.commitTransaction();
      break;
    } catch (error) {
      // UnknownTransactionCommitResult — можна повторити commit
      if (error.hasErrorLabel("UnknownTransactionCommitResult")) {
        console.log("UnknownTransactionCommitResult, retrying commit...");
        continue;
      }
      throw error;
    }
  }
}
```

---

## Коли використовувати транзакції?

**Використовуй:**
- Фінансові операції (переказ коштів)
- Зв'язані зміни в кількох колекціях
- Коли embedding неможливий

**НЕ використовуй (обирай embedding):**
- Дані завжди читаються разом
- One-to-few відношення
- Дані природно належать одному документу

**Правило:** якщо потрібні транзакції часто — можливо, MongoDB не найкращий вибір для цього use case, або schema потребує переосмислення.

---

## Distributed Transactions (Sharded Clusters)

З MongoDB 4.2+ транзакції працюють на sharded кластерах:

```
Client → mongos → Координатор транзакції
                → Shard 1: операція A
                → Shard 2: операція B
                → Two-Phase Commit (2PC)
```

**Two-Phase Commit:**
1. **Prepare** — кожен shard готує зміни
2. **Commit** — координатор каже всім підтвердити

**Ціна:** додаткова latency через координацію між shards.
