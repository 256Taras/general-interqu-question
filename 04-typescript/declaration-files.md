# Declaration Files — Питання для інтерв'ю

## Що таке .d.ts файли?

Declaration files описують типи для JavaScript коду, який не має вбудованих типів. Вони не містять реалізації — тільки сигнатури типів.

```typescript
// math-lib.d.ts
declare function add(a: number, b: number): number;
declare function multiply(a: number, b: number): number;

declare const PI: number;

declare class Calculator {
  add(a: number, b: number): number;
  subtract(a: number, b: number): number;
}
```

---

## declare module

Додати типи для бібліотеки без TypeScript підтримки:

```typescript
// types/untyped-lib.d.ts
declare module "untyped-lib" {
  export function parse(input: string): object;
  export function stringify(data: object): string;
  export default class Client {
    constructor(config: { url: string });
    connect(): Promise<void>;
  }
}

// Wildcard для файлів
declare module "*.css" {
  const styles: Record<string, string>;
  export default styles;
}

declare module "*.svg" {
  const content: string;
  export default content;
}
```

---

## declare global

Додати типи до глобального scope:

```typescript
// types/global.d.ts
declare global {
  interface Window {
    analytics: {
      track(event: string, data?: object): void;
    };
  }

  // Глобальна змінна
  var __DEV__: boolean;

  // Розширення існуючого інтерфейсу
  interface Array<T> {
    customMethod(): T[];
  }
}

export {}; // файл повинен бути модулем
```

---

## Type Augmentation / Module Augmentation

Розширення типів існуючих бібліотек:

```typescript
// Розширення Fastify
import "fastify";

declare module "fastify" {
  interface FastifyInstance {
    authenticate: (request: FastifyRequest) => Promise<void>;
  }

  interface FastifyRequest {
    userId?: string;
    sessionId?: string;
  }
}

// Розширення Express
import "express";

declare module "express" {
  interface Request {
    user?: { id: string; role: string; };
  }
}
```

---

## Ambient Declarations

Описують існуючий код без створення нових значень:

```typescript
// Змінна існує у runtime (через script tag або env)
declare const API_URL: string;
declare const __VERSION__: string;

// Функція існує глобально
declare function gtag(command: string, ...args: any[]): void;

// Namespace
declare namespace NodeJS {
  interface ProcessEnv {
    NODE_ENV: "development" | "production" | "test";
    DATABASE_URL: string;
    JWT_SECRET: string;
  }
}
```

---

## Triple-Slash Directives

Застарілий механізм, але іноді ще потрібний:

```typescript
/// <reference types="node" />       — включити типи пакету
/// <reference path="./types.d.ts" /> — включити файл типів
/// <reference lib="es2022" />        — включити lib

// Сучасна альтернатива: tsconfig.json
{
  "compilerOptions": {
    "types": ["node", "jest"],
    "lib": ["ES2022"]
  }
}
```

**Коли triple-slash ще потрібен:**
- Declaration files (.d.ts) які не є модулями
- Вказати залежності між .d.ts файлами
