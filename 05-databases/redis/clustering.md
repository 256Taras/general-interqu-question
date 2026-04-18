# Redis Clustering — Питання для інтерв'ю

## Способи масштабування Redis

```
1. Standalone       — один сервер
2. Sentinel         — HA (high availability) без шардінгу
3. Cluster          — HA + шардінг (горизонтальне масштабування)
```

---

## Redis Sentinel

```
         ┌── Sentinel 1 ──┐
         │   Sentinel 2   │  (моніторинг + failover)
         └── Sentinel 3 ──┘
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    Master → Replica 1  Replica 2
   (write)   (read)     (read)
```

**Sentinel відповідає за:**
- **Моніторинг** — перевіряє чи Master/Replica доступні
- **Сповіщення** — повідомляє адміни при проблемах
- **Automatic Failover** — промоутить Replica → Master
- **Service Discovery** — клієнти запитують у Sentinel адресу Master

```redis
-- Конфігурація sentinel.conf
sentinel monitor mymaster 127.0.0.1 6379 2
-- "2" = quorum — скільки Sentinel повинні погодитись що Master недоступний

sentinel down-after-milliseconds mymaster 5000
-- Через 5 сек без відповіді вважати недоступним

sentinel failover-timeout mymaster 60000
```

**Failover процес:**
1. Sentinel помічає що Master не відповідає (`SDOWN` — subjective down)
2. Інші Sentinel-и підтверджують (`ODOWN` — objective down, quorum)
3. Один Sentinel обирається лідером
4. Лідер обирає найкращу Replica (найбільше даних)
5. Replica промоутиться в Master
6. Інші Replica переключаються на новий Master

---

## Redis Cluster

```
┌──────────────────────────────────────────────┐
│                16384 Hash Slots              │
│  [0─5460]      [5461─10922]    [10923─16383] │
│     │               │               │        │
│  Master A        Master B        Master C    │
│  Replica A1      Replica B1      Replica C1  │
└──────────────────────────────────────────────┘
```

**Ключові концепції:**
- **16384 hash slots** — дані розподіляються по слотах
- **Hash function**: `CRC16(key) % 16384` → номер слота
- **Мінімум 3 master** ноди для кластера
- Кожен master має 1+ replica для HA

---

## Hash Slots та Routing

```redis
-- Клієнт відправляє команду
SET user:123 "John"

-- Redis Cluster:
-- 1. CRC16("user:123") % 16384 = slot 5649
-- 2. Slot 5649 належить Master B
-- 3. Якщо запит прийшов на Master A:
--    → MOVED 5649 192.168.1.2:6379 (перенаправлення)
```

### Hash Tags — примусово в один слот

```redis
-- Без hash tags
SET user:1:profile "..."    -- slot X
SET user:1:settings "..."   -- slot Y (інший!)
-- Multi-key операції не працюють!

-- З hash tags {user:1}
SET {user:1}:profile "..."  -- slot = CRC16("user:1")
SET {user:1}:settings "..." -- той самий slot!
-- Тепер можна використовувати MGET, транзакції
```

---

## Cluster Failover

```
Master A падає
    ↓
Replica A1 помічає (heartbeat timeout)
    ↓
Replica A1 запитує голоси у інших Masters
    ↓
Majority Masters голосують ЗА
    ↓
Replica A1 стає новим Master A
    ↓
Hash slots [0-5460] тепер обслуговує новий Master A
```

**Час failover:** зазвичай 1-2 секунди.

---

## Cluster vs Sentinel

| Характеристика | Sentinel | Cluster |
|---------------|----------|---------|
| Шардінг | Ні | Так (hash slots) |
| Max пам'ять | RAM одного сервера | Сума RAM всіх master |
| Failover | Через Sentinel | Вбудований |
| Multi-key | Будь-які ключі | Тільки в одному слоті |
| Складність | Низька | Висока |
| Коли | < 25GB, HA потрібен | > 25GB, горизонтальне масштабування |

---

## Обмеження Redis Cluster

```
1. Multi-key операції — тільки якщо ключі в одному hash slot
   MGET key1 key2        -- ❌ якщо різні слоти
   MGET {tag}:k1 {tag}:k2 -- ✅ один слот

2. Транзакції (MULTI/EXEC) — тільки один слот

3. Lua scripts — всі ключі повинні бути в одному слоті

4. Pub/Sub — повідомлення пересилаються на ВСІ ноди
   (Redis 7.0: Sharded Pub/Sub вирішує це)

5. Database — тільки db 0 (SELECT не підтримується)
```

---

## Resharding (додавання ноди)

```
Додаємо Master D:

До:   A[0-5460]  B[5461-10922]  C[10923-16383]
Після: A[0-4095]  B[4096-8191]  C[8192-12287]  D[12288-16383]

-- Процес:
redis-cli --cluster reshard 127.0.0.1:6379
-- Redis мігрує слоти з існуючих master на новий
-- Клієнти автоматично перенаправляються (ASK → MOVED)
```

**ASK vs MOVED:**
- `MOVED` — слот постійно переїхав (оновити конфігурацію)
- `ASK` — слот тимчасово мігрує (один запит на нову ноду)

---

## Розгортання у продакшні

```
Рекомендована топологія:
- 6 нод мінімум (3 master + 3 replica)
- Replica на іншому фізичному сервері ніж її master
- Мінімум 3 зони доступності (AZ)

AZ-1:  Master A, Replica B1
AZ-2:  Master B, Replica C1
AZ-3:  Master C, Replica A1
```

**Managed сервіси:**
- AWS ElastiCache / MemoryDB
- GCP Memorystore
- Azure Cache for Redis

Managed сервіси обробляють failover, patching, backups автоматично.

---

## Connection Pooling

```javascript
import { createCluster } from "redis";

const cluster = createCluster({
  rootNodes: [
    { url: "redis://node1:6379" },
    { url: "redis://node2:6379" },
    { url: "redis://node3:6379" },
  ],
  defaults: {
    socket: {
      connectTimeout: 5000,
      reconnectStrategy: (retries) => Math.min(retries * 100, 5000),
    },
  },
});

await cluster.connect();
await cluster.set("key", "value");
await cluster.get("key");
```
