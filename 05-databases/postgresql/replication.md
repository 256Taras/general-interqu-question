# PostgreSQL Replication — Питання для інтерв'ю

## Streaming Replication (фізична)

Primary відправляє WAL (Write-Ahead Log) records на Replica:
```
Primary (read/write) ──WAL stream──→ Replica 1 (read-only)
                                  ──→ Replica 2 (read-only)
```

- Replica є точною копією Primary
- Async за замовчуванням (може бути sync)
- Replica — read-only (hot standby)

---

## Logical Replication

Реплікація на рівні змін (INSERT/UPDATE/DELETE), не WAL:
```sql
-- На publisher
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- На subscriber
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=primary dbname=mydb'
  PUBLICATION my_pub;
```

**Переваги над streaming:**
- Selective (окремі таблиці)
- Різні версії PostgreSQL
- Різна схема (додаткові колонки)
- Subscriber може мати свої записи

---

## Read Replicas — масштабування reads

```
Application
  ├── Writes → Primary
  └── Reads  → Load Balancer → Replica 1
                             → Replica 2
                             → Replica 3
```

**Replication lag:** replica може відставати від primary (eventual consistency). Для критичних reads — читай з primary.

---

## Failover / Switchover

**Failover** — автоматичне переключення при crash primary:
```
Primary (down!) → Replica 1 промоутиться → New Primary
                  Replica 2 → перенаправляється на New Primary
```

**Інструменти:** Patroni, repmgr, pg_auto_failover, AWS RDS Multi-AZ.

---

## Connection Pooling

### PgBouncer
Proxy між додатком і PostgreSQL:
```
App (100 connections) → PgBouncer (20 connections) → PostgreSQL
```

**Modes:**
- **Session** — одне з'єднання на сесію клієнта
- **Transaction** — з'єднання тільки на час транзакції (найефективніший)
- **Statement** — з'єднання на один запит

### Application-level pool
```javascript
const pool = new pg.Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

**Чому важливо:**
- PostgreSQL max_connections зазвичай 100-200
- Кожне з'єднання = ~10MB RAM
- Pool перевикористовує з'єднання

---

## High Availability архітектура

```
                  ┌─────────────┐
                  │ HAProxy/    │
                  │ PgBouncer   │
                  └──────┬──────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
        ┌─────┴──┐ ┌────┴───┐ ┌───┴────┐
        │Primary │ │Replica1│ │Replica2│
        │(write) │ │(read)  │ │(read)  │
        └────────┘ └────────┘ └────────┘
              │
        ┌─────┴──┐
        │Patroni │  ← автоматичний failover
        └────────┘
```

- **RPO** (Recovery Point Objective) — скільки даних можемо втратити
- **RTO** (Recovery Time Objective) — як швидко відновимось
- Synchronous replication: RPO = 0 (no data loss), але повільніше
- Async replication: RPO > 0 (можлива втрата), але швидше
