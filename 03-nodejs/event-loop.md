# Event Loop в Node.js

## Як працює Event Loop в Node.js?

Event Loop -- це механізм, який дозволяє Node.js виконувати неблокуючі I/O операції, незважаючи на те, що JavaScript є однопотоковою мовою. Він делегує операції ядру операційної системи, коли це можливо.

Основна ідея: Node.js запускає один головний потік, в якому виконується JavaScript-код. Коли зустрічається асинхронна операція (читання файлу, мережевий запит, таймер), вона передається відповідному системному механізму (libuv thread pool, OS kernel). Коли операція завершується, її callback додається у відповідну чергу Event Loop для подальшого виконання.

```
   ┌───────────────────────────────────────────────┐
   │              Вхідний скрипт                    │
   │         (синхронний код виконується)            │
   └───────────────────┬───────────────────────────┘
                       │
                       ▼
   ┌───────────────────────────────────────────────┐
   │              EVENT LOOP                        │
   │                                               │
   │   ┌─────────────┐    ┌─────────────────┐     │
   │   │   Timers     │───▶│ Pending Callbacks│     │
   │   └─────────────┘    └────────┬────────┘     │
   │                               │               │
   │   ┌─────────────┐    ┌───────▼────────┐     │
   │   │Close Callbacks│◀──│  Idle/Prepare   │     │
   │   └──────┬──────┘    └────────┬────────┘     │
   │          │                    │               │
   │   ┌──────▼──────┐    ┌───────▼────────┐     │
   │   │    Check     │◀──│     Poll        │     │
   │   └─────────────┘    └────────────────┘     │
   │                                               │
   └───────────────────────────────────────────────┘
```

Event Loop працює як нескінченний цикл, який перевіряє наявність задач у кожній фазі. Якщо задач немає і немає запланованих таймерів, процес завершується.

```javascript
// Приклад: послідовність виконання
console.log("1: Синхронний код");

setTimeout(() => {
  console.log("2: setTimeout (macrotask - timers phase)");
}, 0);

setImmediate(() => {
  console.log("3: setImmediate (check phase)");
});

Promise.resolve().then(() => {
  console.log("4: Promise (microtask)");
});

process.nextTick(() => {
  console.log("5: process.nextTick (microtask, найвищий пріоритет)");
});

console.log("6: Ще синхронний код");

// Результат:
// 1: Синхронний код
// 6: Ще синхронний код
// 5: process.nextTick (microtask, найвищий пріоритет)
// 4: Promise (microtask)
// 2: setTimeout (macrotask - timers phase)
// 3: setImmediate (check phase)
```

---

## Які фази Event Loop?

Event Loop складається з шести основних фаз, кожна з яких має FIFO-чергу callbacks для виконання:

### 1. Timers (Таймери)

Виконує callbacks, заплановані через `setTimeout()` та `setInterval()`. Таймер визначає **мінімальний** час, після якого callback може бути виконаний, а не точний час виконання.

```javascript
// Callback виконається не раніше ніж через 100мс,
// але може бути пізніше, якщо Event Loop зайнятий
setTimeout(() => {
  console.log("Виконано після >= 100мс");
}, 100);
```

### 2. Pending Callbacks (Відкладені Callbacks)

Виконує callbacks для деяких системних операцій, наприклад, помилки TCP-з'єднання (ECONNREFUSED). Ці callbacks відкладені з попередньої ітерації циклу.

### 3. Idle, Prepare (Очікування, Підготовка)

Використовується виключно внутрішньо Node.js. Розробники не мають прямого доступу до цієї фази. Тут відбувається внутрішня підготовка до фази Poll.

### 4. Poll (Опитування)

Найважливіша фаза. Виконує два основних завдання:
- Обчислює, як довго слід блокувати та чекати на I/O
- Обробляє події з черги poll

```
Poll фаза:
├── Якщо черга poll НЕ порожня:
│   └── Виконує callbacks синхронно, поки черга не спорожніє
│       або не досягне системного ліміту
├── Якщо черга poll порожня:
│   ├── Якщо є setImmediate() → переходить до фази Check
│   └── Якщо немає setImmediate() → чекає на нові callbacks
└── Перевіряє таймери: якщо таймер готовий → повертається до Timers
```

### 5. Check (Перевірка)

