# Streams в Node.js

## Які типи потоків є в Node.js?

Node.js має чотири основних типи streams, кожен з яких наслідує від базового класу `EventEmitter`:

### 1. Readable Stream (Потік для читання)

Джерело даних, з якого можна читати. Працює у двох режимах: flowing та paused.

```javascript
import { createReadStream } from "node:fs";
import { Readable } from "node:stream";

// Приклад 1: Читання файлу
const fileStream = createReadStream("large-file.txt", {
  encoding: "utf-8",
  highWaterMark: 64 * 1024, // 64KB буфер
});

fileStream.on("data", (chunk) => {
  console.log(`Отримано ${chunk.length} байт`);
});

fileStream.on("end", () => {
  console.log("Файл прочитано повністю");
});

// Приклад 2: Власний Readable stream
const customReadable = new Readable({
  read(size) {
    this.push("Порція даних\n");
    this.push(null); // Сигнал завершення
  },
});
```

### 2. Writable Stream (Потік для запису)

Призначення даних, куди можна записувати.

```javascript
import { createWriteStream } from "node:fs";
import { Writable } from "node:stream";

// Приклад 1: Запис у файл
const fileWriter = createWriteStream("output.txt");
fileWriter.write("Рядок 1\n");
fileWriter.write("Рядок 2\n");
fileWriter.end("Останній рядок\n");

// Приклад 2: Власний Writable stream
const customWritable = new Writable({
  write(chunk, encoding, callback) {
    console.log(`Записано: ${chunk.toString()}`);
    callback(); // Сигналізує про готовність приймати наступний chunk
  },
});
```

### 3. Duplex Stream (Двонаправлений потік)

Поєднує Readable та Writable. Може одночасно читати та записувати.

```javascript
import { Duplex } from "node:stream";
import net from "node:net";

// TCP socket -- класичний приклад Duplex stream
const server = net.createServer((socket) => {
  // socket є Duplex: можна читати з нього і писати в нього
  socket.on("data", (data) => {
    socket.write(`Echo: ${data}`);
  });
});

// Власний Duplex stream
const customDuplex = new Duplex({
  read(size) {
    this.push("Дані для читання");
    this.push(null);
  },
  write(chunk, encoding, callback) {
    console.log(`Отримано для запису: ${chunk.toString()}`);
    callback();
  },
});
```

### 4. Transform Stream (Потік трансформації)

Спеціальний вид Duplex, де вихід обчислюється на основі входу.

```javascript
import { Transform } from "node:stream";
import { createGzip } from "node:zlib";

// zlib.createGzip() -- вбудований Transform stream
// Він читає нестиснуті дані та виводить стиснуті

// Власний Transform stream
const upperCaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    const upperCased = chunk.toString().toUpperCase();
    callback(null, upperCased); // Перший аргумент -- помилка, другий -- результат
  },
});

// Використання
process.stdin
  .pipe(upperCaseTransform)
  .pipe(process.stdout);
```

### Порівняльна таблиця

| Тип | Читання | Запис | Приклад |
|-----|---------|-------|---------|
| Readable | Так | Ні | `fs.createReadStream`, `http.IncomingMessage` |
| Writable | Ні | Так | `fs.createWriteStream`, `http.ServerResponse` |
| Duplex | Так | Так | `net.Socket`, `crypto.Cipher` |
| Transform | Так | Так | `zlib.createGzip`, `crypto.createHash` |

---

## Що таке backpressure і як з ним працювати?

Backpressure -- це механізм, який запобігає переповненню пам'яті, коли швидкість виробника (Readable) перевищує швидкість споживача (Writable).

```
Без backpressure:
Readable (швидкий) ──────────▶ [████████████ БУФЕР ПЕРЕПОВНЕНИЙ] ──▶ Writable (повільний)
                                           OOM!

З backpressure:
Readable (пауза) ─ ─ ─ ─ ─ ▶ [████ БУФЕР OK] ──▶ Writable (повільний)
                 ◀── "зачекай!" ──┘
```

### Як працює backpressure

1. Кожен stream має внутрішній буфер розміром `highWaterMark` (за замовчуванням 16KB)
2. Коли `writable.write()` повертає `false`, буфер заповнений
3. Readable повинен зупинити відправку даних
4. Коли буфер звільниться, Writable емітить подію `drain`
5. Readable відновлює відправку даних

