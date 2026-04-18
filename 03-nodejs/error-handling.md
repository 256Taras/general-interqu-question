# Error Handling в Node.js — Питання для інтерв'ю

## Різниця між throw та reject

```javascript
// throw — синхронна помилка
function divide(a, b) {
  if (b === 0) throw new Error("Division by zero");
  return a / b;
}

// reject — асинхронна помилка
function fetchUser(id) {
  return new Promise((resolve, reject) => {
    if (!id) reject(new Error("ID required"));
    // ...
  });
}

// async/await — throw всередині async = reject
async function getUser(id) {
  if (!id) throw new Error("ID required"); // автоматично reject
  return await db.findById(id);
}
```

---

## Unhandled Promise Rejections

```javascript
// ❌ Небезпечно — unhandled rejection
fetchUser(null); // reject, але ніхто не ловить

// ✅ Обробка
fetchUser(null).catch(err => console.error(err));

// Глобальний обробник (safety net)
process.on("unhandledRejection", (reason, promise) => {
  logger.error({ reason }, "Unhandled Rejection");
  // В Node 15+ — це crash за замовчуванням
});
```

**Важливо:** з Node.js 15+ unhandled rejection завершує процес. Завжди обробляй Promise помилки.

---

## process.on("uncaughtException")

```javascript
process.on("uncaughtException", (error) => {
  logger.fatal(error, "Uncaught Exception");
  // ОБОВ'ЯЗКОВО завершити процес!
  process.exit(1);
});
```

**Правило:** після `uncaughtException` процес в невизначеному стані — ЗАВЖДИ завершуй. Використовуй process manager (PM2) для перезапуску.

---

## Operational vs Programmer Errors

### Operational errors (очікувані)
- Мережева помилка, timeout
- Файл не знайдено
- Валідація не пройшла
- DB constraint violation

**Обробка:** catch, retry, fallback, повідомити користувача.

### Programmer errors (баги)
- TypeError, ReferenceError
- Невалідний аргумент
- Null pointer
- Assertion failure

**Обробка:** фіксити код. Не ловити, давати crash → restart.

```javascript
// Operational — обробляємо
try {
  await fetch(url);
} catch (err) {
  if (err.code === "ECONNREFUSED") return fallbackData;
  throw err;
}

// Programmer — НЕ ловимо, фіксимо
function getUser(id) {
  if (typeof id !== "string") throw new TypeError("ID must be string"); // fail fast
}
```

---

## Graceful Error Handling Strategies

### Domain-specific errors
```typescript
class AppError extends Error {
  constructor(message, public statusCode: number, public code: string) {
    super(message);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, "NOT_FOUND");
  }
}

class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, "CONFLICT");
  }
}
```

### Centralized error handler (Fastify)
```typescript
app.setErrorHandler((error, request, reply) => {
  if (error instanceof AppError) {
    return reply.status(error.statusCode).send({
      error: { code: error.code, message: error.message }
    });
  }
  // Unexpected error
  logger.error(error);
  reply.status(500).send({ error: { code: "INTERNAL", message: "Internal error" } });
});
```

---

## Retry Pattern з Exponential Backoff

```javascript
async function withRetry(fn, { maxRetries = 3, baseDelay = 1000 } = {}) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      if (!isRetryable(error)) throw error;

      const delay = baseDelay * Math.pow(2, attempt);
      const jitter = Math.random() * 1000;
      await new Promise(r => setTimeout(r, delay + jitter));
    }
  }
}

function isRetryable(error) {
  return error.code === "ECONNRESET"
    || error.code === "ETIMEDOUT"
    || error.status === 503
    || error.status === 429;
}

// Використання
const data = await withRetry(() => fetch(url));
```

---

## Circuit Breaker Pattern

```
CLOSED → (помилки > threshold) → OPEN
OPEN → (timeout) → HALF-OPEN
HALF-OPEN → (успіх) → CLOSED
HALF-OPEN → (помилка) → OPEN
```

Захищає від каскадних збоїв. Якщо зовнішній сервіс не відповідає — не тратимо ресурси на повторні запити, повертаємо fallback одразу.

```javascript
// Використання бібліотеки opossum
import CircuitBreaker from "opossum";

const breaker = new CircuitBreaker(callExternalService, {
  timeout: 3000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000,
});

breaker.fallback(() => cachedData);
breaker.on("open", () => logger.warn("Circuit opened"));

const result = await breaker.fire(params);
```

---

## Graceful Shutdown

```javascript
async function gracefulShutdown(signal) {
  logger.info(`Received ${signal}. Starting graceful shutdown...`);

  // 1. Перестати приймати нові запити
  await server.close();

  // 2. Завершити поточні запити (timeout)
  await Promise.race([
    finishActiveRequests(),
    new Promise(r => setTimeout(r, 30000)), // max 30s
  ]);

  // 3. Закрити з'єднання
  await db.end();
  await redis.quit();

  logger.info("Shutdown complete");
  process.exit(0);
}

process.on("SIGTERM", () => gracefulShutdown("SIGTERM"));
process.on("SIGINT", () => gracefulShutdown("SIGINT"));
```
