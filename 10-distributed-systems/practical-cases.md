# Практичні кейси розподілених систем — Питання для інтерв'ю

## Система нотифікацій для 10M користувачів

```
"Ми будуємо систему нотифікацій для 10M користувачів.
Кожен користувач може отримати до 1000 нотифікацій на день.
Як зберігати та доставляти нотифікації?"
```

### Оцінка масштабу

```
10M користувачів × 1000 нотифікацій/день = 10B нотифікацій/день
10B / 86400 секунд ≈ 115K записів/секунду (write-heavy!)
Розмір нотифікації: ~500 bytes → 5TB/день (потрібен TTL)
```

### Архітектура

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Event Source  │────→│ Kafka Topics │────→│ Notification │
│ (API, Cron)  │     │ (by type)    │     │ Workers      │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                          ┌───────────────────────┼────────────────┐
                          ▼                       ▼                ▼
                    ┌──────────┐          ┌──────────┐     ┌──────────┐
                    │ Push     │          │ Email    │     │ SMS      │
                    │ Service  │          │ Service  │     │ Service  │
                    │ (FCM/APNs)│         │ (SES)   │     │ (Twilio) │
                    └──────────┘          └──────────┘     └──────────┘
```

### Data Model

```sql
-- Шардування по user_id (hash-based)
-- TTL на 30 днів (автоматичне видалення старих)

notifications (
    user_id       BIGINT,        -- partition key
    notification_id UUID,        -- clustering key
    type          VARCHAR,       -- push, email, sms, in_app
    title         VARCHAR,
    body          TEXT,
    data          JSONB,
    is_read       BOOLEAN,
    created_at    TIMESTAMP,
    expires_at    TIMESTAMP      -- TTL
)
```

### Write Path

```
1. Сервіс генерує нотифікацію
2. Запис в БД (Cassandra — write-optimized)
3. Публікація в Kafka topic (по типу: push, email, sms)
4. Worker читає з Kafka → доставляє через відповідний канал
5. Оновлення статусу доставки
```

### Read Path

```
GET /notifications?user_id=123&limit=50

1. Перевіряємо Redis cache (останні 50 нотифікацій)
2. Cache hit → повертаємо
3. Cache miss → читаємо з відповідного шарда → кешуємо → повертаємо
```

### Масштабування

```
Проблема: знаменитості/корпоративні акаунти отримують набагато більше нотифікацій

Рішення:
1. Sub-sharding для popular users (як у hotspot проблемі)
2. Bloom filters для перевірки "чи є непрочитані" без читання з БД
3. Окремі Kafka consumer groups для різних пріоритетів
4. Rate limiting: не більше N нотифікацій на годину для одного юзера
```

### Гарантія доставки

```
At-least-once delivery:
1. Worker читає з Kafka
2. Доставляє нотифікацію
3. Commit offset ТІЛЬКИ після успішної доставки
4. При failure — повторна спроба (exponential backoff)

Дедуплікація на стороні клієнта:
- notification_id як idempotency key
- Клієнт ігнорує вже показані нотифікації
```

---

## E-commerce: система кошиків на Black Friday

```
"У нас є онлайн-магазин з 1M товарів. На Black Friday
очікуємо 100k одночасних користувачів. Як побудувати систему кошиків?"
```

### Оцінка масштабу

```
100K одночасних користувачів
Середньо 10 дій з кошиком на сесію
→ ~1M операцій з кошиками на годину
→ ~280 операцій/секунду (peak: 1000+/s)
```

### Архітектура

```
Client → API Gateway → Cart Service → Redis (primary storage)
                                     → PostgreSQL (persistent backup)
                                     → Inventory Service (stock check)
