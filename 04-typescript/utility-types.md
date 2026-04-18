# TypeScript Utility Types — Питання для інтерв'ю

## Основні Utility Types

### Partial\<T\> — всі поля optional
```typescript
interface User { name: string; email: string; age: number; }

type UpdateUser = Partial<User>;
// { name?: string; email?: string; age?: number; }

function updateUser(id: string, data: Partial<User>) {
  // data може містити лише частину полів
}
updateUser("123", { name: "New Name" }); // OK
```

### Required\<T\> — всі поля обов'язкові
```typescript
interface Config { host?: string; port?: number; }
type StrictConfig = Required<Config>;
// { host: string; port: number; }
```

### Readonly\<T\> — всі поля readonly
```typescript
type ReadonlyUser = Readonly<User>;
const user: ReadonlyUser = { name: "John", email: "j@j.com", age: 30 };
user.name = "Jane"; // ❌ Error: cannot assign to readonly
```

---

## Pick, Omit, Record

### Pick\<T, K\> — вибрати конкретні поля
```typescript
type UserPreview = Pick<User, "name" | "email">;
// { name: string; email: string; }
```

### Omit\<T, K\> — виключити поля
```typescript
type UserWithoutPassword = Omit<User, "password">;
// все крім password
```

### Record\<K, T\> — об'єкт з типізованими ключами
```typescript
type Roles = "admin" | "user" | "guest";
type RolePermissions = Record<Roles, string[]>;

const permissions: RolePermissions = {
  admin: ["read", "write", "delete"],
  user: ["read", "write"],
  guest: ["read"],
};
```

---

## Exclude, Extract, NonNullable

### Exclude\<T, U\> — видалити з union
```typescript
type T = Exclude<"a" | "b" | "c", "a">;  // "b" | "c"
type Status = Exclude<"active" | "inactive" | "deleted", "deleted">;
// "active" | "inactive"
```

### Extract\<T, U\> — залишити тільки спільне
```typescript
type T = Extract<"a" | "b" | "c", "a" | "f">;  // "a"
type StringOrNumber = Extract<string | number | boolean, string | number>;
// string | number
```

### NonNullable\<T\> — видалити null і undefined
```typescript
type T = NonNullable<string | null | undefined>;  // string
```

---

## ReturnType, Parameters, Awaited

### ReturnType\<T\> — тип повернення функції
```typescript
function getUser() { return { name: "John", age: 30 }; }
type UserType = ReturnType<typeof getUser>;
// { name: string; age: number; }
```

### Parameters\<T\> — tuple типів аргументів
```typescript
function createUser(name: string, age: number, email: string) {}
type Params = Parameters<typeof createUser>;
// [name: string, age: number, email: string]

type FirstParam = Params[0]; // string
```

### InstanceType\<T\> — тип інстансу класу
```typescript
class UserService { findAll() { return []; } }
type Instance = InstanceType<typeof UserService>;
// UserService
```

### Awaited\<T\> — розгортає Promise
```typescript
type A = Awaited<Promise<string>>;           // string
type B = Awaited<Promise<Promise<number>>>;  // number (рекурсивно)
```

---

## Як створити власний Utility Type?

```typescript
// Deep Partial — рекурсивний Partial
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

// Deep Readonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

// Mutable — видалити readonly
type Mutable<T> = {
  -readonly [K in keyof T]: T[K];
};

// RequireAtLeastOne
type RequireAtLeastOne<T, Keys extends keyof T = keyof T> =
  Pick<T, Exclude<keyof T, Keys>> &
  { [K in Keys]-?: Required<Pick<T, K>> & Partial<Pick<T, Exclude<Keys, K>>> }[Keys];

// Використання
type Filter = RequireAtLeastOne<{
  name?: string;
  email?: string;
  age?: number;
}>;
// Мінімум одне поле обов'язкове
```

---

## Практичне використання

```typescript
// API response wrapper
type ApiResponse<T> = {
  data: T;
  meta: { total: number; page: number; };
};

type UserListResponse = ApiResponse<Pick<User, "name" | "email">[]>;

// Form state
type FormState<T> = {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  touched: Partial<Record<keyof T, boolean>>;
};

type LoginForm = FormState<{ email: string; password: string; }>;
```
