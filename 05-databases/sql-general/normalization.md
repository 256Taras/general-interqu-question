# Нормалізація БД — Питання для інтерв'ю

## 1NF (Перша нормальна форма)

**Правила:**
- Кожна клітинка містить одне атомарне значення
- Кожен рядок унікальний (є primary key)

```
❌ Не 1NF:
| id | name  | phones                |
|----|-------|-----------------------|
| 1  | John  | 380991234567, 380501234567 |

✅ 1NF:
| id | name  | phone        |
|----|-------|--------------|
| 1  | John  | 380991234567 |
| 1  | John  | 380501234567 |
(або окрема таблиця phones)
```

---

## 2NF (Друга нормальна форма)

**Правила:** 1NF + кожне не-ключове поле залежить від ВСЬОГО primary key (не частини).

```
❌ Не 2NF (composite key: student_id + course_id):
| student_id | course_id | student_name | grade |
|------------|-----------|--------------|-------|
student_name залежить тільки від student_id, не від course_id!

✅ 2NF — розділити:
students: | student_id | student_name |
grades:   | student_id | course_id | grade |
```

---

## 3NF (Третя нормальна форма)

**Правила:** 2NF + немає транзитивних залежностей (не-ключове поле не залежить від іншого не-ключового).

```
❌ Не 3NF:
| id | name | city   | country   |
|----|------|--------|-----------|
country залежить від city, а city від id → транзитивна залежність

✅ 3NF:
users:  | id | name | city_id |
cities: | id | city | country |
```

---

## BCNF (Boyce-Codd Normal Form)

**Правила:** 3NF + кожен детермінант є candidate key.

Рідко потрібна на практиці. Різниця з 3NF проявляється при складних залежностях між candidate keys.

---

## Денормалізація — коли потрібна?

**Нормалізація:**
- Зменшує дублювання
- Забезпечує consistency
- Складніші JOIN-и, повільніші читання

**Денормалізація:**
- Швидші читання (менше JOIN-ів)
- Дублювання даних
- Ризик inconsistency

**Коли денормалізувати:**
- Read-heavy системи (90% reads)
- Аналітичні запити (OLAP)
- Кешування агрегатів (counter cache, materialized views)
- Після профілювання — коли JOIN реально bottleneck

**Приклади:**
```sql
-- Денормалізований counter
ALTER TABLE posts ADD COLUMN comments_count INTEGER DEFAULT 0;
-- Замість COUNT(*) FROM comments WHERE post_id = ?

-- Materialized View
CREATE MATERIALIZED VIEW user_stats AS
SELECT user_id, COUNT(*) as posts_count, AVG(rating) as avg_rating
FROM posts GROUP BY user_id;

REFRESH MATERIALIZED VIEW user_stats; -- оновлення вручну
```

**Правило:** нормалізуй за замовчуванням, денормалізуй тільки коли є доведена проблема з performance.
