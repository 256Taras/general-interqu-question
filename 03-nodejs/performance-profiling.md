# Performance та Profiling — Питання для інтерв'ю

## Як профілювати Node.js додаток?

### 1. Node.js Built-in Profiler (--prof)
```bash
node --prof app.js
# Генерує isolate-*.log файл
node --prof-process isolate-*.log > profile.txt
```
Показує де процесор витрачає найбільше часу (C++ vs JS).

### 2. Chrome DevTools
```bash
node --inspect app.js
# або --inspect-brk для зупинки на старті
```
Відкрити `chrome://inspect` → Performance tab → Record.

**Що шукати:**
- Довгі синхронні операції (блокують Event Loop)
- Часті GC паузи (memory issues)
- Bottlenecks у конкретних функціях

### 3. Clinic.js
```bash
npx clinic doctor -- node app.js      # загальна діагностика
npx clinic flame -- node app.js       # flame graph (CPU)
npx clinic bubbleprof -- node app.js  # async операції
npx clinic heapprofile -- node app.js # memory
```
Генерує HTML звіт з візуалізацією.

### 4. Flame Graphs
Показують стек викликів і час кожної функції:
```
───────────────────────────── main()
    ─────────────── processRequest()
        ────── queryDB()
        ─── transformData()
    ── sendResponse()
```
Широкі блоки = довгий час виконання = bottleneck.

---

## Performance Hooks (perf_hooks)

```javascript
import { performance, PerformanceObserver } from "node:perf_hooks";

// Вимірювання часу
performance.mark("start");
await heavyOperation();
performance.mark("end");
performance.measure("heavy-op", "start", "end");

// Observer
const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.duration.toFixed(2)}ms`);
  }
});
obs.observe({ entryTypes: ["measure"] });

// timerify — обгортка для вимірювання функцій
const measuredFn = performance.timerify(myFunction);
```

---

## Оптимізація CPU-інтенсивних операцій

**Проблема:** CPU-intensive код блокує Event Loop → всі запити чекають.

**Рішення:**

### 1. Worker Threads
```javascript
import { Worker } from "node:worker_threads";

function runInWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker("./heavy-task.js", { workerData: data });
    worker.on("message", resolve);
    worker.on("error", reject);
  });
}
```

### 2. Розбити на chunks
```javascript
// ❌ Блокує Event Loop
function processAll(items) {
  for (const item of items) processItem(item); // 1 мільйон items
}

// ✅ Порціями з поверненням у Event Loop
async function processAll(items) {
  const CHUNK_SIZE = 1000;
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    chunk.forEach(processItem);
    await new Promise(resolve => setImmediate(resolve)); // дати Event Loop працювати
  }
}
```

### 3. Streams для великих даних
Обробка порціями замість завантаження всього у пам'ять.

---

## Connection Pooling

```javascript
// PostgreSQL з pg
import pg from "pg";
const pool = new pg.Pool({
  max: 20,              // максимум з'єднань
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Drizzle з connection pool
const pool = new Pool({ connectionString: DATABASE_URL, max: 20 });
const db = drizzle(pool);
```

**Чому важливо:**
- Створення TCP з'єднання з БД = 20-100ms
- Pool перевикористовує з'єднання
- Обмежує навантаження на БД

**Налаштування:**
- `max` = кількість CPU cores × 2 + кількість дисків (формула від PostgreSQL)
- Не більше, ніж `max_connections` у PostgreSQL

---

## Caching Strategies

### In-memory (Map / LRU Cache)
```javascript
import { LRUCache } from "lru-cache";

const cache = new LRUCache({
  max: 500,           // макс елементів
  ttl: 1000 * 60 * 5, // TTL 5 хвилин
});

async function getUser(id) {
  const cached = cache.get(id);
  if (cached) return cached;

  const user = await db.findUser(id);
  cache.set(id, user);
  return user;
}
```

### Redis (distributed cache)
```javascript
async function getUser(id) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);

  const user = await db.findUser(id);
  await redis.set(`user:${id}`, JSON.stringify(user), "EX", 300);
  return user;
}
```

### HTTP Caching
```javascript
// ETag
reply.header("ETag", hash(data));
reply.header("Cache-Control", "max-age=300, stale-while-revalidate=60");
```

---

## Типові bottlenecks та рішення

| Bottleneck | Симптом | Рішення |
| --- | --- | --- |
| N+1 queries | Повільні API, багато DB запитів | Eager loading, DataLoader |
| Sync operations | Event Loop lag | async/await, Worker Threads |
| Missing indexes | Повільні SELECT | EXPLAIN ANALYZE, додати індекси |
| No connection pool | Connection timeouts | Pool з правильним max |
| No caching | Повторні запити до DB | Redis / in-memory cache |
| Large payloads | Повільні responses | Pagination, compression |
| Memory leaks | RSS зростає з часом | Profiling, WeakRef, cleanup |
