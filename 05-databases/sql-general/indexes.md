# SQL Indexes — Питання для інтерв'ю

## Що таке індекс і як працює B-Tree?

Індекс — структура даних, яка прискорює пошук за рахунок додаткового місця на диску та уповільнення INSERT/UPDATE.

### B-Tree (Balanced Tree) — найпоширеніший тип
```
                    [50]
                   /    \
           [20, 35]      [70, 85]
          /   |   \     /   |    \
      [10,15][25,30][40,45][60,65][75,80][90,95]
```
- Пошук: O(log n) замість O(n) для sequential scan
- Сортовані дані → підтримує ORDER BY, >, <, BETWEEN
- Кожен вузол містить кілька ключів (зменшує кількість disk reads)

---

## Типи індексів

| Тип | Опис | Коли |
| --- | --- | --- |
| B-Tree | Збалансоване дерево | За замовчуванням, =, <, >, BETWEEN, ORDER BY |
| Hash | Hash table | Тільки = (exact match) |
| GIN | Generalized Inverted | Масиви, JSONB, full-text search |
| GiST | Generalized Search Tree | Геодані, ranges, full-text |
| BRIN | Block Range Index | Великі таблиці з природнім порядком (timestamp) |

---

## Composite Index (складений)

```sql
CREATE INDEX idx_users_country_city ON users(country, city);
```

**Leftmost prefix rule** — індекс працює тільки зліва направо:
```sql
WHERE country = 'UA' AND city = 'Kyiv'  -- ✅ обидва
WHERE country = 'UA'                     -- ✅ перший
WHERE city = 'Kyiv'                      -- ❌ другий без першого
```

**Порядок колонок важливий:** перша колонка — найбільш selective.

---

## Covering Index

Індекс містить ВСІ колонки, потрібні для запиту — не потрібно читати таблицю:

```sql
CREATE INDEX idx_users_email_name ON users(email) INCLUDE (name, created_at);

-- Index-only scan (не торкається таблиці)
SELECT name, created_at FROM users WHERE email = 'john@example.com';
```

---

## Partial Index

Індекс тільки для підмножини рядків:

```sql
CREATE INDEX idx_active_users ON users(email) WHERE deleted_at IS NULL;
-- Менший індекс, швидший пошук серед активних
```

---

## Unique Index

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
-- Забороняє дублікати + працює як звичайний індекс
```

---

## Коли індекс НЕ потрібен?

- Маленькі таблиці (< 1000 рядків) — seq scan швидший
- Колонки з низькою selectivity (boolean, status з 3 значеннями)
- Таблиці з частими INSERT/UPDATE/DELETE — індекс уповільнює запис
- Колонки, які рідко використовуються в WHERE/JOIN/ORDER BY

---

## Вплив індексів на INSERT/UPDATE/DELETE

```
Операція    | Без індексу | З 1 індексом | З 5 індексами
INSERT      | Швидкий     | +10-20%      | +50-100%
UPDATE      | Залежить    | Якщо indexed col — повільніше
DELETE      | Залежить    | Повільніше (видалення з індексу)
SELECT      | Повільний   | Набагато швидший
```

**Баланс:** не створюй індекси "на всяк випадок". Кожен індекс — overhead на запис.

---

## Index vs Sequential Scan

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'john@example.com';
```

**Index Scan** — використовує індекс, потім читає рядок з таблиці
**Index Only Scan** — вся інформація в індексі (covering)
**Bitmap Index Scan** — для великої кількості результатів
**Sequential Scan** — читає всю таблицю

PostgreSQL сам обирає оптимальний план. Може ігнорувати індекс якщо:
- Таблиця маленька
- Запит повертає > 10-20% рядків
- Статистика застаріла (потрібен `ANALYZE`)
