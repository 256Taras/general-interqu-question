# Docker Compose

## Що таке Docker Compose і коли його використовувати?

Docker Compose -- це інструмент для визначення і запуску multi-container застосунків через декларативний YAML-файл. Замість десятка `docker run` команд, ви описуєте весь стек в `docker-compose.yml` і стартуєте його однією командою.

**Коли використовувати:**
- **Локальна розробка** -- швидко підняти застосунок + БД + Redis + інші залежності.
- **Інтеграційні тести** -- ізольоване тестове середовище в CI.
- **Прості production-деплої** -- один сервер, кілька сервісів (не для масштабованих систем).

**Коли НЕ використовувати:**
- Production з декількома серверами, auto-scaling, rolling updates -- використовуйте Kubernetes.
- Складні deployments з різними регіонами -- Kubernetes / Nomad.

---

## Базовий приклад `docker-compose.yml`

```yaml
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

```bash
docker compose up -d          # запустити все у фоні
docker compose logs -f api    # дивитись логи конкретного сервісу
docker compose ps             # список сервісів
docker compose exec api sh    # зайти всередину
docker compose down           # зупинити і видалити контейнери
docker compose down -v        # + видалити volumes (обережно!)
```

---

## Як працюють мережі в Docker Compose?

За замовчуванням Compose створює bridge-мережу для проекту, де всі сервіси можуть звертатися один до одного за іменем сервісу як DNS.

```yaml
services:
  api:
    image: myapp
    environment:
      DB_HOST: db        # ← DNS-ім'я = ім'я сервісу
  db:
    image: postgres
```

Під капотом Docker піднімає embedded DNS-сервер, який резолвить імена контейнерів в IP. Це працює тільки в межах однієї мережі.

**Кастомні мережі** для сегментації:
```yaml
services:
  api:
    networks:
      - frontend
      - backend
  db:
    networks:
      - backend        # ← БД недоступна з frontend мережі
  nginx:
    networks:
      - frontend

networks:
  frontend:
  backend:
```

Це корисно, наприклад, щоб публічний Nginx мав доступ до API, але не до БД напряму.

---

## Volumes vs bind mounts: у чому різниця?

**Bind mount** -- монтування директорії з хоста в контейнер:
```yaml
services:
  api:
    volumes:
      - ./src:/app/src        # ./src на хості → /app/src у контейнері
      - ./logs:/var/log/app
```

Використовується для локальної розробки (hot reload) -- зміни в коді на хості одразу видно в контейнері.

**Named volume** -- Docker керує сховищем:
```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:       # Docker створює в /var/lib/docker/volumes/
```

Використовується для persistent даних (БД, файли користувачів). Переваги:
- Docker керує lifecycle.
- Можна бекапити через `docker run --rm -v pgdata:/data -v $(pwd):/backup alpine tar czf /backup/pgdata.tar.gz /data`.
- Переносяться між хостами.
- На Mac/Windows швидші за bind mounts (які йдуть через віртуалізацію).

| Аспект               | Bind mount                 | Named volume              |
|----------------------|----------------------------|---------------------------|
| Шлях                 | Довільний на хості          | Керується Docker          |
| Продуктивність (Mac) | Повільніше                  | Швидше                    |
| Backup               | Звичайними файловими інструментами | Через Docker volumes |
| Use case             | Код у dev                   | Persistent production data |

---

## Як робити `depends_on` з очікуванням готовності?

Проста форма `depends_on` гарантує тільки порядок запуску, а не готовність:
```yaml
api:
  depends_on:
    - db        # ❌ API стартує, як тільки db контейнер запущений,
                #    навіть якщо Postgres всередині ще не готовий
```

**Правильно** -- через `healthcheck` + `service_healthy`:
```yaml
api:
  depends_on:
    db:
      condition: service_healthy    # ← чекати healthy статус
    cache:
      condition: service_started    # ← достатньо запуску

db:
  image: postgres:16
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
    interval: 5s
    timeout: 3s
    retries: 10
    start_period: 10s
