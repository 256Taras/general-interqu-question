# Software Architecture — Питання для інтерв'ю

## Чим відрізняється мікросервісна архітектура від моноліту?

### Моноліт
Весь додаток — один deployable unit:
```
┌─────────────────────────────┐
│  Monolith Application       │
│  ├── User Module            │
│  ├── Order Module           │
│  ├── Payment Module         │
│  ├── Notification Module    │
│  └── Shared Database        │
└─────────────────────────────┘
```

**Переваги:** простий deployment, простіший debugging, транзакції в одній БД, менше мережевих затримок.
**Недоліки:** складно масштабувати окремі модулі, один баг може зламати все, великий codebase.

### Мікросервіси
Кожен сервіс — окремий process з власною БД:
```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ User Svc │  │Order Svc │  │Pay Svc   │
│   + DB   │  │   + DB   │  │   + DB   │
└──────────┘  └──────────┘  └──────────┘
     ↕              ↕             ↕
          Message Bus / API Gateway
```

**Переваги:** незалежний deployment, масштабування окремих сервісів, ізоляція помилок, різні технології.
**Недоліки:** складність інфраструктури, мережеві затримки, distributed transactions, дублювання даних.

### Modular Monolith — золота середина
```
┌──────────────────────────────────────┐
│  Application                         │
│  ┌──────┐  ┌──────┐  ┌──────┐       │
│  │User  │  │Order │  │Pay   │       │
│  │Module│  │Module│  │Module│       │
│  └──┬───┘  └──┬───┘  └──┬───┘       │
│     └─────────┴────────┘            │
│           Shared DB                  │
└──────────────────────────────────────┘
```

Модулі чітко розділені (як мікросервіси), але деплояться як один unit. Можна пізніше виділити в окремий сервіс.

---

## Що таке Domain-Driven Design (DDD)?

### Основні концепції

**Ubiquitous Language** — єдина мова між розробниками та бізнесом. Якщо бізнес каже "замовлення", в коді має бути `Order`, а не `Transaction`.

**Bounded Context** — чіткі межі контексту, де терміни мають конкретне значення:
```
E-commerce Context:           Shipping Context:
  Order = items + payment       Order = package + address
  Customer = buyer              Customer = recipient
```

**Entities** — об'єкти з унікальною ідентичністю (User, Order):
```typescript
class Order {
  readonly id: string;     // identity
  items: OrderItem[];
  status: OrderStatus;
}
```

**Value Objects** — об'єкти без ідентичності, визначаються значенням (Money, Address):
```typescript
class Money {
  constructor(readonly amount: number, readonly currency: string) {}
  equals(other: Money) {
    return this.amount === other.amount && this.currency === other.currency;
  }
}
```

**Aggregates** — кластер пов'язаних entities з одним кореневим (Aggregate Root):
```
Order (Aggregate Root)
  ├── OrderItem
  ├── OrderItem
  └── ShippingAddress (Value Object)
```

**Domain Events** — значущі події в домені:
```typescript
OrderPlaced { orderId, customerId, totalAmount, timestamp }
PaymentReceived { orderId, amount, paymentMethod }
```

**Repository** — абстракція доступу до даних:
```typescript
interface OrderRepository {
  findById(id: string): Promise<Order>;
  save(order: Order): Promise<void>;
}
```

---

## SOLID принципи — пояснити кожен з прикладами

### S — Single Responsibility Principle
Клас має одну причину для зміни.

```typescript
// ❌ Порушення SRP
class User {
  save() { /* запис у БД */ }
  sendEmail() { /* відправка email */ }
  generateReport() { /* генерація звіту */ }
}

// ✅ Дотримання SRP
class UserRepository { save(user: User) {} }
class EmailService { sendEmail(to: string) {} }
class ReportGenerator { generate(data: any) {} }
```

### O — Open/Closed Principle
Відкритий для розширення, закритий для модифікації.

```typescript
// ❌ Порушення OCP — при новому типі потрібно змінювати функцію
function calculateDiscount(type: string) {
  if (type === "student") return 0.2;
  if (type === "senior") return 0.3;
  // потрібно додавати нові if...
}

// ✅ Дотримання OCP
interface DiscountStrategy { calculate(): number; }
class StudentDiscount implements DiscountStrategy { calculate() { return 0.2; } }
class SeniorDiscount implements DiscountStrategy { calculate() { return 0.3; } }
// Новий тип — новий клас, без зміни існуючого коду
```