Виконує callbacks `setImmediate()`. Ця фаза дозволяє виконати код одразу після завершення фази Poll.

```javascript
// setImmediate гарантовано виконується після фази Poll
const fs = require("fs");

fs.readFile("file.txt", () => {
  // Всередині I/O callback setImmediate завжди виконується перед setTimeout
  setTimeout(() => console.log("setTimeout"), 0);
  setImmediate(() => console.log("setImmediate")); // завжди першим
});
```

### 6. Close Callbacks (Callbacks закриття)

Виконує callbacks подій закриття, наприклад `socket.on('close', ...)`. Якщо socket або handle закривається раптово (через `socket.destroy()`), подія 'close' буде оброблена тут.

```javascript
const net = require("net");

const server = net.createServer((socket) => {
  socket.on("close", () => {
    console.log("З'єднання закрите"); // Close Callbacks фаза
  });
});
```

---

## Різниця між process.nextTick() та setImmediate()

Це одне з найпопулярніших запитань на співбесідах. Назви цих функцій інтуїтивно заплутані:

| Характеристика | `process.nextTick()` | `setImmediate()` |
|---|---|---|
| Коли виконується | Після поточної операції, перед продовженням Event Loop | На наступній ітерації Event Loop (фаза Check) |
| Черга | Microtask queue (nextTick queue) | Macrotask queue (check queue) |
| Пріоритет | Найвищий серед microtasks | Нижчий, виконується у фазі Check |
| Рекурсія | Може заблокувати Event Loop (starvation) | Безпечний для рекурсії |
| Рекомендація | Використовувати обережно | Рекомендований для більшості випадків |

```javascript
// Демонстрація пріоритету
setImmediate(() => console.log("1: setImmediate"));
process.nextTick(() => console.log("2: nextTick"));
Promise.resolve().then(() => console.log("3: Promise"));

// Результат:
// 2: nextTick (найвищий пріоритет серед microtasks)
// 3: Promise (microtask, але після nextTick)
// 1: setImmediate (macrotask, фаза Check)
```

```javascript
// Небезпека рекурсивного process.nextTick
function recursiveNextTick() {
  process.nextTick(recursiveNextTick); // Event Loop НІКОЛИ не просунеться далі!
}
// recursiveNextTick(); // НЕ робіть так!

// Безпечна альтернатива з setImmediate
function recursiveImmediate() {
  setImmediate(recursiveImmediate); // Event Loop продовжує працювати нормально
}
```

### Коли використовувати process.nextTick()

1. Для обробки помилок перед продовженням Event Loop
2. Для виконання callback після поточного стеку, але перед I/O
3. Для забезпечення консистентності API (завжди асинхронний)

```javascript
// Класичний приклад: забезпечення асинхронності API
function apiCall(callback) {
  // Якщо дані в кеші, все одно викликаємо callback асинхронно
  if (cache.has(key)) {
    process.nextTick(() => callback(null, cache.get(key)));
    return;
  }
  fetchFromDb(key, callback);
}
```

---

## Що таке microtasks та macrotasks?

Event Loop оперує двома типами асинхронних задач:

### Microtasks (Мікрозадачі)

Виконуються **між** кожною фазою Event Loop та після кожного macrotask. Мають вищий пріоритет.

Джерела microtasks:
- `process.nextTick()` (найвищий пріоритет серед microtasks)
- `Promise.then()`, `Promise.catch()`, `Promise.finally()`
- `queueMicrotask()`

### Macrotasks (Макрозадачі)

Виконуються у відповідних фазах Event Loop.

Джерела macrotasks:
- `setTimeout()`, `setInterval()` (фаза Timers)
- `setImmediate()` (фаза Check)
- I/O callbacks (фаза Poll)
- UI rendering (у браузері)

```javascript
console.log("1: Синхронний");

setTimeout(() => console.log("2: Macrotask - setTimeout"), 0);

queueMicrotask(() => console.log("3: Microtask - queueMicrotask"));

Promise.resolve()
  .then(() => console.log("4: Microtask - Promise.then"))
  .then(() => console.log("5: Microtask - ланцюжок Promise"));

process.nextTick(() => console.log("6: Microtask - nextTick"));

console.log("7: Синхронний");

// Результат:
// 1: Синхронний
// 7: Синхронний
// 6: Microtask - nextTick (найвищий пріоритет)
// 3: Microtask - queueMicrotask
// 4: Microtask - Promise.then
// 5: Microtask - ланцюжок Promise
// 2: Macrotask - setTimeout
```

