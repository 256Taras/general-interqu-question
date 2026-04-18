# TypeScript Generics — Питання для інтерв'ю

## Що таке Generics і навіщо?

Generics дозволяють створювати компоненти, які працюють з різними типами, зберігаючи type safety.

```typescript
// ❌ Без generics — втрачаємо тип
function first(arr: any[]): any {
  return arr[0];
}
const x = first([1, 2, 3]); // any — не знаємо що number

// ✅ З generics — тип зберігається
function first<T>(arr: T[]): T {
  return arr[0];
}
const x = first([1, 2, 3]);     // number
const y = first(["a", "b"]);    // string
```

---

## Generic Functions

```typescript
// Одна type variable
function identity<T>(value: T): T {
  return value;
}

// Кілька type variables
function pair<K, V>(key: K, value: V): [K, V] {
  return [key, value];
}

// Arrow function
const wrap = <T>(value: T): { data: T } => ({ data: value });

// З default type
function createArray<T = string>(length: number, value: T): T[] {
  return Array(length).fill(value);
}
```

---

## Generic Interfaces та Classes

```typescript
// Interface
interface Repository<T> {
  findById(id: string): Promise<T>;
  create(data: T): Promise<T>;
  update(id: string, data: Partial<T>): Promise<T>;
}

// Реалізація
class UserRepository implements Repository<User> {
  async findById(id: string): Promise<User> { /* ... */ }
  async create(data: User): Promise<User> { /* ... */ }
  async update(id: string, data: Partial<User>): Promise<User> { /* ... */ }
}

// Generic class
class Queue<T> {
  private items: T[] = [];
  enqueue(item: T) { this.items.push(item); }
  dequeue(): T | undefined { return this.items.shift(); }
}

const numberQueue = new Queue<number>();
numberQueue.enqueue(42);
```

---

## Generic Constraints (extends)

Обмежують, які типи можна передати:

```typescript
// T повинен мати властивість length
function logLength<T extends { length: number }>(value: T): void {
  console.log(value.length);
}
logLength("hello");     // OK
logLength([1, 2, 3]);   // OK
logLength(42);           // ❌ Error: number не має length

// T повинен бути ключем K
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const user = { name: "John", age: 30 };
getProperty(user, "name"); // OK → string
getProperty(user, "foo");  // ❌ Error: "foo" не є ключем

// Constraint з interface
interface HasId { id: string; }
function findById<T extends HasId>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}
```

---

## Conditional Types з Generics

```typescript
// Базовий синтаксис: T extends U ? X : Y
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;  // true
type B = IsString<number>;  // false

// Практичний приклад — різна поведінка для різних типів
type ApiResponse<T> = T extends string
  ? { message: T }
  : T extends number
  ? { count: T }
  : { data: T };

type R1 = ApiResponse<string>; // { message: string }
type R2 = ApiResponse<number>; // { count: number }
type R3 = ApiResponse<User>;   // { data: User }
```

---

## infer Keyword

`infer` дозволяє "витягнути" тип зсередини іншого типу:

```typescript
// Витягти тип повернення функції
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type A = MyReturnType<() => string>;           // string
type B = MyReturnType<(x: number) => boolean>; // boolean

// Витягти тип елемента масиву
type ElementType<T> = T extends (infer E)[] ? E : never;

type C = ElementType<string[]>;  // string
type D = ElementType<number[]>;  // number

// Витягти тип Promise
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

type E = UnwrapPromise<Promise<string>>; // string
type F = UnwrapPromise<number>;          // number

// Витягти перший аргумент функції
type FirstArg<T> = T extends (first: infer A, ...rest: any[]) => any ? A : never;

type G = FirstArg<(name: string, age: number) => void>; // string
```

---

## Mapped Types з Generics

```typescript
// Зробити всі поля optional
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Зробити всі поля readonly
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Зробити всі поля nullable
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

// Перейменувати ключі
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User { name: string; age: number; }
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; }
```

---

## Практичні задачі

### Типізований EventEmitter
```typescript
type EventMap = Record<string, any[]>;

class TypedEmitter<Events extends EventMap> {
  private listeners: Partial<{ [K in keyof Events]: ((...args: Events[K]) => void)[] }> = {};

  on<K extends keyof Events>(event: K, fn: (...args: Events[K]) => void) {
    (this.listeners[event] ??= []).push(fn);
  }

  emit<K extends keyof Events>(event: K, ...args: Events[K]) {
    this.listeners[event]?.forEach(fn => fn(...args));
  }
}

// Використання — повна типізація
const bus = new TypedEmitter<{
  userCreated: [id: string, name: string];
  error: [error: Error];
}>();

bus.on("userCreated", (id, name) => {}); // id: string, name: string
bus.emit("userCreated", "123", "John");  // OK
bus.emit("userCreated", 123);            // ❌ Error
```

### Builder Pattern з Generics
```typescript
class QueryBuilder<T extends Record<string, unknown>> {
  private conditions: Partial<T> = {};

  where<K extends keyof T>(key: K, value: T[K]): this {
    this.conditions[key] = value;
    return this;
  }

  build() { return this.conditions; }
}

const query = new QueryBuilder<User>()
  .where("name", "John")   // OK
  .where("age", 30)        // OK
  .where("name", 42)       // ❌ Error: 42 не string
  .build();
```
