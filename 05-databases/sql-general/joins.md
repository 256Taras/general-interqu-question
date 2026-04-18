# SQL JOINs

## Що таке JOIN і навіщо він потрібен?

JOIN — це операція в SQL, яка дозволяє об'єднувати рядки з двох або більше таблиць на основі пов'язаної колонки між ними. JOIN є фундаментальною операцією реляційних баз даних, оскільки дані зазвичай нормалізовані та розподілені по кількох таблицях.

Без JOIN нам довелося б робити окремі запити до кожної таблиці, а потім об'єднувати результати на рівні застосунку, що значно менш ефективно.

---

## INNER JOIN

INNER JOIN повертає лише ті рядки, які мають відповідні значення в обох таблицях. Якщо рядок з однієї таблиці не має відповідника в іншій — він не потрапить у результат.

```sql
-- Знайти всіх користувачів та їх замовлення
SELECT users.name, orders.total_amount
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

**Коли використовувати:**
- Коли потрібні лише записи, що мають зв'язок в обох таблицях
- Коли відсутність зв'язку означає невалідні дані
- Найпоширеніший тип JOIN у повсякденній роботі

**Важливо:** INNER JOIN є комутативним — порядок таблиць не впливає на результат (але може впливати на план виконання запиту).

```sql
-- Ці два запити дадуть однаковий результат
SELECT * FROM users INNER JOIN orders ON users.id = orders.user_id;
SELECT * FROM orders INNER JOIN users ON orders.user_id = users.id;
```

---

## LEFT JOIN (LEFT OUTER JOIN)

LEFT JOIN повертає всі рядки з лівої таблиці та відповідні рядки з правої. Якщо для рядка з лівої таблиці немає відповідника в правій — колонки правої таблиці заповнюються NULL.

```sql
-- Знайти всіх користувачів, навіть якщо вони не мають замовлень
SELECT users.name, orders.total_amount
FROM users
LEFT JOIN orders ON users.id = orders.user_id;
```

**Результат:**
```
| name    | total_amount |
|---------|-------------|
| Олексій | 500.00      |
| Марія   | 300.00      |
| Іван    | NULL        |  -- Іван не має замовлень
```

**Коли використовувати:**
- Коли потрібно показати всі записи з основної таблиці незалежно від наявності зв'язків
- Для пошуку записів без зв'язків (сирот)
- Для звітів, де потрібна повна картина

```sql
-- Знайти користувачів БЕЗ замовлень
SELECT users.name
FROM users
LEFT JOIN orders ON users.id = orders.user_id
WHERE orders.id IS NULL;
```

---

## RIGHT JOIN (RIGHT OUTER JOIN)

RIGHT JOIN — це дзеркальне відображення LEFT JOIN. Повертає всі рядки з правої таблиці та відповідні рядки з лівої. Якщо відповідника немає — колонки лівої таблиці заповнюються NULL.

```sql
-- Знайти всі замовлення, навіть якщо користувач видалений
SELECT users.name, orders.total_amount
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;
```

**На практиці:** RIGHT JOIN використовується рідко, оскільки його завжди можна переписати як LEFT JOIN, змінивши порядок таблиць. Більшість розробників віддають перевагу LEFT JOIN для кращої читабельності.

```sql
-- Ці два запити еквівалентні
SELECT * FROM users RIGHT JOIN orders ON users.id = orders.user_id;
SELECT * FROM orders LEFT JOIN users ON orders.user_id = users.id;
```

---

## FULL OUTER JOIN

FULL OUTER JOIN повертає всі рядки з обох таблиць. Якщо рядок з однієї таблиці не має відповідника в іншій — відсутні значення заповнюються NULL.

```sql
-- Показати всіх користувачів та всі замовлення, навіть непов'язані
SELECT users.name, orders.total_amount
FROM users
FULL OUTER JOIN orders ON users.id = orders.user_id;
```

**Результат:**
```
| name    | total_amount |
|---------|-------------|
| Олексій | 500.00      |
| Марія   | 300.00      |
| Іван    | NULL        |  -- Користувач без замовлень
| NULL    | 150.00      |  -- Замовлення без користувача
```

**Коли використовувати:**
- Для аудиту та пошуку невідповідностей між таблицями
- Для міграції даних, коли потрібно побачити повну картину
- Для звірки двох наборів даних

```sql
-- Знайти всі невідповідності між таблицями
SELECT users.id AS user_id, orders.id AS order_id
FROM users
FULL OUTER JOIN orders ON users.id = orders.user_id
WHERE users.id IS NULL OR orders.id IS NULL;
```

---

## CROSS JOIN

CROSS JOIN повертає декартів добуток двох таблиць — кожен рядок першої таблиці поєднується з кожним рядком другої. Якщо в першій таблиці N рядків, а в другій M — результат матиме N * M рядків.

```sql
-- Генерація всіх можливих комбінацій розмір-колір
SELECT sizes.name AS size, colors.name AS color
FROM sizes
CROSS JOIN colors;
```

**Результат (якщо 3 розміри та 3 кольори = 9 рядків):**
```
| size | color   |
|------|---------|
| S    | Червоний|
| S    | Синій   |
| S    | Зелений |
| M    | Червоний|
| M    | Синій   |
| M    | Зелений |
| L    | Червоний|
| L    | Синій   |
| L    | Зелений |
```

**Коли використовувати:**
- Генерація всіх можливих комбінацій
- Створення календарних таблиць
- Тестові дані

**Обережно:** CROSS JOIN може генерувати величезні набори даних. Таблиця з 1000 рядків CROSS JOIN з таблицею з 1000 рядків дасть 1 000 000 рядків.

---

## Self JOIN

Self JOIN — це об'єднання таблиці з самою собою. Використовується, коли потрібно порівняти рядки всередині однієї таблиці або для ієрархічних структур.

```sql
-- Знайти менеджера для кожного співробітника
SELECT
  e.name AS employee,
  m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

