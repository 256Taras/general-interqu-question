# Buffers та Binary Data — Питання для інтерв'ю

## Що таке Buffer в Node.js?

Buffer — клас для роботи з бінарними даними (raw memory). Виділяється **поза V8 heap** (external memory).

```javascript
// Створення
const buf1 = Buffer.alloc(10);               // 10 bytes, заповнені 0
const buf2 = Buffer.allocUnsafe(10);          // 10 bytes, НЕ очищені (швидше)
const buf3 = Buffer.from("Hello", "utf-8");   // з рядка
const buf4 = Buffer.from([0x48, 0x65, 0x6c]); // з масиву байтів

// Читання
buf3.toString("utf-8");     // "Hello"
buf3.toString("base64");    // "SGVsbG8="
buf3.toString("hex");       // "48656c6c6f"
buf3[0];                    // 72 (ASCII код "H")

// Порівняння
Buffer.compare(buf1, buf2); // -1, 0, або 1
buf1.equals(buf2);          // true/false
```

---

## Buffer vs ArrayBuffer vs TypedArray

| Тип | Середовище | Опис |
| --- | --- | --- |
| `Buffer` | Node.js | Підклас Uint8Array, зручний API |
| `ArrayBuffer` | Browser + Node | Raw binary data, без методів |
| `TypedArray` | Browser + Node | View на ArrayBuffer (Int32Array, Float64Array) |

```javascript
// ArrayBuffer + TypedArray
const ab = new ArrayBuffer(16);        // 16 bytes
const view = new Int32Array(ab);       // 4 int32 числа
view[0] = 42;

// Buffer → ArrayBuffer
const buf = Buffer.from("hello");
const arrayBuffer = buf.buffer;        // underlying ArrayBuffer

// ArrayBuffer → Buffer
const buf2 = Buffer.from(arrayBuffer);
```

---

## alloc() vs allocUnsafe()

```javascript
Buffer.alloc(1024);       // Заповнений нулями, безпечний, повільніший
Buffer.allocUnsafe(1024); // НЕ очищений, може містити старі дані, швидший
```

**Коли allocUnsafe безпечний:**
- Одразу заповнюєш весь buffer (read file, network data)
- Performance критичний і ти контролюєш вміст

**Коли НЕ використовувати allocUnsafe:**
- Buffer може бути повернутий користувачу без заповнення
- Security-sensitive контекст

---

## Encoding

```javascript
const buf = Buffer.from("Привіт", "utf-8");

// Різні encoding
buf.toString("utf-8");    // "Привіт"
buf.toString("base64");   // base64 рядок
buf.toString("hex");      // hex рядок
buf.toString("ascii");    // тільки ASCII символи
buf.toString("latin1");   // ISO-8859-1

// Base64 encode/decode
const encoded = Buffer.from("Hello World").toString("base64");
const decoded = Buffer.from(encoded, "base64").toString("utf-8");
```

---

## Практичні задачі

### Читання бінарного файлу
```javascript
import { createReadStream } from "node:fs";

const chunks = [];
const stream = createReadStream("image.png");
stream.on("data", (chunk) => chunks.push(chunk));
stream.on("end", () => {
  const buffer = Buffer.concat(chunks);
  console.log(`Size: ${buffer.length} bytes`);
  console.log(`PNG?: ${buffer[0] === 0x89 && buffer.toString("ascii", 1, 4) === "PNG"}`);
});
```

### Hash обчислення
```javascript
import { createHash } from "node:crypto";

function sha256(data) {
  return createHash("sha256").update(data).digest("hex");
}

const hash = sha256(Buffer.from("password123"));
```

### Конвертація між форматами
```javascript
// Hex → Buffer → Base64
const hex = "48656c6c6f";
const buf = Buffer.from(hex, "hex");
const base64 = buf.toString("base64"); // "SGVsbG8="
const text = buf.toString("utf-8");     // "Hello"
```