### L — Liskov Substitution Principle
Підтип повинен бути замінюваним на базовий тип.

```typescript
// ❌ Порушення LSP
class Rectangle {
  setWidth(w: number) { this.width = w; }
  setHeight(h: number) { this.height = h; }
}
class Square extends Rectangle {
  setWidth(w: number) { this.width = w; this.height = w; } // неочікувана поведінка
}

// ✅ Рішення — окремі типи або спільний інтерфейс
interface Shape { area(): number; }
class Rectangle implements Shape { area() { return this.width * this.height; } }
class Square implements Shape { area() { return this.side * this.side; } }
```

### I — Interface Segregation Principle
Багато специфічних інтерфейсів краще одного великого.

```typescript
// ❌ Порушення ISP
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}
// Robot не їсть і не спить

// ✅ Дотримання ISP
interface Workable { work(): void; }
interface Eatable { eat(): void; }
interface Sleepable { sleep(): void; }

class Human implements Workable, Eatable, Sleepable { /*...*/ }
class Robot implements Workable { /*...*/ }
```

### D — Dependency Inversion Principle
Залежність від абстракцій, не від реалізацій.

```typescript
// ❌ Порушення DIP
class UserService {
  private db = new PostgresDatabase(); // пряма залежність
}

// ✅ Дотримання DIP
interface Database { query(sql: string): Promise<any>; }
class UserService {
  constructor(private db: Database) {} // залежність від абстракції
}
```

---

## Clean Architecture vs Hexagonal Architecture

### Clean Architecture (Robert C. Martin)
```
┌──────────────────────────────────┐
│        Frameworks & Drivers      │  ← зовнішній шар (DB, Web, UI)
│  ┌───────────────────────────┐   │
│  │   Interface Adapters      │   │  ← контролери, презентери
│  │  ┌────────────────────┐   │   │
│  │  │  Application Logic │   │   │  ← use cases
│  │  │  ┌─────────────┐   │   │   │
│  │  │  │  Entities    │   │   │   │  ← бізнес-логіка
│  │  │  └─────────────┘   │   │   │
│  │  └────────────────────┘   │   │
│  └───────────────────────────┘   │
└──────────────────────────────────┘
```
**Правило залежностей:** залежності спрямовані ТІЛЬКИ всередину. Зовнішні шари залежать від внутрішніх.

### Hexagonal Architecture (Ports & Adapters)
```
                  ┌─────────────┐
  HTTP Adapter ──→│             │←── Database Adapter
  CLI Adapter  ──→│   Domain    │←── Email Adapter
  gRPC Adapter ──→│   (Core)    │←── Queue Adapter
                  │             │
                  └─────────────┘
                   Ports (interfaces)
```
**Порти** — інтерфейси (what), **Адаптери** — реалізації (how).

### Спільне
Обидва підходи:
- Ізолюють бізнес-логіку від інфраструктури
- Використовують Dependency Inversion
- Легко тестувати (mock зовнішні залежності)
- Дозволяють замінювати інфраструктуру без зміни бізнес-логіки

---

## Event-Driven Architecture

### Типи

**Event Notification** — "щось сталось", мінімум даних:
```
{ event: "OrderPlaced", orderId: "123" }
// Споживач сам запитує деталі
```

**Event-Carried State Transfer** — подія містить всі дані:
```
{ event: "OrderPlaced", order: { id: "123", items: [...], total: 99.99 } }
// Споживач НЕ потребує додаткових запитів
```

**Event Sourcing** — стан = послідовність подій:
```
Events: [Created, ItemAdded, ItemAdded, ItemRemoved, Placed]
State = replay(events)
```

### Переваги
- Loose coupling між компонентами
- Масштабованість (async processing)
- Auditability (history подій)
- Resilience (retry, dead letter queue)

### Недоліки
- Eventual consistency
- Складніший debugging
- Порядок подій не гарантований (без додаткових зусиль)
- Складніший моніторинг

---

## 12-Factor App — що це?

Методологія для побудови cloud-native додатків:

