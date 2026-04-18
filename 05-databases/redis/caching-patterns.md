# Redis Caching Patterns — Питання для інтерв'ю

## Основні стратегії кешування

### 1. Cache-Aside (Lazy Loading)

```
Read:
  App → Cache (hit?) → Yes → повернути
                     → No  → DB → записати в Cache → повернути

Write:
  App → DB → видалити з Cache (invalidate)
```

```javascript
async function getUser(userId) {
  // 1. Перевірити кеш
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);

  // 2. Якщо немає — з бази
  const user = await db.query("SELECT * FROM users WHERE id = $1", [userId]);

  // 3. Зберегти в кеш з TTL
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));

  return user;
}

async function updateUser(userId, data) {
  await db.query("UPDATE users SET name = $1 WHERE id = $2", [data.name, userId]);
  // Видалити з кешу (не оновлювати!)
  await redis.del(`user:${userId}`);
}
```

**Плюси:** простий, кешується тільки те що запитується.
**Мінуси:** перший запит завжди cache miss, можливі stale data.

---

### 2. Write-Through

```
Write:
  App → Cache → DB (синхронно)
Read:
  App → Cache (завжди hit)
```

```javascript
async function createOrder(order) {
  // Запис і в кеш, і в БД одночасно
  await db.query("INSERT INTO orders ...", [order]);
  await redis.setex(`order:${order.id}`, 3600, JSON.stringify(order));
}
```

**Плюси:** кеш завжди актуальний, ніколи stale.
**Мінуси:** повільніший write (два запити), кешує все (навіть непотрібне).

---

### 3. Write-Behind (Write-Back)

```
Write:
  App → Cache (швидко!) → DB (асинхронно, пізніше)
```

```javascript
async function incrementPageView(pageId) {
  // Тільки в кеш (миттєво)
  await redis.hincrby(`pageviews`, pageId, 1);
}

// Periodic flush в БД (кожні 5 хвилин)
setInterval(async () => {
  const views = await redis.hgetall("pageviews");
  for (const [pageId, count] of Object.entries(views)) {
    await db.query("UPDATE pages SET views = views + $1 WHERE id = $2", [count, pageId]);
  }
  await redis.del("pageviews");
}, 5 * 60 * 1000);
```

**Плюси:** найшвидший write, batch updates.
**Мінуси:** ризик втрати даних якщо Redis впаде.

---

### 4. Read-Through

```
Read:
  App → Cache (автоматично завантажує з DB якщо miss)
```

Відрізняється від Cache-Aside тим, що **кеш сам** відповідає за завантаження з БД. Зазвичай реалізується через бібліотеку/фреймворк.

---

## Порівняння стратегій

| Стратегія | Read speed | Write speed | Consistency | Complexity |
|-----------|-----------|------------|-------------|-----------|
| Cache-Aside | Cache hit: швидко, miss: повільно | Середній | Eventual | Простий |
| Write-Through | Завжди швидко | Повільніший | Strong | Середній |
| Write-Behind | Завжди швидко | Найшвидший | Eventual | Складний |
| Read-Through | Cache hit: швидко | Не впливає | Eventual | Середній |

---

## Cache Invalidation Patterns

### 1. TTL (Time-To-Live) — найпростіший

```javascript
await redis.setex("user:1", 3600, data); // видалиться через 1 годину
```

### 2. Event-Based Invalidation

```javascript
// При зміні даних — видалити кеш
eventBus.on("user.updated", async (userId) => {
  await redis.del(`user:${userId}`);
  await redis.del(`user:${userId}:profile`);
  await redis.del(`user:${userId}:settings`);
});
```

### 3. Version-Based

```javascript
// Версія як частина ключа
const version = await redis.incr("user:1:version");
await redis.setex(`user:1:v${version}`, 3600, data);
```

---

## Типові проблеми

### Cache Stampede (Thundering Herd)

**Проблема:** TTL закінчився → тисячі запитів одночасно йдуть в БД.

```javascript
// Рішення 1: Mutex Lock
async function getUserWithLock(userId) {
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);

  // Спробувати взяти lock
  const lock = await redis.set(`lock:user:${userId}`, "1", "EX", 10, "NX");

  if (lock) {
    // Отримали lock — завантажити з БД
    const user = await db.query("...");
    await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));
    await redis.del(`lock:user:${userId}`);
    return user;
  }

  // Не отримали lock — зачекати і повторити
  await sleep(100);
  return getUserWithLock(userId);
}

// Рішення 2: Stale-While-Revalidate
// Зберігати з подвійним TTL: data TTL + stale TTL
// Повертати stale дані поки фонова задача оновлює кеш
```

### Cache Penetration

**Проблема:** запити по неіснуючих ключах завжди потрапляють в БД.

```javascript
// Рішення: кешувати "порожній" результат
async function getUser(userId) {
  const cached = await redis.get(`user:${userId}`);
  if (cached === "NULL") return null; // кешований null
  if (cached) return JSON.parse(cached);

  const user = await db.query("...");
  if (!user) {
    await redis.setex(`user:${userId}`, 300, "NULL"); // кешувати null на 5 хв
    return null;
  }

  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));
  return user;
}

// Рішення 2: Bloom Filter
// Перевірити через Bloom Filter чи ключ існує перед запитом
```

### Cache Avalanche

**Проблема:** багато ключів з однаковим TTL закінчуються одночасно.

```javascript
// Рішення: random jitter до TTL
const ttl = 3600 + Math.floor(Math.random() * 300); // 3600-3900 сек
await redis.setex(key, ttl, data);
```

---

## Практичні поради

1. **Завжди ставити TTL** — ніколи не кешувати без терміну
2. **Invalidate, а не update** — видаляй ключ замість оновлення (уникає race conditions)
3. **Namespace ключі** — `service:entity:id:field` (наприклад `api:user:123:profile`)
4. **Моніторити hit rate** — якщо менше 80% — щось не так
5. **Serialization** — JSON для простоти, MessagePack/Protobuf для швидкості
6. **Не кешувати великі об'єкти** — Redis в пам'яті, пам'ять коштує

```javascript
// Перевірка hit rate
const hits = await redis.info("stats");
// keyspace_hits / (keyspace_hits + keyspace_misses) > 0.8
```