```

**Альтернатива без healthcheck** -- скрипт `wait-for-it.sh` або `dockerize` всередині контейнера, який чекає TCP-порту:
```dockerfile
CMD ["./wait-for-it.sh", "db:5432", "--", "node", "server.js"]
```

---

## Як розділяти конфігурацію між середовищами?

Docker Compose підтримує **override-файли**. За замовчуванням підхоплюється `docker-compose.override.yml` поверх `docker-compose.yml`.

**Базовий `docker-compose.yml` (production-like):**
```yaml
services:
  api:
    image: myapp:1.0
    environment:
      NODE_ENV: production
    restart: always
```

**`docker-compose.override.yml` (dev, auto-merge):**
```yaml
services:
  api:
    build: .                    # замість готового образу -- білдимо локально
    environment:
      NODE_ENV: development
    volumes:
      - ./src:/app/src          # hot reload
    command: npm run dev
    ports:
      - "9229:9229"             # debug port
```

**`docker-compose.prod.yml`:**
```yaml
services:
  api:
    image: myregistry.com/myapp:${VERSION}
    environment:
      NODE_ENV: production
    deploy:
      replicas: 3
```

```bash
# Dev (підхоплюється override автоматично)
docker compose up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**Змінні середовища** з `.env` файлу:
```
# .env
VERSION=1.2.3
POSTGRES_PASSWORD=supersecret
```

```yaml
services:
  api:
    image: myapp:${VERSION}
  db:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

**НЕ коммітьте `.env` з production-секретами в git** -- додайте до `.gitignore`, тримайте шаблон в `.env.example`.

---

## Profiles: як запускати підмножину сервісів?

```yaml
services:
  api:
    image: myapp

  db:
    image: postgres

  # Сервіси тільки для dev-середовища
  pgadmin:
    image: dpage/pgadmin4
    profiles: ["dev-tools"]

  mailhog:
    image: mailhog/mailhog
    profiles: ["dev-tools"]

  # Сервіс тільки для тестів
  test-runner:
    build: ./tests
    profiles: ["test"]
```

```bash
docker compose up                        # тільки api + db
docker compose --profile dev-tools up    # + pgadmin + mailhog
docker compose --profile test up         # + test-runner
```

---

## Як керувати ресурсами контейнерів?

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

Для Compose у standalone-режимі (не Swarm) ліміти застосовуються через:
```yaml
services:
  api:
    cpus: 1.5
    mem_limit: 512m
    mem_reservation: 256m
```

**Важливо для Node.js:** встановлюйте `--max-old-space-size` відповідно до memory limit, інакше V8 не знатиме про обмеження і може впасти з OOM:
```yaml
environment:
  NODE_OPTIONS: "--max-old-space-size=400"    # трохи менше за 512M limit
```

---

## Типові пастки

**1. Порт зайнятий на хості:**
```
Error: port 5432 is already allocated
```
Або інший сервіс вже слухає цей порт, або залишився зупинений контейнер. `docker compose down` або змініть мапінг: `"5433:5432"`.

**2. `depends_on` без healthcheck:**
Розглянуто вище -- сервіс стартує до того, як залежність справді готова приймати запити.

**3. Volumes перекривають COPY:**
```yaml
services:
  api:
    build: .
    volumes:
      - ./src:/app/src        # ← локальні файли перекриють те, що COPY-нуто в образ
      - /app/node_modules     # ← named volume, щоб node_modules у контейнері не перезаписався
```

**4. DNS не резолвить кастомні hostname:**
Сервіси доступні за ім'ям сервісу (`db`, `api`), не за `container_name`. Використовуйте `networks` з `aliases`, якщо треба кілька імен.

**5. `docker compose down -v` видалить всі дані:**
Якщо volume містив БД -- дані зникнуть. Завжди розрізняйте `down` (безпечно) і `down -v` (деструктивно).

---

## `docker compose` vs `docker-compose`

- `docker-compose` (з дефісом) -- старий Python-інструмент (Compose V1), deprecated.
- `docker compose` (без дефіса) -- плагін до Docker CLI (Compose V2), написаний на Go. Швидший, інтегрований. Використовуйте саме його.

Синтаксис однаковий, команди ідентичні.