1. **Codebase** — один репозиторій, багато deployments
2. **Dependencies** — явна декларація (package.json, requirements.txt)
3. **Config** — конфігурація через environment variables
4. **Backing Services** — БД, queue, cache як зовнішні ресурси (підключаються через URL)
5. **Build, Release, Run** — чітке розділення етапів
6. **Processes** — stateless процеси (стан у зовнішніх сервісах)
7. **Port Binding** — додаток експортує HTTP через порт
8. **Concurrency** — масштабування через процеси
9. **Disposability** — швидкий старт і graceful shutdown
10. **Dev/Prod Parity** — мінімальна різниця між середовищами
11. **Logs** — як потік подій (stdout)
12. **Admin Processes** — одноразові задачі як окремі процеси

---

## Serverless архітектура — переваги і недоліки

### Переваги
- **Немає серверів** для управління
- **Pay per use** — платиш тільки за виконання
- **Auto-scaling** — автоматичне масштабування
- **Швидкий time-to-market** — фокус на бізнес-логіці

### Недоліки
- **Cold start** — перший виклик повільніший
- **Vendor lock-in** — прив'язка до AWS/GCP/Azure
- **Обмеження** — час виконання, розмір payload, пам'ять
- **Debugging** — складніше дебажити та моніторити
- **Stateless** — немає локального стану
- **Ціна при високому навантаженні** — може бути дорожче за dedicated servers

### Коли використовувати
✅ Event-driven обробка (webhooks, file processing)
✅ API з непередбачуваним навантаженням
✅ Scheduled tasks (cron)
✅ MVP і прототипи

### Коли НЕ використовувати
❌ Long-running processes
❌ Real-time (WebSocket)
❌ Високе та стабільне навантаження
❌ Потрібен повний контроль над інфраструктурою

---

## Як правильно розбити моноліт на мікросервіси?

### Крок 1: Визначити Bounded Contexts (DDD)
Знайди природні межі в бізнес-домені:
- Які частини можуть розвиватися незалежно?
- Які команди відповідають за які модулі?
- Де найменше зв'язків між модулями?

### Крок 2: Strangler Fig Pattern
Поступово замінюй частини моноліту:
1. Нові фічі — одразу як окремі сервіси
2. Виділяй найбільш незалежні модулі першими
3. Facade (API Gateway) ховає внутрішню структуру

### Крок 3: Data Separation
Найскладніший етап — розділення даних:
1. Спочатку — окремий сервіс, shared DB
2. Потім — read-only access до чужих таблиць через views/API
3. Фінально — повністю окрема БД

### Anti-patterns
- **Big Bang Rewrite** — переписати все одразу (майже завжди провал)
- **Distributed Monolith** — мікросервіси, які не можуть деплоїтися незалежно
- **Nano-services** — занадто маленькі сервіси (overhead > benefit)

---

## Як забезпечити consistency між мікросервісами?

### Проблема
Немає distributed transactions як в моноліті. Як забезпечити, що дані консистентні?

### Підходи

**1. Saga Pattern**
Послідовність локальних транзакцій з компенсаційними діями:
```
Create Order → Reserve Inventory → Process Payment → Ship
     ↕                ↕                  ↕
Cancel Order ← Restore Inventory ← Refund Payment (компенсація)
```

**2. Eventual Consistency**
Дані стають консистентними з часом через events:
```
Order Service → OrderPlaced event → Inventory Service оновлює stock
```

**3. Outbox Pattern**
Атомарний запис у БД + events:
```
BEGIN TRANSACTION
  INSERT INTO orders (...)
  INSERT INTO outbox (event_type, payload)
COMMIT
-- окремий процес читає outbox і публікує events
```

**4. Change Data Capture (CDC)**
Слідкуємо за змінами у БД і публікуємо events:
```
PostgreSQL WAL → Debezium → Kafka → Consumers
```

---

## Service-Oriented Architecture (SOA) vs Microservices

| Аспект | SOA | Microservices |
|---|---|---|
| Розмір сервісів | Великі, enterprise-level | Маленькі, focused |
| Комунікація | ESB (Enterprise Service Bus) | Lightweight (HTTP, gRPC, messaging) |
| Дані | Shared database | Database per service |
| Governance | Централізоване | Децентралізоване |
| Технології | Стандартизовані (SOAP, WSDL) | Різні для різних сервісів |
| Deployment | Часто разом | Незалежний |

**SOA** — архітектурний стиль enterprise рівня, фокус на reusability.
**Microservices** — еволюція SOA з фокусом на незалежності та simplicity.
