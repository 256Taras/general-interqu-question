# Advanced TypeScript Types — Питання для інтерв'ю

## Union vs Intersection Types

```typescript
// Union — АБО (один з типів)
type StringOrNumber = string | number;

// Intersection — І (об'єднання типів)
type Employee = Person & { company: string; role: string; };

// Практичний приклад
type ApiResult<T> = { data: T; status: "success" } | { error: string; status: "error" };
```

---

## Literal Types

```typescript
type Direction = "north" | "south" | "east" | "west";
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";
type StatusCode = 200 | 201 | 400 | 404 | 500;

function move(direction: Direction) {} // тільки 4 значення
```

---

## Template Literal Types

```typescript
type EventName = `on${Capitalize<string>}`;
// "onClick", "onLoad", "onSubmit", ...

type CssProperty = `${string}-${string}`;
// "font-size", "border-color", ...

// Комбінації
type Color = "red" | "blue" | "green";
type Shade = "light" | "dark";
type Theme = `${Shade}-${Color}`;
// "light-red" | "light-blue" | "light-green" | "dark-red" | "dark-blue" | "dark-green"

// Getter/Setter
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
```

---

## Recursive Types

```typescript
// JSON type
type Json = string | number | boolean | null | Json[] | { [key: string]: Json };

// Deeply nested object
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// Tree structure
type TreeNode<T> = {
  value: T;
  children: TreeNode<T>[];
};

// Flatten nested arrays
type Flatten<T> = T extends (infer U)[] ? Flatten<U> : T;
type F = Flatten<number[][][]>; // number
```

---

## Branded Types (Nominal Typing)

TypeScript має structural typing. Branded types емулюють nominal typing:

```typescript
// Проблема: обидва string, легко сплутати
function getUser(userId: string) {}
function getOrder(orderId: string) {}
getUser(orderId); // TypeScript не помітить помилку!

// Рішення: branded types
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };

function createUserId(id: string): UserId { return id as UserId; }
function createOrderId(id: string): OrderId { return id as OrderId; }

function getUser(userId: UserId) {}

const userId = createUserId("user-123");
const orderId = createOrderId("order-456");

getUser(userId);   // ✅ OK
getUser(orderId);  // ❌ Error: OrderId не можна передати як UserId
```

---

## Variadic Tuple Types

```typescript
// Spread у tuple types
type Concat<A extends unknown[], B extends unknown[]> = [...A, ...B];
type R = Concat<[1, 2], [3, 4]>; // [1, 2, 3, 4]

// Typed function composition
function concat<A extends unknown[], B extends unknown[]>(
  a: [...A], b: [...B]
): [...A, ...B] {
  return [...a, ...b];
}
```

---

## Key Remapping в Mapped Types

```typescript
type RemovePrefix<T, Prefix extends string> = {
  [K in keyof T as K extends `${Prefix}${infer Rest}` ? Uncapitalize<Rest> : K]: T[K];
};

interface ApiUser {
  user_name: string;
  user_email: string;
  user_age: number;
}

type User = RemovePrefix<ApiUser, "user_">;
// { name: string; email: string; age: number; }

// Фільтрація ключів
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
```

---

## satisfies Operator

Перевіряє тип БЕЗ розширення (widening):

```typescript
// ❌ Без satisfies — тип розширюється
const colors: Record<string, string | number[]> = {
  red: "#ff0000",
  green: [0, 255, 0],
};
colors.red.toUpperCase(); // Error: може бути number[]

// ✅ З satisfies — зберігає narrow type
const colors = {
  red: "#ff0000",
  green: [0, 255, 0],
} satisfies Record<string, string | number[]>;

colors.red.toUpperCase();  // OK: TypeScript знає що red — string
colors.green.map(x => x);  // OK: TypeScript знає що green — number[]
```

---

## Conditional Types — Advanced

```typescript
// Distributive conditional types
type ToArray<T> = T extends any ? T[] : never;
type R = ToArray<string | number>; // string[] | number[]

// Запобігти distribution
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type R2 = ToArrayNonDist<string | number>; // (string | number)[]

// Практичний приклад
type IsNever<T> = [T] extends [never] ? true : false;
type IsAny<T> = 0 extends (1 & T) ? true : false;
```
