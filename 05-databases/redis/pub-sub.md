# Redis Pub/Sub — Питання для інтерв'ю

## Базова концепція

```
Publisher ──PUBLISH──→ Channel ──→ Subscriber 1
                               ──→ Subscriber 2
                               ──→ Subscriber 3
```

**Pub/Sub** — механізм "fire-and-forget" повідомлень. Публікатор не знає хто слухає, підписники не знають хто відправляє.

---

## Основні команди

```redis
-- Підписка на канал
SUBSCRIBE notifications
SUBSCRIBE chat:room:1 chat:room:2

-- Підписка за патерном
PSUBSCRIBE chat:*        -- всі канали що починаються з "chat:"
PSUBSCRIBE news.*        -- news.sport, news.tech, etc.

-- Публікація
PUBLISH notifications "New order #123"
PUBLISH chat:room:1 "Hello everyone!"

-- Відписка
UNSUBSCRIBE notifications
PUNSUBSCRIBE chat:*
```

---

## Приклад у Node.js

```javascript
import { createClient } from "redis";

// Publisher (окремий клієнт)
const publisher = createClient();
await publisher.connect();

// Subscriber (ОКРЕМИЙ клієнт! — обов'язково)
const subscriber = createClient();
await subscriber.connect();

// Підписка
await subscriber.subscribe("orders", (message, channel) => {
  console.log(`[${channel}] ${message}`);
  // [orders] {"orderId":"123","status":"created"}
});

// Публікація
await publisher.publish("orders", JSON.stringify({
  orderId: "123",
  status: "created",
}));
```

**Важливо:** підписник не може виконувати інші команди! Потрібен окремий клієнт.

---

## Обмеження Pub/Sub

| Обмеження | Деталі |
|-----------|--------|
| Без гарантії доставки | Якщо підписника немає — повідомлення втрачається |
| Без persistence | Повідомлення не зберігаються на диску |
| Без replay | Не можна отримати історію повідомлень |
| Без ACK | Немає підтвердження обробки |
| At-most-once | Повідомлення доставляється максимум раз |
| Блокує з'єднання | Підписник не може виконувати інші команди |

---

## Pub/Sub vs Streams

```
                    Pub/Sub              Streams
Persistence:        Ні                   Так
Replay:             Ні                   Так (XRANGE)
Consumer Groups:    Ні                   Так (XGROUP)
ACK:                Ні                   Так (XACK)
Гарантія:           At-most-once         At-least-once
Історія:            Ні                   Так
Швидкість:          Дуже швидко          Швидко
Пам'ять:            Мінімально           Зберігає дані
```

### Коли Pub/Sub?
- Real-time нотифікації (WebSocket broadcast)
- Chat повідомлення (не критично якщо втрачено)
- Cache invalidation (повідомити всі інстанси)
- Live dashboards

### Коли Streams?
- Обробка замовлень (не можна втратити)
- Event sourcing
- Task queue з гарантією
- Логи та аудит

---

## Патерни використання

### 1. Cache Invalidation

```javascript
// Сервіс A: змінив дані
await db.query("UPDATE users SET name = $1 WHERE id = $2", [name, id]);
await publisher.publish("cache:invalidate", JSON.stringify({
  type: "user",
  id: userId,
}));

// Сервіс B, C, D: очистили кеш
await subscriber.subscribe("cache:invalidate", (message) => {
  const { type, id } = JSON.parse(message);
  cache.delete(`${type}:${id}`);
});
```

### 2. WebSocket Broadcasting

```javascript
// API сервер 1 (отримав подію)
await publisher.publish("ws:broadcast", JSON.stringify({
  room: "room:123",
  event: "message",
  data: { text: "Hello!", userId: "user:1" },
}));

// WebSocket сервер (кілька інстансів)
await subscriber.subscribe("ws:broadcast", (message) => {
  const { room, event, data } = JSON.parse(message);
  io.to(room).emit(event, data);
});
```

### 3. Microservice Events

```javascript
// Order Service
await publisher.publish("events:order", JSON.stringify({
  event: "order.created",
  orderId: "123",
  userId: "user:1",
}));

// Notification Service
await subscriber.pSubscribe("events:*", (message, channel) => {
  const event = JSON.parse(message);
  if (event.event === "order.created") {
    sendNotification(event.userId, "Your order was created!");
  }
});
```

---

## Масштабування Pub/Sub

У Redis Cluster:
- Повідомлення пересилаються на **всі ноди** кластера
- Це може бути bottleneck при великому трафіку
- **Sharded Pub/Sub** (Redis 7.0+) — повідомлення тільки на потрібні ноди

```redis
-- Sharded Pub/Sub (Redis 7.0+)
SSUBSCRIBE channel
SPUBLISH channel "message"
-- Повідомлення маршрутизується через hash slot
```
