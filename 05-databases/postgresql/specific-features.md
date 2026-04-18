# PostgreSQL Specific Features — Питання для інтерв'ю

## JSONB

```sql
CREATE TABLE events (
  id SERIAL PRIMARY KEY,
  data JSONB NOT NULL
);

INSERT INTO events (data) VALUES ('{"type": "click", "page": "/home", "meta": {"browser": "Chrome"}}');

-- Запити
SELECT data->>'type' FROM events;                    -- text: "click"
SELECT data->'meta'->>'browser' FROM events;         -- text: "Chrome"
SELECT * FROM events WHERE data @> '{"type": "click"}'; -- containment
SELECT * FROM events WHERE data ? 'type';            -- has key
SELECT * FROM events WHERE data->>'type' = 'click';  -- equality

-- Індекси для JSONB
CREATE INDEX idx_events_data ON events USING GIN (data);           -- для @>, ?, ?|, ?&
CREATE INDEX idx_events_type ON events ((data->>'type'));           -- для конкретного поля
CREATE INDEX idx_events_data_path ON events USING GIN (data jsonb_path_ops); -- для @>
```

**JSON vs JSONB:**
- JSON — зберігає як текст, зберігає форматування
- JSONB — бінарний формат, швидші запити, підтримує індекси

---

## Arrays

```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  tags TEXT[] DEFAULT '{}'
);

INSERT INTO users (tags) VALUES (ARRAY['admin', 'active']);

SELECT * FROM users WHERE 'admin' = ANY(tags);     -- містить елемент
SELECT * FROM users WHERE tags @> ARRAY['admin'];  -- contains
SELECT * FROM users WHERE tags && ARRAY['admin', 'vip']; -- overlap (будь-який)

-- GIN індекс для масивів
CREATE INDEX idx_users_tags ON users USING GIN (tags);
```

---

## Full-Text Search

```sql
-- tsvector — документ, tsquery — запит
SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
-- 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2

-- Пошук
SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'quick & fox');

-- Stored generated column + GIN index (production)
ALTER TABLE articles ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))) STORED;

CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

SELECT * FROM articles WHERE search_vector @@ to_tsquery('english', 'quick & fox');
```

---

## Row Level Security (RLS)

```sql
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_documents ON documents
  USING (owner_id = current_setting('app.user_id')::int);

-- Кожен користувач бачить тільки свої документи
SET app.user_id = '42';
SELECT * FROM documents; -- тільки документи user 42
```

---

## LISTEN / NOTIFY

Real-time нотифікації через PostgreSQL:
```sql
-- Listener
LISTEN new_order;

-- Sender
NOTIFY new_order, '{"order_id": 123, "total": 99.99}';
```

```javascript
// Node.js
const client = new pg.Client();
await client.connect();
client.on("notification", (msg) => {
  console.log(msg.channel, JSON.parse(msg.payload));
});
await client.query("LISTEN new_order");
```

---

## Advisory Locks

Application-level locks через PostgreSQL:
```sql
-- Session-level lock
SELECT pg_advisory_lock(12345);         -- блокує
SELECT pg_advisory_unlock(12345);       -- розблоковує

-- Transaction-level lock (автоматично розблоковується при COMMIT)
SELECT pg_advisory_xact_lock(12345);

-- Try lock (non-blocking)
SELECT pg_try_advisory_lock(12345);     -- повертає true/false
```

**Use case:** cron jobs — гарантувати що тільки один інстанс виконує задачу.

---

## Generated Columns

```sql
CREATE TABLE products (
  price NUMERIC,
  tax_rate NUMERIC DEFAULT 0.2,
  total NUMERIC GENERATED ALWAYS AS (price * (1 + tax_rate)) STORED
);
-- total обчислюється автоматично при INSERT/UPDATE
```

---

## pg_stat_statements

Збирає статистику виконання запитів:
```sql
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
-- Топ-10 найповільніших запитів
```

---

## UUID

```sql
-- uuid-ossp extension
CREATE EXTENSION "uuid-ossp";
SELECT uuid_generate_v4(); -- random UUID

-- Або gen_random_uuid() (вбудований у PG 13+)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);
```

**UUID v4 vs ULID vs nanoid:**
- UUID v4: стандартний, random, поганий для B-Tree (не sequential)
- UUID v7: sequential (timestamp-based), краще для індексів
- ULID: sequential, sortable, compact
