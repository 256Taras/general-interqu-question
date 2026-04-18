# Module Systems — Питання для інтерв'ю

## CommonJS vs ESM — різниця

| Аспект | CommonJS | ESM |
|---|---|---|
| Синтаксис | `require` / `module.exports` | `import` / `export` |
| Завантаження | Синхронне | Асинхронне |
| Парсинг | Runtime | Compile time (static) |
| Tree shaking | Неможливе | Можливе |
| Top-level await | Ні | Так |
| `this` верхній рівень | `module.exports` | `undefined` |
| `__dirname` | Доступний | `import.meta.dirname` (Node 21+) |
| Circular deps | Часткові exports | Live bindings |

```javascript
// CommonJS
const { readFile } = require("node:fs/promises");
module.exports = { myFunction };

// ESM
import { readFile } from "node:fs/promises";
export function myFunction() {}
export default class MyService {}
```

---

## Як працює require()? Module Resolution Algorithm

```javascript
require("express");     // 1. Core module? Ні
                        // 2. node_modules/express

require("./utils");     // Відносний шлях:
                        // 1. ./utils.js
                        // 2. ./utils/index.js
                        // 3. ./utils.json
                        // 4. ./utils.node
```

**Пошук node_modules:**
```
/project/src/controllers/user.js → require("lodash")
  1. /project/src/controllers/node_modules/lodash
  2. /project/src/node_modules/lodash
  3. /project/node_modules/lodash
  4. /node_modules/lodash
```

**Кешування:** require кешує модулі. Повторний require повертає той самий об'єкт.
```javascript
require("./module"); // завантажує і кешує
require("./module"); // повертає з кешу
```

---

## Circular Dependencies — як вирішувати?

**Проблема:** модулі A і B імпортують один одного → частковий export.

```javascript
// a.js
const b = require("./b");
console.log(b.value); // undefined! (B ще не завершив export)
module.exports = { value: "A" };

// b.js
const a = require("./a");
console.log(a.value); // undefined!
module.exports = { value: "B" };
```

**Рішення:**
1. **Restructure** — виділити спільну залежність C, A→C, B→C
2. **Lazy require** — `require` всередині функції, не на верхньому рівні
3. **Dependency injection** — передавати залежності як параметри
4. **ESM live bindings** — краще обробляє circular deps

---

## Dynamic imports — import()

```javascript
// Умовний імпорт
if (process.env.NODE_ENV === "development") {
  const devTools = await import("./dev-tools.js");
  devTools.setup();
}

// Lazy loading
async function processImage(file) {
  const sharp = await import("sharp");
  return sharp.default(file).resize(200).toBuffer();
}
```

---

## Top-level Await

Доступний тільки в ESM:
```javascript
// config.js
const response = await fetch("https://config.example.com");
export const config = await response.json();

// app.js — чекає поки config завантажиться
import { config } from "./config.js";
```

---

## package.json "type" field

```json
{ "type": "module" }   // всі .js = ESM, для CJS використовуй .cjs
{ "type": "commonjs" } // всі .js = CJS, для ESM використовуй .mjs
```

| Розширення | Завжди |
|---|---|
| `.mjs` / `.mts` | ESM |
| `.cjs` / `.cts` | CommonJS |
| `.js` / `.ts` | Залежить від "type" |

---

## import.meta

```javascript
import.meta.url;       // "file:///project/src/app.js"
import.meta.dirname;   // "/project/src" (Node 21+)
import.meta.filename;  // "/project/src/app.js" (Node 21+)

// Для старих версій
import { fileURLToPath } from "node:url";
const __filename = fileURLToPath(import.meta.url);
```

---

## Node.js Native TypeScript Support

```bash
node --experimental-strip-types app.ts  # Node 22
node app.ts                             # Node 23+
```

- Node.js **видаляє** типи (не перевіряє!) через swc
- `tsc --noEmit` для перевірки типів окремо
- Використовуй `imports` у package.json замість `paths` з tsconfig:

```json
{
  "imports": {
    "#libs/*": "./src/libs/*",
    "#modules/*": "./src/modules/*"
  }
}
```
