# System Design Patterns — Довідник

## Load Balancing

### Алгоритми балансування

**Round Robin** — найпростіший, запити розподіляються по черзі:
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (повторення)
```

**Weighted Round Robin** — з урахуванням потужності:
```
Server A (weight: 3) — отримує 3 запити
Server B (weight: 1) — отримує 1 запит
Server C (weight: 2) — отримує 2 запити
```

**Least Connections** — на сервер з найменшою кількістю активних з'єднань. Найкращий для запитів з різним часом виконання.

**IP Hash** — hash(client_ip) % num_servers. Гарантує, що один клієнт завжди потрапляє на один сервер. Корисний для sticky sessions.

**Least Response Time** — на сервер, який відповідає найшвидше. Враховує і кількість з'єднань, і latency.

### L4 vs L7 Load Balancing

| | L4 (Transport) | L7 (Application) |
|---|---|---|
| Рівень | TCP/UDP | HTTP/HTTPS |
| Швидкість | Швидший | Повільніший |
| Можливості | IP + Port routing | URL, Headers, Cookies routing |
| SSL termination | Ні | Так |
| Приклад | AWS NLB, HAProxy | AWS ALB, Nginx |

---

## Caching Strategies

### Cache-Aside (Lazy Loading)
```
1. Читання: перевіряємо cache
2. Cache hit → повертаємо з cache
3. Cache miss → читаємо з DB → зберігаємо в cache → повертаємо
4. Запис: записуємо в DB → інвалідуємо cache
```
**Переваги:** тільки потрібні дані в cache, resilient до падіння cache
**Недоліки:** cache miss = повільніше (DB + cache write), stale data можливий

### Write-Through
```
1. Запис: записуємо в cache + DB одночасно (синхронно)
2. Читання: завжди з cache
```
**Переваги:** cache завжди актуальний
**Недоліки:** повільніший write (два записи), cache може містити рідко читані дані

### Write-Behind (Write-Back)
```
1. Запис: записуємо ТІЛЬКИ в cache
2. Cache асинхронно записує в DB (batch)
3. Читання: завжди з cache
```
**Переваги:** дуже швидкий write
**Недоліки:** ризик втрати даних при падінні cache

### Read-Through
```
1. Читання: запит йде в cache
2. Cache miss → cache САМА читає з DB → повертає
```
Схоже на Cache-Aside, але cache відповідає за завантаження даних.

---

## Database Replication

### Master-Slave (Primary-Replica)
```
Writes → Master ──replication──→ Slave 1 (reads)
                                → Slave 2 (reads)
                                → Slave 3 (reads)
```
**Переваги:** масштабування reads, backup, географічне розподілення
**Недоліки:** replication lag (eventual consistency), master — single point of failure

### Master-Master (Multi-Master)
```
Writes/Reads → Master 1 ←──replication──→ Master 2 ← Writes/Reads
```
**Переваги:** висока доступність для writes, географічне розподілення
**Недоліки:** конфлікти записів, складніша consistency

### Synchronous vs Asynchronous replication
- **Sync:** master чекає підтвердження від slave → сильна consistency, повільніше
- **Async:** master не чекає → швидше, але можлива втрата даних при збої

---

## Message Queues

### RabbitMQ
- **AMQP** протокол
- **Routing** через exchanges (direct, topic, fanout, headers)
- **Acknowledgements** — гарантія доставки
- **Найкраще для:** task queues, RPC, складний routing

### Apache Kafka
- **Distributed log** — повідомлення зберігаються на диску
- **Consumer groups** — паралельна обробка
- **Replay** — можна перечитати повідомлення
- **Найкраще для:** event streaming, high throughput, audit log

### Redis Pub/Sub
- **Fire-and-forget** — без persistence
- **Найпростіший** у налаштуванні
- **Найкраще для:** real-time notifications, lightweight messaging

### Порівняння

| | RabbitMQ | Kafka | Redis Pub/Sub |
|---|---|---|---|
| Throughput | Середній | Дуже високий | Високий |
| Persistence | Так | Так (disk) | Ні |
| Ordering | Per queue | Per partition | Ні |
| Replay | Ні | Так | Ні |
| Складність | Середня | Висока | Низька |

---

## Circuit Breaker Pattern

Захист від каскадних збоїв у розподілених системах.

### Стани
```
CLOSED (нормальна робота)
  ↓ (помилки перевищили поріг)
OPEN (всі запити відхиляються)
  ↓ (після timeout)
HALF-OPEN (пропускаємо тестовий запит)
  ↓ (успіх → CLOSED / помилка → OPEN)
```

### Параметри
- **Failure threshold** — кількість помилок для переходу в OPEN
- **Timeout** — час в стані OPEN перед HALF-OPEN
- **Success threshold** — кількість успішних запитів для повернення в CLOSED

### Реалізація (концептуально)
```
function callService(request) {
  if (circuitBreaker.state === "OPEN") {
    if (timeout не минув) return fallback();
    circuitBreaker.state = "HALF-OPEN";
  }

  try {
    result = service.call(request);
    circuitBreaker.onSuccess();
    return result;
  } catch (error) {
    circuitBreaker.onFailure();
    return fallback();
  }
}
```

**Бібліотеки:** opossum (Node.js), Hystrix (Java), Polly (.NET)

---

## Saga Pattern

Управління розподіленими транзакціями без 2PC.

### Choreography (хореографія)
Кожен сервіс публікує подію, наступний реагує:
```
Order Service → OrderCreated
                    ↓
