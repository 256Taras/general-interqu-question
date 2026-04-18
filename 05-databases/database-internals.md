# Database Internals — питання для співбесіди

## Page & Storage

### 1. Чому БД зберігає дані посторінково (page), а не окремими рядками? [Junior]

**Підказка:** Подумай про фізику диска — яку мінімальну одиницю він вміє читати?

**Відповідь:** Диск (SSD/HDD) не вміє читати окремий байт — тільки блоками (мін. 4KB). Тому БД організує дані у pages фіксованого розміру (8KB у Postgres), щоб одне читання з диска обслуговувало багато рядків. Без сторінок кожен SELECT — це окреме звернення до диска. Зі сторінками одне читання кешує ~100 рядків у RAM (buffer pool), і наступні запити до тих самих рядків не чіпають диск взагалі.

---

### 2. Що таке slotted page і навіщо потрібен slot array? [Junior]

**Підказка:** Подумай — рядки різного розміру. Як знайти конкретний рядок всередині 8KB?

**Відповідь:** Slotted page — структура сторінки де Header + slot array ростуть зліва направо, а tuples ростуть справа наліво, вільне місце посередині. Slot array — це масив (offset, size) пар, де кожен слот вказує де фізично лежить tuple. Потрібен тому що рядки різного розміру (VARCHAR), і без слотів неможливо обчислити позицію n-го рядка. Зовнішні структури (індекси) посилаються на запис як (page_id, slot_number) — це TID у Postgres. Коли tuple переміщується всередині сторінки (наприклад при UPDATE) — оновлюється тільки offset у слоті, а TID залишається тим самим, тому індекси не ламаються.

---

### 3. Що станеться якщо рядок не влазить у сторінку 8KB? [Middle]

**Підказка:** Postgres має спеціальний механізм для oversized attributes...

**Відповідь:** Postgres використовує TOAST (The Oversized-Attribute Storage Technique). Крок 1: спроба стиснути значення (LZ-компресія). Якщо все одно не влазить — розрізає великі поля на ~2KB чанки і зберігає їх у прихованій TOAST-таблиці (pg_toast_XXXXX). В основному tuple залишається pointer (18 байт). Ключова перевага: SELECT title FROM articles не чіпає TOAST — якщо ти не запитуєш великі поля, їх навіть не треба читати з диска. У MySQL InnoDB аналогічно: overflow pages у тому ж tablespace з pointer'ом у основному записі.

---

### 4. Поясни різницю між row-oriented і column-oriented зберіганням. Коли що краще? [Middle]

**Підказка:** Подумай який запит ти робиш частіше: SELECT * WHERE id=... чи SELECT AVG(salary)...

**Відповідь:** Row-oriented (Postgres, MySQL): всі поля одного рядка зберігаються поряд. Оптимально для OLTP — SELECT * WHERE id=X читає одну сторінку і має весь рядок. Column-oriented (ClickHouse, DuckDB, BigQuery): всі значення однієї колонки зберігаються поряд. Оптимально для OLAP — SELECT AVG(salary) читає тільки колонку salary, не чіпає name, address тощо. Плюс колоночне зберігання краще стискається (однакові типи поряд). TiDB поєднує обидва підходи: TiKV (row) для OLTP + TiFlash (column) для OLAP — це HTAP.

---

## Buffer Pool

### 5. Що таке Buffer Pool і навіщо він потрібен? Що відбувається при page fault? [Middle]

**Підказка:** Де живуть сторінки коли БД з ними працює — на диску чи в RAM?

**Відповідь:** Buffer Pool — це кеш сторінок у RAM. БД ніколи не працює з даними на диску напряму — спочатку сторінка має потрапити в buffer pool. Page fault: запит потребує сторінку якої немає в buffer pool → читаємо з диска в вільний frame → додаємо запис у page table (map page_id → frame). Page table — це hash map для швидкого пошуку. Кожна сторінка має pin count (скільки транзакцій зараз її використовують) і dirty flag (чи була змінена — якщо так, при eviction треба записати на диск). Eviction policy — зазвичай LRU або clock algorithm. У Postgres це shared_buffers, у MySQL — innodb_buffer_pool_size.

---

## Index

### 6. Поясни різницю між clustered і unclustered індексом. [Middle]

**Підказка:** Чи збігається фізичний порядок записів на диску з порядком ключа індексу?

**Відповідь:** Clustered: фізичний порядок рядків на диску збігається з порядком ключа. Може бути тільки один на таблицю. В InnoDB таблиця IS the clustered index по primary key — дані зберігаються в листках B+Tree. Unclustered: індекс містить pointer'и (TID) на рядки в heap, які лежать у довільному порядку. Може бути багато на таблицю. В Postgres всі індекси unclustered — таблиця це heap, індекси окремо вказують на (page, slot). Наслідок: range scan по clustered індексу = sequential read (швидко). Range scan по unclustered = random I/O в різні сторінки heap (повільно).

