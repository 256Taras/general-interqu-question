# Transactions та ACID — Питання для інтерв'ю

## ACID Properties

**A — Atomicity (Атомарність)**
Транзакція або виконується повністю, або не виконується взагалі.
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- обидва або жоден
```

**C — Consistency (Узгодженість)**
Транзакція переводить БД з одного валідного стану в інший. Constraints (UNIQUE, FK, CHECK) зберігаються.

**I — Isolation (Ізоляція)**
Паралельні транзакції не бачать незавершені зміни одна одної (залежить від рівня ізоляції).

**D — Durability (Стійкість)**
Після COMMIT дані збережені навіть при crash (WAL — Write-Ahead Log).

---

## Isolation Levels

| Рівень | Dirty Read | Non-Repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| READ UNCOMMITTED | Можливий | Можливий | Можливий |
| READ COMMITTED | Ні | Можливий | Можливий |
| REPEATABLE READ | Ні | Ні | Можливий* |
| SERIALIZABLE | Ні | Ні | Ні |

*PostgreSQL REPEATABLE READ також запобігає phantom reads через MVCC.

### Dirty Read
Транзакція бачить незакомічені зміни іншої:
```
T1: UPDATE users SET name = 'Jane' WHERE id = 1;  (не COMMIT)
T2: SELECT name FROM users WHERE id = 1;  → 'Jane' (dirty!)
T1: ROLLBACK;  → name знову 'John', але T2 вже прочитала 'Jane'
```

### Non-Repeatable Read
Повторне читання дає інший результат:
```
T1: SELECT balance FROM accounts WHERE id = 1;  → 100
T2: UPDATE accounts SET balance = 50 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  → 50 (змінилось!)
```

### Phantom Read
Нові рядки з'являються у повторному запиті:
```
T1: SELECT * FROM users WHERE age > 20;  → 5 рядків
T2: INSERT INTO users (name, age) VALUES ('New', 25); COMMIT;
T1: SELECT * FROM users WHERE age > 20;  → 6 рядків (phantom!)
```

---

## Deadlocks

```
T1: LOCK row A → чекає на row B
T2: LOCK row B → чекає на row A
→ Deadlock! PostgreSQL автоматично відміняє одну транзакцію.
```

**Запобігання:**
- Завжди блокуй ресурси в одному порядку
- Тримай транзакції короткими
- Використовуй `SET lock_timeout = '5s'`
- Використовуй advisory locks для application-level locking

---

## Optimistic vs Pessimistic Locking

### Pessimistic — блокуємо ДО зміни
```sql
BEGIN;
SELECT * FROM products WHERE id = 1 FOR UPDATE; -- блокує рядок
-- інші транзакції чекають
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;
```
**Коли:** високий contention, критичні дані (фінанси).

### Optimistic — перевіряємо ПІСЛЯ зміни
```sql
-- Читаємо з version
SELECT id, stock, version FROM products WHERE id = 1;
-- stock = 10, version = 5

-- Оновлюємо з перевіркою version
UPDATE products SET stock = 9, version = 6
WHERE id = 1 AND version = 5;
-- Якщо affected_rows = 0 → хтось змінив → retry
```
**Коли:** низький contention, read-heavy.

---

## MVCC (Multi-Version Concurrency Control)

PostgreSQL не блокує рядки при читанні. Кожна транзакція бачить **snapshot** даних:

```
Рядок: { id: 1, name: 'John', xmin: 100, xmax: 105 }

T1 (txid: 103): бачить 'John' (100 < 103, xmax 105 > 103)
T2 (txid: 106): НЕ бачить цей рядок (xmax 105 < 106 → видалений)
```

- `xmin` — транзакція що створила рядок
- `xmax` — транзакція що видалила/оновила
- Readers не блокують Writers, Writers не блокують Readers
- VACUUM очищає старі версії рядків
