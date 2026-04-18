# Типові помилки TypeScript — Питання для інтерв'ю

## any vs unknown

```typescript
// any — вимикає type checking повністю
let x: any = "hello";
x.foo.bar.baz(); // Компілюється! Crash у runtime.
x = 42;
x.toUpperCase(); // Компілюється! Crash у runtime.

// unknown — безпечна альтернатива "не знаю тип"
let y: unknown = "hello";
y.toUpperCase(); // ❌ Error: Object is of type 'unknown'

// Потрібно перевірити тип перед використанням
if (typeof y === "string") {
  y.toUpperCase(); // ✅ OK
}
```

**Правило:** використовуй `unknown` замість `any`. `any` допустимий тільки для міграції з JS або для escape hatches.

---

## Type Assertions (as) — зловживання

```typescript
// ❌ Небезпечне assertion
const user = {} as User; // TypeScript вірить тобі!
console.log(user.name.toUpperCase()); // Runtime crash!

// ❌ Подвійне assertion (code smell)
const x = "hello" as unknown as number; // дуже підозріло

// ✅ Правильний підхід — валідація
function parseUser(data: unknown): User {
  if (typeof data === "object" && data !== null && "name" in data) {
    return data as User; // assertion після перевірки
  }
  throw new Error("Invalid user data");
}

// ✅ Або використовуй schema validation
import { Type } from "@sinclair/typebox";
const UserSchema = Type.Object({ name: Type.String() });
```

---

## @ts-ignore vs @ts-expect-error

```typescript
// @ts-ignore — ігнорує БУДЬ-ЯКУ помилку на наступному рядку
// @ts-ignore
const x: number = "string"; // помилка ігнорується

// @ts-expect-error — ігнорує, АЛЕ якщо помилки немає — сам стає помилкою
// @ts-expect-error — testing invalid input
const result = add("not", "numbers");

// Якщо виправити код і помилки більше немає:
// @ts-expect-error ← TypeScript покаже: "Unused '@ts-expect-error' directive"
const result = add(1, 2);   // це правильний код, @ts-expect-error зайвий
```

**Правило:** `@ts-expect-error` краще за `@ts-ignore` — воно попереджує коли стає зайвим. Але обидва — stop-gap, не рішення.

---

## interface vs type — коли що?

```typescript
// Interface — можна extend і merge
interface User { name: string; }
interface User { age: number; } // declaration merging!
// User = { name: string; age: number; }

interface Admin extends User { permissions: string[]; }

// Type — більш гнучкий
type StringOrNumber = string | number;       // union (interface не може)
type Pair<T> = [T, T];                       // tuple
type Callback = (data: string) => void;      // function type
type Status = "active" | "inactive";         // literal union
type UserKeys = keyof User;                  // computed types
```

**Рекомендація:**
- `interface` — для об'єктів і класів (extendable)
- `type` — для всього іншого (unions, tuples, computed types)
- Для проекту — обрати одне і дотримуватись (consistency)

---

## Забуті strict mode опції

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true // включає ВСЕ нижче:
    // "strictNullChecks": true       — null/undefined перевіряються
    // "strictFunctionTypes": true    — суворіші function types
    // "strictBindCallApply": true    — типізовані bind/call/apply
    // "noImplicitAny": true          — забороняє implicit any
    // "noImplicitThis": true         — забороняє implicit this
    // "strictPropertyInitialization": true — перевірка ініціалізації
  }
}
```

**Помилка:** працювати без `strict: true`. Без нього TypeScript пропускає багато помилок.

---

## Типові помилки з Generics

```typescript
// ❌ Generic без constraint — занадто широкий
function getLength<T>(value: T) {
  return value.length; // Error: T може не мати length
}

// ✅ З constraint
function getLength<T extends { length: number }>(value: T) {
  return value.length;
}

// ❌ Зайвий generic
function identity<T>(value: T): T { return value; }
// Якщо T використовується один раз — він зайвий
function log<T>(value: T): void { console.log(value); }
// Краще:
function log(value: unknown): void { console.log(value); }

// ❌ Неправильний constraint порядок
function merge<T extends object, U extends T>(a: T, b: U) {} // U extends T?
// Запитай себе: чи U дійсно повинен розширювати T?
```

---

## Проблеми з Third-Party типами

```typescript
// 1. Типи застарілі або неправильні
// @types/library@2.0 але library@3.0 — breaking changes

// 2. Типи занадто широкі
// Бібліотека повертає any замість конкретного типу
// Рішення: wrapper з правильними типами

// 3. Module augmentation
// Якщо бібліотека не експортує потрібний тип:
declare module "some-lib" {
  interface Config {
    customField: string; // додаємо своє поле
  }
}

// 4. DefinitelyTyped vs bundled types
// @types/* — community maintained (може відставати)
// Bundled types — від авторів бібліотеки (надійніше)
```

---

## Надмірне ускладнення типів

```typescript
// ❌ Overcomplicated
type DeepRequired<T> = T extends object
  ? { [K in keyof T]-?: T[K] extends object
    ? T[K] extends (...args: any[]) => any
      ? T[K]
      : DeepRequired<T[K]>
    : T[K]
  }
  : T;

// Запитай себе:
// 1. Чи хтось зрозуміє цей тип через 6 місяців?
// 2. Чи є простіший спосіб?
// 3. Чи це вирішує реальну проблему?

// ✅ Простіше — часто достатньо
type Config = Required<AppConfig>;
// Або навіть просто написати interface руками
```

**Правило:** якщо тип потребує більше 30 секунд щоб зрозуміти — він занадто складний. Розбий на менші або спрости.
