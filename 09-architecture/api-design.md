# API Design — Питання для інтерв'ю

## REST vs GraphQL vs gRPC — порівняння

### REST
```
GET    /api/v1/users          — список
GET    /api/v1/users/123      — один
POST   /api/v1/users          — створити
PUT    /api/v1/users/123      — замінити
PATCH  /api/v1/users/123      — оновити частково
DELETE /api/v1/users/123      — видалити
```

**Переваги:** простий, кешується (HTTP cache), зрозумілий
**Недоліки:** over-fetching, under-fetching, N+1 запитів для зв'язаних ресурсів

### GraphQL
```graphql
query {
  user(id: "123") {
    name
    posts {
      title
      comments { text }
    }
  }
}
```

**Переваги:** клієнт запитує тільки потрібні дані, один endpoint, самодокументований
**Недоліки:** складніший caching, N+1 на бекенді (без DataLoader), security (depth attacks)

### gRPC
```protobuf
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}
```

**Переваги:** найшвидший (Protocol Buffers, binary), streaming, code generation, HTTP/2
**Недоліки:** не працює з браузерами напряму (потрібен grpc-web), складніший debugging

### Коли що обирати

| Сценарій | Рекомендація |
|---|---|
| Public API | REST (найзрозуміліший) |
| Mobile додатки | GraphQL (мінімізація трафіку) |
| Мікросервіси (internal) | gRPC (performance) |
| Real-time | gRPC streaming або WebSocket |
| CRUD | REST |
| Складні зв'язки даних | GraphQL |

---

## Як проектувати RESTful API?

### Основні принципи

**1. Ресурси як іменники (не дієслова):**
```
✅ GET /users           ❌ GET /getUsers
✅ POST /orders         ❌ POST /createOrder
✅ DELETE /users/123    ❌ POST /deleteUser
```

**2. Правильні HTTP методи:**
- GET — читання (idempotent, cacheable)
- POST — створення
- PUT — повна заміна
- PATCH — часткове оновлення
- DELETE — видалення

**3. Вкладені ресурси:**
```
GET /users/123/orders          — замовлення користувача 123
GET /users/123/orders/456      — конкретне замовлення
POST /users/123/orders         — створити замовлення для користувача
```
Але не більше 2 рівнів вкладеності.

**4. Фільтрація, сортування, пагінація:**
```
GET /users?role=admin&status=active          — фільтрація
GET /users?sort=-createdAt,name              — сортування
GET /users?page=2&limit=20                   — пагінація
GET /users?fields=id,name,email              — часткова відповідь
```

**5. HTTP статус коди:**
```
200 OK                  — успішне читання
201 Created             — ресурс створено
204 No Content          — успішне видалення
400 Bad Request         — невалідний input
401 Unauthorized        — не авторизований
403 Forbidden           — немає доступу
404 Not Found           — ресурс не знайдено
409 Conflict            — конфлікт (дублікат)
422 Unprocessable Entity — валідація
429 Too Many Requests   — rate limit
500 Internal Server Error — помилка сервера
```

---

## Версіонування API — стратегії

### URL versioning (найпоширеніший)
```
GET /api/v1/users
GET /api/v2/users
```
**Плюси:** зрозумілий, простий, легко кешувати
**Мінуси:** не RESTful (URL = resource, не версія)

### Header versioning
```
GET /api/users
Accept: application/vnd.myapp.v2+json
```
**Плюси:** чистий URL
**Мінуси:** складніше тестувати (потрібні headers)

### Query parameter
```
GET /api/users?version=2
```
**Плюси:** простий
**Мінуси:** не стандартний, може конфліктувати з іншими параметрами

**Рекомендація:** URL versioning — найпростіший і найпоширеніший підхід.

### Стратегія підтримки версій
- Підтримуй максимум 2 версії одночасно (current + previous)
- Deprecation notice за 6 місяців до видалення
- Sunset header: `Sunset: Sat, 01 Jan 2025 00:00:00 GMT`

---

## Pagination — Cursor vs Offset

### Offset-based (класичний)
```
GET /users?page=3&limit=20
// SQL: SELECT * FROM users LIMIT 20 OFFSET 40
```

**Плюси:** простий, можна перейти на будь-яку сторінку
**Мінуси:**
- Повільний для великих offset (OFFSET 100000)
- Inconsistency при вставці/видаленні (пропуск/дублювання елементів)

### Cursor-based (рекомендований)
```
GET /users?limit=20&cursor=eyJpZCI6MTIzfQ==
// SQL: SELECT * FROM users WHERE id > 123 LIMIT 20
```

**Плюси:**
- Стабільний при вставці/видаленні
- Швидкий (використовує INDEX)
- Масштабується для великих datasets

**Мінуси:**
- Не можна перейти на довільну сторінку
- Складніша реалізація

**Відповідь:**
```json
{
  "data": [...],
  "meta": {
    "hasNextPage": true,
    "nextCursor": "eyJpZCI6MTQzfQ=="
  }
}
```

