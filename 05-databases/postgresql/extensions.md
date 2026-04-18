# PostgreSQL Extensions — Питання для інтерв'ю

## pg_stat_statements
Статистика запитів — знайти повільні запити:
```sql
CREATE EXTENSION pg_stat_statements;
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

## pgcrypto
Криптографічні функції:
```sql
CREATE EXTENSION pgcrypto;
-- Хешування паролів
SELECT crypt('password123', gen_salt('bf', 12));
-- UUID
SELECT gen_random_uuid();
-- Шифрування
SELECT pgp_sym_encrypt('secret data', 'encryption_key');
```

## pg_trgm (Trigram)
Нечіткий пошук (similarity, fuzzy matching):
```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);
-- Пошук по схожості
SELECT * FROM users WHERE name % 'Jhon'; -- знайде "John"
SELECT similarity('John', 'Jhon'); -- 0.5
```

## PostGIS
Геопросторові дані:
```sql
CREATE EXTENSION postgis;
SELECT ST_Distance(
  ST_MakePoint(30.5234, 50.4501)::geography, -- Kyiv
  ST_MakePoint(24.0297, 49.8397)::geography   -- Lviv
); -- відстань у метрах
```

## TimescaleDB
Time-series дані (IoT, метрики, логи):
```sql
CREATE EXTENSION timescaledb;
SELECT create_hypertable('metrics', 'time');
-- Автоматичне partitioning по часу, compression, retention policies
```

## Citus
Distributed PostgreSQL (horizontal scaling):
- Sharding таблиць по кластеру
- Distributed queries
- Columnar storage

## pgvector
Векторний пошук (AI/ML embeddings):
```sql
CREATE EXTENSION vector;
CREATE TABLE items (id SERIAL, embedding vector(1536)); -- OpenAI dimensions
CREATE INDEX ON items USING ivfflat (embedding vector_cosine_ops);
SELECT * FROM items ORDER BY embedding <=> '[0.1, 0.2, ...]' LIMIT 10;
```
