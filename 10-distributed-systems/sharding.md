# Шардування — Питання для інтерв'ю

## Дизайн шардування таблиці users

```
"У нас є таблиця users з 500 мільйонами записів.
Кожен користувач має:
- user_id (int, auto-increment)
- email (string)
- country_code (string)
- created_at (timestamp)

Запити:
1. Отримати користувача по ID (часто)
2. Знайти користувачів з країни X (рідко)
3. Отримати останніх зареєстрованих користувачів (часто)

Як би ви шардували цю таблицю?"
```

### Аналіз варіантів

**Варіант 1: Range-based по user_id**
```sql
shard1: user_id 1-100M
shard2: user_id 100M-200M
shard3: user_id 200M-300M
shard4: user_id 300M-400M
shard5: user_id 400M-500M
```

| | |
|---|---|
| Плюси | Простота, легко додавати нових користувачів, діапазонні запити по ID |
| Мінуси | **Hotspot** — останній шард перевантажений (нові реєстрації), нерівномірне навантаження |
| Запит 1 (по ID) | Знаємо шард одразу |
| Запит 2 (по країні) | Scatter-gather на всі шарди |
| Запит 3 (останні) | Тільки останній шард (hotspot!) |

**Варіант 2: Hash-based по user_id**
```sql
shard_id = hash(user_id) % num_shards

-- user_id=123 → hash(123) % 5 = 2 → shard2
-- user_id=456 → hash(456) % 5 = 0 → shard0
```

| | |
|---|---|
| Плюси | Рівномірний розподіл, немає hotspot-ів |
| Мінуси | Складно робити діапазонні запити (created_at), resharding при додаванні шардів |
| Запит 1 (по ID) | Знаємо шард (hash) |
| Запит 2 (по країні) | Scatter-gather на всі шарди |
| Запит 3 (останні) | Scatter-gather на всі шарди |

**Варіант 3: Composite key (country_code + user_id)**
```sql
-- Користувачі однієї країни на одному шарді
shard_ua: country_code = 'UA'
shard_us: country_code = 'US'
shard_de: country_code = 'DE'
...
```

| | |
|---|---|
| Плюси | Локалізація даних, швидкі запити по країні |
| Мінуси | **Дисбаланс** — US/India/China матимуть величезні шарди, маленькі країни = порожні шарди |
| Запит 1 (по ID) | Потрібно знати country_code |
| Запит 2 (по країні) | Один шард — дуже швидко |
| Запит 3 (останні) | Scatter-gather |

### Рекомендоване рішення

```
Основний шардинг: Hash-based по user_id (рівномірний розподіл)

Для запиту 2 (по країні): Глобальний вторинний індекс
  → Окрема таблиця: country_code → [user_ids]
  → Або ElasticSearch для повнотекстового пошуку

Для запиту 3 (останні): Окремий sorted set в Redis
  → ZADD recent_users <timestamp> <user_id>
  → ZREVRANGE recent_users 0 99
  → Або окрема таблиця recent_registrations з TTL
```

---

## Hotspot проблема

```
"Ми шардували таблицу замовлень по customer_id (hash-based).
Але один корпоративний клієнт робить 50% всіх замовлень.
Тепер один шард перевантажений. Що робити?"
```

### Діагностика

```
Стандартний hash-based sharding:
shard_id = hash(customer_id) % num_shards

Проблема: hash("big_corp_123") % 5 = 3
→ Всі 50% замовлень big_corp йдуть на shard3
→ shard3 перевантажений, інші недовантажені
```

### Рішення

**1. Sub-sharding (додатковий розподіл великих клієнтів)**
```python
BIG_CUSTOMERS = {"big_corp_123", "mega_corp_456"}

def get_shard_for_order(customer_id, order_id):
    if customer_id in BIG_CUSTOMERS:
        # Великий клієнт — розподіляємо по order_id
        return hash(order_id) % num_shards
    else:
        # Маленький клієнт — по customer_id
        return hash(customer_id) % num_shards
```
- Замовлення великого клієнта розподіляються рівномірно
- Замовлення маленьких клієнтів залишаються на одному шарді (зручно для запитів)

**2. Composite key**
```python
# Завжди використовуємо (customer_id, order_id) як ключ шардування
shard_id = hash(customer_id + ":" + order_id) % num_shards

# Мінус: запит "всі замовлення клієнта X" → scatter-gather
```

**3. Директорний підхід з ручним розподілом**
```python
# Routing table в Redis/etcd
shard_directory = {
    "big_corp_123": [0, 1, 2, 3, 4],  # розподілено по всіх шардах
    "small_client_1": [2],              # один шард
    "small_client_2": [0],              # один шард
}

def get_shard(customer_id, order_id):
    shards = shard_directory[customer_id]
    if len(shards) > 1:
        return shards[hash(order_id) % len(shards)]
    return shards[0]
```

