# Redis Interview Guide

## Рівень підготовки

Ти вже знаєш: key-value, in-memory, TTL, аналог Map, locks, pub/sub.

Файли з деталями:
- `data-structures.md` — String, List, Hash, Set, Sorted Set, HyperLogLog, Bitmap, Stream
- `persistence-rdb-aof.md` — RDB snapshots, AOF log, гібридний підхід
- `caching-patterns.md` — Cache-Aside, Write-Through, Write-Behind, invalidation
- `clustering.md` — Sentinel, Cluster, hash slots, failover
- `pub-sub.md` — Pub/Sub vs Streams, патерни

Цей файл покриває **прогалини** — те, що ще не описано, але часто питають.

---

## 1. Чому Redis швидкий? (питають завжди)

```
1. In-memory     — дані в RAM, не на диску (мікросекунди vs мілісекунди)
2. Single-thread — один потік для команд, нема overhead на locks/context switch
3. I/O multiplexing — epoll/kqueue обробляє тисячі з'єднань в одному потоці
4. Прості структури — оптимізовані під RAM (ziplist, intset, quicklist)
5. Протокол RESP — простий текстовий протокол, мінімальний parsing
```

### Single-threaded — це не мінус?

```
"Redis single-threaded" — це ПРО ОБРОБКУ КОМАНД.

Один потік обробляє всі команди послідовно:
  SET → GET → INCR → DEL → SET → ...
  (немає race conditions, немає locks)

Але Redis використовує додаткові потоки для:
  - I/O (Redis 6.0+) — читання/запис мережі в окремих потоках
  - BGSAVE / BGREWRITEAOF — fork(), child process для persistence
  - Lazy free — видалення великих ключів в background (UNLINK замість DEL)

Чому single-thread достатньо?
  - Bottleneck Redis — мережа та пам'ять, НЕ CPU
  - Одна команда виконується за ~1 мікросекунду
  - 100,000+ ops/sec на одному ядрі
```

---

## 2. Eviction Policies (коли пам'ять закінчується)

```redis
maxmemory 2gb
maxmemory-policy allkeys-lru
```

| Policy | Що робить |
|--------|-----------|
| `noeviction` | Помилка OOM на write (default) |
| `allkeys-lru` | Видаляє найменш нещодавно використані (рекомендовано для кешу) |
| `allkeys-lfu` | Видаляє найменш часто використані (Redis 4.0+) |
| `volatile-lru` | LRU тільки серед ключів з TTL |
| `volatile-lfu` | LFU тільки серед ключів з TTL |
| `volatile-ttl` | Видаляє з найменшим TTL |
| `allkeys-random` | Випадковий ключ |
| `volatile-random` | Випадковий серед ключів з TTL |

**Типове питання:** "Яку policy обрати?"
- Чистий кеш → `allkeys-lru` або `allkeys-lfu`
- Кеш + важливі дані → `volatile-lru` (важливі без TTL не видаляться)
- Не знаєш → `allkeys-lru` (найбезпечніший default)

**LRU vs LFU:**
- LRU — "давно не використовувався" (один запит — ключ "нещодавній")
- LFU — "рідко використовується" (потрібно багато запитів щоб стати "частим")
- LFU краще для hot data (популярний ключ не витіснить одноразовий scan)

---

## 3. Transactions (MULTI/EXEC)

```redis
MULTI                          -- початок транзакції
SET user:1:balance 100
DECRBY user:1:balance 30
INCRBY user:2:balance 30
EXEC                           -- виконати все атомарно
```

### Redis транзакції != SQL транзакції!

```
SQL транзакції (ACID):
  - Rollback при помилці ✅
  - Ізоляція між транзакціями ✅

Redis MULTI/EXEC:
  - Rollback при помилці ❌ (часткове виконання!)
  - Атомарність виконання ✅ (нічого між командами)
  - Команди буферизуються, виконуються всі разом
```

**Важливо:** якщо INCRBY фейлить (не число), SET все одно виконається. Немає rollback!

### WATCH — Optimistic Locking

```redis
WATCH user:1:balance           -- слідкувати за ключем
val = GET user:1:balance       -- прочитати
MULTI
SET user:1:balance (val - 30)  -- оновити
EXEC                           -- якщо хтось змінив balance → EXEC поверне nil
                               -- потрібно повторити (retry loop)
```

