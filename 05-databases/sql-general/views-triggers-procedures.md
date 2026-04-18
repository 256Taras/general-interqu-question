# Views, Triggers, Procedures — Питання для інтерв'ю

## Views vs Materialized Views

### View — віртуальна таблиця (збережений запит)
```sql
CREATE VIEW active_users AS
SELECT id, name, email FROM users WHERE deleted_at IS NULL;

SELECT * FROM active_users; -- виконує запит кожного разу
```
- Не зберігає дані
- Завжди актуальні
- Можуть бути повільними (складний запит)

### Materialized View — кешований результат
```sql
CREATE MATERIALIZED VIEW user_stats AS
SELECT user_id, COUNT(*) as posts, AVG(rating) as avg_rating
FROM reviews GROUP BY user_id;

-- Оновлення вручну
REFRESH MATERIALIZED VIEW user_stats;
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats; -- без блокування reads
```
- Зберігає дані на диску
- Потребує ручного/scheduled оновлення
- Швидке читання (як звичайна таблиця)
- Можна додавати індекси

**Коли Materialized View:** агрегації, dashboards, звіти — де допустимо stale data.

---

## Triggers

Автоматично виконується при INSERT/UPDATE/DELETE:

```sql
-- Trigger function
CREATE FUNCTION update_modified_at() RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger
CREATE TRIGGER set_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_modified_at();
```

### BEFORE vs AFTER
- **BEFORE** — може модифікувати дані перед записом (RETURN NEW)
- **AFTER** — для побічних ефектів (логування, нотифікації)

### Коли використовувати
- ✅ updated_at timestamps
- ✅ Audit logging
- ✅ Денормалізовані counters
- ❌ Складна бізнес-логіка (краще в application)
- ❌ Зовнішні API виклики

---

## Stored Procedures vs Application Logic

### Stored Procedures
```sql
CREATE PROCEDURE transfer_funds(sender INT, receiver INT, amount DECIMAL)
LANGUAGE plpgsql AS $$
BEGIN
  UPDATE accounts SET balance = balance - amount WHERE id = sender;
  UPDATE accounts SET balance = balance + amount WHERE id = receiver;
END;
$$;

CALL transfer_funds(1, 2, 100.00);
```

### Коли Stored Procedures
- ✅ Критична атомарність (фінансові операції)
- ✅ Зменшення мережевих round-trips
- ✅ Обробка великих обсягів даних

### Коли Application Logic
- ✅ Бізнес-логіка (легше тестувати, версіонувати)
- ✅ Інтеграція з зовнішніми сервісами
- ✅ Складні бізнес-правила
- ✅ Потрібна портабельність між БД

**Сучасний підхід:** бізнес-логіка в application, stored procedures тільки для performance-critical операцій.

---

## Functions vs Procedures (PostgreSQL)

```sql
-- Function — повертає значення, можна в SELECT
CREATE FUNCTION full_name(first TEXT, last TEXT) RETURNS TEXT AS $$
  SELECT first || ' ' || last;
$$ LANGUAGE SQL IMMUTABLE;

SELECT full_name(first_name, last_name) FROM users;

-- Procedure — не повертає, може управляти транзакціями
CREATE PROCEDURE cleanup_old_data() LANGUAGE plpgsql AS $$
BEGIN
  DELETE FROM logs WHERE created_at < NOW() - INTERVAL '90 days';
  COMMIT;
  DELETE FROM sessions WHERE expires_at < NOW();
  COMMIT;
END;
$$;

CALL cleanup_old_data();
```