```javascript
import { createReadStream, createWriteStream } from "node:fs";

const readable = createReadStream("huge-file.txt");
const writable = createWriteStream("output.txt");

// НЕПРАВИЛЬНИЙ спосіб -- ігнорування backpressure
readable.on("data", (chunk) => {
  writable.write(chunk); // Ігноруємо return value -- можливе переповнення пам'яті!
});

// ПРАВИЛЬНИЙ спосіб -- ручна обробка backpressure
readable.on("data", (chunk) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    // Буфер заповнений, ставимо на паузу
    readable.pause();
  }
});

writable.on("drain", () => {
  // Буфер звільнився, продовжуємо читання
  readable.resume();
});

readable.on("end", () => {
  writable.end();
});
```

```javascript
// НАЙКРАЩИЙ спосіб -- використовувати pipeline() або pipe()
import { pipeline } from "node:stream/promises";

await pipeline(
  createReadStream("huge-file.txt"),
  createWriteStream("output.txt"),
);
// pipeline автоматично обробляє backpressure та помилки
```

---

## pipeline() vs pipe() - різниця

### pipe()

Простий метод для з'єднання streams. Має суттєвий недолік -- не обробляє помилки автоматично.

```javascript
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";

// pipe() -- старий підхід
const source = createReadStream("input.txt");
const gzip = createGzip();
const destination = createWriteStream("output.txt.gz");

source
  .pipe(gzip)
  .pipe(destination);

// Проблема: якщо помилка в gzip, source НЕ буде знищений автоматично
// Потрібно вручну обробляти помилки кожного stream
source.on("error", handleError);
gzip.on("error", handleError);
destination.on("error", handleError);

function handleError(err) {
  console.error("Помилка:", err);
  source.destroy();
  gzip.destroy();
  destination.destroy();
}
```

### pipeline()

Рекомендований підхід. Автоматично обробляє помилки та знищує streams.

```javascript
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";
import { createGzip } from "node:zlib";

// pipeline() -- сучасний підхід
try {
  await pipeline(
    createReadStream("input.txt"),
    createGzip(),
    createWriteStream("output.txt.gz"),
  );
  console.log("Стиснення завершено");
} catch (err) {
  console.error("Помилка під час стиснення:", err);
  // Всі streams автоматично знищені
}
```

### Порівняння

| Характеристика | `pipe()` | `pipeline()` |
|---|---|---|
| Обробка помилок | Вручну для кожного stream | Автоматична |
| Знищення streams | Вручну | Автоматичне |
| Promise-based | Ні | Так (`stream/promises`) |
| Callback-based | Ні | Так (`stream.pipeline`) |
| Рекомендований | Ні | Так |
| AbortSignal | Ні | Так |

```javascript
// pipeline з AbortController
import { pipeline } from "node:stream/promises";

const ac = new AbortController();

setTimeout(() => ac.abort(), 5000); // Скасувати через 5 секунд

try {
  await pipeline(
    createReadStream("very-large-file.txt"),
    createWriteStream("output.txt"),
    { signal: ac.signal },
  );
} catch (err) {
  if (err.name === "AbortError") {
    console.log("Операцію скасовано");
  }
}
```

---

## Як створити власний Transform stream?

```javascript
import { Transform } from "node:stream";

// Спосіб 1: Через конструктор
const csvToJson = new Transform({
  objectMode: true,
  transform(chunk, encoding, callback) {
    const line = chunk.toString().trim();
    const [name, age, city] = line.split(",");

    const jsonObject = { name, age: Number(age), city };
    callback(null, jsonObject);
  },
});

// Спосіб 2: Через наслідування класу
class FilterTransform extends Transform {
  #predicate;

  constructor(predicate) {
    super({ objectMode: true });
    this.#predicate = predicate;
  }

  _transform(chunk, encoding, callback) {
    if (this.#predicate(chunk)) {
      this.push(chunk);
    }
    callback();
  }

  _flush(callback) {
    // Викликається коли всі дані оброблені
    // Тут можна відправити фінальні дані
    callback();
  }
}

// Використання
const adultFilter = new FilterTransform((person) => person.age >= 18);

// Спосіб 3: Через stream.compose (Node.js 18+)
import { compose } from "node:stream";

const transformPipeline = compose(
  csvToJson,
  adultFilter,
);
```

### Практичний приклад: JSON Line Parser

