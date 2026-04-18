# Docker: основи

## Що таке Docker і яку проблему він вирішує?

Docker -- це платформа контейнеризації, яка дозволяє запакувати застосунок разом з усіма залежностями (бібліотеки, runtime, конфігурація) у легкий, портативний контейнер. Контейнер гарантує, що застосунок запуститься однаково на будь-якій машині, де є Docker Engine.

Проблеми, які вирішує Docker:
- **"Works on my machine"** -- різниця між dev, staging, production середовищами.
- **Ізоляція процесів** -- кілька сервісів на одному хості не конфліктують між собою.
- **Швидкий деплой** -- образ будується один раз, запускається будь-де.
- **Масштабування** -- легко підняти N копій контейнера.

```
┌─────────────────────────────────────────────────────────┐
│               Віртуальна машина (VM)                    │
│  ┌────────┐  ┌────────┐  ┌────────┐                     │
│  │  App A │  │  App B │  │  App C │                     │
│  ├────────┤  ├────────┤  ├────────┤                     │
│  │ Bins/  │  │ Bins/  │  │ Bins/  │                     │
│  │ Libs   │  │ Libs   │  │ Libs   │                     │
│  ├────────┤  ├────────┤  ├────────┤                     │
│  │ Guest  │  │ Guest  │  │ Guest  │   ← Кожна VM        │
│  │ OS     │  │ OS     │  │ OS     │     має свою ОС     │
│  └────────┘  └────────┘  └────────┘                     │
│              Hypervisor                                 │
│              Host OS                                    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 Docker контейнери                       │
│  ┌────────┐  ┌────────┐  ┌────────┐                     │
│  │  App A │  │  App B │  │  App C │                     │
│  ├────────┤  ├────────┤  ├────────┤                     │
│  │ Bins/  │  │ Bins/  │  │ Bins/  │                     │
│  │ Libs   │  │ Libs   │  │ Libs   │                     │
│  └────────┘  └────────┘  └────────┘                     │
│              Docker Engine                              │
│              Host OS (спільне ядро)   ← Спільне ядро    │
└─────────────────────────────────────────────────────────┘
```

---

## Чим контейнер відрізняється від віртуальної машини?

| Характеристика      | Контейнер                  | VM                             |
|---------------------|----------------------------|--------------------------------|
| Ізоляція            | Процесна (namespaces)      | Апаратна (hypervisor)          |
| Ядро ОС             | Спільне з хостом            | Власне (Guest OS)              |
| Розмір              | МБ                          | ГБ                             |
| Час запуску         | Секунди                     | Хвилини                        |
| Щільність на хості  | Сотні                       | Десятки                        |
| Безпека             | Нижча (ядро спільне)        | Вища (повна ізоляція)          |

Контейнер -- це **ізольований процес** на хості, а не повноцінна машина. Ізоляція досягається завдяки Linux-механізмам:
- **Namespaces** -- ізолюють PID, мережу, точки монтування, users, IPC, UTS (hostname).
- **cgroups** -- обмежують ресурси (CPU, RAM, I/O).
- **Union filesystems** (OverlayFS) -- шарова файлова система для образів.

---

## Чим відрізняється образ (image) від контейнера (container)?

**Образ** -- це незмінний (immutable) шаблон, який містить файлову систему, залежності та метадані для запуску застосунку. Образ складається з шарів (layers), кожен з яких є результатом однієї інструкції в Dockerfile.

**Контейнер** -- це запущений екземпляр образу з додатковим writable-шаром (copy-on-write). Можна створити багато контейнерів з одного образу.

Аналогія: образ -- це клас, контейнер -- об'єкт цього класу.

```
┌─────────────────────────────┐
│   Container (writable)      │  ← Тут змінюються файли
├─────────────────────────────┤
│   Layer: CMD ["node","app"] │
├─────────────────────────────┤
│   Layer: COPY . /app        │  ← Read-only шари
├─────────────────────────────┤
│   Layer: RUN npm install    │
├─────────────────────────────┤
│   Layer: FROM node:20       │
└─────────────────────────────┘
```

```bash
# Побудувати образ
docker build -t myapp:1.0 .

# Запустити контейнер з образу
docker run -d --name myapp-1 myapp:1.0
docker run -d --name myapp-2 myapp:1.0  # другий контейнер з того ж образу

# Контейнери ізольовані один від одного
```

---

## Як працює шарова файлова система Docker?

Кожна інструкція в Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`) створює окремий шар. Шари кешуються і переюзяться між образами.

Переваги:
- **Економія місця** -- якщо два образи мають однакові базові шари, вони зберігаються на диску один раз.
- **Швидкий білд** -- якщо шар не змінився, Docker бере його з кешу.
- **Швидке завантаження** -- з registry качаються тільки відсутні шари.

```dockerfile
FROM node:20-alpine          # Layer 1 (~50MB, з registry)
WORKDIR /app                 # Layer 2 (metadata only)
COPY package*.json ./        # Layer 3 (залежить від package.json)
RUN npm ci                   # Layer 4 (залежить від Layer 3)
COPY . .                     # Layer 5 (залежить від всього коду)
CMD ["node", "server.js"]    # Layer 6 (metadata only)
```

Якщо ви змінили код (але не package.json) -- Docker переюзне шари 1-4 з кешу і перебудує тільки шар 5.

---

## Які основні команди Docker потрібно знати?

```bash
# ---- Образи ----
docker build -t myapp:1.0 .       # Зібрати образ з Dockerfile
docker images                      # Список локальних образів
docker pull nginx:latest           # Завантажити з Docker Hub
docker push myapp:1.0              # Відправити в registry
docker rmi myapp:1.0               # Видалити образ
docker history myapp:1.0           # Показати шари образу

