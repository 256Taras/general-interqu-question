# PostgreSQL Partitioning — Питання для інтерв'ю

## Що таке Partitioning і навіщо?

Розділення великої таблиці на менші фізичні частини (partitions). Логічно — одна таблиця, фізично — кілька.

**Коли потрібен:** таблиця > 100M рядків, запити фільтрують по partition key, потрібен швидкий DELETE старих даних.

---

## Range Partitioning

```sql
CREATE TABLE logs (
  id BIGSERIAL,
  created_at TIMESTAMPTZ NOT NULL,
  message TEXT
) PARTITION BY RANGE (created_at);

CREATE TABLE logs_2024_01 PARTITION OF logs
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE logs_2024_02 PARTITION OF logs
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Запит автоматично сканує тільки потрібну partition
SELECT * FROM logs WHERE created_at BETWEEN '2024-01-15' AND '2024-01-20';
-- → тільки logs_2024_01

-- Швидке видалення старих даних
DROP TABLE logs_2024_01; -- миттєво, замість DELETE мільйонів рядків
```

---

## List Partitioning

```sql
CREATE TABLE orders (
  id SERIAL,
  country TEXT NOT NULL,
  total NUMERIC
) PARTITION BY LIST (country);

CREATE TABLE orders_ua PARTITION OF orders FOR VALUES IN ('UA');
CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US');
CREATE TABLE orders_other PARTITION OF orders DEFAULT;
```

---

## Hash Partitioning

```sql
CREATE TABLE sessions (
  id UUID NOT NULL,
  user_id INT,
  data JSONB
) PARTITION BY HASH (id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

## Partition Pruning

PostgreSQL автоматично виключає непотрібні partitions з запиту:
```sql
EXPLAIN SELECT * FROM logs WHERE created_at = '2024-01-15';
-- Scan ТІЛЬКИ logs_2024_01, інші partitions ігноруються
```

`SET enable_partition_pruning = on;` — увімкнено за замовчуванням.

---

## Практичні поради

- **Partition key** — колонка, яка найчастіше в WHERE
- **Індекси** створюються на кожній partition окремо
- **Автоматизуй** створення нових partitions (cron або pg_partman extension)
- **Не занадто багато** partitions (< 1000, оптимально 50-200)
