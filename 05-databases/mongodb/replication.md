# MongoDB Replication — Питання для інтерв'ю

## Replica Set

```
Primary (read/write) ←──replication──→ Secondary 1 (read)
                     ←──replication──→ Secondary 2 (read)
```

- **Primary** — приймає всі writes, за замовчуванням reads
- **Secondary** — копія даних, read-only (якщо налаштовано)
- **Arbiter** — голосує на виборах, не зберігає дані (для парної кількості)

Мінімум 3 члени для automatic failover (majority vote).

---

## Election (вибори)

Коли Primary стає недоступним:
1. Secondary помічає heartbeat timeout (10 сек)
2. Ініціює election
3. Majority голосів (2 з 3) обирає нового Primary
4. Новий Primary приймає writes

**Пріоритет:** Secondary з `priority: 10` має перевагу над `priority: 1`.

---

## Read Preferences

```javascript
// primary — тільки з Primary (за замовчуванням, strong consistency)
// primaryPreferred — Primary, якщо доступний, інакше Secondary
// secondary — тільки з Secondary
// secondaryPreferred — Secondary, якщо доступний, інакше Primary
// nearest — найближчий по latency

db.users.find().readPref("secondaryPreferred");
```

**Увага:** читання з Secondary може повернути stale data (replication lag).

---

## Write Concerns

```javascript
// w: 1 — підтвердження від Primary (за замовчуванням)
db.orders.insertOne({ total: 99 }, { writeConcern: { w: 1 } });

// w: "majority" — підтвердження від majority
db.orders.insertOne({ total: 99 }, { writeConcern: { w: "majority" } });

// w: 0 — fire-and-forget (без підтвердження)
// j: true — дочекатись запису в journal (durability)
```

---

## Oplog

**Oplog** (operations log) — capped collection що зберігає всі write операції:
```javascript
// Кожен запис:
{ ts: Timestamp, op: "i", ns: "mydb.users", o: { _id: 1, name: "John" } }
// op: "i" (insert), "u" (update), "d" (delete)
```

Secondary реплікують, застосовуючи операції з oplog Primary.

**Розмір oplog** важливий — якщо Secondary відстає більше ніж oplog window, потрібна повна ресинхронізація.