```javascript
import { Transform } from "node:stream";

class JsonLineParser extends Transform {
  #buffer = "";

  constructor() {
    super({ readableObjectMode: true });
  }

  _transform(chunk, encoding, callback) {
    this.#buffer += chunk.toString();
    const lines = this.#buffer.split("\n");

    // Зберігаємо останній неповний рядок у буфері
    this.#buffer = lines.pop();

    for (const line of lines) {
      if (line.trim()) {
        try {
          this.push(JSON.parse(line));
        } catch (err) {
          this.destroy(new Error(`Невалідний JSON: ${line}`));
          return;
        }
      }
    }
    callback();
  }

  _flush(callback) {
    if (this.#buffer.trim()) {
      try {
        this.push(JSON.parse(this.#buffer));
      } catch (err) {
        return callback(new Error(`Невалідний JSON: ${this.#buffer}`));
      }
    }
    callback();
  }
}
```

---

## Коли використовувати Streams замість буферизації?

### Використовуйте Streams коли:

1. **Великі файли** -- якщо файл не поміщається в пам'ять
2. **Поточна обробка** -- дані можна обробляти порціями
3. **Time to First Byte** -- потрібно швидко почати відправку
4. **ETL процеси** -- Extract, Transform, Load

```javascript
// ПОГАНО: Завантаження всього файлу в пам'ять
import { readFile } from "node:fs/promises";

const data = await readFile("10gb-file.csv", "utf-8"); // OOM!
const lines = data.split("\n");
for (const line of lines) {
  processLine(line);
}

// ДОБРЕ: Поточна обробка через streams
import { createReadStream } from "node:fs";
import { createInterface } from "node:readline";

const lineReader = createInterface({
  input: createReadStream("10gb-file.csv"),
  crlfDelay: Infinity,
});

for await (const line of lineReader) {
  processLine(line);
}
```

### Використовуйте буферизацію коли:

1. **Малий обсяг даних** -- файл легко поміщається в пам'ять
2. **Потрібен повний вміст** -- наприклад, JSON.parse
3. **Простота важливіша** -- streams додають складності

```javascript
// Для малих файлів буферизація простіша
import { readFile } from "node:fs/promises";

const config = JSON.parse(await readFile("config.json", "utf-8"));
```

---

## Object mode streams

За замовчуванням streams працюють з `Buffer` або `string`. Object mode дозволяє передавати будь-які JavaScript-об'єкти.

```javascript
import { Transform, Readable, Writable } from "node:stream";

// Readable в object mode
const usersStream = new Readable({
  objectMode: true,
  read() {
    this.push({ id: 1, name: "Олексій" });
    this.push({ id: 2, name: "Марія" });
    this.push(null); // Кінець потоку
  },
});

// Transform в object mode
const enrichUser = new Transform({
  objectMode: true,
  transform(user, encoding, callback) {
    user.enrichedAt = new Date().toISOString();
    user.role = "user";
    callback(null, user);
  },
});

// Writable в object mode
const saveToDb = new Writable({
  objectMode: true,
  async write(user, encoding, callback) {
    await db.insert(user);
    callback();
  },
});

// З'єднуємо
import { pipeline } from "node:stream/promises";

await pipeline(usersStream, enrichUser, saveToDb);
```

### Важливі особливості object mode

- `highWaterMark` означає кількість об'єктів (не байт). За замовчуванням 16 об'єктів
- Неможливо змішувати object mode і byte mode в одному stream
- `readableObjectMode` та `writableObjectMode` дозволяють різні режими для Duplex/Transform

```javascript
// Transform з різними режимами: byte -> object
const parser = new Transform({
  writableObjectMode: false,  // Вхід -- bytes
  readableObjectMode: true,   // Вихід -- objects
  transform(chunk, encoding, callback) {
    const obj = JSON.parse(chunk.toString());
    callback(null, obj);
  },
});
```

---

## Stream events

### Readable stream events

```javascript
import { createReadStream } from "node:fs";

const readable = createReadStream("file.txt");

// data -- кожен chunk даних
readable.on("data", (chunk) => {
  console.log(`Chunk: ${chunk.length} bytes`);
});

// end -- всі дані прочитані
readable.on("end", () => {
  console.log("Читання завершено");
});

// error -- помилка під час читання
readable.on("error", (err) => {
  console.error("Помилка читання:", err);
});

// close -- ресурс закрито (файловий дескриптор)
readable.on("close", () => {
  console.log("Ресурс закрито");
});

