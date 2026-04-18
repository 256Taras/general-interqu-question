# Drizzle ORM vs TypeORM — Питання для інтерв'ю

## Філософія

| | Drizzle | TypeORM |
|---|---------|---------|
| Підхід | SQL-first (Query Builder) | Active Record / Data Mapper |
| Типи | Type-safe з виведенням | Декоратори + reflect-metadata |
| SQL знання | Потрібне | Абстрагує SQL |
| Кодова база | Легка (~30KB) | Важка (~500KB) |
| Міграції | Генерує з schema diff | CLI генерує або ручні |

---

## Визначення Schema

### Drizzle
```typescript
import { pgTable, text, integer, timestamp, uuid } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  age: integer("age"),
  createdAt: timestamp("created_at").defaultNow(),
});

// Типи автоматично виводяться
type User = typeof users.$inferSelect;
type NewUser = typeof users.$inferInsert;
```

### TypeORM
```typescript
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn } from "typeorm";

@Entity()
export class User {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column()
  name: string;

  @Column({ unique: true })
  email: string;

  @Column({ nullable: true })
  age: number;

  @CreateDateColumn()
  createdAt: Date;
}
```

**Різниця:** Drizzle використовує функції, TypeORM — декоратори. Drizzle не потребує `reflect-metadata` та `experimentalDecorators`.

---

## Запити

### Select
```typescript
// Drizzle — SQL-like
const user = await db
  .select()
  .from(users)
  .where(eq(users.email, "john@example.com"));

// Вибір конкретних полів
const names = await db
  .select({ name: users.name, email: users.email })
  .from(users);

// TypeORM — ORM-style
const user = await userRepo.findOne({
  where: { email: "john@example.com" },
});

// Вибір конкретних полів
const names = await userRepo.find({
  select: ["name", "email"],
});
```

### JOIN
```typescript
// Drizzle
const result = await db
  .select({
    userName: users.name,
    orderTotal: orders.total,
  })
  .from(users)
  .leftJoin(orders, eq(users.id, orders.userId))
  .where(gt(orders.total, 100));

// TypeORM
const result = await userRepo
  .createQueryBuilder("user")
  .leftJoinAndSelect("user.orders", "order")
  .where("order.total > :total", { total: 100 })
  .getMany();
```

### Insert / Update / Delete
```typescript
// Drizzle
await db.insert(users).values({ name: "John", email: "john@example.com" });
await db.update(users).set({ name: "Jane" }).where(eq(users.id, userId));
await db.delete(users).where(eq(users.id, userId));

// TypeORM
await userRepo.save({ name: "John", email: "john@example.com" });
await userRepo.update(userId, { name: "Jane" });
await userRepo.delete(userId);
```

---

## Міграції

### Drizzle
```bash
# Генерує міграцію з різниці schema
npx drizzle-kit generate
# → drizzle/0001_initial.sql (чистий SQL!)

# Застосувати
npx drizzle-kit migrate

# Або push напряму (dev)
npx drizzle-kit push
```

### TypeORM
```bash
# Генерує міграцію
npx typeorm migration:generate -n CreateUsers
# → TypeScript файл з up() та down()

# Застосувати
npx typeorm migration:run
```

**Drizzle:** міграції — чистий SQL (можна review, редагувати).
**TypeORM:** міграції — TypeScript код (складніше review).

---

## Relations

### Drizzle (Relations API)
```typescript
import { relations } from "drizzle-orm";

export const usersRelations = relations(users, ({ many }) => ({
  orders: many(orders),
}));

export const ordersRelations = relations(orders, ({ one }) => ({
  user: one(users, {
    fields: [orders.userId],
    references: [users.id],
  }),
}));

// Relational query (новий API)
const usersWithOrders = await db.query.users.findMany({
  with: { orders: true },
});
```

### TypeORM
```typescript
@Entity()
export class User {
  @OneToMany(() => Order, (order) => order.user)
  orders: Order[];
}

@Entity()
export class Order {
  @ManyToOne(() => User, (user) => user.orders)
  user: User;
}

// Eager loading
const users = await userRepo.find({ relations: ["orders"] });
```

---

## Порівняння

| Критерій | Drizzle | TypeORM |
|---------|---------|---------|
| **Type Safety** | Відмінна (виведення типів) | Добра (декоратори) |
| **Performance** | Швидший (тонший шар) | Повільніший (більше абстракцій) |
| **SQL Control** | Повний | Обмежений (QueryBuilder) |
| **Міграції** | SQL файли | TypeScript файли |
| **Learning Curve** | Потрібен SQL | Легше без SQL |
| **Bundle Size** | ~30KB | ~500KB |
| **ESM Support** | Нативний | Проблематичний |
| **Зрілість** | Молодий (2023+) | Зрілий (2016+) |
| **Community** | Зростає | Великий |
| **Документація** | Хороша | Обширна |

---

## Коли що обирати?

**Drizzle:**
- Знаєш SQL і хочеш контроль
- Performance критичний
- Новий проект з TypeScript
- Потрібен легкий bundle (serverless)
- Fastify / Hono / сучасний стек

**TypeORM:**
- Команда не знає SQL добре
- Потрібен Active Record pattern
- Legacy проект вже на TypeORM
- NestJS (інтеграція з коробки)
- Потрібна зріла екосистема

---

## Чому Drizzle для нового проекту?

```
1. SQL-first → менше "магії", легше дебажити
2. Type inference → не потрібні декоратори
3. Легший → швидший cold start (serverless)
4. ESM native → сумісність з сучасним Node.js
5. Drizzle Kit → зручні міграції
6. Relational API → зручні nested queries
```