**Аналогія:** `WATCH` = CAS (Compare-And-Swap). Якщо між WATCH і EXEC хтось змінив ключ → транзакція відкидається.

---

## 4. Lua Scripts (атомарні операції)

```redis
-- Rate limiter: максимум 10 запитів за 60 секунд
EVAL "
  local current = redis.call('INCR', KEYS[1])
  if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
  end
  if current > tonumber(ARGV[2]) then
    return 0
  end
  return 1
" 1 ratelimit:user:123 60 10
```

**Чому Lua, а не MULTI/EXEC?**
- Lua script виконується **атомарно** (нічого між командами)
- Можна читати і писати в одному скрипті
- Є умовна логіка (if/else)
- MULTI/EXEC не може використати результат однієї команди в іншій

**Обмеження:**
- Скрипт блокує Redis (не повинен бути довгим!)
- В кластері — всі ключі повинні бути в одному hash slot
- `redis.call()` — помилка зупиняє скрипт
- `redis.pcall()` — помилка повертається як значення

---

## 5. Pipelining (батчінг команд)

```
Без pipeline (round-trip на кожну команду):
  Client → SET k1 v1 → Server → OK
  Client → SET k2 v2 → Server → OK
  Client → SET k3 v3 → Server → OK
  = 3 round-trips (~3ms мережі)

З pipeline (один round-trip):
  Client → SET k1 v1
           SET k2 v2   → Server → OK, OK, OK
           SET k3 v3
  = 1 round-trip (~1ms мережі)
```

```javascript
const pipeline = redis.multi(); // або redis.pipeline() в ioredis
pipeline.set("k1", "v1");
pipeline.set("k2", "v2");
pipeline.set("k3", "v3");
const results = await pipeline.exec();
```

**Pipeline vs MULTI/EXEC:**
- Pipeline — оптимізація мережі (команди можуть виконатися "між" іншими клієнтами)
- MULTI/EXEC — атомарність (нічого між командами)
- Можна комбінувати: `MULTI` в pipeline для атомарного батчу

---

## 6. Distributed Locks (Redlock)

### Простий lock (один Redis)

```redis
-- Взяти lock
SET lock:order:123 "owner-uuid" NX EX 30
-- NX = тільки якщо не існує
-- EX 30 = автоматичне звільнення через 30 сек (safety timeout)

-- Звільнити lock (тільки якщо ми власник!)
-- Lua скрипт щоб уникнути race condition
EVAL "
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  end
  return 0
" 1 lock:order:123 "owner-uuid"
```

**Чому UUID?** Щоб не видалити чужий lock. Якщо наш lock expired і хтось інший взяв — DEL без перевірки видалить чужий.

### Redlock (розподілений lock)

```
Проблема: один Redis — single point of failure.
Рішення: Redlock алгоритм (Martin Kleppmann критикував, але широко використовується).

Алгоритм:
1. Взяти timestamp
2. Спробувати SET NX EX на N незалежних Redis серверах (зазвичай 5)
3. Якщо отримали lock на majority (≥ N/2 + 1) серверів
   І загальний час < TTL lock → lock отримано
4. Інакше → звільнити lock на всіх серверах

Приклад з 5 серверами:
  Lock на Redis 1 ✅
  Lock на Redis 2 ✅
  Lock на Redis 3 ❌ (timeout)
  Lock на Redis 4 ✅
  Lock на Redis 5 ✅
  → 4/5 > majority → lock отримано
```

**Критика Redlock (Martin Kleppmann):**
- Залежить від синхронізації часу
- Можливий split-brain при GC паузах
- Для критичних систем краще Zookeeper / etcd

**Практична відповідь на інтерв'ю:**
- Для більшості задач — простий lock на одному Redis достатній
- Для розподіленого lock — краще використати бібліотеку (Redlock, Redisson)
- Для критичних фінансових операцій — Zookeeper або etcd

---

## 7. Redis vs Memcached

| Характеристика | Redis | Memcached |
|---------------|-------|-----------|
| Структури даних | Багато (String, List, Hash, Set...) | Тільки String |
| Persistence | RDB, AOF | Ні |
| Pub/Sub | Так | Ні |
| Lua scripts | Так | Ні |
| Кластеринг | Вбудований | Client-side sharding |
| Потоки | Single-threaded (+I/O threads) | Multi-threaded |
| Максимальний value | 512MB | 1MB |
| Replication | Вбудована | Ні |