---

### 7. Що таке Bloom filter? Які гарантії він дає і де застосовується? [Middle]

**Підказка:** Масив біт + кілька хеш-функцій. Які два можливих результати перевірки?

**Відповідь:** Bloom filter — імовірнісна структура даних: масив біт + k хеш-функцій. Додавання: обчислити k хешів ключа, поставити відповідні біти в 1. Перевірка: обчислити k хешів, перевірити чи всі біти = 1. Якщо хоч один = 0 → ключа ТОЧНО НЕМАЄ. Якщо всі = 1 → МОЖЛИВО Є (false positive). False negative неможливий. Типова конфігурація: 10 біт/ключ, 7 хешів → FPR ~1%. Використання в LSM: кожен SST файл має Bloom filter у RAM. При GET перевіряємо filter замість читання файлу з диска. Також: Chrome (malicious URLs), CDN (cache check), Bitcoin (SPV clients). Обмеження: не можна видалити ключ (рішення — counting Bloom filter).

---

## LSM Tree

### 8. Опиши write path в LSM-tree. Чому LSM оптимізований під записи? [Senior]

**Підказка:** WAL → MemTable → flush → SST → compaction

**Відповідь:** Write path: 1) Запис у WAL (append на диск — для crash recovery). 2) Запис у MemTable (sorted структура в RAM, зазвичай skip list). 3) Коли MemTable заповнюється (~64-128MB) — заморожується, flush'ається на диск як SST файл (Sorted String Table) на L0. 4) Новий WAL + нова MemTable створюються. 5) Періодично compaction мерджить SST файли вниз по рівнях (L0→L1→...→L6). Чому швидко: всі записи — sequential append (WAL + flush). Немає random I/O як у B-Tree де треба знайти правильну сторінку, можливо зробити split. Trade-off: reads повільніші бо треба перевіряти кілька рівнів (Bloom filter компенсує).

---

### 9. Що таке compaction і навіщо він потрібен? Що таке write amplification? [Senior]

**Підказка:** Що буде якщо ніколи не мерджити SST файли?

**Відповідь:** Compaction — процес злиття (merge sort) кількох SST файлів у один більший. Навіщо: без compaction L0 накопичив би тисячи SST файлів, кожен GET мусив би перевіряти їх усі. Compaction зменшує кількість файлів, видаляє дублікати (залишає найновішу версію), прибирає tombstones (маркери DELETE). На L1+ файли non-overlapping (один ключ може бути максимум в одному файлі на рівень). Write amplification — один запис реально перезаписується на диск багато разів (WAL + flush + compaction L0→L1, L1→L2, ...). Типова write amplification в RocksDB: 10-30×. Це ціна за швидкий write + добру компресію. B-Tree має нижчу write amplification (~2-5×), але повільніші writes.

---

### 10. Що таке skip list і чому його обрали для MemTable замість B-Tree? [Senior]

**Підказка:** Подумай про concurrent writes від багатьох тредів...

**Відповідь:** Skip list — кілька рівнів відсортованих зв'язних списків. Верхні рівні = "експреси" з меншою кількістю елементів. Пошук/вставка O(log n). При вставці "монетка" визначає на скільки рівнів додається елемент. Чому для MemTable: 1) Lock-free конкурентність — вставка = зміна кількох pointer'ів через CAS, багато тредів можуть писати одночасно. B-Tree потребує синхронізації при split'ах. 2) Простота реалізації (~200 рядків vs тисячи для concurrent B-Tree). 3) Завжди відсортований — flush на диск = лінійна ітерація по L0, без додаткового сортування. Використовується в: RocksDB/LevelDB MemTable, Redis sorted sets, CockroachDB.

---

## VACUUM & MVCC

### 11. Навіщо Postgres потрібен VACUUM? Що буде якщо він не працює? [Senior]

**Підказка:** Як Postgres робить UPDATE? Чи видаляє він старий рядок?

**Відповідь:** Postgres MVCC: UPDATE не змінює рядок на місці, а створює нову версію tuple + позначає стару як мертву. DELETE теж тільки позначає — фізично дані залишаються. Без VACUUM мертві tuples накопичуються → table bloat (файл росте нескінченно), index bloat, повільніші seq scan. VACUUM: проходить по сторінках, звільняє місце від мертвих tuples для перевикористання (але НЕ повертає місце ОС). VACUUM FULL: перепакує таблицю щільно, повертає місце ОС, але бере ACCESS EXCLUSIVE lock — блокує все. Найстрашніше без VACUUM: transaction ID wraparound — 32-біт XID вичерпується за ~50 днів при 1000 tx/sec, і Postgres відмовляється приймати нові транзакції. Autovacuum за замовчуванням спрацьовує при 20% мертвих рядків — для великих таблиць це занадто пізно.

---

## Concurrency

### 12. Поясни різницю між Dirty Read, Lost Update, Unrepeatable Read, Phantom Read. [Middle]

