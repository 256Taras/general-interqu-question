# Type Guards та Narrowing — Питання для інтерв'ю

## typeof Guards

```typescript
function process(value: string | number) {
  if (typeof value === "string") {
    console.log(value.toUpperCase()); // TypeScript знає: string
  } else {
    console.log(value.toFixed(2));    // TypeScript знає: number
  }
}

// typeof працює з: "string", "number", "boolean", "object", "function", "undefined", "bigint", "symbol"
```

---

## instanceof Guards

```typescript
class ApiError extends Error {
  constructor(message: string, public statusCode: number) {
    super(message);
  }
}

function handleError(error: unknown) {
  if (error instanceof ApiError) {
    console.log(error.statusCode); // TypeScript знає: ApiError
  } else if (error instanceof Error) {
    console.log(error.message);    // TypeScript знає: Error
  }
}
```

---

## in Operator

```typescript
interface Bird { fly(): void; }
interface Fish { swim(): void; }

function move(animal: Bird | Fish) {
  if ("fly" in animal) {
    animal.fly();   // Bird
  } else {
    animal.swim();  // Fish
  }
}
```

---

## Custom Type Guards (is keyword)

```typescript
interface User { type: "user"; name: string; }
interface Admin { type: "admin"; name: string; permissions: string[]; }

// Custom type guard
function isAdmin(person: User | Admin): person is Admin {
  return person.type === "admin";
}

function greet(person: User | Admin) {
  if (isAdmin(person)) {
    console.log(`Admin: ${person.permissions}`); // TypeScript знає: Admin
  } else {
    console.log(`User: ${person.name}`);         // TypeScript знає: User
  }
}

// Для unknown
function isString(value: unknown): value is string {
  return typeof value === "string";
}

function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}

// Використання з filter
const items: (string | null)[] = ["a", null, "b", null, "c"];
const strings: string[] = items.filter(isNonNullable); // string[]
```

---

## Discriminated Unions

Union типи з спільним полем (discriminant):

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "rectangle"; width: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;   // TypeScript знає: circle
    case "square":
      return shape.side ** 2;                // TypeScript знає: square
    case "rectangle":
      return shape.width * shape.height;     // TypeScript знає: rectangle
  }
}

// Дуже поширений патерн для:
// - Redux actions
// - API responses
// - Event handling
// - State machines
```

---

## Exhaustive Checks з never

Перевірка, що всі випадки оброблені:

```typescript
function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "rectangle":
      return shape.width * shape.height;
    default:
      // Якщо додати новий kind — TypeScript покаже помилку тут
      const _exhaustive: never = shape;
      throw new Error(`Unknown shape: ${_exhaustive}`);
  }
}

// Або helper function
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}
```

---

## Assertion Functions (asserts keyword)

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error("Expected string");
  }
}

function process(input: unknown) {
  assertIsString(input);
  // Після assertion — TypeScript знає: string
  console.log(input.toUpperCase());
}

// Assert non-null
function assertDefined<T>(value: T | null | undefined): asserts value is T {
  if (value == null) {
    throw new Error("Value is null or undefined");
  }
}

const user = await findUser(id);
assertDefined(user); // після цього user: User, не User | null
console.log(user.name);
```