### Порядок виконання microtasks

```
┌─────────────────────────────┐
│  1. process.nextTick queue  │  <- Найвищий пріоритет
├─────────────────────────────┤
│  2. Promise microtask queue │
├─────────────────────────────┤
│  3. queueMicrotask queue    │  <- Те ж, що Promise queue
└─────────────────────────────┘
```

---

## Як Promise впливає на Event Loop?

Promise callbacks (`.then()`, `.catch()`, `.finally()`) додаються до microtask queue. Це означає, що вони виконуються після поточного синхронного коду, але перед наступною фазою Event Loop.

```javascript
// Детальний приклад взаємодії Promise з Event Loop

async function example() {
  console.log("1: Початок async функції (синхронно)");

  const result = await fetch("https://api.example.com/data");
  // Все після await -- це microtask

  console.log("2: Після await (microtask)");
  return result;
}

console.log("3: Перед викликом");
example();
console.log("4: Після виклику (синхронно)");

// Результат:
// 3: Перед викликом
// 1: Початок async функції (синхронно)
// 4: Після виклику (синхронно)
// 2: Після await (microtask) -- коли fetch завершиться
```

### Ланцюжки Promise та Event Loop

```javascript
Promise.resolve()
  .then(() => {
    console.log("Promise 1");
    // Новий microtask додається до черги
    return Promise.resolve();
  })
  .then(() => {
    console.log("Promise 2");
  });

Promise.resolve().then(() => {
  console.log("Promise 3");
});

// Результат:
// Promise 1
// Promise 3
// Promise 2
```

Важливо розуміти: якщо `.then()` повертає новий Promise, наступний `.then()` у ланцюжку відкладається на одну додаткову microtask ітерацію.

---

## Що таке starvation і як його уникнути?

Starvation (голодування) -- це ситуація, коли певні задачі ніколи не отримують шансу виконатися, тому що microtask queue постійно поповнюється новими задачами.

```javascript
// Приклад starvation: Event Loop не просувається далі
function starvation() {
  process.nextTick(() => {
    console.log("nextTick");
    starvation(); // Рекурсивно додаємо нові microtasks
  });
}

setTimeout(() => {
  console.log("Цей код НІКОЛИ не виконається!");
}, 0);

starvation(); // Event Loop заблокований у microtask queue
```

### Як уникнути starvation

1. **Використовуйте `setImmediate()` замість `process.nextTick()` для рекурсії**

```javascript
function safe() {
  setImmediate(() => {
    doWork();
    safe(); // Event Loop встигає обробити інші фази
  });
}
```

2. **Обмежуйте кількість microtasks за ітерацію**

```javascript
const BATCH_SIZE = 100;

async function processBatch(items) {
  for (let i = 0; i < items.length; i++) {
    await processItem(items[i]);

    // Кожні BATCH_SIZE елементів даємо Event Loop "подихати"
    if (i % BATCH_SIZE === 0) {
      await new Promise((resolve) => setImmediate(resolve));
    }
  }
}
```

3. **Використовуйте `queueMicrotask()` свідомо**

```javascript
// Не створюйте нескінченний ланцюжок microtasks
let count = 0;
function controlled() {
  if (count++ > 1000) return;
  queueMicrotask(controlled);
}
```

---

## Як Event Loop обробляє I/O операції?

Node.js використовує бібліотеку **libuv** для обробки I/O операцій. libuv надає thread pool (за замовчуванням 4 потоки) та використовує механізми ядра ОС.

```
┌──────────────────────────────────────────┐
│          JavaScript (V8 Engine)           │
│          Однопотокове виконання           │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│             libuv                         │
│                                          │
│  ┌──────────────┐  ┌─────────────────┐  │
│  │  Thread Pool  │  │  OS Kernel APIs  │  │
│  │  (4 потоки)   │  │                 │  │
│  │              │  │  - epoll (Linux) │  │
│  │  - fs.*      │  │  - kqueue (Mac)  │  │
│  │  - dns.*     │  │  - IOCP (Win)    │  │
│  │  - crypto.*  │  │                 │  │
│  │  - zlib.*    │  │  - TCP/UDP       │  │
│  │              │  │  - Pipes         │  │
│  └──────────────┘  └─────────────────┘  │
└──────────────────────────────────────────┘
```

