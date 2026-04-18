# Dockerfile: best practices

## Що таке multi-stage build і навіщо він потрібен?

Multi-stage build -- це техніка, коли один Dockerfile містить кілька стадій `FROM`, і фінальний образ отримує тільки потрібні артефакти з попередніх стадій. Це дозволяє мати важке build-середовище (компілятори, dev-залежності, інструменти) і легкий runtime-образ без зайвого.

**Проблема без multi-stage:**
```dockerfile
# ❌ Фінальний образ ~1.2GB
FROM node:20
WORKDIR /app
COPY . .
RUN npm ci                  # + dev-залежності
RUN npm run build           # компілятор, TypeScript
CMD ["node", "dist/server.js"]
```

**З multi-stage:**
```dockerfile
# ---- Build stage ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                  # встановити всі залежності, включно з dev
COPY . .
RUN npm run build
RUN npm prune --production  # видалити dev-залежності

# ---- Runtime stage ----
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
USER node
CMD ["node", "dist/server.js"]
# ✅ Фінальний образ ~180MB
```

Переваги:
- **Менший розмір** -- швидший pull/push, менше disk usage.
- **Менше attack surface** -- немає компіляторів і dev-інструментів у production.
- **Краще кешування** -- стадії кешуються незалежно.

---

## Як правильно оптимізувати кеш Docker-шарів?

Docker кешує шари, поки контекст (вхідні дані інструкції) не змінився. Як тільки один шар інвалідується -- всі наступні теж перебудовуються. Тому порядок інструкцій має значення.

**Правило:** спочатку ставимо інструкції, що змінюються рідко; в кінці -- ті, що змінюються часто.

```dockerfile
# ❌ Поганий порядок
FROM node:20-alpine
WORKDIR /app
COPY . .                    # будь-яка зміна в коді ламає кеш
RUN npm ci                  # npm ci буде перезапускатися щоразу

# ✅ Правильний порядок
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./        # змінюється рідко
RUN npm ci                   # кеш зберігається, поки package.json не змінився
COPY . .                     # код копіюється в останню чергу
```

**BuildKit cache mounts** для ще більшого прискорення:
```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build
```

Cache mount зберігає кеш npm між білдами, навіть якщо `package.json` змінився.

---

## Як зменшити розмір Docker-образу?

**1. Базовий образ:**
| Образ                | Розмір |
|----------------------|--------|
| `node:20`            | ~1GB   |
| `node:20-slim`       | ~240MB |
| `node:20-alpine`     | ~180MB |
| `gcr.io/distroless/nodejs20` | ~150MB |
| `scratch` (Go, Rust) | ~10MB  |

`alpine` -- мінімальний Linux, але використовує `musl libc` замість `glibc`. Іноді це ламає бінарні залежності (`bcrypt`, `sharp`).

`distroless` -- образ без shell, package manager, нічого зайвого. Безпечно, але дебажити складніше (`docker exec -it ... sh` не працює).

**2. Multi-stage build** (див. вище).

**3. `.dockerignore`:**
```
node_modules
.git
.env
*.log
tests/
coverage/
Dockerfile
README.md
```

**4. Об'єднання RUN-інструкцій:**
```dockerfile
# ❌ Три шари
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Один шар (менше, бо `rm` фізично видаляє з того ж шару)
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

**5. Не копіюйте зайве:**
```dockerfile
# ❌ Копіюємо весь код, потім білдимо
COPY . .
RUN npm run build

# ✅ Копіюємо тільки потрібне для runtime
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
```

---

## Чим відрізняється `CMD` від `ENTRYPOINT`?

Обидві інструкції визначають, що виконається під час запуску контейнера, але поводяться по-різному.

**`CMD`** -- дефолтна команда. Легко перевизначається аргументами `docker run`.

**`ENTRYPOINT`** -- фіксована команда. Аргументи `docker run` додаються до неї як параметри.

```dockerfile
# Варіант 1: тільки CMD
CMD ["node", "server.js"]

# docker run myapp                → node server.js
# docker run myapp bash           → bash (CMD перевизначено)
```

```dockerfile
# Варіант 2: тільки ENTRYPOINT
ENTRYPOINT ["node", "server.js"]

# docker run myapp                → node server.js
# docker run myapp --port 3000    → node server.js --port 3000
# docker run myapp bash           → node server.js bash ❌
```

```dockerfile
# Варіант 3: ENTRYPOINT + CMD (рекомендовано)
ENTRYPOINT ["node"]
CMD ["server.js"]

