# Redis — 50 Питань для співбесіди з детальними відповідями

> Написано так, щоб зрозумів навіть новачок.
> Кожна відповідь пояснює ЩО, ЧОМУ і КОЛИ — з прикладами та аналогіями.

---

## Частина 1: Базові питання (Junior)

---

### 1. Що таке Redis?

**Коротко:** Redis — це база даних, яка зберігає дані в оперативній пам'яті (RAM), а не на диску.

**Детально:**
Уяви два типи сховищ:
- **PostgreSQL** — це шафа з документами. Надійно, але щоб дістати потрібний документ, треба відчинити шафу, знайти папку, дістати файл. Це повільно.
- **Redis** — це стіл перед тобою. Все під рукою, миттєво. Але місця на столі менше, ніж у шафі.

```
PostgreSQL (диск):  ~1-10 мс на запит
Redis (RAM):        ~0.1-0.5 мс на запит (в 10-100 разів швидше)
```

Redis зберігає дані у форматі "ключ → значення" (key-value):
```redis
SET name "John"        -- зберегти: ключ "name", значення "John"
GET name               -- отримати: → "John"
```

**Де використовують:**
- Кеш (щоб не ходити в базу кожен раз)
- Сесії користувачів (хто залогінений)
- Черги задач (обробка email, повідомлень)
- Лічильники (перегляди, лайки)
- Рейтинги та лідерборди

---

### 2. Чим Redis відрізняється від PostgreSQL/MySQL?

| | Redis | PostgreSQL/MySQL |
|--|-------|-----------------|
| Де дані | В оперативній пам'яті (RAM) | На жорсткому диску |
| Швидкість | Дуже швидко (~100K операцій/сек) | Повільніше (~1-10K операцій/сек) |
| Структура | Key-Value (ключ → значення) | Таблиці з рядками і колонками (SQL) |
| Зв'язки | Немає JOIN, foreign keys | Є повна підтримка зв'язків |
| Надійність | Може втратити дані (секунди) | ACID — дані не втрачаються |
| Об'єм | Обмежений розміром RAM | Терабайти на диску |

**Головне правило:** вони не конкуренти, а партнери.
- PostgreSQL — "джерело правди", зберігає все надійно
- Redis — "швидкий помічник", кешує найчастіші запити

```
Користувач → Redis (є в кеші?) → Так → відповідь за 0.1 мс
                                → Ні  → PostgreSQL → Redis (зберегти) → відповідь
```

---

### 3. Чому Redis такий швидкий? (питають завжди!)

5 причин:

**1. Дані в RAM (оперативній пам'яті)**
```
Диск (HDD):  ~10 мс     — це як їхати машиною
Диск (SSD):  ~0.1 мс    — це як їхати потягом
RAM:         ~0.0001 мс  — це як телепортація
```

**2. Один потік для команд (single-threaded)**
Звучить як мінус, але це плюс! Уяви касу в магазині:
- Одна каса, один касир — обслуговує ДУЖЕ швидко, без плутанини
- Багато кас — потрібна координація, хто що робить (locks, context switch)
Redis обробляє одну команду за ~1 мікросекунду, тому один потік встигає.

**3. I/O multiplexing (epoll/kqueue)**
Один потік може слухати тисячі клієнтів одночасно через спеціальний механізм ОС.
Аналогія: офіціант, який приймає замовлення від усіх столиків, а не чекає поки один доїсть.

**4. Оптимізовані структури даних**
Redis використовує спеціальні формати для малих даних (ziplist, intset), які займають менше пам'яті та працюють швидше.

**5. Простий протокол (RESP)**
Формат спілкування між клієнтом і Redis — текстовий та мінімальний, парситься миттєво.

---

### 4. Що таке TTL?

**TTL (Time-To-Live)** — час життя ключа. Після закінчення Redis автоматично видаляє ключ.

```redis
SET session:abc "user-data"    -- зберегти сесію
EXPIRE session:abc 3600        -- видалити через 3600 секунд (1 годину)

-- Або одразу при створенні:
SETEX session:abc 3600 "user-data"   -- SET + EXPIRE в одній команді

-- Перевірити скільки залишилось:
TTL session:abc                -- → 3547 (секунд до видалення)

-- Зняти TTL (зробити ключ постійним):
PERSIST session:abc
```

**Навіщо TTL?**
- Кеш: дані актуальні 1 годину, потім оновити з бази
- Сесії: автоматичний logout через 24 години
- Rate limiting: лічильник скидається щохвилини
- Тимчасові locks: автоматичне розблокування

**Правило:** завжди ставити TTL на кеш. Кеш без TTL = пам'ять, яка ніколи не звільниться.

---

### 5. Як Redis зберігає дані, якщо він in-memory?

Два механізми збереження на диск:

```
           ┌── RDB (фото) ──→ dump.rdb
Redis ─────┤
           └── AOF (щоденник) ──→ appendonly.aof
```

**RDB (snapshot)** — як фото кімнати.
Redis робить "знімок" всіх даних, наприклад, кожні 5 хвилин.
- ✅ Компактний файл, швидке відновлення
- ❌ Можна втратити до 5 хвилин даних

**AOF (Append Only File)** — як щоденник.
Redis записує кожну операцію: "SET name John", "DEL name", ...
При перезапуску — "перечитує щоденник" і відтворює всі дані.
- ✅ Втрата максимум 1 секунда
- ❌ Файл більший, перезапуск повільніший

**Для продакшну:** RDB + AOF разом (швидкий restart від RDB + мінімальна втрата від AOF).

---

### 6. Які структури даних підтримує Redis?

Redis — це не просто "ключ → рядок". Він підтримує 8 типів:

| Структура | Аналогія | Приклад використання |
|-----------|----------|---------------------|
| **String** | Звичайна змінна | Кеш, лічильники, сесії |
| **List** | Масив (двозв'язний список) | Черги, стрічка новин |
| **Hash** | Об'єкт / словник | Профіль користувача |
| **Set** | Множина (без дублів) | Теги, унікальні відвідувачі |
| **Sorted Set** | Множина з рейтингом | Лідерборди, рейтинги |
| **HyperLogLog** | Лічильник унікальних | Підрахунок унікальних IP |
| **Bitmap** | Масив бітів (0/1) | Щоденна активність |
| **Stream** | Лог подій (як Kafka) | Event sourcing, черги |

```redis
-- String: простий кеш
SET user:1:name "John"

-- Hash: об'єкт з полями
HSET user:1 name "John" age "30" email "john@example.com"

-- List: черга завдань
LPUSH queue:emails "send welcome email"

-- Set: унікальні теги
SADD tags:post:1 "redis" "nodejs" "backend"

-- Sorted Set: рейтинг з очками
ZADD leaderboard 100 "player:1"
ZADD leaderboard 250 "player:2"
```

---

### 7. Коли використати Hash замість String для зберігання об'єкта?

**String підхід** — зберігаємо весь об'єкт як JSON:
```redis
SET user:1 '{"name":"John","age":30,"email":"john@example.com"}'

-- Щоб змінити тільки email, треба:
-- 1. GET → десеріалізувати JSON
-- 2. Змінити поле
-- 3. SET → серіалізувати назад
```

**Hash підхід** — кожне поле окремо:
```redis
HSET user:1 name "John" age "30" email "john@example.com"

-- Змінити тільки email:
HSET user:1 email "new@example.com"   -- одна команда!

-- Отримати одне поле:
HGET user:1 name   -- → "John" (не треба тягнути весь об'єкт)
```

**Правило:**
- Об'єкт часто оновлюється частково → **Hash**
- Об'єкт завжди читається/пишеться цілком → **String** (простіше)

---

### 8. Чим Set відрізняється від Sorted Set?

**Set** — мішок з унікальними предметами. Порядку немає.
```redis
SADD fruits "apple" "banana" "apple"
SMEMBERS fruits   -- → "banana", "apple" (порядок випадковий, дублів немає)
```

**Sorted Set** — ті самі унікальні предмети, але кожен має "оцінку" (score), і вони відсортовані.
```redis
ZADD exam 85 "Alice" 92 "Bob" 78 "Charlie"
ZREVRANGE exam 0 -1 WITHSCORES
-- → Bob(92), Alice(85), Charlie(78) — відсортовано за score
```

**Аналогія:**
- Set — список гостей на вечірці (хто прийшов, без порядку)
- Sorted Set — таблиця результатів іспиту (кожен має бал, є рейтинг)

---

### 9. Що таке HyperLogLog і навіщо він потрібен?

**Проблема:** порахувати скільки унікальних користувачів відвідали сайт за місяць.

**Рішення через Set:**
```redis
SADD visitors:january "user:1" "user:2" ... "user:5000000"
-- 5 мільйонів записів ≈ 300+ МБ пам'яті!
```

**Рішення через HyperLogLog:**
```redis
PFADD visitors:january "user:1" "user:2" ... "user:5000000"
PFCOUNT visitors:january   -- → ≈ 5,000,000 (похибка ~0.81%)
-- Завжди займає 12 КБ пам'яті! Навіть для мільярдів елементів.
```

**Компроміс:** неточний підрахунок (~0.81% похибка), але в 25,000 разів менше пам'яті.
Ідеально для аналітики, де точність "±1%" допустима.

---

### 10. Як реалізувати просту чергу на Redis?

Redis List працює як черга (FIFO — First In, First Out):

```
Виробник (Producer)             Споживач (Consumer)
     │                               │
     └──→ LPUSH (додати зліва) ──→ List ──→ BRPOP (витягнути справа) ──→
```

```javascript
// Producer: додає задачу в чергу
await redis.lpush("queue:emails", JSON.stringify({
  to: "user@example.com",
  subject: "Welcome!",
}));

// Consumer: чекає і обробляє задачі
while (true) {
  // BRPOP блокує і чекає до 30 секунд поки з'явиться задача
  const [, job] = await redis.brpop("queue:emails", 30);
  if (job) {
    const email = JSON.parse(job);
    await sendEmail(email);
  }
}
```

**BRPOP** — "blocking right pop". Не споживає CPU в очікуванні (на відміну від циклу з RPOP).

---

### 11. Що таке Cache-Aside (Lazy Loading)?

Найпопулярніший патерн кешування. Логіка проста:

```
Запит "дай мені юзера #1":

1. Перевірити кеш Redis
   ├── Є (cache hit) → повернути з кешу (швидко!)
   └── Немає (cache miss) →
       2. Запитати з бази PostgreSQL
       3. Зберегти в Redis з TTL
       4. Повернути результат
```

```javascript
async function getUser(userId) {
  // 1. Спочатку в кеш
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached); // cache hit!

  // 2. Якщо немає — з бази
  const user = await db.query("SELECT * FROM users WHERE id = $1", [userId]);

  // 3. Зберегти в кеш на 1 годину
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));

  return user;
}
```

**"Lazy"** — тому що кешується тільки те, що реально запитують. Не витрачаємо пам'ять на те, що нікому не потрібно.

---

### 12. Чому при оновленні краще видаляти кеш, а не оновлювати?

**Проблема з оновленням кешу (race condition):**
```
Потік A: UPDATE user SET name="Alice"
Потік B: UPDATE user SET name="Bob"
Потік B: SET cache:user = "Bob"      ← B оновив кеш
Потік A: SET cache:user = "Alice"    ← A оновив кеш ПІСЛЯ B!
-- В базі: "Bob", В кеші: "Alice" — НЕУЗГОДЖЕНІСТЬ!
```

**Рішення — видаляти (invalidate):**
```
Потік A: UPDATE user SET name="Alice" → DEL cache:user
Потік B: UPDATE user SET name="Bob"   → DEL cache:user
-- Наступний GET → cache miss → зчитає актуальні дані з бази
```

Видалення безпечне, тому що при miss Redis просто завантажить свіжі дані з бази.

---

## Частина 2: Середні питання (Middle)

---

### 13. Чому Redis single-threaded і як він обробляє тисячі з'єднань?

**"Redis single-threaded" = тільки обробка команд в одному потоці.**

Одна команда виконується за ~1 мікросекунду. Це означає 1,000,000 команд за секунду теоретично.
Bottleneck — мережа, не CPU.

**Як він обробляє тисячі з'єднань?**

I/O Multiplexing (epoll на Linux, kqueue на macOS):
```
Аналогія:
Ресторан з одним офіціантом, але дуже швидким.
Він не сидить біля одного столику і чекає.
Він швидко обходить ВСІ столики:
  - Столик 1: є замовлення? → прийняти
  - Столик 2: нічого? → далі
  - Столик 3: є замовлення? → прийняти
  - ...
  - Повернутися до столика 1
```

```
1000 клієнтів підключені до Redis
    ↓
epoll каже: "Клієнти 5, 23, 147 мають дані для читання"
    ↓
Redis обробляє команди від 5, 23, 147
    ↓
epoll каже: "Клієнти 8, 42 мають дані"
    ↓
...і так далі, в одному потоці, без блокувань
```

**Redis 6.0+:** додав I/O threads — мережеве читання/запис в окремих потоках, але команди все ще в одному.

---

### 14. Redis vs Memcached — коли що обирати?

| | Redis | Memcached |
|--|-------|-----------|
| Структури даних | String, List, Hash, Set, Sorted Set... | Тільки String |
| Persistence | RDB, AOF | Ні |
| Pub/Sub | Так | Ні |
| Потоки | Single-threaded (команди) | Multi-threaded |
| Max value | 512 МБ | 1 МБ |
| Реплікація | Вбудована | Ні |
| Кластер | Вбудований | Client-side |

**Memcached обирають коли:**
- Потрібен тільки простий кеш "ключ → рядок"
- Дуже великий трафік (multi-threaded дає перевагу)
- Не потрібна persistence

**Redis обирають коли:**
- Потрібні складні структури (списки, множини, рейтинги)
- Потрібна persistence, pub/sub, streams, locks
- **99% випадків — обирають Redis**

---

### 15. Чим відрізняється RDB від AOF?

**RDB = фотографія кімнати.**
Кожні N хвилин Redis робить "знімок" всіх даних і зберігає файл `dump.rdb`.
```
Час:    0 хв ──── 5 хв (фото) ──── 10 хв (фото) ──── 15 хв (фото)
Втрата: якщо Redis впаде на 8-й хвилині → втрачено 3 хвилини даних
```

**AOF = щоденник подій.**
Redis записує кожну команду в файл `appendonly.aof`:
```
SET user:1 "John"
DEL user:2
INCR counter
SET user:1 "Jane"
...
```
При перезапуску — "перечитує щоденник" з початку.

```
Якщо fsync кожну секунду → максимальна втрата = 1 секунда
```

**Як працює BGSAVE (фонове збереження RDB):**
```
1. Redis викликає fork() — створює копію процесу (child process)
2. Child process пише дані на диск
3. Батьківський процес продовжує обслуговувати клієнтів
4. Copy-on-Write: пам'ять спільна, копіюються тільки змінені сторінки
```

---

### 16. Які fsync policy є у AOF?

fsync — це коли дані з буфера ОС реально записуються на диск.

| Policy | Що робить | Втрата даних | Швидкість |
|--------|-----------|-------------|-----------|
| `always` | fsync після кожної команди | 0 | Найповільніша |
| `everysec` | fsync раз на секунду | до 1 секунди | Рекомендована |
| `no` | ОС сама вирішує коли | до 30 секунд | Найшвидша |

**Рекомендація:** `appendfsync everysec` — баланс між надійністю та швидкістю.

---

### 17. Чому для production рекомендують RDB + AOF?

```
RDB + AOF (гібрид, Redis 7.0+):
  aof-use-rdb-preamble yes

Файл AOF = [RDB snapshot на початку] + [AOF лог змін після snapshot]
```

Отримуємо найкраще з обох:
- **Швидкий restart** — завантажити RDB (бінарний, швидко)
- **Мінімальна втрата** — AOF "хвіст" з останніми змінами (максимум 1 сек)
- **Компактніший файл** — RDB стискає краще ніж тисячі AOF команд

---

### 18. Що таке Cache Stampede (Thundering Herd)?

**Проблема:**
```
TTL ключа "popular:data" закінчився
    ↓
1000 користувачів одночасно запитують цей ключ
    ↓
Всі 1000 отримують cache miss
    ↓
Всі 1000 йдуть в PostgreSQL
    ↓
PostgreSQL під навантаженням, можливий crash!
```

**Рішення 1: Mutex Lock**
Тільки один запит йде в базу, решта чекають:
```javascript
async function getUserSafe(userId) {
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);

  // Спробувати взяти "замок"
  const gotLock = await redis.set(`lock:user:${userId}`, "1", "EX", 10, "NX");

  if (gotLock) {
    // Ми єдині — завантажуємо з бази
    const user = await db.query("SELECT * FROM users WHERE id = $1", [userId]);
    await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));
    await redis.del(`lock:user:${userId}`);
    return user;
  }

  // Хтось вже завантажує — почекати і повторити
  await sleep(100);
  return getUserSafe(userId);
}
```

**Рішення 2: Random Jitter до TTL**
```javascript
const ttl = 3600 + Math.floor(Math.random() * 300); // 3600-3900 сек
await redis.setex(key, ttl, data);
// Тепер ключі "протухають" в різний час, не всі одразу
```

---

### 19. Що таке Cache Penetration?

**Проблема:** запити по ключах, яких НЕМАЄ ні в кеші, ні в базі.
```
GET user:999999 → cache miss → SELECT FROM users WHERE id=999999 → NULL
GET user:999999 → cache miss → SELECT ... → NULL
GET user:999999 → cache miss → SELECT ... → NULL
-- Кожен запит потрапляє в базу! Атакуючий може цим скористатись.
```

**Рішення 1: Кешувати NULL**
```javascript
const user = await db.query("...");
if (!user) {
  // Кешувати "порожній" результат на 5 хвилин
  await redis.setex(`user:${userId}`, 300, "NULL");
  return null;
}
```

**Рішення 2: Bloom Filter**
Структура, яка швидко каже "точно немає" або "можливо є".
```
Запит user:999999
  → Bloom Filter: "такого ID точно немає"
  → Навіть не йдемо в Redis і базу
```

---

### 20. Що таке Cache Avalanche?

**Проблема:** багато ключів з однаковим TTL закінчуються одночасно.
```
12:00 — масове завантаження кешу з TTL = 3600
13:00 — ВСЕ протухає одночасно → база під навалою
```

**Рішення: додати випадковість (jitter) до TTL:**
```javascript
// Замість:
await redis.setex(key, 3600, data);

// Краще:
const jitter = Math.floor(Math.random() * 300); // 0-300 секунд
await redis.setex(key, 3600 + jitter, data);
// Ключі протухнуть між 13:00 і 13:05 — навантаження розмазується
```

---

### 21. Порівняйте Cache-Aside, Write-Through, Write-Behind

**Cache-Aside (Lazy Loading):**
```
Read:  App → Кеш? → hit → відповідь
                  → miss → БД → зберегти в кеш → відповідь
Write: App → БД → видалити з кешу
```
- ✅ Кешується тільки те що запитують
- ❌ Перший запит завжди повільний (miss)

**Write-Through:**
```
Write: App → Кеш → БД (обидва одночасно)
Read:  App → Кеш (завжди є дані)
```
- ✅ Кеш завжди актуальний
- ❌ Повільніший write, кешує навіть непотрібне

**Write-Behind (Write-Back):**
```
Write: App → Кеш (швидко!)
       Кеш → БД (асинхронно, пізніше, батчами)
```
- ✅ Найшвидший write
- ❌ Якщо Redis впаде — дані втрачені (ще не записані в БД)

**Приклад Write-Behind — лічильник переглядів:**
```javascript
// Миттєво записуємо в Redis
await redis.hincrby("pageviews", pageId, 1);

// Кожні 5 хвилин зберігаємо в базу
setInterval(async () => {
  const views = await redis.hgetall("pageviews");
  for (const [pageId, count] of Object.entries(views)) {
    await db.query("UPDATE pages SET views = views + $1 WHERE id = $2", [count, pageId]);
  }
  await redis.del("pageviews");
}, 5 * 60 * 1000);
```

---

### 22. Як працюють Redis транзакції (MULTI/EXEC)?

```redis
MULTI                          -- "почни збирати команди"
SET user:1:balance 100         -- буферизується (не виконується!)
DECRBY user:1:balance 30       -- буферизується
INCRBY user:2:balance 30       -- буферизується
EXEC                           -- "виконай все зібране одразу"
```

**Головна відмінність від SQL транзакцій:**

| | SQL (PostgreSQL) | Redis (MULTI/EXEC) |
|--|-----------------|---------------------|
| Rollback при помилці | ✅ Так | ❌ НІ! |
| Атомарне виконання | ✅ Так | ✅ Так |

```
Redis: якщо INCRBY фейлить (значення не число) → SET все одно виконається!
Немає rollback. Частина команд може пройти, частина — ні.
```

**Коли використовувати:** коли треба гарантувати, що ніхто не вклиниться між командами. Але НЕ для фінансових транзакцій де потрібен rollback.

---

### 23. Що таке WATCH? (Optimistic Locking)

WATCH — це механізм "я слідкую за ключем, і якщо хтось його змінить поки я готую транзакцію — скасувати все".

```redis
WATCH user:1:balance           -- "слідкуй за цим ключем"
val = GET user:1:balance       -- прочитати поточне значення (100)

MULTI
SET user:1:balance (val - 30)  -- підготувати зміну
EXEC
-- Якщо ніхто не змінив balance → EXEC пройде
-- Якщо хтось змінив → EXEC поверне nil → треба повторити
```

**Аналогія:** Compare-And-Swap (CAS).
Як редагування Google Doc: ви відкрили документ, зробили зміни, натиснули "Зберегти". Якщо хтось змінив документ поки ви редагували — конфлікт, треба перечитати.

---

### 24. Чому Lua скрипти кращі за MULTI/EXEC для складної логіки?

**Проблема MULTI/EXEC:** не можна прочитати результат команди і використати його в наступній.

```redis
-- Це НЕ працює в MULTI/EXEC:
MULTI
val = GET counter        -- отримаєш "QUEUED", не значення!
if val > 10 then         -- неможливо
  DEL counter
EXEC
```

**Lua скрипт — може все:**
```redis
EVAL "
  local val = redis.call('GET', KEYS[1])
  if tonumber(val) > 10 then
    redis.call('DEL', KEYS[1])
    return 1
  end
  return 0
" 1 counter
```

**Переваги Lua:**
- Виконується **атомарно** (нічого між командами)
- Є умовна логіка (if/else)
- Можна читати і писати в одному скрипті
- Результат однієї команди використовується в наступній

**Обмеження:**
- Скрипт блокує Redis (не робіть довгих скриптів!)
- В кластері — всі ключі мають бути в одному hash slot

---

### 25. Що таке Pipeline?

**Без Pipeline:**
```
Клієнт → SET k1 v1 → Сервер (чекаємо відповідь) → OK
Клієнт → SET k2 v2 → Сервер (чекаємо відповідь) → OK
Клієнт → SET k3 v3 → Сервер (чекаємо відповідь) → OK
= 3 round-trips по мережі (наприклад, 3 × 1 мс = 3 мс)
```

**З Pipeline:**
```
Клієнт → SET k1 v1 ─┐
         SET k2 v2 ──┼──→ Сервер → OK, OK, OK
         SET k3 v3 ─┘
= 1 round-trip (1 мс замість 3 мс)
```

```javascript
const pipeline = redis.pipeline();
pipeline.set("k1", "v1");
pipeline.set("k2", "v2");
pipeline.set("k3", "v3");
const results = await pipeline.exec();
```

**Pipeline vs MULTI/EXEC:**
- **Pipeline** — мережева оптимізація (команди відправляються пачкою, але між ними МОЖУТЬ вклинитись інші клієнти)
- **MULTI/EXEC** — гарантія атомарності (нічого між командами)
- Можна комбінувати: MULTI всередині pipeline

---

### 26. Як реалізувати distributed lock?

**Розподілений lock — це "замок", який бачать усі сервери.**

```redis
-- Взяти lock:
SET lock:order:123 "server-1-uuid" NX EX 30
-- NX  = тільки якщо ключа ще немає (тільки один отримає lock)
-- EX 30 = автоматичне відкриття через 30 секунд (щоб не заблокувати назавжди)
```

**Чому UUID?** Щоб не видалити чужий lock:
```
Сервер A взяв lock (TTL 30 сек)
Сервер A завис на 35 секунд
Lock автоматично знявся (TTL)
Сервер B взяв lock
Сервер A "прокинувся" і робить DEL lock → видалив lock сервера B!
```

**Правильне звільнення через Lua:**
```redis
EVAL "
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  end
  return 0
" 1 lock:order:123 "server-1-uuid"
-- Видаляє lock ТІЛЬКИ якщо UUID збігається (наш lock)
```

---

### 27. Redis Sentinel vs Redis Cluster — коли що?

**Sentinel = один master, кілька replica, автоматичний failover.**
```
Sentinel (сторож) слідкує за серверами:
    Master (пише) ←──── Replica 1 (читає)
                  ←──── Replica 2 (читає)

Якщо Master впав → Sentinel промоутить одну з Replica в новий Master.
```
- Дані: до ~25 ГБ (обмежено RAM одного сервера)
- Шардінг: немає
- Простота: висока

**Cluster = кілька master, дані розподілені між ними.**
```
16384 hash slots розподілені між master-ами:

Master A [слоти 0-5460]      + Replica A1
Master B [слоти 5461-10922]  + Replica B1
Master C [слоти 10923-16383] + Replica C1
```
- Дані: необмежено (додаєш master — додаєш RAM)
- Шардінг: автоматичний
- Складність: висока

**Правило:** менше 25 ГБ → Sentinel. Більше → Cluster.

---

### 28. Що таке hash slots? Як працює routing у Cluster?

Redis Cluster ділить всі можливі ключі на **16384 слоти**.

```
Кожен ключ "потрапляє" в слот через формулу:
  slot = CRC16("user:123") % 16384 = 5649

Кожен master відповідає за діапазон слотів:
  Master A: слоти 0-5460
  Master B: слоти 5461-10922   ← слот 5649 тут!
  Master C: слоти 10923-16383
```

**Що відбувається при запиті:**
```
Клієнт → Master A: SET user:123 "John"
Master A: "Слот 5649 не мій, це Master B"
Master A → Клієнт: MOVED 5649 192.168.1.2:6379
Клієнт → Master B: SET user:123 "John" → OK
```

Клієнтські бібліотеки кешують маппінг слотів, тому після першого MOVED далі запити йдуть одразу на правильний master.

---

### 29. Що таке hash tags і навіщо вони?

**Проблема:** в кластері multi-key операції працюють тільки якщо ключі в одному слоті.
```redis
SET user:1:profile "..."    -- слот X
SET user:1:settings "..."   -- слот Y (інший!)
MGET user:1:profile user:1:settings   -- ❌ ПОМИЛКА! Різні слоти!
```

**Рішення — hash tags `{}`:**
```redis
SET {user:1}:profile "..."    -- слот = CRC16("user:1")
SET {user:1}:settings "..."   -- слот = CRC16("user:1") — той самий!
MGET {user:1}:profile {user:1}:settings   -- ✅ Працює!
```

Redis бере для хешування тільки текст в `{}`, тому всі ключі з `{user:1}` потрапляють в один слот.

---

### 30. Pub/Sub vs Streams — коли що обирати?

```
Pub/Sub = радіо. Якщо ти не слухаєш — повідомлення втрачене.
Streams = поштова скринька. Повідомлення чекає поки ти його прочитаєш.
```

|  | Pub/Sub | Streams |
|--|---------|---------|
| Зберігає повідомлення | ❌ Ні | ✅ Так |
| Можна переглянути історію | ❌ Ні | ✅ Так (XRANGE) |
| Підтвердження обробки | ❌ Ні | ✅ Так (XACK) |
| Consumer Groups | ❌ Ні | ✅ Так |
| Гарантія доставки | At-most-once | At-least-once |
| Швидкість | Дуже швидко | Швидко |

**Pub/Sub використовують для:**
- Real-time нотифікацій (WebSocket broadcast)
- Cache invalidation (повідомити всі інстанси)
- Chat (не критично якщо одне повідомлення втрачено)

**Streams використовують для:**
- Обробка замовлень (не можна втратити!)
- Event sourcing
- Task queue з гарантією доставки
- Аудит логи

---

### 31. Як працює Redis Pub/Sub?

```redis
-- Термінал 1 (підписник):
SUBSCRIBE news:sport
-- Чекає повідомлень...

-- Термінал 2 (публікатор):
PUBLISH news:sport "Україна виграла!"
-- Підписник отримає: "Україна виграла!"
```

```javascript
// Підписник (ОБОВ'ЯЗКОВО окремий клієнт!)
const subscriber = createClient();
await subscriber.connect();
await subscriber.subscribe("orders", (message) => {
  console.log("Нове замовлення:", message);
});

// Публікатор
const publisher = createClient();
await publisher.connect();
await publisher.publish("orders", JSON.stringify({ id: 123 }));
```

**Важливо:** підписник НЕ МОЖЕ виконувати інші Redis команди. Потрібен окремий клієнт для підписки.

---

## Частина 3: Просунуті питання (Senior)

---

### 32. Що таке Eviction Policies?

Коли пам'ять Redis закінчується — що робити з новими даними?

```redis
maxmemory 2gb                  -- максимум 2 ГБ
maxmemory-policy allkeys-lru   -- що робити коли 2 ГБ вичерпано
```

| Policy | Що робить | Коли використовувати |
|--------|-----------|---------------------|
| `noeviction` | Помилка на write (default) | Дані не можна втрачати |
| `allkeys-lru` | Видаляє "давно не використовувані" | **Кеш (рекомендовано)** |
| `allkeys-lfu` | Видаляє "рідко використовувані" | Кеш з hot data |
| `volatile-lru` | LRU тільки серед ключів з TTL | Кеш + важливі дані без TTL |
| `volatile-ttl` | Видаляє з найменшим TTL | Пріоритет на свіжі дані |
| `allkeys-random` | Випадковий ключ | Рівномірне навантаження |

---

### 33. LRU vs LFU — різниця?

**LRU (Least Recently Used)** — "давно не чіпали":
```
Ключі:  A(10 хв тому)  B(5 сек тому)  C(2 год тому)
Видалити → C (найдавніше використаний)

Проблема: один випадковий запит робить ключ "свіжим",
навіть якщо він непопулярний.
```

**LFU (Least Frequently Used)** — "рідко використовують":
```
Ключі:  A(100 запитів/день)  B(2 запити/день)  C(50 запитів/день)
Видалити → B (найменш популярний)

Перевага: один випадковий запит не "врятує" непопулярний ключ.
```

**Правило:**
- Більшість випадків → `allkeys-lru` (простий, надійний)
- Є явні "гарячі" ключі → `allkeys-lfu` (краще тримає популярне)

---

### 34. Hot Key — що це і як вирішити?

**Hot Key** — один ключ отримує непропорційно багато запитів.

```
Приклад: сторінка "Головна" на новинному сайті.
  cache:homepage → 100,000 запитів/сек
  Інші ключі → 10-100 запитів/сек
```

**Проблеми:**
- Одна нода обробляє весь трафік (в кластері hot key на одному master)
- Мережевий bottleneck

**Рішення:**

**1. Local Cache (L1 + L2):**
```
L1 (in-process, 1-5 сек TTL) → L2 (Redis) → DB

Кожен сервер тримає копію hot key в пам'яті процесу.
Більшість запитів навіть не йдуть в Redis.
```

**2. Реплікація hot key:**
```redis
-- Замість одного ключа:
cache:homepage

-- Кілька копій з випадковим суфіксом:
cache:homepage:1
cache:homepage:2
cache:homepage:3

-- Клієнт вибирає випадкову копію:
const shard = Math.floor(Math.random() * 3) + 1;
const data = await redis.get(`cache:homepage:${shard}`);
```

---

### 35. Big Key — чим небезпечний? Як вирішити?

**Big Key** — ключ з дуже великим значенням (наприклад, 100 МБ).

**Проблеми:**
```
DEL big_key → блокує Redis на секунди!
(Redis single-threaded — всі інші клієнти чекають)
```

**Рішення:**
```redis
-- 1. UNLINK замість DEL (асинхронне видалення в background)
UNLINK big_key    -- повертається миттєво, видалення у фоні

-- 2. Моніторити великі ключі:
redis-cli --bigkeys

-- 3. Розбити на менші ключі:
-- Замість: SET user:1:data "{100MB JSON}"
-- Краще:   HSET user:1 profile "{...}" settings "{...}" history "{...}"
```

**Правило:** один ключ < 10 КБ — ідеально, < 1 МБ — прийнятно, > 10 МБ — проблема.

---

### 36. Чому не можна використовувати KEYS * у production?

```redis
KEYS user:*    -- знайти всі ключі що починаються з "user:"
```

**Проблема:** KEYS сканує ВСІ ключі в базі. Складність O(n).
Якщо в Redis 10 мільйонів ключів → Redis заблокується на секунди.
В цей час ніхто не може виконати жодну команду.

**Рішення: SCAN — курсорний ітератор:**
```javascript
let cursor = "0";
do {
  const [newCursor, keys] = await redis.scan(
    cursor,
    "MATCH", "user:*",
    "COUNT", 100     // обробити ~100 ключів за раз
  );
  cursor = newCursor;
  // обробити знайдені keys
} while (cursor !== "0");
```

SCAN повертає ключі порціями, не блокуючи Redis.

---

### 37. Що таке Redlock? Як працює?

**Проблема:** якщо Redis з lock-ом впаде — lock зникне, і два процеси одночасно виконають критичну секцію.

**Redlock — розподілений lock на кількох незалежних Redis серверах:**

```
Алгоритм (зазвичай 5 серверів):

1. Записати timestamp
2. Спробувати SET NX EX на кожному з 5 серверів
3. Якщо отримав lock на majority (≥ 3 з 5)
   І загальний час < TTL lock
   → Lock отримано!
4. Інакше → звільнити lock на всіх серверах

Приклад:
  Redis 1: ✅ lock
  Redis 2: ✅ lock
  Redis 3: ❌ timeout
  Redis 4: ✅ lock
  Redis 5: ✅ lock
  → 4/5 ≥ majority → lock отримано
```

**Критика (Martin Kleppmann):**
- Залежить від синхронізації часу між серверами
- GC пауза може призвести до split-brain
- Для критичних систем краще Zookeeper / etcd

**Практична порада:**
- Для більшості задач — простий lock на одному Redis
- Для розподіленого — бібліотека (Redlock, Redisson)
- Для фінансових операцій — Zookeeper або etcd

---

### 38. Як уникнути data loss при Redis failover?

Коли Master падає і Replica стає новим Master — деякі дані можуть бути втрачені (async replication).

**4 рівні захисту:**

```
1. AOF з fsync everysec
   → Максимум 1 секунда втрати на одному сервері

2. WAIT command
   await redis.set("key", "value");
   await redis.wait(1, 5000);
   // Чекати поки хоча б 1 Replica підтвердить запис (до 5 сек)

3. min-replicas-to-write 1
   // Redis не приймає write якщо немає жодної replica
   // Краще відмовити, ніж втратити дані

4. Redis як кеш, PostgreSQL як source of truth
   // Найнадійніший підхід:
   // Записати в PostgreSQL → записати в Redis
   // Якщо Redis втратить дані → перечитати з PostgreSQL
```

---

### 39. Як працює failover у Redis Cluster?

```
1. Master A перестає відповідати

2. Replica A1 помічає відсутність heartbeat
   → PFAIL (Probable Fail) — "можливо впав"

3. Replica A1 питає інші Masters:
   "Чи Master A відповідає вам?"

4. Якщо majority Masters підтверджують що Master A недоступний
   → FAIL (точно впав)

5. Replica A1 запитує голоси у Masters:
   "Чи можу я стати новим Master A?"

6. Majority Masters голосують ЗА
   → Replica A1 стає Master A

7. Нові hash slots (які обслуговував старий Master A)
   тепер обслуговує новий Master A (колишня Replica A1)

Час failover: зазвичай 1-2 секунди
```

---

### 40. ASK vs MOVED redirect — різниця?

Обидва — відповіді кластера "цей ключ не у мене":

**MOVED** — слот ПОСТІЙНО переїхав:
```
Клієнт → Node A: GET key
Node A → Клієнт: MOVED 5649 node-b:6379
-- "Слот 5649 тепер назавжди на Node B. Оновіть свою конфігурацію."
```

**ASK** — слот ТИМЧАСОВО мігрує (resharding у процесі):
```
Клієнт → Node A: GET key
Node A → Клієнт: ASK 5649 node-b:6379
-- "Цей конкретний ключ вже на Node B, але слот ще мігрує.
--  Одноразово запитай у Node B, але не оновлюй конфігурацію."
```

---

### 41. Як спроєктувати real-time leaderboard для 10M гравців?

**Sorted Set — ідеальна структура:**

```redis
-- Додати/оновити score гравця — O(log n)
ZADD leaderboard 1500 "player:123"

-- Топ-10 — O(log n + 10)
ZREVRANGE leaderboard 0 9 WITHSCORES

-- Позиція конкретного гравця — O(log n)
ZREVRANK leaderboard "player:123"   -- → 42 (43-тє місце, від 0)

-- Score гравця — O(1)
ZSCORE leaderboard "player:123"     -- → 1500
```

**Масштабування для 10M:**
- Sorted Set з 10M елементів ≈ 1.5 ГБ RAM
- Одна нода Redis справиться
- Для більшого масштабу — sharding по регіону або лізі:
  ```redis
  leaderboard:europe
  leaderboard:asia
  leaderboard:global   -- агрегація з регіональних
  ```

---

### 42. Як реалізувати Rate Limiter?

**4 алгоритми від простого до складного:**

**1. Fixed Window (найпростіший):**
```redis
-- Максимум 100 запитів за хвилину
INCR ratelimit:user:123:202401151430    -- ключ = user + хвилина
EXPIRE ratelimit:user:123:202401151430 60
-- Якщо INCR повернув > 100 → заблокувати
```
- ❌ Проблема: 99 запитів в 14:30:59 + 99 запитів в 14:31:01 = 198 за 2 секунди

**2. Sliding Window (точніший):**
```javascript
async function isRateLimited(userId, limit, windowSec) {
  const key = `ratelimit:${userId}`;
  const now = Date.now();
  const windowStart = now - windowSec * 1000;

  const pipe = redis.multi();
  pipe.zremrangebyscore(key, 0, windowStart); // видалити старі
  pipe.zadd(key, now, `${now}`);              // додати поточний
  pipe.zcard(key);                            // порахувати
  pipe.expire(key, windowSec);

  const results = await pipe.exec();
  return results[2] > limit;
}
```

**3. Token Bucket** — як відро з жетонами:
```
Кожну секунду в "відро" додається 10 жетонів (максимум 100).
Кожен запит забирає 1 жетон.
Немає жетонів → запит відхилено.
Дозволяє burst (якщо відро повне — можна 100 запитів одразу).
```

**4. Leaky Bucket** — як відро з дірочкою:
```
Запити потрапляють у чергу (відро).
Обробляються з фіксованою швидкістю (дірочка).
Відро переповнене → запит відхилено.
Згладжує трафік — навіть burst обробляється рівномірно.
```

---

### 43. Redis sessions vs JWT — коли що?

| | Redis Sessions | JWT |
|--|---------------|-----|
| Де зберігається | На сервері (Redis) | У клієнта (cookie/header) |
| Можна інвалідувати | ✅ Так (видалити з Redis) | ❌ Ні (до закінчення терміну) |
| Масштабування | Потрібен shared Redis | Без стану (будь-який сервер) |
| Розмір | Маленький (session ID) | Більший (payload + підпис) |

**Redis Session:**
```javascript
// Login
const sessionId = crypto.randomUUID();
await redis.setex(`session:${sessionId}`, 86400, JSON.stringify({
  userId: "user:1", role: "admin"
}));
// Відправити sessionId в cookie

// Logout — миттєва інвалідація!
await redis.del(`session:${sessionId}`);
```

**JWT:**
```javascript
// Login
const token = jwt.sign({ userId: "user:1", role: "admin" }, SECRET, { expiresIn: "15m" });

// Logout — НЕ МОЖНА інвалідувати JWT до закінчення!
// Рішення: blacklist в Redis
await redis.setex(`blacklist:${token}`, 900, "1");
```

**Практичний підхід (гібрид):**
- **JWT** — короткий access token (15 хвилин), без Redis
- **Redis** — refresh token або session blacklist, для logout/ban

---

### 44. Redis vs Kafka для event streaming?

| | Redis Streams | Kafka |
|--|--------------|-------|
| Зберігання | В пам'яті | На диску |
| Масштабування | Вертикальне | Горизонтальне (partitions) |
| Гарантія | At-least-once | Exactly-once (з налаштуванням) |
| Retention | Обмежено RAM | Необмежено (диск) |
| Складність | Проста | Складна (Zookeeper/KRaft) |
| Throughput | ~100K msg/sec | ~1M+ msg/sec |

**Redis Streams:**
- Малий масштаб (один сервіс, тисячі msg/sec)
- Вже використовуєте Redis
- Не потрібна довга історія подій

**Kafka:**
- Великий масштаб (мікросервіси, мільйони msg/sec)
- Потрібна довга історія (retention days/weeks)
- Потрібен exactly-once
- Event sourcing з replay

**Redis Pub/Sub** — fire-and-forget (взагалі без persistence).

---

### 45. Як спроєктувати cache invalidation у мікросервісах?

**Проблема:** User Service оновив юзера, але Cache Service на іншому сервері має старі дані.

**Рішення — Event-Based Invalidation через Pub/Sub:**

```javascript
// User Service: оновив дані
await db.query("UPDATE users SET name = $1 WHERE id = $2", [name, id]);
await publisher.publish("cache:invalidate", JSON.stringify({
  type: "user",
  id: userId,
}));

// API Service 1, 2, 3... (всі інстанси підписані)
await subscriber.subscribe("cache:invalidate", (message) => {
  const { type, id } = JSON.parse(message);
  // Видалити з локального кешу
  localCache.delete(`${type}:${id}`);
  // Видалити з Redis
  redis.del(`${type}:${id}`);
});
```

**Патерн: Event → Invalidate → Lazy Reload**
```
1. Дані змінились → опублікувати подію
2. Всі інстанси отримали подію → видалили кеш
3. Наступний запит → cache miss → завантажити свіжі дані з бази
```

---

### 46. Як організувати naming convention для ключів?

**Формат:** `service:entity:id:field`

```redis
-- Хороші приклади:
api:user:123:profile         -- профіль юзера
api:session:abc123           -- сесія
api:ratelimit:ip:1.2.3.4    -- rate limit по IP
cache:query:users:page:1     -- кеш запиту
lock:order:456               -- розподілений lock
queue:emails                 -- черга emails

-- Погані приклади:
user_123                     -- незрозуміло що це
data                         -- надто загально
myapp_cache_user_profile_123 -- нечитабельно
```

**Правила:**
1. Розділювач — двокрапка `:` (стандарт Redis)
2. Від загального до конкретного
3. Lowercase
4. Не використовувати пробіли та спецсимволи

---

### 47. Які проблеми Redis в мікросервісній архітектурі?

| Проблема | Опис | Рішення |
|----------|------|---------|
| **Hot Key** | Один ключ отримує весь трафік | L1 local cache + реплікація ключа |
| **Big Key** | Великий value блокує при DELETE | UNLINK, розбити на менші |
| **Memory Fragmentation** | Часті alloc/dealloc фрагментують RAM | `activedefrag yes`, jemalloc |
| **Network Partition** | Split-brain в кластері | `min-replicas-to-write`, quorum |
| **Thundering Herd** | TTL закінчився → всі в базу | Mutex lock, stale-while-revalidate |
| **Data Loss** | Async replication → втрата при failover | WAIT, AOF fsync, source of truth в SQL |

---

### 48. Обмеження Redis Cluster

```
1. Multi-key операції — тільки якщо ключі в одному hash slot
   MGET key1 key2           -- ❌ якщо різні слоти
   MGET {tag}:k1 {tag}:k2   -- ✅ один слот (hash tags)

2. Транзакції (MULTI/EXEC) — тільки один слот

3. Lua scripts — всі ключі повинні бути в одному слоті

4. Pub/Sub — повідомлення йдуть на ВСІ ноди (до Redis 7.0)
   Redis 7.0+: Sharded Pub/Sub вирішує це (SSUBSCRIBE, SPUBLISH)

5. SELECT (вибір БД) — тільки db 0

6. Resharding — slot migration під час роботи може додати latency
```

---

### 49. Розгортання Redis у production — best practices

```
Топологія (мінімум для Cluster):
  6 нод = 3 master + 3 replica
  Replica на ІНШОМУ фізичному сервері ніж її master

  AZ-1:  Master A, Replica B1
  AZ-2:  Master B, Replica C1
  AZ-3:  Master C, Replica A1
  (якщо одна AZ впаде — дані не втрачаться)
```

**Managed сервіси (рекомендовано):**
- AWS ElastiCache / MemoryDB
- GCP Memorystore
- Azure Cache for Redis
- Автоматичний failover, patching, backups

---

### 50. Cheat Sheet — швидке повторення перед співбесідою

```
Швидкий?        → RAM + single-thread + epoll + RESP протокол
Persistence?    → RDB (snapshot) + AOF (log) → гібрид для production
Eviction?       → allkeys-lru (кеш), volatile-lru (кеш + important keys)
Транзакції?     → MULTI/EXEC (без rollback!), WATCH (optimistic lock)
Атомарність?    → Lua scripts (єдиний спосіб "прочитай → вирішив → запиши")
Масштабування?  → Sentinel (HA) або Cluster (HA + sharding)
Lock?           → SET key uuid NX EX ttl + Lua DEL з перевіркою UUID
Pipeline?       → Батчінг команд в один round-trip
Pub/Sub?        → Fire-and-forget. Для гарантій → Streams
KEYS?           → НІКОЛИ в production! SCAN замість цього
Big Key?        → UNLINK замість DEL. Ключ < 1 МБ
Cache Stampede? → Mutex lock або stale-while-revalidate
Cache Penetration? → Кешувати NULL або Bloom Filter
Cache Avalanche?   → Random jitter до TTL
Hot Key?        → L1 local cache + реплікація ключа
```
