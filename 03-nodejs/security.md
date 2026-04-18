# Node.js Security — Питання для інтерв'ю

## Основні вразливості Node.js додатків

1. **Injection** (SQL, NoSQL, Command) — найнебезпечніша
2. **XSS** через server-rendered HTML
3. **Path Traversal** — доступ до файлів поза дозволеною директорією
4. **Prototype Pollution** — модифікація Object.prototype
5. **ReDoS** (Regular Expression Denial of Service)
6. **Dependency vulnerabilities** — вразливості у npm пакетах

---

## npm audit та Supply Chain Attacks

```bash
npm audit              # показати вразливості
npm audit fix          # автоматично оновити
npm audit --production # тільки production залежності
```

### Supply Chain Attacks
Зловмисник публікує шкідливий пакет або захоплює існуючий:
- **Typosquatting** — `loadsh` замість `lodash`
- **Dependency confusion** — приватний пакет з публічним ім'ям
- **Malicious maintainer** — maintainer додає шкідливий код

**Захист:**
- `npm audit` у CI/CD
- Використовуй `package-lock.json` (фіксує версії)
- Мінімізуй кількість залежностей
- Перевіряй нові залежності перед додаванням
- Використовуй Snyk або Socket.dev

---

## Input Validation

```typescript
// ✅ Використовуй schema validation (TypeBox / Zod / Joi)
import { Type } from "@sinclair/typebox";

const CreateUserSchema = Type.Object({
  email: Type.String({ format: "email", maxLength: 255 }),
  name: Type.String({ minLength: 2, maxLength: 100 }),
  age: Type.Integer({ minimum: 0, maximum: 150 }),
});

// ❌ Ніколи не довіряй user input
app.post("/users", async (req) => {
  const { email, name } = req.body; // може бути БУДЬ-ЩО
});
```

### Prototype Pollution
```javascript
// ❌ Вразливий код
function merge(target, source) {
  for (const key in source) {
    target[key] = source[key]; // __proto__ може бути в source!
  }
}
// Атака: merge({}, JSON.parse('{"__proto__": {"isAdmin": true}}'))
// Тепер ВСІ об'єкти мають isAdmin === true

// ✅ Захист
function safeMerge(target, source) {
  for (const key of Object.keys(source)) { // Object.keys ігнорує prototype
    if (key === "__proto__" || key === "constructor") continue;
    target[key] = source[key];
  }
}
```

---

## Helmet.js

Встановлює security HTTP headers:
```javascript
import helmet from "@fastify/helmet";
app.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
    },
  },
});
```

**Headers які встановлює:**
- `Content-Security-Policy` — обмежує джерела ресурсів
- `X-Content-Type-Options: nosniff` — запобігає MIME sniffing
- `X-Frame-Options: DENY` — захист від clickjacking
- `Strict-Transport-Security` — примусовий HTTPS
- `Referrer-Policy` — контроль Referer header

---

## Rate Limiting

```javascript
import rateLimit from "@fastify/rate-limit";

app.register(rateLimit, {
  max: 100,            // максимум запитів
  timeWindow: "1 minute",
  keyGenerator: (req) => req.ip, // по IP
});

// Per-route rate limit
app.get("/login", {
  config: { rateLimit: { max: 5, timeWindow: "1 minute" } },
}, handler);
```

---

## CORS

```javascript
import cors from "@fastify/cors";

app.register(cors, {
  origin: ["https://frontend.com"],  // НЕ "*" для auth!
  methods: ["GET", "POST", "PUT", "DELETE"],
  credentials: true,                  // для cookies
});
```

---

## Environment Variables — Best Practices

```bash
# ❌ Ніколи не комітьте .env файли
# .gitignore повинен містити: .env, .env.*

# ✅ Використовуй .env.example (без значень)
DATABASE_URL=
JWT_SECRET=
```

```typescript
// ✅ Валідуй env при старті
import { Type } from "@sinclair/typebox";

const EnvSchema = Type.Object({
  DATABASE_URL: Type.String(),
  JWT_SECRET: Type.String({ minLength: 32 }),
  PORT: Type.Number({ default: 3000 }),
});
// Якщо валідація не пройде — додаток не запуститься
```

---

## ReDoS (Regular Expression DoS)

```javascript
// ❌ Вразливий regex
const emailRegex = /^([a-zA-Z0-9]+)+@[a-zA-Z0-9]+\.[a-zA-Z]+$/;
// Input: "aaaaaaaaaaaaaaaaaaaaaaaaa!" — зависне на кілька секунд

// ✅ Захист:
// 1. Використовуй перевірені regex
// 2. Обмежуй довжину input ПЕРЕД regex
// 3. Використовуй timeout
// 4. Тестуй regex на ReDoS (safe-regex, rxxr2)
```

---

## Node.js Permission Model

```bash
# Node.js 20+ — experimental
node --experimental-permission --allow-fs-read=/app/data app.js
node --experimental-permission --allow-fs-write=/tmp app.js
node --experimental-permission --allow-child-process app.js
node --experimental-permission --allow-worker app.js
```

Обмежує доступ додатку до файлової системи, child processes, worker threads.

---

## Path Traversal

```javascript
// ❌ Вразливий
app.get("/files/:name", (req, reply) => {
  const filePath = `/uploads/${req.params.name}`;
  // name = "../../etc/passwd" → читає системний файл!
  return reply.sendFile(filePath);
});

// ✅ Захист
import path from "node:path";

app.get("/files/:name", (req, reply) => {
  const safeName = path.basename(req.params.name); // видаляє ../
  const filePath = path.join("/uploads", safeName);

  // Додатково перевіряємо що шлях всередині дозволеної директорії
  if (!filePath.startsWith("/uploads/")) {
    return reply.status(403).send("Forbidden");
  }
  return reply.sendFile(filePath);
});
```
