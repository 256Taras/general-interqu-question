# MongoDB Sharding — Питання для інтерв'ю

## Архітектура

```
Client → mongos (router) → Config Servers (metadata)
                         → Shard 1 (data)
                         → Shard 2 (data)
                         → Shard 3 (data)
```

- **mongos** — роутер, направляє запити на потрібний shard
- **Config Servers** — зберігають metadata (яка chunk на якому shard)
- **Shard** — містить підмножину даних (кожен shard — replica set)

---

## Shard Key — вибір

Shard key визначає розподіл даних між shards. **Найважливіше рішення — не можна змінити після створення.**

**Хороший shard key:**
- Висока кардинальність (багато унікальних значень)
- Рівномірний розподіл
- Підтримує типові запити (targeted queries)

```javascript
// ✅ Хороший: userId (висока кардинальність, рівномірний)
sh.shardCollection("mydb.orders", { userId: 1 });

// ❌ Поганий: status (3 значення — все на 3 shards, нерівномірно)
sh.shardCollection("mydb.orders", { status: 1 });

// ❌ Поганий: createdAt (monotonically increasing — все на останній shard)
sh.shardCollection("mydb.logs", { createdAt: 1 });
```

---

## Range vs Hash Sharding

### Range
```javascript
sh.shardCollection("mydb.orders", { userId: 1 });
// userId 1-1000 → Shard A, 1001-2000 → Shard B
```
Добре для range queries. Погано якщо ключ монотонно зростає (hotspot).

### Hash
```javascript
sh.shardCollection("mydb.orders", { userId: "hashed" });
// hash(userId) → рівномірний розподіл
```
Рівномірний розподіл. Погано для range queries (scatter-gather).

---

## Targeted vs Scatter-Gather Queries

**Targeted** — mongos знає на який shard відправити:
```javascript
// Shard key: userId
db.orders.find({ userId: "user123" }); // → один shard
```

**Scatter-Gather** — запит на ВСІ shards:
```javascript
// Shard key: userId, але шукаємо по email
db.orders.find({ email: "john@example.com" }); // → всі shards
```

---

## Chunks та Balancer

- Дані розділені на **chunks** (~128MB за замовчуванням)
- **Balancer** автоматично переміщує chunks між shards для рівномірного розподілу
- Якщо один shard має набагато більше chunks — balancer мігрує їх
