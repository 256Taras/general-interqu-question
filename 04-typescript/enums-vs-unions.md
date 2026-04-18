# Enums vs Union Types — Питання для інтерв'ю

## const enum vs enum

### Звичайний enum
```typescript
enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}
// Компілюється у JavaScript об'єкт:
// var Direction = { Up: "UP", Down: "DOWN", ... }
// Залишається у bundle!
```

### const enum
```typescript
const enum Direction {
  Up = "UP",
  Down = "DOWN",
}
// Інлайниться при компіляції:
// const d = "UP"; (замість Direction.Up)
// НЕ створює об'єкт у runtime
```

**Але:** `const enum` має проблеми з declaration files та isolatedModules. TypeScript team не рекомендує у бібліотеках.

---

## Чому Union Types часто кращі за Enums?

```typescript
// Enum
enum Status { Active = "active", Inactive = "inactive", Deleted = "deleted" }

// Union — простіше, no runtime overhead
type Status = "active" | "inactive" | "deleted";

// Порівняння:
// 1. Union НЕ генерує JS код (zero runtime cost)
// 2. Union працює з tree shaking
// 3. Union простіший для розуміння
// 4. Union краще працює з type narrowing
// 5. Union не потребує import

function setStatus(status: Status) {} // обидва варіанти працюють однаково
setStatus("active"); // з union — напряму
setStatus(Status.Active); // з enum — через namespace
```

---

## as const Assertions

```typescript
// Без as const — TypeScript розширює тип
const colors = ["red", "green", "blue"]; // string[]

// З as const — literal type
const colors = ["red", "green", "blue"] as const;
// readonly ["red", "green", "blue"]

type Color = typeof colors[number]; // "red" | "green" | "blue"

// Об'єкт з as const
const config = {
  port: 3000,
  host: "localhost",
  env: "production",
} as const;

type Env = typeof config.env; // "production" (не string!)

// Заміна enum
const Direction = {
  Up: "UP",
  Down: "DOWN",
  Left: "LEFT",
  Right: "RIGHT",
} as const;

type Direction = typeof Direction[keyof typeof Direction];
// "UP" | "DOWN" | "LEFT" | "RIGHT"
```

---

## Коли Enums доречні?

**Використовуй enum коли:**
- Потрібен reverse mapping (numeric enums): `Direction[0] === "Up"`
- Потрібне значення у runtime як об'єкт (iterate, Object.keys)
- Команда звикла до enums і це конвенція проекту

**Використовуй union коли:**
- Простий набір string literals
- Важливий bundle size
- Працюєш з API (JSON значення — це strings, не enums)
- Потрібен tree shaking

---

## Tree Shaking та Enums

```typescript
// enum — НЕ tree-shakeable (залишається у bundle)
enum Animal { Cat, Dog, Fish }
// Навіть якщо не використовуєш — об'єкт у bundle

// union — zero cost
type Animal = "cat" | "dog" | "fish";
// Існує тільки у type system, зникає після компіляції

// as const object — частково tree-shakeable
const Animal = { Cat: "cat", Dog: "dog" } as const;
// Об'єкт у bundle, але bundler може видалити якщо не використовується
```

---

## Numeric vs String Enums

```typescript
// Numeric (автоматичне значення)
enum Direction { Up, Down, Left, Right } // 0, 1, 2, 3
// Має reverse mapping: Direction[0] === "Up"
// Проблема: будь-яке число проходить type check

// String (явне значення)
enum Status { Active = "active", Inactive = "inactive" }
// Немає reverse mapping
// Більш type-safe: тільки задані значення

// ❌ Numeric enum проблема
enum Fruit { Apple, Banana }
function eat(fruit: Fruit) {}
eat(42); // Компілюється без помилки! Небезпечно.

// ✅ String enum — безпечніший
enum Color { Red = "red", Blue = "blue" }
function paint(color: Color) {}
paint("green"); // ❌ Error
```