```

### Чому Redis як primary

```
1. Latency < 1ms (vs 5-10ms PostgreSQL)
2. Кошик — тимчасові дані (TTL 24h)
3. Атомарні операції (HINCRBY, HSET)
4. Не критично якщо втратимо кошик при аварії Redis
```

### Основні операції

```python
class ShoppingCartService:
    async def add_to_cart(self, user_id, product_id, quantity):
        # 1. Перевіряємо наявність товару
        available = await self.inventory.check_available(product_id, quantity)
        if not available:
            raise InsufficientStockError()

        # 2. Атомарно оновлюємо кошик в Redis
        cart_key = f"cart:{user_id}"
        await self.redis.hincrby(cart_key, product_id, quantity)
        await self.redis.expire(cart_key, 86400)  # TTL 24 години

        # 3. Асинхронно синхронізуємо в БД (backup)
        await self.queue.publish("cart.updated", {
            "user_id": user_id,
            "product_id": product_id,
            "quantity": quantity
        })

        # 4. Soft-reserve товар (не блокуємо, але враховуємо)
        await self.inventory.soft_reserve(product_id, quantity, ttl=600)


    async def checkout(self, user_id):
        # 1. Distributed lock на кошик (запобігаємо race conditions)
        async with self.redis.lock(f"lock:cart:{user_id}", timeout=30):

            # 2. Отримуємо весь кошик
            cart = await self.redis.hgetall(f"cart:{user_id}")
            if not cart:
                raise EmptyCartError()

            # 3. Перевіряємо доступність ВСІХ товарів
            for product_id, quantity in cart.items():
                if not await self.inventory.check_available(product_id, int(quantity)):
                    raise InsufficientStockError(product_id)

            # 4. Створюємо замовлення (транзакційно в PostgreSQL)
            async with self.db.transaction():
                order = await self.db.create_order(user_id, cart)

                # 5. Резервуємо товари (hard reserve)
                for product_id, quantity in cart.items():
                    await self.inventory.hard_reserve(
                        product_id, int(quantity), order.id
                    )

            # 6. Очищаємо кошик
            await self.redis.delete(f"cart:{user_id}")

            # 7. Публікуємо подію для подальшої обробки
            await self.queue.publish("order.created", {"order_id": order.id})

            return order
```

### Inventory Management (запобігання overselling)

```
Два рівні резервування:

1. Soft Reserve (при додаванні в кошик):
   - Зменшуємо "available" count в Redis
   - TTL 10 хвилин (автоматичне звільнення)
   - Не блокує покупку іншими

2. Hard Reserve (при checkout):
   - Атомарне списання в PostgreSQL
   - Транзакція з перевіркою
   - Незворотне до скасування замовлення

Atomic inventory check (Redis + Lua):
```

```lua
-- Redis Lua script для атомарної перевірки та резервування
local stock = tonumber(redis.call('GET', KEYS[1]))
local requested = tonumber(ARGV[1])

if stock >= requested then
    redis.call('DECRBY', KEYS[1], requested)
    return 1  -- success
else
    return 0  -- insufficient stock
end
```

### Обробка edge cases

```
1. Користувач додав товар, але не оформив замовлення
   → Soft reserve з TTL → автоматичне звільнення через 10 хвилин

2. Два користувачі купують останній товар одночасно
   → Lua script в Redis для атомарної перевірки
   → Тільки перший отримує товар, другий — "Out of stock"

3. Redis впав під час Black Friday
   → Fallback на PostgreSQL (повільніше, але працює)
   → Кошики відновлюються з backup

4. Ціна змінилась поки товар у кошику
   → При checkout — перевіряємо актуальну ціну
   → Показуємо різницю користувачу
```

### Масштабування під Black Friday

```
1. Redis Cluster — шардування по user_id
2. Read replicas для перегляду кошика
3. Rate limiting — max 10 add-to-cart на хвилину
4. Queue-based checkout — при піковому навантаженні
   ставимо checkout в чергу замість синхронної обробки
5. Pre-warming cache — завантажуємо популярні товари заздалегідь
6. Circuit breaker — якщо Inventory Service перевантажений
```