**Підказка:** WR, WW, RW конфлікти + INSERT/DELETE...

**Відповідь:** Lost Update (WW): дві транзакції читають X=100, обидві модифікують, хто пише останнім — перезаписує першого. Приклад: два зняття з банкомата одночасно. Dirty Read (WR): транзакція читає незакомічене значення іншої, яка потім робить ROLLBACK — прочитане ніколи не існувало. Unrepeatable Read (RW): той самий SELECT в одній транзакції повертає РІЗНІ значення ТОГО САМОГО рядка, бо інша транзакція зробила UPDATE + COMMIT між ними. Phantom Read: той самий SELECT WHERE... повертає РІЗНУ КІЛЬКІСТЬ рядків — з'явився новий рядок через INSERT іншої транзакції. Різниця unrepeatable vs phantom: перше про зміну існуючого рядка, друге про появу/зникнення рядків.

---

### 13. Чим pessimistic locking (2PL) відрізняється від optimistic (MVCC)? [Senior]

**Підказка:** Блокувати заздалегідь vs дозволити працювати і перевірити при commit...

**Відповідь:** Pessimistic (2PL): перед операцією бери lock (Shared для читання, Exclusive для запису). Читачі блокують письменників і навпаки. Deadlock'и можливі — БД детектує і вбиває одну транзакцію. Приклад: MySQL InnoDB row-level locks, SELECT FOR UPDATE. Optimistic (MVCC): кожна транзакція працює зі своїм snapshot даних. Читачі НІКОЛИ не блокують письменників. При commit перевірка: якщо хтось уже змінив ті ж рядки — abort + retry. Приклад: Postgres (snapshot isolation), CockroachDB (SSI). Де що: high-contention workloads (багато конфліктів на тих самих рядках) → pessimistic ефективніше (менше retries). Read-heavy або low-contention → optimistic ефективніше (немає блокувань). Postgres дефолт: READ COMMITTED + MVCC. CockroachDB: SERIALIZABLE + MVCC.

---

### 14. Як вирішити проблему lost update в Postgres при read-modify-write? [Senior]

**Підказка:** Є мінімум 3 способи...

**Відповідь:** Спосіб 1: Атомарний UPDATE — `UPDATE users SET counter = counter + 1 WHERE id = 1` (БД робить read+modify+write атомарно, без вікна для конфлікту). Спосіб 2: SELECT FOR UPDATE — бере exclusive lock на рядок перед читанням, інші транзакції чекають. Спосіб 3: REPEATABLE READ + retry — якщо інша транзакція змінила рядок, commit зафейлиться з serialization error → ловимо помилку і повторюємо транзакцію. Спосіб 4: Optimistic locking на рівні додатку — `UPDATE ... WHERE id=1 AND version=5`, якщо 0 affected rows → хтось уже оновив → retry. Найпростіше: атомарний UPDATE. Якщо треба складна логіка між read і write — SELECT FOR UPDATE.

---

## Architecture

### 15. Опиши архітектуру Neon (serverless Postgres). В чому ключова інновація? [Senior]

**Підказка:** Separation of compute and storage. Де живуть Postgres процеси, де WAL?

**Відповідь:** Neon розділяє compute і storage. Compute: звичайні Postgres процеси (можна scale-to-zero, instant branching). Safekeepers (3 ноди): приймають WAL stream від Postgres, забезпечують durability через консенсус (кворум 2/3). Pageserver: реконструює сторінки з WAL on-demand, віддає їх Postgres'у. Object Storage (S3): холодне довгострокове сховище. Postgres думає що пише на локальний диск, але WAL летить у safekeepers. Ключова інновація: branching як у git (створити копію бази за мілісекунди без фізичного копіювання), scale-to-zero (вимкнути compute коли немає запитів), point-in-time queries (pageserver зберігає історію змін). Databricks купив Neon за $1B — це показує цінність підходу.

---

### 16. Порівняй B-Tree vs LSM-tree storage engine. Коли що обрати? [Senior]

**Підказка:** Write speed vs read speed, space amplification, write amplification...

**Відповідь:** B-Tree: in-place updates, дані відсортовані в сторінках, random I/O при записі. Reads швидкі (одне дерево). Writes повільніші (знайти сторінку + можливий split). Write amplification ~2-5×. Space amplification ~1.5× (bloat). Краще для: OLTP з read-heavy, складні запити, range scans. Приклади: Postgres, MySQL InnoDB. LSM-tree: append-only writes, дані у відсортованих SST файлах на рівнях. Writes дуже швидкі (sequential). Reads повільніші (перевірка кількох рівнів, компенсується Bloom filters). Write amplification ~10-30× (compaction). Space amplification ~1.1-1.3× (добра компресія). Краще для: write-heavy (логи, метрики, IoT, черги, time-series). Приклади: RocksDB, CockroachDB Pebble, Cassandra.
