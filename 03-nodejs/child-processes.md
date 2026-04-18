# Child Processes — Питання для інтерв'ю

## exec() vs execFile() vs spawn() vs fork()

| Метод | Повертає | Buffer/Stream | Shell | Use case |
| --- | --- | --- | --- | --- |
| `exec` | buffer | Buffer (весь output) | Так | Прості команди з малим output |
| `execFile` | buffer | Buffer | Ні | Безпечніший exec (без shell) |
| `spawn` | ChildProcess | Stream | Ні | Великий output, довгі процеси |
| `fork` | ChildProcess | Stream + IPC | Ні | Node.js процеси з IPC |

```javascript
import { exec, execFile, spawn, fork } from "node:child_process";

// exec — через shell, весь output у buffer
exec("ls -la | grep .js", (err, stdout, stderr) => {
  console.log(stdout);
});

// execFile — без shell, безпечніший
execFile("node", ["--version"], (err, stdout) => {
  console.log(stdout); // v22.0.0
});

// spawn — streaming output
const child = spawn("find", [".", "-name", "*.ts"]);
child.stdout.on("data", (data) => console.log(data.toString()));
child.on("close", (code) => console.log(`Exit: ${code}`));

// fork — Node.js процес з IPC каналом
const worker = fork("./worker.js");
worker.send({ task: "process", data: [1, 2, 3] });
worker.on("message", (result) => console.log(result));
```

---

## IPC (Inter-Process Communication)

```javascript
// parent.js
const child = fork("./child.js");
child.send({ type: "START", payload: { userId: 123 } });
child.on("message", (msg) => {
  console.log("From child:", msg);
});

// child.js
process.on("message", (msg) => {
  if (msg.type === "START") {
    const result = heavyComputation(msg.payload);
    process.send({ type: "RESULT", data: result });
  }
});
```

**Обмеження IPC:**
- Дані серіалізуються (JSON) — не можна передати functions, циклічні об'єкти
- Overhead серіалізації для великих об'єктів
- Для великих даних — SharedArrayBuffer (Worker Threads) або файли/streams

---

## Сигнали процесів

```javascript
// SIGTERM — "завершись коректно" (default від kill, Docker stop)
process.on("SIGTERM", () => {
  console.log("Graceful shutdown...");
  server.close(() => process.exit(0));
});

// SIGINT — Ctrl+C
process.on("SIGINT", () => {
  console.log("Interrupted");
  process.exit(0);
});

// SIGKILL — "вбий зараз" (не можна перехопити!)
// kill -9 pid — неможливо обробити

// SIGHUP — terminal closed, часто для reload config
process.on("SIGHUP", () => reloadConfig());
```

**Відправка сигналів child process:**
```javascript
const child = spawn("long-task");
child.kill("SIGTERM"); // м'яке завершення
// Якщо не завершився через 5 сек:
setTimeout(() => child.kill("SIGKILL"), 5000); // примусове
```

---

## Pipe between Processes

```javascript
// ls | grep .ts | wc -l
const ls = spawn("ls", ["-la"]);
const grep = spawn("grep", [".ts"]);
const wc = spawn("wc", ["-l"]);

ls.stdout.pipe(grep.stdin);
grep.stdout.pipe(wc.stdin);

wc.stdout.on("data", (data) => {
  console.log(`TypeScript files: ${data.toString().trim()}`);
});
```

---

## Коли використовувати Child Processes?

**Використовуй коли:**
- Потрібно запустити зовнішню програму (ffmpeg, ImageMagick)
- CPU-intensive задача і Worker Threads не підходять
- Потрібна повна ізоляція (crash не впливає на основний процес)
- Legacy код на іншій мові

**НЕ використовуй коли:**
- Можна вирішити async I/O
- Worker Threads достатньо (CPU tasks у Node.js)
- Потрібна shared memory (Worker Threads краще)