**Коли Memcached?**
- Потрібен простий кеш string → string
- Multi-threaded важливіший (дуже великий traffic)
- Не потрібна persistence

**Коли Redis?**
- Потрібні складні структури даних
- Потрібна persistence
- Потрібні pub/sub, streams, locks
- Вже 99% випадків — Redis

---

## 8. Key Design Patterns

### Naming Convention

```
service:entity:id:field

Приклади:
  api:user:123:profile       -- профіль юзера
  api:session:abc123         -- сесія
  api:ratelimit:ip:1.2.3.4   -- rate limit по IP
  cache:query:users:page:1   -- кеш запиту
  lock:order:456             -- розподілений lock
  queue:emails               -- черга emails
```

### Key Expiration Patterns

```javascript
// Проблема: KEYS * — O(n), блокує Redis!
// Ніколи не використовуй KEYS в продакшні!

// Правильно: SCAN (курсорний ітератор)
let cursor = "0";
do {
  const [newCursor, keys] = await redis.scan(cursor, "MATCH", "user:*", "COUNT", 100);
  cursor = newCursor;
  // обробити keys
} while (cursor !== "0");
```

### Big Key Problem

```
Проблема: один ключ = 100MB → DEL блокує Redis!

Рішення:
  1. UNLINK замість DEL (async видалення в background)
  2. Розбити на менші ключі
  3. Моніторити: redis-cli --bigkeys

Правило: один ключ < 10KB ідеально, < 1MB прийнятно, > 10MB — проблема
```

---

## 9. Redis в реальних проєктах (use cases)

### Session Storage

```javascript
// Зберегти сесію
await redis.setex(`session:${sessionId}`, 86400, JSON.stringify({
  userId: "user:1",
  role: "admin",
  createdAt: Date.now(),
}));

// Отримати сесію
const session = JSON.parse(await redis.get(`session:${sessionId}`));
```

**Чому Redis, а не JWT?**
- Redis session: можна інвалідувати (logout, ban)
- JWT: не можна інвалідувати до закінчення терміну
- Практика: JWT (access, 15 хв) + Redis (refresh token / session blacklist)

### Rate Limiting

```
Алгоритми:
1. Fixed Window    — INCR + EXPIRE (простий, але burst на межі вікна)
2. Sliding Window  — Sorted Set з timestamp як score
3. Token Bucket    — Lua script (Stripe так робить)
4. Leaky Bucket    — List як queue з фіксованою швидкістю
```

```javascript
// Sliding Window Rate Limiter
async function isRateLimited(userId, limit, windowSec) {
  const key = `ratelimit:${userId}`;
  const now = Date.now();
  const windowStart = now - windowSec * 1000;

  const pipe = redis.multi();
  pipe.zremrangebyscore(key, 0, windowStart);  // видалити старі
  pipe.zadd(key, now, `${now}`);               // додати поточний
  pipe.zcard(key);                              // порахувати
  pipe.expire(key, windowSec);                  // TTL

  const results = await pipe.exec();
  const count = results[2];
  return count > limit;
}
```

### Leaderboard

```javascript
// Додати/оновити score
await redis.zadd("leaderboard:weekly", score, `player:${playerId}`);

// Топ-10
const top10 = await redis.zrevrange("leaderboard:weekly", 0, 9, "WITHSCORES");

// Позиція гравця
const rank = await redis.zrevrank("leaderboard:weekly", `player:${playerId}`);
// rank = 0 означає перше місце
```

### Job Queue (простий)

```javascript
// Producer
await redis.lpush("queue:emails", JSON.stringify({
  to: "user@example.com",
  subject: "Welcome!",
}));

// Consumer (blocking)
while (true) {
  const [, job] = await redis.brpop("queue:emails", 30); // чекає 30 сек
  if (job) {
    await processEmail(JSON.parse(job));
  }
}
```

**Для production:** використовуй BullMQ (побудований на Redis Streams).

---

## 10. Типові питання на інтерв'ю

### Рівень Junior