# ---- Контейнери ----
docker run -d -p 8080:80 --name web nginx   # Запустити в detached режимі
docker ps                                    # Запущені контейнери
docker ps -a                                 # Всі, включно зі зупиненими
docker logs -f web                           # Дивитись логи в реальному часі
docker exec -it web sh                       # Зайти всередину контейнера
docker stop web && docker rm web             # Зупинити та видалити
docker stats                                 # Моніторинг ресурсів

# ---- Volumes та мережі ----
docker volume create mydata
docker run -v mydata:/var/lib/postgresql/data postgres
docker network create mynet
docker run --network mynet --name api myapp

# ---- Очистка ----
docker system prune -a             # Видалити все невикористане
docker system df                   # Скільки місця займає Docker
```

---

## Що таке Docker Registry?

Registry -- це сховище Docker-образів. Найпоширеніші:
- **Docker Hub** -- публічний, дефолтний.
- **AWS ECR, Google GCR, Azure ACR** -- хмарні приватні registries.
- **GitHub Container Registry (ghcr.io)** -- інтегрований з GitHub.
- **Harbor, Nexus, Artifactory** -- self-hosted рішення.

Tag -- це версія образу. Правила іменування: `registry/namespace/image:tag`.

```bash
# Повне ім'я образу
ghcr.io/myorg/myapp:v1.2.3

# Локальні теги
docker tag myapp:1.0 myapp:latest
docker tag myapp:1.0 ghcr.io/myorg/myapp:v1.0

# Логін та push
docker login ghcr.io -u USERNAME
docker push ghcr.io/myorg/myapp:v1.0
```

**Важливо:** не використовуйте тег `latest` в production. Завжди прив'язуйтесь до конкретної версії (semver або git SHA), щоб мати відтворюваність деплоїв.

---

## Як Docker ізолює ресурси контейнерів?

Docker використовує вбудовані Linux-механізми:

**1. Namespaces** (ізоляція видимості):
- `pid` -- контейнер бачить тільки свої процеси.
- `net` -- власний мережевий стек (інтерфейси, IP, routing).
- `mnt` -- власні точки монтування.
- `uts` -- свій hostname.
- `ipc` -- ізоляція IPC (shared memory, semaphores).
- `user` -- маппінг user ID між контейнером і хостом.

**2. cgroups** (обмеження ресурсів):
```bash
# Обмежити CPU і пам'ять
docker run --cpus="1.5" --memory="512m" myapp

# Під капотом Docker створює cgroup з лімітами
```

**3. Capabilities** (обмеження привілеїв root-процесу):
```bash
# Видалити capability CAP_NET_RAW (заборонити raw sockets)
docker run --cap-drop=NET_RAW myapp

# Запустити без привілеїв (рекомендується)
docker run --user 1000:1000 myapp
```

**4. Seccomp, AppArmor, SELinux** -- додаткові рівні ізоляції системних викликів.

---

## Що відбувається, коли ви виконуєте `docker run`?

Послідовність дій:

1. **Docker CLI** відправляє запит до **Docker Daemon** через UNIX-сокет.
2. Daemon перевіряє, чи є образ локально. Якщо немає -- завантажує з registry (`docker pull`).
3. Створюється новий контейнер: розпаковуються шари, додається writable-шар.
4. Створюються namespaces та cgroups для ізоляції.
5. Налаштовується мережа (IP, iptables правила для port mapping).
6. Монтуються volumes.
7. Запускається процес `ENTRYPOINT` + `CMD` всередині namespace-ів.
8. STDOUT/STDERR стримиться назад в Docker Daemon (і доступний через `docker logs`).

```
docker CLI  →  Docker Daemon (dockerd)  →  containerd  →  runc  →  процес
                      │
                      ├─ Registry (pull образу)
                      ├─ Storage driver (OverlayFS)
                      └─ Networking (bridge, CNI)
```

---

## Типові пастки та антипатерни

**1. Використання `latest` у production:**
```dockerfile
FROM node:latest   # ❌ непередбачувані оновлення
FROM node:20.11.1-alpine3.19   # ✅ конкретна версія
```

**2. Запуск від root:**
```dockerfile
# ❌ Процес з правами root -- ризик для безпеки
CMD ["node", "server.js"]

# ✅ Створити та використати non-root user
RUN adduser -D appuser
USER appuser
CMD ["node", "server.js"]
```

**3. Збереження state всередині контейнера:**
Контейнер може бути знищений у будь-який момент. Всі дані, які мають пережити перезапуск, -- виносьте у volumes або зовнішні сховища (S3, RDS).

**4. Великі образи:**
Використовуйте `alpine` або `distroless` базові образи, multi-stage build (про це -- окрема тема), `.dockerignore` для виключення непотрібних файлів.

**5. Один контейнер -- один процес:**
Не запускайте systemd, supervisor, cron всередині контейнера. Якщо потрібно кілька процесів -- це кілька контейнерів, які оркеструються (Docker Compose, Kubernetes).
