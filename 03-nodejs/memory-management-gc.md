# Memory Management та GC — Питання для інтерв'ю

## Як працює Garbage Collector у V8?

V8 використовує **generational garbage collection** — розділяє heap на два покоління:

### Young Generation (New Space)
- Маленький (1-8 MB)
- Нові об'єкти створюються тут
- **Scavenge** алгоритм (copying GC)
- Два semi-spaces: From і To
- Швидкий, але часто запускається

```
From Space:  [obj1] [obj2] [obj3] [dead] [obj4]
                ↓      ↓              ↓
To Space:    [obj1] [obj2]         [obj4]
```

Об'єкти, що пережили 2 збирання → переходять у Old Generation.

### Old Generation (Old Space)
- Великий (сотні MB — кілька GB)
- Довгоживучі об'єкти
- **Mark-Sweep-Compact** алгоритм:
  1. **Mark** — обходить граф об'єктів від roots, позначає живі
  2. **Sweep** — видаляє непозначені (мертві)
  3. **Compact** — ущільнює пам'ять (дефрагментація)
- Повільніший, запускається рідше

### Incremental та Concurrent GC
- **Incremental marking** — розбиває Mark на маленькі кроки
- **Concurrent marking** — працює у фоновому потоці
- **Lazy sweeping** — очищає по мірі необхідності
- Мета: зменшити паузи (stop-the-world)

---

## Heap vs Stack

| Stack | Heap |
| --- | --- |
| Примітиви (number, string, boolean) | Об'єкти, масиви, функції |
| Фіксований розмір | Динамічний розмір |
| LIFO (Last In First Out) | Довільний доступ |
| Автоматичне очищення (вихід із scope) | Garbage Collector |
| Дуже швидкий | Повільніший |

```javascript
function example() {
  const num = 42;          // Stack
  const str = "hello";     // Stack (reference) → Heap (value)
  const obj = { x: 1 };   // Stack (reference) → Heap (object)
  // Після виходу з функції:
  // - num видаляється зі Stack
  // - obj reference видаляється зі Stack
  // - { x: 1 } на Heap — чекає на GC
}
```

---

## Що таке Memory Leak і як знайти?

**Memory leak** — коли об'єкт більше не потрібен, але GC не може його видалити (є посилання).

### Типові причини

**1. Забуті Event Listeners:**
```javascript
// ❌ Leak — listener не видаляється
function createHandler() {
  const bigData = new Array(1000000);
  emitter.on("event", () => process(bigData));
}
// bigData ніколи не буде зібрано GC

// ✅ Правильно
function createHandler() {
  const bigData = new Array(1000000);
  const handler = () => process(bigData);
  emitter.on("event", handler);
  return () => emitter.removeListener("event", handler); // cleanup
}
```

**2. Closures що тримають великі об'єкти:**
```javascript
// ❌ Leak
function getHandler() {
  const hugeArray = new Array(1000000);
  return function() {
    // hugeArray живе поки живе ця функція
    return hugeArray.length;
  };
}

// ✅ Витягни тільки потрібне
function getHandler() {
  const hugeArray = new Array(1000000);
  const length = hugeArray.length; // копіюємо потрібне
  return function() { return length; };
}
```

**3. Global variables:**
```javascript
// ❌ Leak — global ніколи не очищається
global.cache = {};
function addToCache(key, value) {
  global.cache[key] = value; // росте безкінечно
}

// ✅ Обмежений кеш
const cache = new Map();
const MAX_SIZE = 1000;
function addToCache(key, value) {
  if (cache.size >= MAX_SIZE) {
    const firstKey = cache.keys().next().value;
    cache.delete(firstKey);
  }
  cache.set(key, value);
}
```

**4. Незавершені таймери:**
```javascript
// ❌ Leak — interval ніколи не очищається
setInterval(() => fetchData(), 5000);

// ✅ Зберігаємо reference для cleanup
const timer = setInterval(() => fetchData(), 5000);
// При shutdown:
clearInterval(timer);
```

---

## Як профілювати пам'ять?

### 1. process.memoryUsage()
```javascript
const usage = process.memoryUsage();
console.log({
  rss: `${Math.round(usage.rss / 1024 / 1024)} MB`,       // Resident Set Size
  heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)} MB`,
  heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)} MB`,
  external: `${Math.round(usage.external / 1024 / 1024)} MB`,
});
```

### 2. Chrome DevTools
```bash
node --inspect app.js
# Відкрити chrome://inspect → Memory tab → Take heap snapshot
```

### 3. Heapdump
```javascript
import v8 from "node:v8";
// Зберегти heap snapshot
v8.writeHeapSnapshot(); // створить .heapsnapshot файл
```

### 4. Clinic.js
```bash
npx clinic heapprofile -- node app.js
```

---

## WeakRef та FinalizationRegistry

### WeakRef — слабке посилання (не запобігає GC)
```javascript
let obj = { data: "large data" };
const weakRef = new WeakRef(obj);

obj = null; // об'єкт може бути зібраний GC

const deref = weakRef.deref();
if (deref) {
  console.log(deref.data); // якщо ще живий
} else {
  console.log("Object was garbage collected");
}
```

### FinalizationRegistry — callback при GC
```javascript
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`${heldValue} was garbage collected`);
});

let obj = { data: "important" };
registry.register(obj, "my-object");
obj = null; // коли GC збере → виведе "my-object was garbage collected"
```

**Use case:** кеш з автоматичним очищенням, resource cleanup.

---

## V8 прапорці для управління пам'яттю

```bash
# Збільшити heap (за замовчуванням ~1.7GB для 64-bit)
node --max-old-space-size=4096 app.js  # 4GB

# Показати GC активність
node --trace-gc app.js

# Expose GC для ручного виклику
node --expose-gc app.js
# global.gc(); // примусовий GC

# Heap snapshots
node --heap-prof app.js
```

---

## Buffer та управління пам'яттю

```javascript
// Buffer виділяється ПОЗА V8 heap (external memory)
const buf = Buffer.alloc(1024);      // заповнений нулями, безпечний
const unsafe = Buffer.allocUnsafe(1024); // може містити старі дані, швидший

// Buffer.from копіює дані
const fromStr = Buffer.from("hello", "utf-8");
const fromArr = Buffer.from([1, 2, 3]);

// Buffer pool — Node.js переиспользує пул для маленьких буферів (<4KB)
// allocUnsafe використовує pool → швидший, але може містити чужі дані
```
