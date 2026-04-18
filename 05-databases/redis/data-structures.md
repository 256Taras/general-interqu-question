
# Redis Data Structures — Питання для інтерв'ю

## Основні структури даних

### String
```redis
SET user:1:name "John"          -- зберегти
GET user:1:name                 -- отримати → "John"
INCR counter                    -- атомарний інкремент
INCRBY counter 10               -- +10
SETNX lock:order "locked"       -- SET if Not eXists (для locks)
SETEX session:abc 3600 "data"   -- SET з TTL (3600 сек)
MSET k1 "v1" k2 "v2"           -- множинний SET
MGET k1 k2                     -- множинний GET
```

**Використання:** кеш, лічильники, сесії, розподілені блокування.

---

### List (двозв'язний список)
```redis
LPUSH queue:emails "email1"     -- додати зліва
RPUSH queue:emails "email2"     -- додати справа
LPOP queue:emails               -- витягнути зліва
RPOP queue:emails               -- витягнути справа
BRPOP queue:emails 30           -- blocking POP (чекає 30 сек)
LRANGE queue:emails 0 -1        -- всі елементи
LLEN queue:emails               -- довжина
```

**Використання:** черги (FIFO/LIFO), recent activity feed, обмежені списки.

```redis
-- Тримати тільки останні 100 елементів
LPUSH recent:actions "action"
LTRIM recent:actions 0 99
```

---

### Hash (об'єкт/map)
```redis
HSET user:1 name "John" age "30" email "john@example.com"
HGET user:1 name                -- → "John"
HGETALL user:1                  -- всі поля
HINCRBY user:1 age 1            -- атомарний інкремент поля
HDEL user:1 email               -- видалити поле
HEXISTS user:1 email            -- чи існує поле
```

**Використання:** об'єкти/профілі, сесії, конфігурації.

**Перевага:** оновлення одного поля без перезапису всього об'єкта.

---

### Set (унікальні значення, без порядку)
```redis
SADD tags:post:1 "nodejs" "redis" "backend"
SMEMBERS tags:post:1            -- всі елементи
SISMEMBER tags:post:1 "redis"   -- чи є елемент → 1/0
SCARD tags:post:1               -- кількість
SREM tags:post:1 "backend"      -- видалити

-- Операції між множинами
SINTER tags:post:1 tags:post:2  -- перетин
SUNION tags:post:1 tags:post:2  -- об'єднання
SDIFF tags:post:1 tags:post:2   -- різниця
```

**Використання:** теги, унікальні відвідувачі, фільтрація дублікатів, взаємні друзі.

---

### Sorted Set (ZSet — унікальні значення з score)
```redis
ZADD leaderboard 100 "player:1"
ZADD leaderboard 250 "player:2"
ZADD leaderboard 150 "player:3"

ZRANGE leaderboard 0 -1 WITHSCORES    -- від найменшого score
ZREVRANGE leaderboard 0 2 WITHSCORES  -- топ-3 (від найбільшого)
ZRANK leaderboard "player:2"          -- позиція (від 0)
ZSCORE leaderboard "player:1"         -- score гравця
ZINCRBY leaderboard 50 "player:1"     -- збільшити score
ZRANGEBYSCORE leaderboard 100 200     -- елементи зі score 100-200
ZCARD leaderboard                     -- кількість
```

**Використання:** рейтинги, лідерборди, пріоритетні черги, часові ряди.

---

## Додаткові структури

### HyperLogLog (приблизний підрахунок унікальних)
```redis
PFADD visitors:2024-01 "user:1" "user:2" "user:3"
PFADD visitors:2024-01 "user:1"       -- дублікат, не рахується
PFCOUNT visitors:2024-01              -- ≈ 3 (похибка ~0.81%)
PFMERGE visitors:q1 visitors:2024-01 visitors:2024-02 visitors:2024-03
```

**12KB пам'яті** незалежно від кількості елементів (мільйони!).

**Використання:** підрахунок унікальних відвідувачів, унікальних IP, унікальних запитів.

---

### Bitmap (бітові операції)
```redis
SETBIT login:2024-01-15 1001 1        -- user 1001 логінився
GETBIT login:2024-01-15 1001          -- → 1
BITCOUNT login:2024-01-15             -- скільки логінилось

-- Хто логінився І 15, І 16 січня?
BITOP AND active:both login:2024-01-15 login:2024-01-16
BITCOUNT active:both
```

**Використання:** щоденна активність, feature flags, онлайн-статуси.

---

### Stream (лог подій, як Kafka)
```redis
XADD events * action "purchase" userId "user:1" amount "99"
XADD events * action "signup" userId "user:2"

XLEN events                           -- кількість записів
XRANGE events - +                     -- всі записи
XRANGE events - + COUNT 10            -- перші 10

-- Consumer Group (як Kafka consumer group)
XGROUP CREATE events mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 COUNT 1 BLOCK 5000 STREAMS events >
XACK events mygroup "1234567890-0"    -- підтвердити обробку
```

**Використання:** event sourcing, логи, черги з гарантією доставки.

---

## Порівняльна таблиця

| Структура | Складність | Пам'ять | Типове використання |
|-----------|-----------|---------|---------------------|
| String | O(1) get/set | Залежить від значення | Кеш, лічильники |
| List | O(1) push/pop, O(n) index | ~overhead на елемент | Черги, стрічки |
| Hash | O(1) get/set поля | Ефективний для об'єктів | Профілі, сесії |
| Set | O(1) add/remove/check | ~64 bytes на елемент | Теги, унікальні |
| Sorted Set | O(log n) add/remove | ~128 bytes на елемент | Рейтинги |
| HyperLogLog | O(1) | **12KB фіксовано** | Підрахунок унікальних |
| Bitmap | O(1) per bit | 1 bit на елемент | Активність, прапорці |
| Stream | O(1) append, O(n) read | Залежить від записів | Черги подій |

---

## Вибір структури

```
Потрібен простий кеш? → String
Потрібна черга? → List (простий) або Stream (надійний)
Потрібен об'єкт? → Hash
Потрібні унікальні значення? → Set
Потрібен рейтинг? → Sorted Set
Потрібен підрахунок унікальних? → HyperLogLog
Потрібні бітові прапорці? → Bitmap
```