**4. Динамічне перебалансування**
```
Моніторинг → Виявлення hotspot (шард > 80% CPU)
→ Автоматичне розділення перевантаженого шарду
→ Оновлення routing table
→ Фонова міграція даних
```

### Порівняння підходів

| Підхід | Складність | Ефективність | Вплив на запити |
|---|---|---|---|
| Sub-sharding | Середня | Висока | Мінімальний |
| Composite key | Низька | Середня | Scatter-gather для "всі замовлення клієнта" |
| Директорний | Висока | Найвища | Залежить від конфігурації |
| Динамічне | Дуже висока | Найвища | Прозорий |

---

## Consistent Hashing

```
"Поясніть Consistent Hashing. Як воно допомагає при додаванні/видаленні шардів?"
```

### Проблема звичайного hash-based sharding

```
Звичайний хеш: shard = hash(key) % N

N = 3 шарди:
  hash("user1") % 3 = 0 → shard0
  hash("user2") % 3 = 1 → shard1
  hash("user3") % 3 = 2 → shard2

Додаємо shard3 (N = 4):
  hash("user1") % 4 = 1 → shard1  ← ЗМІНИВСЯ!
  hash("user2") % 4 = 2 → shard2  ← ЗМІНИВСЯ!
  hash("user3") % 4 = 3 → shard3  ← ЗМІНИВСЯ!

→ ~75% ключів потрібно перемістити при додаванні одного шарда!
```

### Як працює Consistent Hashing

```
Hash ring (0 ... 2^32):

         shard1 (pos: 1000)
        /
  ─────●──────────────────●───── shard2 (pos: 5000)
  |                            |
  |          Hash Ring         |
  |                            |
  ─────●──────────────────●─────
        \                /
    shard3 (pos: 8000)  shard4 (pos: 3000)

Ключ "user123" → hash = 4500
→ Йдемо по кільцю за годинниковою стрілкою
→ Перший шард = shard2 (pos: 5000)
```

### Додавання/видалення шарда

```
Видаляємо shard2:
- Тільки ключі, які були на shard2, переміщуються на наступний шард
- Решта ключів НЕ переміщуються

→ Переміщується лише ~1/N ключів (замість ~(N-1)/N)
```

### Virtual Nodes (vnodes)

**Проблема:** з малою кількістю шардів розподіл нерівномірний.
**Рішення:** кожен фізичний шард має кілька віртуальних позицій на кільці.

```
shard1 → vnode1_0 (pos: 100), vnode1_1 (pos: 3500), vnode1_2 (pos: 7200)
shard2 → vnode2_0 (pos: 900), vnode2_1 (pos: 4800), vnode2_2 (pos: 6100)
shard3 → vnode3_0 (pos: 2300), vnode3_1 (pos: 5500), vnode3_2 (pos: 8900)
```
- Більше vnodes = рівномірніший розподіл
- Типово: 100-200 vnodes на фізичний шард

### Імплементація

```python
import hashlib
from bisect import bisect

class ConsistentHash:
    def __init__(self, nodes=None, virtual_nodes=100):
        self.virtual_nodes = virtual_nodes
        self.ring = []          # sorted list of hash positions
        self.node_map = {}      # hash position → physical node

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring.append(hash_value)
            self.node_map[hash_value] = node
        self.ring.sort()

    def remove_node(self, node):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring.remove(hash_value)
            del self.node_map[hash_value]

    def get_node(self, key):
        if not self.ring:
            return None
        hash_value = self._hash(key)
        # Знаходимо наступну позицію на кільці (за годинниковою стрілкою)
        idx = bisect(self.ring, hash_value) % len(self.ring)
        return self.node_map[self.ring[idx]]


# Приклад використання
ch = ConsistentHash(["shard1", "shard2", "shard3"])
print(ch.get_node("user123"))  # → shard2

# Видаляємо shard2 — тільки частина ключів переміститься
ch.remove_node("shard2")
print(ch.get_node("user123"))  # → shard1 або shard3
```

### Де використовується

| Система | Як використовує |
|---|---|
| Cassandra | Розподіл даних по нодах (vnodes) |
| DynamoDB | Внутрішній розподіл partitions |
| Redis Cluster | 16384 hash slots (не classic consistent hashing, але схожий підхід) |
| CDN (Akamai) | Розподіл контенту по edge серверах |
| Load Balancers | Sticky sessions з мінімальним disruption |