```sql
-- Знайти співробітників з однаковою зарплатою
SELECT
  a.name AS employee_1,
  b.name AS employee_2,
  a.salary
FROM employees a
INNER JOIN employees b ON a.salary = b.salary
WHERE a.id < b.id;  -- Уникаємо дублікатів та порівняння з собою
```

**Типові випадки використання:**
- Ієрархічні дані (організаційна структура, категорії)
- Порівняння рядків в одній таблиці
- Пошук дублікатів
- Графові структури (друзі, підписники)

```sql
-- Рекурсивний запит для повної ієрархії (CTE)
WITH RECURSIVE hierarchy AS (
  -- Базовий випадок: кореневі елементи
  SELECT id, name, manager_id, 1 AS level
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Рекурсивний крок
  SELECT e.id, e.name, e.manager_id, h.level + 1
  FROM employees e
  INNER JOIN hierarchy h ON e.manager_id = h.id
)
SELECT * FROM hierarchy ORDER BY level, name;
```

---

## Коли використовувати який тип JOIN?

| Ситуація | Тип JOIN | Приклад |
|----------|----------|---------|
| Потрібні лише пов'язані записи | INNER JOIN | Користувачі з замовленнями |
| Потрібні всі записи з основної таблиці | LEFT JOIN | Усі користувачі, навіть без замовлень |
| Пошук записів без зв'язків | LEFT JOIN + WHERE IS NULL | Користувачі без замовлень |
| Аудит невідповідностей | FULL OUTER JOIN | Звірка двох таблиць |
| Всі комбінації | CROSS JOIN | Розміри x Кольори |
| Ієрархічні дані | Self JOIN | Підлеглі та менеджери |

**Правило великого пальця:**
- 90% випадків — INNER JOIN або LEFT JOIN
- FULL OUTER JOIN — рідкісний, переважно для аналітики
- CROSS JOIN — ще рідший, обережно з великими таблицями
- RIGHT JOIN — майже ніколи, замініть на LEFT JOIN

---

## Performance Considerations для JOINs

### 1. Індекси на колонках JOIN

Найважливіший фактор продуктивності — наявність індексів на колонках, які використовуються у JOIN.

