# Реплікація — Питання для інтерв'ю

## Master-Slave: Replication Lag проблема

```
"У нас є Master-Slave налаштування. Користувач щойно зареєструвався, отримав відповідь,
але при переході на профіль отримує 404. Що сталося і як виправити?"
```

### Що сталося

Це **Replication Lag** проблема. Запис пішов на Master, відповідь повернулась клієнту, але читання профілю потрапило на Slave, який ще не отримав нові дані.

```
Client → POST /register → Master (записав) → 201 Created
Client → GET /profile   → Slave (ще не отримав дані) → 404 Not Found
```

### Рішення

**1. Read Your Own Writes**
```
Після запису — наступні читання від цього ж користувача йдуть на Master
(протягом N секунд або до підтвердження реплікації)
```

**2. Sticky Sessions**
```
Після запису — прив'язуємо користувача до Master на короткий період
Session cookie або IP-based routing
```

**3. Читання з Master для автентифікованих користувачів**
```
Неавтентифіковані запити → Slave (каталог, пошук)
Автентифіковані запити → Master (профіль, налаштування)
```

**4. Synchronous Replication (для критичних операцій)**
```
Master чекає підтвердження від хоча б одного Slave перед відповіддю
Повільніше, але гарантує consistency
```

**5. Causal Consistency**
```
При відповіді на запис — повертаємо replication token
Клієнт передає token при наступному читанні
Slave чекає поки не отримає дані до цього token
```

---

## Failover процес при падінні Master

```
"Опишіть крок за кроком, що відбувається при падінні Master в 3-нодному кластері.
Як уникнути Split Brain?"
```

### Покроковий процес

**1. Health Check Failure**
```
Slave 1 → ping Master → timeout (5s)
Slave 2 → ping Master → timeout (5s)
```

**2. Виявлення падіння**
```
Кілька послідовних невдалих health checks → Master помічений як DOWN
Зазвичай: 3 невдалих перевірки × 5 секунд = 15 секунд до виявлення
```

**3. Вибори нового Master**
```
Slave 1: replication lag = 100ms, priority = 1
Slave 2: replication lag = 500ms, priority = 2
→ Slave 1 обраний (менший lag = менше втрачених даних)
```

**4. Переконфігурація кластера**
```
Slave 1 → промоутиться в Master
Slave 2 → переключається на реплікацію з нового Master (Slave 1)
```

**5. Переключення клієнтів**
```
DNS update або VIP (Virtual IP) failover
Клієнти починають писати на новий Master
```

### Запобігання Split Brain

**Split Brain** — ситуація коли два вузли вважають себе Master одночасно.

```
Сценарій Split Brain:
Network partition розділяє кластер на дві частини
Кожна частина обирає свій Master
→ Два Master приймають різні записи → конфлікт даних
```

**Рішення:**

| Метод | Як працює |
|---|---|
| Quorum | Для прийняття рішення потрібна більшість (N/2 + 1). У 3-нодному кластері — мінімум 2 вузли |
| Fencing | Старий Master примусово вимикається (STONITH — Shoot The Other Node In The Head) |
| Distributed Locks | Тільки вузол з lock може бути Master (etcd, ZooKeeper) |
| Epoch/Term numbers | Кожні вибори мають номер, старіший Master ігнорується |

**Приклад з Quorum (3 ноди):**
```
Partition: [Master, Slave1] | [Slave2]

Сторона з 2 нодами (більшість) → може обрати новий Master
Сторона з 1 нодою → НЕ може обрати Master (немає quorum)
→ Split Brain неможливий
```

---

## Master-Master: вирішення конфліктів

```
"Два користувачі змінюють один документ одночасно в різних регіонах
(Master-Master реплікація). Значення: A=100, B=200.
Як вирішувати конфлікт?"
```

### Стратегії вирішення конфліктів

**1. Last Write Wins (LWW)**
```python
def resolve_conflict(v1, v2):
    if v1.timestamp > v2.timestamp:
        return v1.value
    return v2.value
```
- Найпростіший підхід
- Може втратити дані (один запис перезаписується)
- Потребує синхронізованих годинників (NTP)
- Підходить для: user preferences, last-seen status

**2. CRDT (Conflict-free Replicated Data Types)**
```python
# G-Counter (тільки зростає)
# Кожен вузол має свій лічильник
node_a_counter = {"A": 5, "B": 0}
node_b_counter = {"A": 0, "B": 3}

# Merge: беремо max по кожному вузлу
merged = {"A": 5, "B": 3}  # total = 8
```
- Автоматичне вирішення без конфліктів
- Обмежений набір операцій (counter, set, register)
- Підходить для: лайки, лічильники, online статус

**3. Application-level resolution**
```python
def resolve_conflict(v1, v2, operation_type):
    if operation_type == "withdraw":
        # Консервативний підхід — беремо менше
        return min(v1.value, v2.value)
    elif operation_type == "deposit":
        # Оптимістичний — беремо суму різниць
        return original + (v1.value - original) + (v2.value - original)
```
- Бізнес-логіка вирішує конфлікт
- Найточніший, але найскладніший
- Підходить для: фінанси, inventory

**4. Vector Clocks**
```python
# Кожен вузол має vector clock
v1 = {"A": 2, "B": 1}  # написано на Node A
v2 = {"A": 1, "B": 2}  # написано на Node B

# Якщо v1 > v2 (кожен елемент >=) → v1 пізніше
# Якщо ні → concurrent → конфлікт → потрібен merge
```
- Точне визначення причинності (causality)
- Визначає чи записи concurrent чи послідовні
- Підходить для: DynamoDB-style системи

**5. Manual resolution (User decides)**
```
Google Docs підхід:
- Показуємо обидві версії користувачу
- Користувач обирає або merge-ить вручну
```
- Найточніший результат
- Не підходить для автоматичних систем

### Порівняння

| Стратегія | Автоматична | Точність | Складність | Втрата даних |
|---|---|---|---|---|
| LWW | Так | Низька | Низька | Можлива |
| CRDT | Так | Висока | Середня | Ні |
| Application-level | Так | Висока | Висока | Залежить |
| Vector Clocks | Частково | Висока | Висока | Ні |
| Manual | Ні | Найвища | Низька | Ні |