// readable -- є дані для читання (paused mode)
readable.on("readable", () => {
  let chunk;
  while ((chunk = readable.read()) !== null) {
    console.log(`Прочитано: ${chunk.length} bytes`);
  }
});
```

### Writable stream events

```javascript
import { createWriteStream } from "node:fs";

const writable = createWriteStream("output.txt");

// drain -- буфер звільнився після переповнення
writable.on("drain", () => {
  console.log("Буфер звільнився, можна продовжувати запис");
});

// finish -- всі дані записані (після .end())
writable.on("finish", () => {
  console.log("Всі дані записані на диск");
});

// error -- помилка під час запису
writable.on("error", (err) => {
  console.error("Помилка запису:", err);
});

// close -- ресурс закрито
writable.on("close", () => {
  console.log("Файл закрито");
});

// pipe -- readable підключився до цього writable
writable.on("pipe", (source) => {
  console.log("Новий джерельний потік підключено");
});
```

### Повний життєвий цикл

```
Readable:  [data] → [data] → [data] → [end] → [close]
                                          │
Writable:  [write] → [write] → [drain] → [finish] → [close]
```

---

## Практичні задачі зі streams

### Задача 1: Обробка великого CSV-файлу

```javascript
import { createReadStream, createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";
import { Transform } from "node:stream";
import { createInterface } from "node:readline";

async function processLargeCsv(inputPath, outputPath) {
  let isHeader = true;
  let headers = [];

  const csvToJson = new Transform({
    objectMode: true,
    transform(line, encoding, callback) {
      const values = line.toString().split(",");

      if (isHeader) {
        headers = values;
        isHeader = false;
        return callback();
      }

      const obj = {};
      headers.forEach((header, i) => {
        obj[header.trim()] = values[i]?.trim();
      });

      callback(null, obj);
    },
  });

  const filterActive = new Transform({
    objectMode: true,
    transform(record, encoding, callback) {
      if (record.status === "active") {
        callback(null, record);
      } else {
        callback();
      }
    },
  });

  const jsonToLine = new Transform({
    objectMode: true,
    transform(obj, encoding, callback) {
      callback(null, JSON.stringify(obj) + "\n");
    },
  });

  const lineReader = createInterface({
    input: createReadStream(inputPath),
    crlfDelay: Infinity,
  });

  const output = createWriteStream(outputPath);

  for await (const line of lineReader) {
    csvToJson.write(line);
  }
  csvToJson.end();

  await pipeline(csvToJson, filterActive, jsonToLine, output);
}
```

### Задача 2: HTTP-проксі з трансформацією

```javascript
import http from "node:http";
import { Transform } from "node:stream";

const injectHeader = new Transform({
  transform(chunk, encoding, callback) {
    const modified = chunk
      .toString()
      .replace("</head>", '<meta name="proxy" content="true"></head>');
    callback(null, modified);
  },
});

const proxy = http.createServer((req, res) => {
  const options = {
    hostname: "target-server.com",
    path: req.url,
    method: req.method,
    headers: req.headers,
  };

  const proxyReq = http.request(options, (proxyRes) => {
    res.writeHead(proxyRes.statusCode, proxyRes.headers);

    if (proxyRes.headers["content-type"]?.includes("text/html")) {
      proxyRes.pipe(injectHeader).pipe(res);
    } else {
      proxyRes.pipe(res);
    }
  });

  req.pipe(proxyReq);
});
```

### Задача 3: ETL з батчінгом

```javascript
import { Transform } from "node:stream";

class BatchTransform extends Transform {
  #batch = [];
  #batchSize;

  constructor(batchSize = 100) {
    super({ objectMode: true });
    this.#batchSize = batchSize;
  }

  _transform(item, encoding, callback) {
    this.#batch.push(item);

    if (this.#batch.length >= this.#batchSize) {
      this.push([...this.#batch]);
      this.#batch = [];
    }

    callback();
  }

  _flush(callback) {
    if (this.#batch.length > 0) {
      this.push([...this.#batch]);
    }
    callback();
  }
}

// Використання: запис у базу батчами
const batcher = new BatchTransform(500);
const dbWriter = new Writable({
  objectMode: true,
  async write(batch, encoding, callback) {
    await db.insertMany(batch); // Вставляємо 500 записів за раз
    callback();
  },
});
```