```sql
-- Без індексу: Seq Scan (O(n*m))
-- З індексом: Index Scan або Hash Join (значно швидше)

CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Тепер JOIN буде використовувати індекс
SELECT users.name, orders.total_amount
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

### 2. Типи фізичних JOIN операцій

PostgreSQL використовує три стратегії виконання JOIN:

**Nested Loop Join:**
- Для кожного рядка зовнішньої таблиці шукає відповідники у внутрішній
- Ефективний, коли одна таблиця маленька, а інша має індекс
- Складність: O(n * m), але з індексом O(n * log m)

**Hash Join:**
- Створює хеш-таблицю з меншої таблиці, потім сканує більшу
- Ефективний для великих таблиць без індексів
- Потребує пам'ять для хеш-таблиці
- Складність: O(n + m)

**Merge Join (Sort-Merge Join):**
- Сортує обидві таблиці та злиття
- Ефективний, коли дані вже відсортовані (або є індекс)
- Складність: O(n log n + m log m) для сортування, O(n + m) для злиття

```sql
-- Перевірити, який тип JOIN обрав планувальник
EXPLAIN ANALYZE
SELECT users.name, orders.total_amount
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

### 3. Порядок таблиць у JOIN

Хоча SQL є декларативною мовою і планувальник сам обирає оптимальний порядок, варто знати:

```sql
-- Планувальник може змінити порядок для INNER JOIN
-- Але для LEFT/RIGHT JOIN порядок фіксований

-- Для складних запитів з багатьма JOIN:
-- Починайте з найменшої таблиці або тієї, що найбільше фільтрується
SELECT *
FROM small_table s
INNER JOIN medium_table m ON s.id = m.small_id
INNER JOIN large_table l ON m.id = l.medium_id
WHERE s.status = 'active';  -- Фільтр зменшує small_table
```

### 4. Уникайте зайвих JOIN

```sql
-- Погано: JOIN, коли потрібен тільки EXISTS
SELECT DISTINCT users.name
FROM users
INNER JOIN orders ON users.id = orders.user_id;

-- Краще: EXISTS більш ефективний
SELECT users.name
FROM users
WHERE EXISTS (
  SELECT 1 FROM orders WHERE orders.user_id = users.id
);
```

### 5. Фільтруйте якомога раніше

```sql
-- Погано: фільтрація після JOIN
SELECT users.name, orders.total_amount
FROM users
INNER JOIN orders ON users.id = orders.user_id
WHERE orders.created_at > '2024-01-01'
  AND users.status = 'active';

-- Краще: планувальник зазвичай оптимізує це сам,
-- але для складних запитів можна використати підзапити
SELECT u.name, o.total_amount
FROM (SELECT * FROM users WHERE status = 'active') u
INNER JOIN (SELECT * FROM orders WHERE created_at > '2024-01-01') o
  ON u.id = o.user_id;
```

---

## Практичні задачі з JOINs

### Задача 1: Знайти записи без зв'язків (сироти)

```sql
-- Спосіб 1: LEFT JOIN + IS NULL (найпоширеніший)
SELECT users.id, users.name
FROM users
LEFT JOIN orders ON users.id = orders.user_id
WHERE orders.id IS NULL;

-- Спосіб 2: NOT EXISTS (часто швидший)
SELECT users.id, users.name
FROM users
WHERE NOT EXISTS (
  SELECT 1 FROM orders WHERE orders.user_id = users.id
);

-- Спосіб 3: NOT IN (обережно з NULL!)
SELECT users.id, users.name
FROM users
WHERE users.id NOT IN (
  SELECT user_id FROM orders WHERE user_id IS NOT NULL
);
```

**Порівняння продуктивності:**
- LEFT JOIN + IS NULL: добре працює з індексом
- NOT EXISTS: зазвичай найшвидший, зупиняється на першому знайденому
- NOT IN: повільний для великих підзапитів, проблеми з NULL

### Задача 2: N+1 проблема

N+1 проблема виникає, коли ORM виконує окремий запит для кожного зв'язаного запису.

