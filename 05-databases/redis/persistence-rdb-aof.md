# Redis Persistence — Питання для інтерв'ю

## Два механізми persistence

Redis зберігає дані **в пам'яті**, але може persist на диск двома способами:

```
           ┌─── RDB (snapshot) ──→ dump.rdb (бінарний файл)
Redis ─────┤
           └─── AOF (log) ──────→ appendonly.aof (текстовий лог)
```

---

## RDB (Redis Database Backup)

**Що:** знімок (snapshot) всіх даних на певний момент часу.

```redis
-- Ручний snapshot
SAVE            -- блокує Redis! (синхронний)
BGSAVE          -- фоновий процес (fork)

-- Автоматичний (redis.conf)
save 900 1      -- snapshot якщо 1+ змін за 900 сек
save 300 10     -- snapshot якщо 10+ змін за 300 сек
save 60 10000   -- snapshot якщо 10000+ змін за 60 сек
```

**Як працює BGSAVE:**
1. Redis виконує `fork()` — створює child process
2. Child process записує дані на диск → `dump.rdb`
3. Батьківський процес продовжує обслуговувати клієнтів
4. Copy-on-Write (COW) — ефективне використання пам'яті

**Плюси:**
- Компактний бінарний файл
- Швидке відновлення (restart)
- Ідеальний для бекапів
- Мінімальний вплив на performance

**Мінуси:**
- Втрата даних між snapshot-ами (до кількох хвилин)
- `fork()` може бути повільним на великих dataset-ах
- Подвоєння пам'яті під час fork (worst case)

---

## AOF (Append Only File)

**Що:** лог кожної write-операції. При restart Redis "відтворює" всі операції.

```redis
-- redis.conf
appendonly yes
appendfilename "appendonly.aof"

-- Fsync policy (коли записувати на диск)
appendfsync always    -- кожна операція (найбезпечніше, найповільніше)
appendfsync everysec  -- раз на секунду (рекомендовано)
appendfsync no        -- OS вирішує (найшвидше, найризикованіше)
```

**Формат AOF файлу:**
```
*3\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n
-- Означає: SET hello world
```

**AOF Rewrite** — стиснення AOF файлу:
```redis
BGREWRITEAOF
-- Замість 1000 INCR counter → одна SET counter 1000
```

```
-- Автоматичний rewrite (redis.conf)
auto-aof-rewrite-percentage 100    -- rewrite коли файл подвоївся
auto-aof-rewrite-min-size 64mb     -- мінімум 64MB
```

**Плюси:**
- Мінімальна втрата даних (максимум 1 сек з `everysec`)
- Читабельний формат
- Автоматичний rewrite

**Мінуси:**
- Більший файл ніж RDB
- Повільніший restart (replay всіх операцій)
- Повільніший throughput (fsync overhead)

---

## RDB + AOF (рекомендований підхід)

```
-- redis.conf (Redis 7.0+)
aof-use-rdb-preamble yes   -- AOF файл починається з RDB snapshot
                            -- потім append-only лог змін
```

**Переваги гібриду:**
1. Швидкий restart (RDB preamble)
2. Мінімальна втрата даних (AOF tail)
3. Компактніший файл ніж чистий AOF

---

## Порівняння

| Характеристика | RDB | AOF | RDB + AOF |
|---------------|-----|-----|-----------|
| Втрата даних | Хвилини | ~1 сек | ~1 сек |
| Розмір файлу | Малий | Великий | Середній |
| Швидкість restart | Швидко | Повільно | Швидко |
| Write overhead | Низький | Середній | Середній |
| Складність | Простий | Середній | Середній |
| Рекомендація | Бекапи | Durability | **Продакшн** |

---

## Коли що використовувати?

**Тільки RDB:**
- Кеш (втрата даних допустима)
- Бекапи (копіювати dump.rdb на інший сервер)
- Development

**Тільки AOF:**
- Критичні дані (мінімальна втрата)
- Коли потрібен audit log

**RDB + AOF (рекомендовано для продакшну):**
- Поєднує переваги обох
- Швидкий restart + мінімальна втрата даних

**Без persistence:**
```redis
save ""            -- вимкнути RDB
appendonly no      -- вимкнути AOF
```
- Чистий кеш (Redis як memcached)
- Тимчасові дані (сесії з коротким TTL)

---

## Redis 7.0+ Multi-Part AOF

```
appenddirname "appendonlydir"
-- Структура:
-- appendonlydir/
--   ├── appendonly.aof.1.base.rdb    (RDB base)
--   ├── appendonly.aof.1.incr.aof    (incremental AOF)
--   └── appendonly.aof.manifest      (manifest файл)
```

Переваги: атомарний rewrite, менше ризику corruption.

---

## Disaster Recovery

```bash
# Бекап RDB (просто скопіювати файл)
cp /var/lib/redis/dump.rdb /backup/redis-$(date +%Y%m%d).rdb

# Перевірка RDB файлу
redis-check-rdb dump.rdb

# Перевірка AOF файлу
redis-check-aof appendonly.aof

# Відновлення з бекапу
# 1. Зупинити Redis
# 2. Скопіювати dump.rdb у data directory
# 3. Запустити Redis
```

**Правило:** завжди тестуйте відновлення з бекапу. Бекап без перевірки — не бекап.
