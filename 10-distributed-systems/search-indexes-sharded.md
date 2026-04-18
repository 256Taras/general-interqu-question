# Пошук по шардованих даних — Питання для інтерв'ю

## Глобальний пошук по шардованій БД

```
"Ми маємо шардовану БД користувачів. Як реалізувати пошук
користувачів за email або іменем?"
```

### Проблема

```
Шардування по user_id (hash-based):
shard0: user_id hash → 0
shard1: user_id hash → 1
shard2: user_id hash → 2

Запит: WHERE email = 'john@example.com'
→ Не знаємо на якому шарді цей email!
→ Потрібно шукати на ВСІХ шардах
```

---

### Варіант 1: Global Secondary Index

```
Окрема таблиця (або сервіс), яка маппить email → user_id + shard_id

global_email_index:
┌──────────────────────┬──────────┬──────────┐
│ email                │ user_id  │ shard_id │
├──────────────────────┼──────────┼──────────┤
│ john@example.com     │ 12345    │ shard2   │
│ jane@example.com     │ 67890    │ shard0   │
└──────────────────────┴──────────┴──────────┘
```

**Як працює:**
```
1. Шукаємо email в global index → отримуємо (user_id, shard_id)
2. Йдемо на конкретний shard → отримуємо повні дані
```

| | |
|---|---|
| Плюси | Швидкий lookup O(1), один запит до шарда |
| Мінуси | Потрібно підтримувати index в sync, додатковий storage |
| Consistency | Може бути stale при асинхронному оновленні |
| Підходить | Для точного пошуку (email, username, phone) |

---

### Варіант 2: Scatter-Gather

```
Запит розсилається на ВСІ шарди паралельно, результати збираються.

Client → Query Router → shard0: SELECT * WHERE email LIKE '%john%'
                       → shard1: SELECT * WHERE email LIKE '%john%'
                       → shard2: SELECT * WHERE email LIKE '%john%'
                       ← Merge results
```

```python
async def search_users(query):
    # Розсилаємо запит на всі шарди паралельно
    tasks = [shard.search(query) for shard in shards]

    # Чекаємо всі відповіді (з timeout)
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # Фільтруємо помилки, об'єднуємо результати
    all_users = []
    for result in results:
        if not isinstance(result, Exception):
            all_users.extend(result)

    # Сортуємо за релевантністю, обмежуємо
    return sorted(all_users, key=lambda x: x["score"])[:100]
```

| | |
|---|---|
| Плюси | Простота, немає додаткової інфраструктури |
| Мінуси | Повільно (чекаємо найповільніший шард), навантаження на всі шарди |
| Latency | Визначається найповільнішим шардом |
| Підходить | Для рідких запитів, адмін-панелі |

**Оптимізації scatter-gather:**
```
1. Timeout per shard — не чекаємо повільний шард вічно
2. Parallel fan-out — запити паралельно, не послідовно
3. Caching результатів — для частих запитів
4. Early termination — якщо знайшли достатньо результатів
```

---

### Варіант 3: Search Engine (ElasticSearch)

```
                    ┌─────────────────────┐
                    │   ElasticSearch      │
                    │   (пошуковий індекс) │
                    └─────────┬───────────┘
                              ↑ sync
┌──────────┐  ┌──────────┐  ┌──────────┐
│  shard0  │  │  shard1  │  │  shard2  │
└──────────┘  └──────────┘  └──────────┘
```

**Як працює:**
```
1. При створенні/оновленні user → асинхронно індексуємо в ElasticSearch
2. Пошук → запит в ElasticSearch → отримуємо user_ids
3. Фетч повних даних → по user_id йдемо на конкретний шард
```

```python
async def search_users(query):
    # 1. Пошук в ElasticSearch (повнотекстовий, fuzzy, фільтри)
    es_results = await elasticsearch.search(
        index="users",
        body={
            "query": {
                "multi_match": {
                    "query": query,
                    "fields": ["name", "email"],
                    "fuzziness": "AUTO"
                }
            },
            "size": 100
        }
    )

    # 2. Отримуємо user_ids та shard mapping
    user_ids = [hit["_source"]["user_id"] for hit in es_results["hits"]["hits"]]

    # 3. Фетч повних даних з відповідних шардів (batch)
    users = await fetch_users_from_shards(user_ids)

    return users
```

| | |
|---|---|
| Плюси | Повнотекстовий пошук, fuzzy matching, фільтри, сортування за релевантністю |
| Мінуси | Додаткова інфраструктура, eventual consistency (sync lag) |
| Sync | CDC (Change Data Capture) через Kafka/Debezium |
| Підходить | Для складного пошуку, автодоповнення, фільтрації |

---

### Варіант 4: Materialized View

```
Окрема read-optimized таблиця, яка оновлюється при змінах.

materialized_users_by_email:
┌──────────────────────┬──────────┬───────────┬────────────┐
│ email                │ user_id  │ name      │ country    │
├──────────────────────┼──────────┼───────────┼────────────┤
│ john@example.com     │ 12345    │ John Doe  │ US         │
└──────────────────────┴──────────┴───────────┴────────────┘

Шардування цієї таблиці по hash(email) — для швидкого пошуку по email.
```

| | |
|---|---|
| Плюси | Швидкий пошук, данні вже denormalized |
| Мінуси | Дублювання даних, потрібна синхронізація |
| Підходить | Для фіксованих паттернів пошуку (по email, по phone) |

---

### Порівняння підходів

| Критерій | Global Index | Scatter-Gather | ElasticSearch | Materialized View |
|---|---|---|---|---|
| Latency | Низька | Висока | Середня | Низька |
| Складність | Середня | Низька | Висока | Середня |
| Full-text search | Ні | Ні | Так | Ні |
| Consistency | Strong/Eventual | Strong | Eventual | Eventual |
| Додаткове storage | Мінімальне | Ні | Значне | Значне |
| Масштабованість | Висока | Обмежена | Висока | Висока |

### Рекомендація

```
Точний пошук (email, phone) → Global Secondary Index
Повнотекстовий пошук (ім'я, опис) → ElasticSearch
Рідкі адмін-запити → Scatter-Gather
Фіксовані паттерни пошуку → Materialized View
```
