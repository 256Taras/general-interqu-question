# CAP Теорема — Питання для інтерв'ю

## Реальний сценарій: георозподілена соцмережа

```
"У нас є георозподілена соцмережа. Є два дата-центри: Європа і США.
Трансатлантичний кабель рветься на 5 хвилин.
Користувачі в Європі ставлять лайки, користувачі в США не бачать цих лайків.
Опишіть:
1. Що трапилось з точки зору CAP
2. Яку властивість ми гарантуємо?
3. Як система повинна поводитись при відновленні зв'язку?"
```

### Відповідь

**1. Що трапилось (з точки зору CAP)**
```
Стався Partition (P) — втрата зв'язку між дата-центрами

Europe DC ──── ✗ ──── USA DC
   ↑                    ↑
 лайки              не бачать
 працюють            лайків
```

Система продовжує працювати в обох регіонах, але дані не синхронізовані.

**2. Яку властивість гарантуємо**
```
Система обрала AP (Availability + Partition Tolerance):
- Availability: обидва дата-центри відповідають на запити
- Partition Tolerance: система працює попри розрив зв'язку
- Consistency: ПОЖЕРТВУВАЛИ — дані в Європі та США різні
```

Альтернативний CP варіант:
```
Заблокувати запити на запис поки Partition не вирішиться
→ Consistency зберігається, але сервіс недоступний для записів
→ Для соцмережі це неприйнятно
```

**3. Поведінка при відновленні зв'язку**
```
Крок 1: Дата-центри виявляють відновлення зв'язку
Крок 2: Починається синхронізація — обмін накопиченими змінами
Крок 3: Conflict Resolution для конфліктних даних
Крок 4: Eventual Consistency досягнута
```

**Можливі конфлікти та їх вирішення:**
```
Сценарій: Користувач лайкнув пост з Європи, а з США видалив свій лайк
→ Vector clocks або timestamps визначають порядок
→ Для лайків: CRDT (G-Counter) — додаємо всі лайки, вирішується автоматично

Сценарій: Один і той самий коментар відредагований з обох регіонів
→ LWW (Last Write Wins) або показуємо обидві версії
```

---

## Вибір БД на основі CAP

```
"Ми будуємо:
1. Систему онлайн-голосування (вибори)
2. Чат-додаток з мільйонами повідомлень
3. Фінансову систему для мікроплатежів

Для кожного випадку поясніть, яку частину CAP обираєте і чому.
Яку БД б вибрали?"
```

### 1. Система онлайн-голосування (вибори)

```
Вибір: CP (Consistency + Partition Tolerance)
```

**Чому Consistency критична:**
- Кожен голос має бути врахований **точно один раз**
- Неприпустимі дублікати або втрати голосів
- Краще тимчасово не приймати голоси, ніж прийняти з помилкою
- Аудит вимагає точних даних

**БД: PostgreSQL з синхронною реплікацією**
```
Client → PostgreSQL Primary (sync replication) → Replica
         ↑
         Чекаємо підтвердження від Replica
         перед відповіддю клієнту
```

**Або:** CockroachDB (distributed SQL, strong consistency)

**Додаткові вимоги:**
- Serializable isolation level
- Idempotency key для кожного голосу
- Audit log всіх операцій

---

### 2. Чат-додаток з мільйонами повідомлень

```
Вибір: AP (Availability + Partition Tolerance)
```

**Чому Availability важливіша:**
- Краще показати повідомлення з затримкою, ніж не показати взагалі
- Користувачі очікують instant response
- Тимчасова неконсистентність прийнятна (повідомлення прийде на секунду пізніше)
- Write-heavy навантаження

**БД: Cassandra або ScyllaDB**
```
Client → Cassandra Cluster (eventual consistency)
         - Partition key: chat_id
         - Clustering key: timestamp
         - Replication factor: 3
         - Consistency level: ONE (для швидкості)
```

**Або:** DynamoDB (managed, auto-scaling)

**Додаткові рішення:**
- Redis для online/offline статусу та typing indicators
- Kafka для надійної доставки повідомлень між сервісами

---

### 3. Фінансова система для мікроплатежів

```
Вибір: CP (Consistency + Partition Tolerance)
```

**Чому Consistency критична:**
- Баланс має бути точним в будь-який момент
- Double spending неприпустимий
- Регуляторні вимоги (аудит, compliance)
- Краще відхилити транзакцію, ніж провести з помилкою

**БД: CockroachDB або Google Spanner**
```
Client → CockroachDB Cluster (serializable transactions)
         - Distributed transactions з TrueTime/HLC
         - Global strong consistency
         - Auto-sharding
```

**Чому не звичайний PostgreSQL:**
- Потрібна горизонтальна масштабованість (мікроплатежі = високий throughput)
- Потрібна multi-region consistency

**Додаткові вимоги:**
- Idempotency keys для кожної транзакції
- Optimistic locking для балансів
- Event sourcing для повної історії транзакцій

---

### Порівняльна таблиця

| | Голосування | Чат | Фінанси |
|---|---|---|---|
| CAP | CP | AP | CP |
| БД | PostgreSQL (sync) | Cassandra/ScyllaDB | CockroachDB/Spanner |
| Consistency | Strong | Eventual | Strong |
| Допустимий downtime | Так (краще ніж помилка) | Ні | Так (краще ніж помилка) |
| Throughput | Середній | Дуже високий | Високий |
| Latency вимоги | Не критична | Критична (< 100ms) | Середня |

---

## PACELC — розширення CAP

```
"Чим PACELC відрізняється від CAP? Коли це важливо?"
```

**CAP** відповідає на питання: що робити **під час Partition**.
**PACELC** додає: що робити **коли Partition немає**.

```
PACELC:
- Якщо є Partition (P):
    → Обираємо між Availability (A) та Consistency (C)
- Else (E), коли немає Partition:
    → Обираємо між Latency (L) та Consistency (C)
```

### Приклади

| Система | Partition | No Partition | PACELC |
|---|---|---|---|
| PostgreSQL (sync) | PC | EC | PC/EC |
| Cassandra | PA | EL | PA/EL |
| DynamoDB | PA | EL | PA/EL |
| MongoDB (default) | PA | EC | PA/EC |
| CockroachDB | PC | EC | PC/EC |

**Практичне значення:**
- Більшість часу Partition не відбувається
- Тому вибір між Latency та Consistency (EL vs EC) важливіший для повсякденної роботи
- Cassandra: швидка завжди (EL), але eventual consistency
- PostgreSQL: повільніша (EC), але завжди consistent