```sql
-- Проблема N+1:
-- 1 запит: SELECT * FROM users LIMIT 100;
-- 100 запитів: SELECT * FROM orders WHERE user_id = ?; (для кожного користувача)

-- Рішення: один JOIN запит
SELECT users.*, orders.*
FROM users
LEFT JOIN orders ON users.id = orders.user_id;

-- Або з агрегацією
SELECT
  users.id,
  users.name,
  COUNT(orders.id) AS order_count,
  SUM(orders.total_amount) AS total_spent
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id, users.name;
```

### Задача 3: Знайти дублікати

```sql
-- Знайти користувачів з однаковим email
SELECT a.id AS id_1, b.id AS id_2, a.email
FROM users a
INNER JOIN users b ON a.email = b.email
WHERE a.id < b.id;
```

### Задача 4: Знайти N останніх записів для кожної групи

```sql
-- Знайти 3 останніх замовлення для кожного користувача
SELECT u.name, o.id AS order_id, o.total_amount, o.created_at
FROM users u
INNER JOIN LATERAL (
  SELECT *
  FROM orders
  WHERE orders.user_id = u.id
  ORDER BY created_at DESC
  LIMIT 3
) o ON true;
```

### Задача 5: Множинні JOIN з агрегацією

```sql
-- Звіт: користувачі, кількість замовлень, загальна сума, кількість відгуків
SELECT
  u.name,
  COUNT(DISTINCT o.id) AS order_count,
  COALESCE(SUM(o.total_amount), 0) AS total_spent,
  COUNT(DISTINCT r.id) AS review_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
LEFT JOIN reviews r ON u.id = r.user_id
GROUP BY u.id, u.name
ORDER BY total_spent DESC;
```

**Увага:** При множинних LEFT JOIN з агрегацією може виникнути проблема "роздування" рядків. Використовуйте COUNT(DISTINCT) замість COUNT(*).

### Задача 6: Conditional JOIN

```sql
-- JOIN з різними умовами залежно від типу
SELECT
  p.name AS product,
  COALESCE(d.discount_percent, 0) AS discount
FROM products p
LEFT JOIN discounts d ON p.id = d.product_id
  AND d.start_date <= CURRENT_DATE
  AND d.end_date >= CURRENT_DATE
  AND d.is_active = true;
```

---

## Поширені помилки при роботі з JOINs

### 1. Картезіанський добуток через пропущену умову

```sql
-- Помилка: забули ON — отримаємо CROSS JOIN
SELECT * FROM users, orders;  -- Це CROSS JOIN!

-- Правильно:
SELECT * FROM users INNER JOIN orders ON users.id = orders.user_id;
```

### 2. Дублювання рядків при JOIN з таблицею "один-до-багатьох"

```sql
-- Якщо у користувача 5 замовлень, його ім'я повториться 5 разів
-- Для унікальних значень використовуйте DISTINCT або агрегацію
SELECT DISTINCT users.name
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

### 3. NULL у JOIN умовах

```sql
-- NULL != NULL в SQL, тому JOIN по колонці з NULL не працює
-- Рядки з NULL не з'єднаються
SELECT *
FROM table_a a
INNER JOIN table_b b ON a.nullable_col = b.nullable_col;
-- Рядки де nullable_col IS NULL не увійдуть у результат

-- Якщо потрібно об'єднати NULL значення:
SELECT *
FROM table_a a
INNER JOIN table_b b ON a.nullable_col IS NOT DISTINCT FROM b.nullable_col;
```

---

## Підсумок

- **INNER JOIN** — найчастіший, повертає лише пов'язані записи
- **LEFT JOIN** — другий за частотою, використовуйте для пошуку сирот та повних звітів
- **RIGHT JOIN** — замініть на LEFT JOIN, змінивши порядок таблиць
- **FULL OUTER JOIN** — для аудиту та міграцій
- **CROSS JOIN** — обережно, генерує декартів добуток
- **Self JOIN** — для ієрархій та порівняння рядків
- Завжди перевіряйте наявність **індексів** на колонках JOIN
- Використовуйте **EXPLAIN ANALYZE** для аналізу продуктивності
- Фільтруйте дані **якомога раніше** в запиті
