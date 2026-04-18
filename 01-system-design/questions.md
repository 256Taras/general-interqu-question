# System Design — Питання для інтерв'ю

## Як би ви спроектували URL Shortener (типу bit.ly)?

### Функціональні вимоги
- Користувач вводить довгий URL → отримує короткий
- Короткий URL перенаправляє на оригінальний
- Можливість кастомних alias-ів
- TTL (термін дії посилання)
- Аналітика (кількість кліків)

### Оцінка масштабу
- 100M нових URL/місяць, 10:1 read/write ratio → 1B reads/місяць
- QPS write: ~40, QPS read: ~400
- Зберігання: 100M × 500 bytes × 12 місяців × 5 років ≈ 300TB

### High-Level дизайн
```
Client → Load Balancer → API Servers → Cache (Redis)
                                      → Database (PostgreSQL/NoSQL)
                                      → Analytics Service
```

### Генерація короткого URL
**Варіант 1: Base62 encoding (a-z, A-Z, 0-9)**
- 7 символів = 62^7 ≈ 3.5 трильйони комбінацій
- Hash довгого URL + collision check

**Варіант 2: Counter-based (pre-generated IDs)**
- Окремий сервіс генерує унікальні ID
- Конвертуємо ID в Base62
- Немає collision-ів

**Варіант 3: Key Generation Service (KGS)**
- Заздалегідь генеруємо ключі і зберігаємо в БД
- При запиті — видаємо з пулу
- Найшвидший варіант

### Database
- **Write:** PostgreSQL або DynamoDB (key-value підходить ідеально)
- **Read:** Redis cache перед БД (cache-aside pattern)
- Партиціонування за hash першого символу ключа

### Масштабування
- Горизонтальне масштабування API серверів
- Redis Cluster для кешу
- Database sharding за діапазоном ключів
- CDN для статичних redirect-ів

---

## Як спроектувати систему чату в реальному часі?

### Функціональні вимоги
- 1-to-1 повідомлення
- Групові чати
- Online/offline статус
- Read receipts
- Typing indicators
- Пошук повідомлень

### Комунікація
- **WebSocket** — для real-time повідомлень (двосторонній)
- **HTTP** — для завантаження історії, пошуку, авторизації

### Архітектура
```
Client ←→ WebSocket Gateway (stateful)
              │
              ▼
         Message Service → Message Queue (Kafka)
              │                    │
              ▼                    ▼
         Message Store        Notification Service
         (Cassandra)          (Push, Email)
```

### Зберігання повідомлень
- **Cassandra** або **ScyllaDB** — оптимізовані для write-heavy навантаження
- Partition key: chat_id, clustering key: timestamp
- Автоматичне видалення старих повідомлень (TTL)

### Доставка повідомлень
1. Клієнт A відправляє повідомлення через WebSocket
2. Gateway визначає WebSocket з'єднання клієнта B
3. Якщо B online — доставка через WebSocket
4. Якщо B offline — зберігання + push notification

### Групові чати
- Fan-out on write: при відправці створюємо копію для кожного учасника
- Fan-out on read: зберігаємо одну копію, читаємо при запиті
- **Вибір:** для маленьких груп (< 100) — fan-out on write, для великих — on read

### Масштабування WebSocket
- Sticky sessions (один користувач → один сервер)
- Redis Pub/Sub для комунікації між серверами
- Connection Gateway → знає, на якому сервері кожен користувач

---

## Як масштабувати додаток до мільйона користувачів?

### Етап 1: Один сервер (< 1000 users)
```
Client → Server (App + DB)
```

### Етап 2: Розділення DB (< 10K users)
```
Client → Server → Database (окремий сервер)
```

