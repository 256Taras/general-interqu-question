# Cluster та Worker Threads — Питання для інтерв'ю

## Зміст

1. [Cluster Module](#cluster-module)
2. [Worker Threads](#worker-threads)
3. [Cluster vs Worker Threads — порівняння](#cluster-vs-worker-threads--порівняння)
4. [SharedArrayBuffer та Atomics](#sharedarraybuffer-та-atomics)
5. [Передача даних між потоками](#передача-даних-між-потоками)
6. [Масштабування Node.js](#масштабування-nodejs)
7. [PM2](#pm2)
8. [Graceful Shutdown](#graceful-shutdown)

---

## Cluster Module

### Що таке Cluster module і навіщо він потрібен?

Cluster module дозволяє створювати дочірні процеси (workers), які розділяють один серверний порт. Node.js за замовчуванням працює в одному потоці, тому Cluster module дає змогу використовувати всі ядра процесора.

Кожен worker — це окремий процес з власною пам'яттю, Event Loop та V8 інстансом. Головний процес (primary) керує дочірніми процесами та розподіляє вхідні з'єднання між ними.

```js
import cluster from "node:cluster";
import http from "node:http";
import os from "node:os";

const numCPUs = os.availableParallelism();

if (cluster.isPrimary) {
  console.log(`Primary процес ${process.pid} запущено`);

  // Створюємо worker для кожного ядра
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} завершився. Перезапускаємо...`);
    cluster.fork();
  });
} else {
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Відповідь від worker ${process.pid}\n`);
  }).listen(8000);

  console.log(`Worker ${process.pid} запущено`);
}
```

### Як Cluster module розподіляє навантаження між workers?

Node.js використовує два підходи до балансування:

1. **Round-Robin** (за замовчуванням на всіх платформах, окрім Windows) — primary процес приймає з'єднання і розподіляє їх між workers по черзі. Це забезпечує рівномірне навантаження.

2. **Системний підхід** (Windows) — серверний socket передається безпосередньо workers, і операційна система сама вирішує, який worker обробить з'єднання. Це може призводити до нерівномірного розподілу.

```js
// Зміна стратегії балансування
import cluster from "node:cluster";

// Round-Robin
cluster.schedulingPolicy = cluster.SCHED_RR;

// Системний підхід
cluster.schedulingPolicy = cluster.SCHED_NONE;
```

### Як workers спілкуються з primary процесом?

Комунікація відбувається через IPC (Inter-Process Communication) канал за допомогою методів `send()` та події `message`.

```js
import cluster from "node:cluster";

if (cluster.isPrimary) {
  const worker = cluster.fork();

  // Надсилаємо повідомлення worker-у
  worker.send({ type: "task", payload: { userId: 42 } });

  // Отримуємо відповідь від worker-а
  worker.on("message", (msg) => {
    console.log(`Відповідь від worker: ${JSON.stringify(msg)}`);
  });
} else {
  // Worker отримує повідомлення
  process.on("message", (msg) => {
    console.log(`Worker отримав: ${JSON.stringify(msg)}`);

    // Надсилаємо відповідь primary процесу
    process.send({ type: "result", payload: { status: "done" } });
  });
}
```

### Які обмеження має Cluster module?

- Кожен worker — це окремий процес з повною копією пам'яті. Це збільшує споживання RAM.
- Workers не розділяють стан. Потрібен зовнішній сховище (Redis, база даних) для спільних даних.
- IPC комунікація серіалізує дані через JSON, що повільніше за спільну пам'ять.
- Створення нового процесу займає значно більше ресурсів, ніж створення потоку.

---

## Worker Threads

### Що таке Worker Threads і чим вони відрізняються від Cluster?

Worker Threads — це легковагі потоки всередині одного процесу. Вони розділяють адресний простір пам'яті, що дозволяє ефективно обмінюватися даними через SharedArrayBuffer.

Worker Threads підходять для CPU-інтенсивних задач: обчислення, обробка зображень, парсинг великих файлів.

```js
// main.js
import { Worker } from "node:worker_threads";

function runHeavyTask(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker("./heavy-task.js", {
      workerData: data,
    });

    worker.on("message", resolve);
    worker.on("error", reject);
    worker.on("exit", (code) => {
      if (code !== 0) {
        reject(new Error(`Worker завершився з кодом ${code}`));
      }
    });
  });
}

const result = await runHeavyTask({ numbers: [1, 2, 3, 4, 5] });
console.log("Результат:", result);
```

```js
// heavy-task.js
import { parentPort, workerData } from "node:worker_threads";

function heavyComputation(numbers) {
  return numbers.reduce((sum, n) => sum + Math.pow(n, 10), 0);
}

const result = heavyComputation(workerData.numbers);
parentPort.postMessage(result);
```

### Як створити пул Worker Threads?

Створення нового потоку щоразу — дороге. Краще тримати пул готових потоків і розподіляти задачі між ними.

```js
import { Worker } from "node:worker_threads";
import os from "node:os";

class WorkerPool {
  #workers = [];
  #queue = [];
  #activeWorkers = 0;

  constructor(workerPath, poolSize = os.availableParallelism()) {
    this.workerPath = workerPath;
    this.poolSize = poolSize;
  }

  execute(data) {
    return new Promise((resolve, reject) => {
      const task = { data, resolve, reject };

      if (this.#activeWorkers < this.poolSize) {
        this.#runTask(task);
      } else {
        this.#queue.push(task);
      }
    });
  }

  #runTask(task) {
    this.#activeWorkers++;
    const worker = new Worker(this.workerPath, {
      workerData: task.data,
    });

    worker.on("message", (result) => {
      task.resolve(result);
      this.#activeWorkers--;
      this.#processQueue();
    });

    worker.on("error", (err) => {
      task.reject(err);
      this.#activeWorkers--;
      this.#processQueue();
    });
  }

  #processQueue() {
    if (this.#queue.length > 0 && this.#activeWorkers < this.poolSize) {
      const nextTask = this.#queue.shift();
      this.#runTask(nextTask);
    }
  }
}

// Використання
const pool = new WorkerPool("./heavy-task.js", 4);

const results = await Promise.all([
  pool.execute({ task: "compute", input: 100 }),
  pool.execute({ task: "compute", input: 200 }),
  pool.execute({ task: "compute", input: 300 }),
]);
```

---

## Cluster vs Worker Threads — порівняння

| Критерій | Cluster | Worker Threads |
|---|---|---|
| Ізоляція | Окремі процеси | Потоки в одному процесі |
| Пам'ять | Окрема для кожного worker | Спільний адресний простір |
| Комунікація | IPC (серіалізація) | MessagePort, SharedArrayBuffer |
| Накладні витрати | Високі (fork процесу) | Низькі (створення потоку) |
| Стабільність | Падіння worker не вбиває primary | Падіння потоку може вплинути на процес |
| Основний сценарій | Масштабування HTTP серверів | CPU-інтенсивні обчислення |
| Доступ до socket | Так (спільний порт) | Ні |
| Спільна пам'ять | Ні | Так (SharedArrayBuffer) |

### Коли використовувати Cluster, а коли Worker Threads?

**Cluster** — коли потрібно масштабувати HTTP сервер на всі ядра процесора. Кожен worker обробляє запити незалежно.

**Worker Threads** — коли потрібно виконати важкі обчислення, не блокуючи основний Event Loop. Наприклад: обробка зображень, шифрування, парсинг великих CSV файлів.

Ці два підходи можна комбінувати: Cluster для масштабування серверу, а Worker Threads всередині кожного worker для CPU-задач.

---

## SharedArrayBuffer та Atomics

### Що таке SharedArrayBuffer?

SharedArrayBuffer — це буфер пам'яті, який можна розділити між потоками без копіювання. Це найшвидший спосіб обміну даними між Worker Threads.

```js
// main.js
import { Worker } from "node:worker_threads";

// Створюємо спільний буфер на 4 цілих числа (4 * 4 байти)
const sharedBuffer = new SharedArrayBuffer(16);
const sharedArray = new Int32Array(sharedBuffer);

sharedArray[0] = 0; // Лічильник

const worker = new Worker("./counter-worker.js", {
  workerData: { sharedBuffer },
});

worker.on("exit", () => {
  console.log(`Фінальне значення лічильника: ${sharedArray[0]}`);
});
```

### Навіщо потрібні Atomics?

Коли декілька потоків одночасно змінюють спільну пам'ять, виникають race conditions. Atomics забезпечують атомарні операції — вони виконуються як єдине ціле, без переривань.

```js
// counter-worker.js
import { workerData } from "node:worker_threads";

const sharedArray = new Int32Array(workerData.sharedBuffer);

// Атомарне збільшення лічильника (без race conditions)
for (let i = 0; i < 1000; i++) {
  Atomics.add(sharedArray, 0, 1);
}

// Очікування на зміну значення
Atomics.wait(sharedArray, 0, 0); // Заблокує потік, поки значення дорівнює 0

// Пробудження іншого потоку
Atomics.notify(sharedArray, 0, 1); // Пробудити один потік, що очікує
```

### Які атомарні операції доступні?

- `Atomics.add()` — атомарне додавання
- `Atomics.sub()` — атомарне віднімання
- `Atomics.load()` — атомарне читання
- `Atomics.store()` — атомарний запис
- `Atomics.compareExchange()` — порівняти і замінити (CAS операція)
- `Atomics.wait()` — заблокувати потік до зміни значення
- `Atomics.notify()` — пробудити заблоковані потоки

---

## Передача даних між потоками

### Які способи передачі даних існують?

1. **Structured Clone** (за замовчуванням) — глибоке копіювання об'єкта. Безпечно, але повільно для великих даних.

2. **Transfer** — передача власності на об'єкт. Оригінал стає недоступним. Швидко, без копіювання.

3. **SharedArrayBuffer** — спільна пам'ять. Найшвидше, але потребує синхронізації.

```js
import { Worker } from "node:worker_threads";

const worker = new Worker("./worker.js");

// 1. Structured Clone (копіювання)
worker.postMessage({ name: "Тест", data: [1, 2, 3] });

// 2. Transfer (передача власності)
const buffer = new ArrayBuffer(1024);
worker.postMessage(buffer, [buffer]);
// buffer тепер недоступний у основному потоці!
console.log(buffer.byteLength); // 0

// 3. SharedArrayBuffer (спільна пам'ять)
const shared = new SharedArrayBuffer(1024);
worker.postMessage({ shared });
// shared залишається доступним в обох потоках
```

### Що таке MessageChannel і навіщо він потрібен?

MessageChannel створює пару зв'язаних портів для прямої комунікації між двома Worker Threads, минаючи основний потік.

```js
import { Worker, MessageChannel } from "node:worker_threads";

const worker1 = new Worker("./worker1.js");
const worker2 = new Worker("./worker2.js");

// Створюємо канал зв'язку
const { port1, port2 } = new MessageChannel();

// Передаємо по одному порту кожному worker
worker1.postMessage({ port: port1 }, [port1]);
worker2.postMessage({ port: port2 }, [port2]);

// Тепер worker1 і worker2 спілкуються напряму
```

---

## Масштабування Node.js

### Які є стратегії масштабування?

1. **Вертикальне масштабування** — збільшення ресурсів одного сервера (CPU, RAM). Cluster module допомагає використати всі ядра.

2. **Горизонтальне масштабування** — додавання нових серверів. Потребує Load Balancer (Nginx, HAProxy) та спільне сховище сесій (Redis).

3. **Мікросервісна архітектура** — розділення додатку на незалежні сервіси, кожен масштабується окремо.

### Що враховувати при масштабуванні?

- **Stateless** — додаток не повинен зберігати стан у пам'яті процесу. Сесії, кеш, тимчасові дані — все у зовнішніх сховищах.
- **Sticky Sessions** — якщо стан неможливо винести, потрібно прив'язувати клієнта до конкретного worker.
- **Shared Nothing** — кожен інстанс має бути незалежним.

---

## PM2

### Що таке PM2 і як він допомагає?

PM2 — це production process manager для Node.js. Він забезпечує: cluster mode, автоматичний перезапуск при падінні, моніторинг, zero-downtime перезавантаження.

```bash
# Запуск у cluster mode з використанням усіх ядер
pm2 start app.js -i max

# Запуск з конкретною кількістю інстансів
pm2 start app.js -i 4

# Zero-downtime перезавантаження
pm2 reload app.js

# Моніторинг
pm2 monit

# Логи
pm2 logs
```

### Як налаштувати PM2 через конфігураційний файл?

```js
// ecosystem.config.cjs
module.exports = {
  apps: [{
    name: "api-server",
    script: "./src/index.ts",
    instances: "max",
    exec_mode: "cluster",
    max_memory_restart: "500M",
    env_production: {
      NODE_ENV: "production",
      PORT: 3000,
    },
    // Автоматичний перезапуск при падінні
    autorestart: true,
    // Максимальна кількість перезапусків за 15 хвилин
    max_restarts: 10,
    min_uptime: "10s",
    // Graceful shutdown timeout
    kill_timeout: 5000,
    // Прослуховування SIGINT
    listen_timeout: 3000,
  }],
};
```

### Як PM2 реалізує zero-downtime reload?

PM2 використовує послідовний перезапуск workers:

1. Запускається новий worker.
2. PM2 чекає, поки новий worker стане готовим (подія `ready`).
3. Старому worker надсилається SIGINT.
4. Старий worker завершує поточні запити і зупиняється.
5. Процес повторюється для кожного worker.

```js
// Сигналізуємо PM2, що worker готовий
process.send("ready");

// Обробка graceful shutdown
process.on("SIGINT", () => {
  console.log("Завершуємо поточні запити...");
  server.close(() => {
    console.log("Сервер зупинено.");
    process.exit(0);
  });
});
```

---

## Graceful Shutdown

### Що таке Graceful Shutdown і чому це важливо?

Graceful Shutdown — це коректне завершення роботи додатку. Сервер перестає приймати нові запити, дочекується завершення поточних, закриває підключення до бази даних, очищує ресурси і лише потім завершує процес.

Без Graceful Shutdown: обрив запитів, втрата даних, незакриті з'єднання з базою, пошкоджені транзакції.

### Як реалізувати Graceful Shutdown?

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end("OK");
});

server.listen(3000);

let isShuttingDown = false;

async function gracefulShutdown(signal) {
  if (isShuttingDown) return;
  isShuttingDown = true;

  console.log(`Отримано сигнал ${signal}. Починаємо graceful shutdown...`);

  // 1. Перестаємо приймати нові з'єднання
  server.close(() => {
    console.log("HTTP сервер закрито.");
  });

  // 2. Встановлюємо таймаут на примусове завершення
  const forceShutdownTimeout = setTimeout(() => {
    console.error("Примусове завершення через таймаут!");
    process.exit(1);
  }, 30_000);

  forceShutdownTimeout.unref();

  try {
    // 3. Завершуємо поточні задачі
    await finishPendingRequests();

    // 4. Закриваємо підключення до бази даних
    await closeDatabaseConnections();

    // 5. Закриваємо з'єднання з Redis
    await closeRedisConnection();

    console.log("Graceful shutdown завершено успішно.");
    process.exit(0);
  } catch (error) {
    console.error("Помилка під час shutdown:", error);
    process.exit(1);
  }
}

// Обробляємо сигнали операційної системи
process.on("SIGTERM", () => gracefulShutdown("SIGTERM"));
process.on("SIGINT", () => gracefulShutdown("SIGINT"));
```

### Як реалізувати Graceful Shutdown у Cluster mode?

```js
import cluster from "node:cluster";
import os from "node:os";

if (cluster.isPrimary) {
  const numCPUs = os.availableParallelism();

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  async function shutdownAllWorkers() {
    const workers = Object.values(cluster.workers);

    console.log(`Завершуємо ${workers.length} workers...`);

    // Надсилаємо сигнал завершення кожному worker
    const shutdownPromises = workers.map((worker) => {
      return new Promise((resolve) => {
        worker.on("exit", resolve);
        worker.send("shutdown");

        // Примусове завершення через 10 секунд
        setTimeout(() => {
          if (!worker.isDead()) {
            worker.kill("SIGKILL");
          }
        }, 10_000);
      });
    });

    await Promise.all(shutdownPromises);
    console.log("Всі workers завершено.");
    process.exit(0);
  }

  process.on("SIGTERM", shutdownAllWorkers);
  process.on("SIGINT", shutdownAllWorkers);
} else {
  // Worker: обробляємо повідомлення про завершення
  process.on("message", (msg) => {
    if (msg === "shutdown") {
      console.log(`Worker ${process.pid}: починаю graceful shutdown`);
      server.close(() => {
        process.exit(0);
      });
    }
  });
}
```

### Які сигнали операційної системи потрібно обробляти?

| Сигнал | Опис | Коли надсилається |
|---|---|---|
| `SIGTERM` | Запит на коректне завершення | Docker stop, Kubernetes, PM2 |
| `SIGINT` | Переривання з терміналу | Ctrl+C у консолі |
| `SIGQUIT` | Завершення з core dump | Рідко використовується |
| `SIGHUP` | Від'єднання терміналу | Закриття SSH сесії |

### Яка типова послідовність Graceful Shutdown?

1. Отримати сигнал (`SIGTERM` / `SIGINT`).
2. Встановити прапорець `isShuttingDown = true`.
3. Перестати приймати нові запити (`server.close()`).
4. Повернути `503 Service Unavailable` для нових запитів (якщо є Load Balancer).
5. Дочекатися завершення поточних запитів.
6. Закрити з'єднання з базою даних.
7. Закрити з'єднання з Redis, черги повідомлень тощо.
8. Зробити flush логів.
9. Завершити процес з кодом `0`.

---

## Питання для самоперевірки

1. Чому Cluster module створює процеси, а не потоки?
2. Як SharedArrayBuffer допомагає уникнути серіалізації при обміні даними?
3. У чому різниця між `worker.postMessage()` з transfer list і без?
4. Чому Graceful Shutdown критично важливий у Docker/Kubernetes середовищі?
5. Як PM2 реалізує zero-downtime deployment?
6. Коли краще використати Worker Threads замість child_process?
7. Що станеться, якщо не обробити `SIGTERM` у Docker контейнері?
8. Як `Atomics.wait()` і `Atomics.notify()` реалізують мютекс між потоками?