# docker run myapp                → node server.js
# docker run myapp worker.js      → node worker.js (CMD перевизначено)
```

**Завжди використовуйте exec-форму** (`["node", "server.js"]`), а не shell-форму (`node server.js`). Shell-форма запускає процес через `/bin/sh -c`, що:
- Створює зайвий shell-процес.
- Ламає обробку сигналів (SIGTERM не доходить до процесу).
- Не працює в `scratch` / `distroless` образах.

---

## Як правильно обробляти сигнали (SIGTERM) у контейнері?

Коли Kubernetes / Docker зупиняє контейнер, він надсилає `SIGTERM` процесу з PID 1. Якщо процес не оброблює сигнал за `grace period` (30 сек за замовчуванням), він отримає `SIGKILL`.

**Проблема shell-форми:**
```dockerfile
# ❌ PID 1 -- це sh, який не проксить SIGTERM до node
CMD node server.js
```

**Рішення 1: exec-форма**
```dockerfile
# ✅ node -- це PID 1, отримає SIGTERM напряму
CMD ["node", "server.js"]
```

**Рішення 2: init-процес** (якщо потрібно запустити кілька процесів або обробити zombie):
```dockerfile
RUN apk add --no-cache tini
ENTRYPOINT ["tini", "--"]
CMD ["node", "server.js"]

# Або в docker run
docker run --init myapp
```

**У застосунку** -- обов'язково обробляйте сигнал для graceful shutdown:
```javascript
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, closing connections...');
  await server.close();          // закрити HTTP
  await db.disconnect();         // закрити БД
  process.exit(0);
});
```

---

## Як писати безпечні Dockerfile?

**1. Non-root user:**
```dockerfile
FROM node:20-alpine
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app
COPY --chown=nodejs:nodejs . .
USER nodejs
CMD ["node", "server.js"]
```

**2. Конкретні версії:**
```dockerfile
FROM node:20.11.1-alpine3.19     # ✅ (замість node:20)
RUN apk add --no-cache curl=8.5.0-r0   # ✅ (замість curl)
```

**3. Не пхайте секрети у шари:**
```dockerfile
# ❌ SECRET_KEY залишиться в історії образу
ARG SECRET_KEY
RUN curl -H "Auth: $SECRET_KEY" ...

# ✅ BuildKit secrets (не зберігаються в шарах)
RUN --mount=type=secret,id=mysecret \
    curl -H "Auth: $(cat /run/secrets/mysecret)" ...
```

```bash
# Білд з секретом
DOCKER_BUILDKIT=1 docker build --secret id=mysecret,src=./secret.txt .
```

**4. Сканування на вразливості:**
```bash
# Trivy -- популярний scanner
trivy image myapp:1.0

# Docker Scout
docker scout cves myapp:1.0

# Snyk
snyk container test myapp:1.0
```

**5. Мінімізуйте attack surface:**
- `distroless` або `scratch` для compiled-мов (Go, Rust).
- Read-only rootfs: `docker run --read-only myapp`.
- Drop capabilities: `--cap-drop=ALL --cap-add=NET_BIND_SERVICE`.

---

## Приклад production-ready Dockerfile для Node.js

```dockerfile
# syntax=docker/dockerfile:1.4

# ---- Stage 1: Dependencies ----
FROM node:20.11.1-alpine3.19 AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

# ---- Stage 2: Build ----
FROM node:20.11.1-alpine3.19 AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .
RUN npm run build

# ---- Stage 3: Runtime ----
FROM node:20.11.1-alpine3.19 AS runner
WORKDIR /app

# Встановити tini для коректної обробки сигналів
RUN apk add --no-cache tini

# Non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

# Скопіювати тільки те, що треба для runtime
COPY --from=deps --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs package.json ./

USER nodejs
EXPOSE 3000

ENV NODE_ENV=production
ENV PORT=3000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/server.js"]
```

Що тут важливо:
- Три стадії: залежності, білд, runtime.
- Версії пінуються.
- BuildKit cache mounts для прискорення.
- Non-root user.
- `tini` для обробки сигналів.
- Healthcheck для оркестраторів.
- Змінні середовища для конфігурації.

---

## Що таке `HEALTHCHECK` і чи треба він?

`HEALTHCHECK` дозволяє Docker періодично перевіряти стан контейнера.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1
```

Параметри:
- `interval` -- як часто перевіряти.
- `timeout` -- максимальний час відповіді.
- `start-period` -- період прогріву (не рахується в retries).
- `retries` -- скільки послідовних фейлів перед `unhealthy`.

**У Kubernetes** `HEALTHCHECK` ігнорується -- там своя система probes (`livenessProbe`, `readinessProbe`). Тому для k8s-деплоїв `HEALTHCHECK` необов'язковий.

**У docker-compose / standalone** -- корисний, щоб інші сервіси чекали, поки залежність стане healthy (`depends_on: condition: service_healthy`).