### Етап 3: Load Balancer + Read Replicas (< 100K users)
```
Client → Load Balancer → [Server 1, Server 2, Server 3]
                              ↓
                    Primary DB → Read Replica 1
                              → Read Replica 2
```
- Reads йдуть на replicas, writes на primary
- Session storage в Redis (не в пам'яті серверів)

### Етап 4: Caching + CDN (< 500K users)
- Redis/Memcached для hot data
- CDN для статики (зображення, CSS, JS)
- Database indexing та query optimization

### Етап 5: Horizontal Scaling (< 1M users)
- Database sharding
- Microservices для різних доменів
- Message queue для async операцій
- Моніторинг та alerting

### Етап 6: Global Scale (1M+ users)
- Multi-region deployment
- Global CDN
- Database per region з cross-region replication
- Rate limiting та circuit breakers

---

## CAP теорема — що це і як впливає на архітектуру?

**CAP теорема** стверджує, що розподілена система може забезпечити лише **дві з трьох** властивостей:

- **C (Consistency)** — всі вузли бачать одні й ті самі дані одночасно
- **A (Availability)** — кожен запит отримує відповідь (без гарантії актуальності)
- **P (Partition Tolerance)** — система працює при втраті зв'язку між вузлами

**Практичний вибір** (Partition Tolerance обов'язковий у розподілених системах):
- **CP** (Consistency + Partition Tolerance) — PostgreSQL cluster, MongoDB (default), Redis Cluster
  - Коли: банківські операції, inventory management
- **AP** (Availability + Partition Tolerance) — Cassandra, DynamoDB, DNS
  - Коли: соцмережі, кеш, аналітика

**PACELC теорема** (розширення CAP):
- Коли немає Partition: вибір між **L**atency та **C**onsistency
- Більшість систем обирають eventual consistency для кращої latency

---

## Мікросервіси vs Моноліт — коли що обрати?

### Моноліт
**Переваги:**
- Простота розробки та деплою
- Простіший debugging (один процес)
- Немає мережевих затримок між модулями
- Транзакції в одній БД

**Коли:** стартап, маленька команда (< 10 людей), MVP, невизначені вимоги.

### Мікросервіси
**Переваги:**
- Незалежний деплой кожного сервісу
- Різні технології для різних задач
- Масштабування окремих компонентів
- Ізоляція помилок

**Недоліки:**
- Складність інфраструктури (service discovery, monitoring, tracing)
- Мережеві затримки
- Distributed transactions (Saga pattern)
- Дублювання даних
- Потрібна зріла DevOps культура

**Коли:** велика команда (10+ людей), чітко визначені домени, потреба у незалежному масштабуванні.

**Правило:** починай з моноліту, переходь на мікросервіси коли є конкретна потреба. "Modular monolith" — часто найкращий компроміс.

---

## Як спроектувати Rate Limiter?

### Алгоритми

**1. Token Bucket**
- Bucket заповнюється токенами з фіксованою швидкістю
- Кожен запит забирає токен
- Якщо bucket порожній — запит відхиляється
- Дозволяє burst traffic

**2. Sliding Window Log**
- Зберігаємо timestamp кожного запиту
- Рахуємо запити у вікні (останні N секунд)
- Точний, але потребує більше пам'яті

**3. Sliding Window Counter**
- Комбінація fixed window та sliding window
- Зважене середнє між поточним та попереднім вікном
- Компроміс між точністю та пам'яттю

**4. Fixed Window Counter**
- Рахуємо запити у фіксованих інтервалах (1 хв, 1 год)
- Простий, але має проблему на межі вікна

### Реалізація (Redis)
```
-- Token Bucket на Redis
MULTI
  GET rate:user:123
  INCR rate:user:123
  EXPIRE rate:user:123 60
EXEC
```

### Де розміщувати?
1. **API Gateway** — глобальний rate limiting
2. **Application level** — per-user, per-endpoint
3. **Distributed** — Redis для shared state між серверами

### HTTP заголовки
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 57
X-RateLimit-Reset: 1640000000
Retry-After: 30
```

---

## Event-Driven архітектура — переваги та недоліки

### Як працює
```
Service A → Event Bus (Kafka/RabbitMQ) → Service B
                                        → Service C
                                        → Service D
```

Замість прямих викликів між сервісами, вони комунікують через **події**.

### Переваги
- **Loose coupling** — сервіси не знають один про одного
- **Масштабованість** — легко додати нового споживача
- **Resilience** — якщо Service B недоступний, подія залишається в черзі
- **Audit trail** — всі події зберігаються

### Недоліки
- **Складність** — важче дебажити та відстежувати flow
- **Eventual consistency** — дані можуть бути неконсистентними деякий час
- **Порядок подій** — гарантувати порядок складно
- **Дублювання подій** — споживачі повинні бути idempotent

### Паттерни
- **Event Notification** — "щось сталось", мінімум даних
- **Event-Carried State Transfer** — подія містить всі необхідні дані
- **Event Sourcing** — стан системи як послідовність подій
- **CQRS** — окремі моделі для read/write

---

## CQRS + Event Sourcing — коли використовувати?

### CQRS (Command Query Responsibility Segregation)
Розділення моделей для **читання** та **запису**:
```
Write → Command → Command Handler → Write Model (PostgreSQL)
                                         ↓ (events)
Read  → Query  → Query Handler   → Read Model (Elasticsearch/Redis)
```

**Коли:** різні вимоги до read/write (складні writes, прості reads), різне масштабування.

### Event Sourcing
Зберігаємо **послідовність подій** замість поточного стану:
```
UserCreated { name: "John", email: "john@mail.com" }
NameChanged { name: "John Doe" }
EmailChanged { email: "johndoe@mail.com" }
```

**Поточний стан** = replay всіх подій (або snapshot + нові події).

**Переваги:**
- Повний audit trail
- Можна відтворити стан на будь-який момент часу
- Можна додати нові read models ретроспективно

**Недоліки:**
- Складність реалізації
- Event versioning
- Eventual consistency
- Snapshot management

**Коли використовувати:** фінансові системи, audit-heavy домени, systems of record.
**Коли НЕ використовувати:** прості CRUD додатки, коли eventual consistency не підходить.

---

## Database Sharding — стратегії та коли використовувати?

### Що таке Sharding
Горизонтальне розділення даних між кількома серверами БД.

### Стратегії
**1. Range-based sharding**
- Дані розділяються за діапазоном ключа (user_id 1-1M → shard 1, 1M-2M → shard 2)
- Простий, але може створити hotspots

**2. Hash-based sharding**
- hash(key) % num_shards = shard number
- Рівномірний розподіл, але складно додавати/видаляти shards
- Consistent hashing вирішує проблему

**3. Geographic sharding**
- Дані розділені за регіоном (EU → shard EU, US → shard US)
- Зменшує latency для користувачів

**4. Directory-based sharding**
- Lookup service знає, де які дані
- Найгнучкіший, але lookup service — single point of failure

### Проблеми
- **Cross-shard queries** — JOIN між shards дуже дорогий
- **Referential integrity** — foreign keys не працюють між shards
- **Resharding** — переміщення даних при додаванні нового shard
- **Hotspots** — один shard отримує непропорційно більше навантаження

### Коли НЕ потрібен sharding
- Read replicas вирішують проблему
- Вертикальне масштабування (більший сервер) ще можливе
- Caching зменшує навантаження на БД
- Архівація старих даних достатня

---

## Як спроектувати API Gateway?

### Функції API Gateway
1. **Routing** — маршрутизація запитів до відповідних сервісів
2. **Authentication/Authorization** — перевірка JWT, API keys
3. **Rate Limiting** — обмеження кількості запитів
4. **Load Balancing** — розподіл навантаження
5. **Caching** — кешування відповідей
6. **Request/Response transformation** — зміна формату
7. **Monitoring/Logging** — збір метрик, логування
8. **Circuit Breaker** — захист від каскадних збоїв

### Архітектура
```
Client → API Gateway → Auth Service
                     → User Service
                     → Order Service
                     → Payment Service
```

### Реалізації
- **Kong** — open-source, на Nginx, плагіни на Lua
- **AWS API Gateway** — managed, serverless
- **Envoy** — cloud-native, service mesh
- **Nginx** — reverse proxy з конфігурацією
- **Express Gateway** — Node.js based

### Паттерни
- **BFF (Backend for Frontend)** — окремий gateway для кожного клієнта (web, mobile)
- **Edge Gateway** — єдиний вхід для всіх клієнтів

### Ризики
- **Single Point of Failure** — потрібна висока доступність
- **Bottleneck** — може стати вузьким місцем
- **Complexity** — додатковий шар, який потрібно підтримувати

---

## Як забезпечити високу доступність системи?

### Рівні доступності
- **99%** — ~3.65 дні downtime/рік
- **99.9%** — ~8.77 годин/рік
- **99.99%** — ~52.6 хвилин/рік
- **99.999%** — ~5.26 хвилин/рік

### Стратегії
1. **Redundancy** — дублювання кожного компонента (servers, DB, LB)
2. **Load Balancing** — розподіл між здоровими серверами
3. **Health Checks** — автоматичне видалення нездорових серверів
4. **Auto-scaling** — автоматичне масштабування під навантаженням
5. **Database Replication** — master-slave, multi-master
6. **Multi-region** — deployment у кількох дата-центрах
7. **Graceful degradation** — зменшення функціональності замість повної відмови
8. **Circuit Breaker** — запобігання каскадних збоїв
9. **Chaos Engineering** — регулярне тестування відмовостійкості

### Паттерни
- **Active-Passive** — один активний, один standby (failover)
- **Active-Active** — обидва обробляють трафік
- **Blue-Green Deployment** — два ідентичних environment-и

---

## Як спроектувати систему сповіщень?

### Типи сповіщень
- Push notifications (mobile)
- Email
- SMS
- In-app notifications
- WebSocket (real-time)

### Архітектура
```
Trigger Service → Notification Service → Priority Queue
                                              │
                                    ┌─────────┼─────────┐
                                    ▼         ▼         ▼
                               Push Service  Email   SMS Service
                               (FCM/APNs)   Service  (Twilio)
```

### Ключові аспекти
- **Template engine** — шаблони для різних каналів
- **User preferences** — які канали увімкнені для кожного типу
- **Rate limiting** — не засипати користувача сповіщеннями
- **Deduplication** — не надсилати одне сповіщення двічі
- **Priority** — критичні (security alert) vs звичайні (marketing)
- **Retry з exponential backoff** — при помилках доставки
- **Analytics** — delivered, opened, clicked

---

## Як спроектувати систему кешування?

### Рівні кешування
1. **Browser cache** — HTTP Cache headers (Cache-Control, ETag)
2. **CDN** — статичні ресурси ближче до користувача
3. **Application cache** — Redis/Memcached
4. **Database cache** — query cache, buffer pool

### Стратегії кешування
**Cache-Aside (Lazy Loading):**
```
Read: Cache hit? → return
      Cache miss? → read DB → write to cache → return
Write: Write DB → invalidate cache
```

**Write-Through:**
```
Write: Write cache → cache writes DB (synchronous)
Read: Always from cache
```

**Write-Behind (Write-Back):**
```
Write: Write cache → cache writes DB asynchronously (batch)
```

### Cache Invalidation
"Є лише дві складні речі в CS: інвалідація кешу та назви речей."

- **TTL** — автоматичне видалення через час
- **Event-based** — видаляємо при зміні даних
- **Version-based** — ключ включає версію
- **Cache stampede prevention** — lock + stale-while-revalidate

### Eviction policies
- **LRU** (Least Recently Used) — найпоширеніший
- **LFU** (Least Frequently Used)
- **FIFO** (First In First Out)
- **TTL-based**

---

## Як працює балансування навантаження?

### Алгоритми
1. **Round Robin** — по черзі між серверами
2. **Weighted Round Robin** — з урахуванням потужності серверів
3. **Least Connections** — на сервер з найменшою кількістю з'єднань
4. **IP Hash** — один клієнт завжди на один сервер (sticky sessions)
5. **Random** — випадковий вибір
6. **Least Response Time** — на сервер з найшвидшою відповіддю

### Рівні балансування
- **L4 (Transport)** — балансування на рівні TCP/UDP (швидше, менше можливостей)
- **L7 (Application)** — балансування на рівні HTTP (routing за URL, headers)

### Реалізації
- **Nginx** — reverse proxy + load balancer
- **HAProxy** — спеціалізований LB
- **AWS ALB/NLB** — managed cloud LB
- **Envoy** — cloud-native proxy
- **DNS Round Robin** — найпростіший (без health checks)

### Health Checks
- **Active** — LB періодично перевіряє /health endpoint
- **Passive** — LB відстежує помилки при реальних запитах
- При failure — сервер видаляється з пулу автоматично
