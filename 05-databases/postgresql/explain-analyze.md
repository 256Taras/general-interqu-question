# EXPLAIN ANALYZE — Питання для інтерв'ю

## Як читати EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.status = 'active'
GROUP BY u.id;
```

```
HashAggregate (cost=245.00..255.00 rows=100 width=40)
              (actual time=12.5..13.2 rows=95 loops=1)
  Group Key: u.id
  -> Hash Left Join (cost=30.00..230.00 rows=1000 width=36)
                     (actual time=2.1..10.8 rows=980 loops=1)
     Hash Cond: (o.user_id = u.id)
     -> Seq Scan on orders o (cost=0.00..150.00 rows=5000 width=8)
                              (actual time=0.01..3.5 rows=5000 loops=1)
     -> Hash (cost=25.00..25.00 rows=100 width=36)
              (actual time=1.8..1.8 rows=95 loops=1)
        -> Index Scan using idx_users_status on users u
           (cost=0.28..25.00 rows=100 width=36)
           (actual time=0.02..1.5 rows=95 loops=1)
           Filter: (status = 'active')
Planning Time: 0.3 ms
Execution Time: 13.8 ms
```

---

## Ключові метрики

**cost=startup..total** — оцінка планувальника (не реальний час)
- startup — час до першого рядка
- total — час до останнього рядка

**rows** — оцінка кількості рядків (estimated vs actual)
Якщо estimated сильно відрізняється від actual → потрібен `ANALYZE` для оновлення статистики.

**width** — середній розмір рядка у байтах

**actual time** — реальний час (мс)

**loops** — скільки разів виконувався цей вузол

---

## Scan Types

### Seq Scan — послідовне сканування всієї таблиці
```
Seq Scan on users (cost=0.00..150.00 rows=5000)
```
Нормально для маленьких таблиць або коли потрібно > 15% рядків.

### Index Scan — пошук по індексу + heap
```
Index Scan using idx_email on users (cost=0.28..8.30 rows=1)
  Index Cond: (email = 'john@example.com')
```
Для маленької кількості рядків.

### Index Only Scan — тільки індекс (covering)
```
Index Only Scan using idx_email_name on users (cost=0.28..4.30 rows=1)
```
Найшвидший — не читає таблицю.

### Bitmap Index Scan — будує bitmap → batch read
```
Bitmap Heap Scan on users
  -> Bitmap Index Scan on idx_status
```
Для середньої кількості рядків.

---

## Join Methods

### Nested Loop
```
Nested Loop (cost=0.28..100.00)
  -> Seq Scan on users
  -> Index Scan on orders (loops=100)
```
Для маленьких таблиць або коли одна сторона дуже мала. O(N * M).

### Hash Join
```
Hash Join
  -> Seq Scan on orders (build hash table)
  -> Seq Scan on users (probe)
```
Для великих таблиць без індексу. Потребує пам'ять для hash table.

### Merge Join
```
Merge Join
  -> Sort on users.id
  -> Sort on orders.user_id
```
Коли обидві сторони відсортовані (або є індекс). Ефективний для великих datasets.

---

## Знаходження bottlenecks

1. **Seq Scan на великій таблиці з WHERE** → додай індекс
2. **rows estimate сильно неправильний** → `ANALYZE tablename`
3. **Sort використовує disk** (Sort Method: external merge) → збільш `work_mem`
4. **Nested Loop з великим loops** → можливо потрібен Hash Join (додай індекс)
5. **Bitmap Heap Scan з великим Recheck** → індекс не selective enough

---

## Корисні прапорці EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;
-- BUFFERS — показує disk I/O (shared hit/read)
-- shared hit = з кешу, shared read = з диску

EXPLAIN (ANALYZE, VERBOSE) SELECT ...;
-- VERBOSE — деталі по колонках

EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;
-- JSON формат — для інструментів візуалізації (explain.dalibo.com)
```
