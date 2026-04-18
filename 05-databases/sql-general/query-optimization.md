# Оптимізація SQL запитів — Питання для інтерв'ю

## EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'john@example.com';

-- Output:
-- Index Scan using idx_users_email on users (cost=0.42..8.44 rows=1 width=128)
--   (actual time=0.028..0.029 rows=1 loops=1)
-- Planning Time: 0.085 ms
-- Execution Time: 0.048 ms
```

**Ключові поля:**
- `cost` — оцінка (перше число = startup, друге = total)
- `rows` — очікувана кількість рядків
- `actual time` — реальний час (мс)
- `loops` — кількість виконань (для nested loops)

---

## Scan Types

| Тип | Опис | Коли |
| --- | --- | --- |
| Seq Scan | Повне сканування таблиці | Немає індексу або повертає > 15% рядків |
| Index Scan | Пошук по індексу + читання таблиці | Мало рядків (< 15%) |
| Index Only Scan | Тільки індекс (covering) | Всі потрібні дані в індексі |
| Bitmap Index Scan | Будує bitmap → читає блоки | Середня кількість рядків |

---

## Підзапити vs JOIN

```sql
-- ❌ Повільно: correlated subquery (виконується для кожного рядка)
SELECT name, (SELECT COUNT(*) FROM orders WHERE orders.user_id = users.id) as order_count
FROM users;

-- ✅ Швидше: JOIN + GROUP BY
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
```

---

## CTE (Common Table Expressions)

```sql
WITH active_users AS (
  SELECT id, name FROM users WHERE status = 'active'
),
user_orders AS (
  SELECT user_id, COUNT(*) as total FROM orders GROUP BY user_id
)
SELECT au.name, COALESCE(uo.total, 0) as orders
FROM active_users au
LEFT JOIN user_orders uo ON uo.user_id = au.id;
```

**PostgreSQL 12+:** CTE можуть бути інлайновані (оптимізатор може "розгорнути" CTE). До 12 — CTE завжди materialized (optimization fence).

---

## Window Functions

```sql
-- Ранжування
SELECT name, salary,
  RANK() OVER (ORDER BY salary DESC) as rank,
  ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num,
  DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank
FROM employees;

-- Running total
SELECT date, amount,
  SUM(amount) OVER (ORDER BY date) as running_total
FROM transactions;

-- По групах
SELECT department, name, salary,
  AVG(salary) OVER (PARTITION BY department) as dept_avg,
  salary - AVG(salary) OVER (PARTITION BY department) as diff_from_avg
FROM employees;
```

---

## Pagination: Offset vs Cursor

### Offset (повільний для великих offset)
```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- PostgreSQL повинен прочитати 10020 рядків і відкинути 10000!
```

### Cursor (швидкий, стабільний)
```sql
-- Перша сторінка
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20;

-- Наступна (cursor = created_at останнього)
SELECT * FROM posts
WHERE created_at < '2024-01-15T10:30:00'
ORDER BY created_at DESC LIMIT 20;
-- Використовує INDEX, завжди O(log n)
```

---

## N+1 проблема

```
❌ N+1: 1 запит на список + N запитів на деталі
SELECT * FROM users;          -- 1 запит → 100 users
SELECT * FROM orders WHERE user_id = 1;  -- }
SELECT * FROM orders WHERE user_id = 2;  -- } 100 запитів!
...

✅ Рішення 1: JOIN
SELECT u.*, o.* FROM users u LEFT JOIN orders o ON o.user_id = u.id;

✅ Рішення 2: Batch (IN clause)
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ..., 100);

✅ Рішення 3: DataLoader (у application layer)
```

---

## Загальні поради з оптимізації

1. **EXPLAIN ANALYZE** перед оптимізацією — не оптимізуй навмання
2. **Індекси** на колонки в WHERE, JOIN, ORDER BY
3. **SELECT тільки потрібні колонки** (не SELECT *)
4. **LIMIT** завжди для pagination
5. **VACUUM ANALYZE** регулярно (або autovacuum)
6. **Уникай функцій** на indexed колонках у WHERE: `WHERE LOWER(email)` не використає індекс (потрібен expression index)
7. **Batch INSERT** замість по одному
8. **Connection pooling** (PgBouncer, node-pg pool)