Payment Service → PaymentProcessed
                    ↓
Inventory Service → InventoryReserved
                    ↓
Shipping Service → ShipmentCreated
```

**При помилці:** compensating transactions у зворотному порядку.

### Orchestration (оркестрація)
Центральний оркестратор координує кроки:
```
Saga Orchestrator → create order
                  → process payment
                  → reserve inventory
                  → create shipment
```

**При помилці:** оркестратор викликає компенсаційні дії.

### Порівняння

| | Choreography | Orchestration |
|---|---|---|
| Coupling | Loose | Більше (через оркестратор) |
| Складність | Складно відстежувати flow | Чіткий flow |
| Single point of failure | Ні | Оркестратор |
| Масштабування | Краще | Оркестратор може стати bottleneck |

---

## CQRS Pattern

### Концепція
```
Command Side (Write):
  Command → Validation → Business Logic → Write DB

Query Side (Read):
  Query → Read-optimized DB → Response
```

### Варіанти
**Проста CQRS** — різні моделі, одна БД:
```
Write Model (normalized) ← PostgreSQL → Read Model (denormalized views)
```

**Повна CQRS** — різні БД:
```
Write → PostgreSQL ──events──→ Elasticsearch (search)
                             → Redis (cache)
                             → MongoDB (documents)
```

### Коли використовувати
- Різні вимоги до read/write performance
- Складні queries, які потребують denormalized даних
- Різне масштабування read/write

### Коли НЕ використовувати
- Простий CRUD
- Маленький проект
- Коли eventual consistency не підходить

---

## Event Sourcing

### Концепція
Замість зберігання поточного стану — зберігаємо **всі події**:
```
Event Store:
  1. AccountCreated { id: 123, balance: 0 }
  2. MoneyDeposited { id: 123, amount: 100 }
  3. MoneyWithdrawn { id: 123, amount: 30 }
  4. MoneyDeposited { id: 123, amount: 50 }

Current state: balance = 0 + 100 - 30 + 50 = 120
```

### Snapshots
Для оптимізації — періодично зберігаємо поточний стан:
```
Snapshot (after event 100): { balance: 500 }
Events 101-150: replay тільки ці
```

### Event Versioning
Коли схема події змінюється:
- **Upcasting** — трансформація старих подій при читанні
- **Versioned events** — v1, v2 з різними схемами

---

## Bulkhead Pattern

Ізоляція компонентів для запобігання каскадних збоїв:
```
Thread Pool A (User Service) ─── max 10 threads
Thread Pool B (Order Service) ── max 20 threads
Thread Pool C (Payment Service) ─ max 5 threads
```

Якщо User Service перевантажений і вичерпав свій пул — Order та Payment продовжують працювати.

**Типи ізоляції:**
- **Thread pool isolation** — окремий пул для кожного сервісу
- **Connection pool isolation** — окремий connection pool
- **Process isolation** — окремі контейнери/pods

---

## Retry Pattern з Exponential Backoff

```
Attempt 1: wait 0s   → request
Attempt 2: wait 1s   → request
Attempt 3: wait 2s   → request
Attempt 4: wait 4s   → request
Attempt 5: wait 8s   → request (max retries reached)
```

**З jitter** (розкидування) — щоб уникнути thundering herd:
```
wait = min(base * 2^attempt + random(0, 1000ms), max_delay)
```

**Важливо:**
- Retry тільки для **transient** помилок (мережа, timeout)
- НЕ retry для **permanent** помилок (400, 401, 404)
- Встановити **max retries** та **max delay**
- Зробити операції **idempotent**

---

## Strangler Fig Pattern

Поступова міграція з моноліту на мікросервіси:

```
Етап 1: Всі запити → Моноліт

Етап 2: /users/* → Моноліт
        /orders/* → Новий Order Service

Етап 3: /users/* → Новий User Service
        /orders/* → Order Service
        /legacy/* → Моноліт

Етап 4: Моноліт видалений
```

**Як реалізувати:**
1. API Gateway або Reverse Proxy як "фасад"
2. Нові фічі — одразу як мікросервіс
3. Поступово виносимо функціональність з моноліту
4. Обидві системи працюють паралельно

---

## Service Mesh

### Що це
Інфраструктурний шар для управління комунікацією між сервісами.

### Як працює (Sidecar pattern)
```
Pod A:
  ├── App Container A
  └── Sidecar Proxy (Envoy)
           ↕
Pod B:
  ├── App Container B
  └── Sidecar Proxy (Envoy)
```

Весь трафік між сервісами проходить через proxy.

### Що дає
- **mTLS** — автоматичне шифрування між сервісами
- **Traffic management** — canary deployments, retries, timeouts
- **Observability** — distributed tracing, metrics
- **Access control** — policy-based routing

### Реалізації
- **Istio** — найпопулярніший, на Envoy
- **Linkerd** — легший, простіший
- **Consul Connect** — від HashiCorp

### Коли потрібен
- 10+ мікросервісів
- Потрібен mTLS між сервісами
- Складний traffic management
- Потрібна observability без зміни коду