### Що використовує Thread Pool

- Файлова система (`fs.readFile`, `fs.writeFile` тощо)
- DNS-запити (`dns.lookup`)
- Криптографічні операції (`crypto.pbkdf2`, `crypto.scrypt`)
- Компресія (`zlib`)

### Що використовує OS Kernel

- Мережеві операції (TCP, UDP, HTTP)
- Pipes
- Таймери
- Сигнали

```javascript
// Розмір thread pool можна змінити
process.env.UV_THREADPOOL_SIZE = "8"; // За замовчуванням 4, максимум 1024

const fs = require("fs");
const crypto = require("crypto");

// Ця операція виконується в thread pool
fs.readFile("large-file.txt", (err, data) => {
  // Callback додається до Poll queue після завершення
  console.log("Файл прочитано");
});

// Ця операція також виконується в thread pool
crypto.pbkdf2("password", "salt", 100000, 64, "sha512", (err, key) => {
  console.log("Хеш обчислено");
});

// Мережева операція -- через OS kernel, без thread pool
const http = require("http");
http.get("http://example.com", (res) => {
  console.log("Відповідь отримана");
});
```

---

## Чому Node.js однопотоковий, але може обробляти багато з'єднань?

Node.js використовує модель **однопотокового Event Loop з неблокуючим I/O**. Це означає:

1. **JavaScript виконується в одному потоці** -- немає проблем з конкурентним доступом до даних, не потрібні мьютекси та локи.

2. **I/O операції делегуються ОС або thread pool** -- головний потік не блокується під час очікування.

3. **Event-driven архітектура** -- замість створення нового потоку для кожного з'єднання (як в Apache), Node.js реєструє callbacks та продовжує обробляти інші запити.

```
Традиційна модель (Apache):           Node.js модель:
┌───────┐                              ┌───────────┐
│Thread 1│── Запит 1 (чекає I/O)       │           │
├───────┤                              │  Event    │
│Thread 2│── Запит 2 (чекає I/O)       │  Loop     │── Обробляє тисячі з'єднань
├───────┤                              │  (1 потік)│   неблокуюче
│Thread 3│── Запит 3 (чекає I/O)       │           │
├───────┤                              └───────────┘
│  ...   │── 10000 потоків = 10GB RAM         │
└───────┘                              ┌───────────┐
                                       │  libuv    │── I/O делегується ОС
                                       │  thread   │
                                       │  pool     │
                                       └───────────┘
```

```javascript
// Приклад: HTTP-сервер що обробляє багато з'єднань
const http = require("http");

const server = http.createServer(async (req, res) => {
  // Кожен запит обробляється як callback в Event Loop
  // Поки чекаємо на базу даних, Event Loop обробляє інші запити

  const data = await queryDatabase(); // Неблокуюча операція
  res.end(JSON.stringify(data));
});

server.listen(3000);
// Один потік може обробляти десятки тисяч одночасних з'єднань
```

### Обмеження однопотокової моделі

```javascript
// CPU-інтенсивна операція БЛОКУЄ Event Loop
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

const server = http.createServer((req, res) => {
  // Цей обчислення заблокує ВСІ інші запити!
  const result = fibonacci(45);
  res.end(`Result: ${result}`);
});

// Рішення: використати Worker Threads для CPU-інтенсивних задач
const { Worker } = require("worker_threads");

const server2 = http.createServer((req, res) => {
  const worker = new Worker("./fibonacci-worker.js", {
    workerData: { n: 45 },
  });

  worker.on("message", (result) => {
    res.end(`Result: ${result}`);
  });
});
```

### Ключові висновки для співбесіди

1. Node.js ефективний для I/O-інтенсивних задач (API-сервери, real-time додатки)
2. Не підходить для CPU-інтенсивних обчислень без Worker Threads
3. Один процес Node.js використовує одне ядро CPU (для масштабування -- Cluster або Worker Threads)
4. Event Loop -- це не потік, а механізм диспетчеризації задач
5. Справжня паралельність досягається через libuv thread pool та OS kernel APIs