```
Q: Що таке Redis?
A: In-memory key-value store. Дані в RAM, тому дуже швидкий (~100K ops/sec).
   Підтримує різні структури даних (String, List, Hash, Set, Sorted Set).
   Може persist дані на диск (RDB/AOF).

Q: Чим Redis відрізняється від бази даних?
A: Redis — в пам'яті, PostgreSQL — на диску.
   Redis для швидкого доступу (кеш, сесії, черги).
   PostgreSQL для надійного зберігання (ACID, relations, SQL).
   Вони доповнюють одне одного.

Q: Що таке TTL?
A: Time-To-Live. Redis автоматично видаляє ключ після закінчення TTL.
   SETEX key 3600 value — видалиться через 1 годину.
   PERSIST key — зняти TTL (ключ стає постійним).

Q: Як Redis зберігає дані якщо він in-memory?
A: Два механізми: RDB (snapshot на диск) і AOF (лог операцій).
   Рекомендовано RDB + AOF для продакшну.
```

### Рівень Middle

```
Q: Чому Redis single-threaded і як він обробляє тисячі з'єднань?
A: I/O multiplexing (epoll). Один потік обробляє всі з'єднання через event loop.
   Bottleneck — мережа, не CPU. Команда ~1 мікросекунда.
   Redis 6.0+ має I/O threads для мережі.

Q: Як реалізувати distributed lock?
A: SET key uuid NX EX 30. NX = тільки якщо не існує. EX = auto-expire.
   Звільнення через Lua script (перевірити UUID перед DEL).
   Для HA — Redlock алгоритм (majority з N серверів).

Q: Що таке Cache Stampede і як боротися?
A: TTL закінчився → тисячі запитів одночасно в БД.
   Рішення: mutex lock (SET NX), stale-while-revalidate, random jitter до TTL.

Q: Redis Cluster vs Sentinel?
A: Sentinel — HA без шардінгу (один master, replicas, auto failover).
   Cluster — HA + шардінг (16384 hash slots, дані розподілені між masters).
   < 25GB → Sentinel. > 25GB → Cluster.
```

### Рівень Senior

```
Q: Які проблеми з Redis в мікросервісній архітектурі?
A: 1. Hot key — один ключ отримує непропорційно багато запитів
      → реплікація hot key, local cache (L1 in-process + L2 Redis)
   2. Big key — великий value блокує Redis при DELETE
      → UNLINK, розбити на менші ключі
   3. Memory fragmentation — часті alloc/dealloc
      → activedefrag yes, jemalloc
   4. Network partition — split-brain в Cluster
      → min-replicas-to-write, proper quorum

Q: Як спроєктувати real-time leaderboard для 10M гравців?
A: Sorted Set (ZADD, ZREVRANGE).
   O(log n) для add/remove. Топ-100 за O(log n + 100).
   Для 10M елементів Sorted Set займе ~1.5GB RAM.
   Sharding по регіону/лізі якщо потрібно більше.

Q: Redis vs Kafka для event streaming?
A: Redis Streams — легковісний, достатній для одного сервісу / малого масштабу.
   Kafka — горизонтально масштабований, дані на диску, retention policies.
   Redis Pub/Sub — fire-and-forget (не persistence).
   Redis Streams — at-least-once з consumer groups.
   Kafka — exactly-once (з idempotent producers + transactions).

Q: Як уникнути data loss при Redis failover?
A: 1. AOF з fsync everysec (max 1 сек втрати)
   2. WAIT command — чекати поки N replicas підтвердять запис
   3. min-replicas-to-write 1 — не приймати write якщо немає replica
   4. Для критичних даних — Redis як кеш, source of truth в PostgreSQL
```

---

## 11. Cheat Sheet для швидкого повторення

```
Швидкий?        → RAM + single-thread + I/O multiplexing + RESP протокол
Persistence?    → RDB (snapshot) + AOF (log) → гібрид для production
Eviction?       → allkeys-lru (кеш), volatile-lru (кеш + important keys)
Транзакції?     → MULTI/EXEC (без rollback!), WATCH (optimistic lock)
Атомарність?    → Lua scripts (одна команда = атомарна операція)
Масштабування?  → Sentinel (HA) або Cluster (HA + sharding)
Lock?           → SET key uuid NX EX ttl + Lua DEL з перевіркою UUID
Pipeline?       → Батчінг команд в один round-trip (мережева оптимізація)
Pub/Sub?        → Fire-and-forget. Для гарантій → Streams
KEYS?           → НІКОЛИ в production! Використовуй SCAN
Big Key?        → UNLINK замість DEL. Ключ < 1MB
```