**Рекомендація:** cursor-based для API, offset-based для адмін-панелей де потрібна навігація по сторінках.

---

## Rate Limiting — алгоритми та реалізація

### Token Bucket
Токени додаються з фіксованою швидкістю. Кожен запит забирає один токен. Дозволяє burst.
```
Bucket capacity: 10
Refill rate: 1 token/second
```

### Sliding Window
Рахує запити в ковзному вікні (наприклад, останні 60 секунд).

### Fixed Window Counter
Рахує запити у фіксованих інтервалах (кожна хвилина — нове вікно).

### HTTP заголовки
```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000060
Retry-After: 30
```

### Рівні rate limiting
1. **Per IP** — захист від одного джерела
2. **Per User** — ліміт для авторизованого користувача
3. **Per Endpoint** — різні ліміти для різних операцій
4. **Global** — загальний ліміт для всього API

---

## API Authentication — OAuth2, JWT, API Keys

### API Keys
```
GET /api/users
X-API-Key: sk_live_abc123
```
**Коли:** server-to-server комунікація, public APIs, third-party інтеграції
**Плюси:** простий
**Мінуси:** якщо скомпрометований — складно відкликати всі

### JWT (JSON Web Token)
```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiMTIzIn0.signature
```
**Структура:** Header.Payload.Signature (base64)
**Коли:** authentication/authorization, stateless sessions
**Плюси:** stateless, містить claims, можна перевірити без БД
**Мінуси:** не можна відкликати до expiration, розмір

**Best practices:**
- Короткий TTL для access token (15 хв)
- Довший TTL для refresh token (7 днів)
- Зберігати refresh token у HttpOnly cookie
- Не зберігати sensitive дані в payload (вони base64, не encrypted)

### OAuth 2.0
Стандарт для делегованої авторизації:
```
User → App → Authorization Server → Token → Resource Server
```

**Flows:**
- **Authorization Code** — для server-side apps (найбезпечніший)
- **Authorization Code + PKCE** — для SPA та mobile
- **Client Credentials** — для machine-to-machine
- **Device Code** — для IoT, CLI (deprecated: Implicit flow)

---

## Error Handling в API

### Стандартний формат помилки
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Помилка валідації",
    "details": [
      {
        "field": "email",
        "message": "Невалідний формат email"
      },
      {
        "field": "password",
        "message": "Мінімум 8 символів"
      }
    ]
  }
}
```

### Принципи
1. **Consistency** — завжди один формат помилки
2. **Корисні повідомлення** — що пішло не так і як виправити
3. **Правильні статус коди** — 400 для клієнтських, 500 для серверних
4. **Без internal деталей** — не показуй stack traces, SQL queries
5. **Request ID** — для трасування: `X-Request-Id: abc-123`
6. **Документація** — опиши всі можливі помилки для кожного endpoint

---

## Backward Compatibility

### Правила
1. **Не видаляй** поля з відповіді (deprecate замість видалення)
2. **Не змінюй** типи полів (string → number = breaking change)
3. **Нові поля** — optional (не required)
4. **Нові endpoints** — додавай, не змінюй існуючі
5. **Deprecation policy** — повідомляй заздалегідь

### Стратегії
- **Additive changes** — тільки додаємо, не видаляємо
- **Feature flags** — нова поведінка за прапорцем
- **Versioning** — нова версія для breaking changes
- **Consumer-Driven Contracts** — тести перевіряють сумісність

### Приклад deprecation
```json
{
  "name": "John Doe",
  "fullName": "John Doe",
  "_deprecated": {
    "name": "Use 'fullName' instead. Will be removed in v3."
  }
}
```

---

## OpenAPI / Swagger — Best Practices

### Навіщо
- **Документація** — автоматична, завжди актуальна
- **Code generation** — клієнти та серверні стаби
- **Тестування** — валідація запитів/відповідей
- **Контракт** — agreement між frontend і backend

### Best practices
1. **Schema-first** або **Code-first** з автогенерацією
2. Описуй **всі** можливі відповіді (200, 400, 401, 404, 500)
3. Використовуй **$ref** для reusable schemas
4. Додавай **приклади** для кожного endpoint
5. Групуй endpoints по **тегах**
6. Вказуй **authentication** requirements

---

## HATEOAS — що це і коли потрібно?

**HATEOAS** (Hypermedia As The Engine Of Application State) — API повертає посилання на доступні дії:

```json
{
  "id": "123",
  "name": "John Doe",
  "status": "active",
  "_links": {
    "self": { "href": "/api/users/123" },
    "orders": { "href": "/api/users/123/orders" },
    "deactivate": { "href": "/api/users/123/deactivate", "method": "POST" }
  }
}
```

**Переваги:** клієнт не хардкодить URL-и, API self-discoverable
**Недоліки:** складніше реалізувати, рідко використовується на практиці

**На інтерв'ю:** "HATEOAS — це Level 3 у Richardson Maturity Model. На практиці більшість API зупиняються на Level 2 (правильні HTTP методи + ресурси), і це нормально для 95% випадків."
