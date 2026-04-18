# Sequelize — Питання для інтерв'ю

## Що таке Sequelize?

**Sequelize** — найстаріший Node.js ORM (з 2011 року). Active Record pattern.

Підтримує: PostgreSQL, MySQL, MariaDB, SQLite, MSSQL.

---

## Визначення моделей

### Class-based (v6+)
```javascript
import { Model, DataTypes } from "sequelize";

class User extends Model {}

User.init({
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true,
  },
  name: {
    type: DataTypes.STRING,
    allowNull: false,
  },
  email: {
    type: DataTypes.STRING,
    allowNull: false,
    unique: true,
    validate: { isEmail: true },
  },
  age: DataTypes.INTEGER,
}, {
  sequelize, // екземпляр підключення
  tableName: "users",
  timestamps: true, // createdAt, updatedAt
  paranoid: true,    // soft delete (deletedAt)
});
```

---

## CRUD операції

```javascript
// Create
const user = await User.create({ name: "John", email: "john@example.com" });

// Read
const user = await User.findByPk(userId);
const user = await User.findOne({ where: { email: "john@example.com" } });
const users = await User.findAll({ where: { age: { [Op.gte]: 18 } } });

// Update
await User.update({ name: "Jane" }, { where: { id: userId } });
// Або через інстанс
user.name = "Jane";
await user.save();

// Delete
await User.destroy({ where: { id: userId } });
// Soft delete (paranoid: true)
await user.destroy(); // встановлює deletedAt
```

---

## Асоціації

```javascript
// One-to-Many
User.hasMany(Order, { foreignKey: "userId" });
Order.belongsTo(User, { foreignKey: "userId" });

// Many-to-Many
User.belongsToMany(Role, { through: "user_roles" });
Role.belongsToMany(User, { through: "user_roles" });

// One-to-One
User.hasOne(Profile, { foreignKey: "userId" });
Profile.belongsTo(User, { foreignKey: "userId" });

// Eager loading
const users = await User.findAll({
  include: [
    { model: Order, where: { status: "active" } },
    { model: Role },
  ],
});
```

---

## Транзакції

```javascript
// Managed transaction (автоматичний commit/rollback)
const result = await sequelize.transaction(async (t) => {
  const user = await User.create({ name: "John" }, { transaction: t });
  await Order.create({ userId: user.id, total: 99 }, { transaction: t });
  return user;
});

// Unmanaged transaction (ручний контроль)
const t = await sequelize.transaction();
try {
  await User.create({ name: "John" }, { transaction: t });
  await t.commit();
} catch (error) {
  await t.rollback();
}
```

---

## Проблеми Sequelize

### 1. TypeScript підтримка — слабка
```typescript
// Потрібно дублювати типи
interface UserAttributes {
  id: string;
  name: string;
  email: string;
}

interface UserCreationAttributes extends Optional<UserAttributes, "id"> {}

class User extends Model<UserAttributes, UserCreationAttributes>
  implements UserAttributes {
  declare id: string;    // declare щоб уникнути подвійної ініціалізації
  declare name: string;
  declare email: string;
}
// Дуже verbose!
```

### 2. N+1 проблема
```javascript
// ❌ N+1 (1 запит на users + N запитів на orders)
const users = await User.findAll();
for (const user of users) {
  const orders = await user.getOrders(); // N запитів!
}

// ✅ Eager loading
const users = await User.findAll({ include: [Order] }); // 1-2 запити
```

### 3. Performance overhead
```
Sequelize додає багато "магії":
- Getter/Setter методи на кожному інстансі
- Validation hooks
- Virtual fields
- Instance methods
- Це все — overhead у пам'яті та CPU
```

### 4. Raw queries — escape hatch
```javascript
// Коли ORM заважає — raw SQL
const [results] = await sequelize.query(
  "SELECT * FROM users WHERE age > :age",
  {
    replacements: { age: 18 },
    type: QueryTypes.SELECT,
  }
);
```

---

## Sequelize vs Drizzle

| Критерій | Sequelize | Drizzle |
|---------|-----------|---------|
| Вік | 2011 (зрілий) | 2023 (молодий) |
| Pattern | Active Record | Query Builder |
| TypeScript | Слабкий | Відмінний |
| Performance | Повільніший | Швидший |
| SQL Control | Абстрагований | Повний |
| Bundle Size | Великий | Малий |
| ESM | Проблематичний | Нативний |
| Міграції | CLI + JS файли | SQL файли |
| Learning Curve | Середній | Потрібен SQL |

---

## Коли Sequelize доречний?

**Використовуй Sequelize коли:**
- Legacy проект вже на Sequelize
- Команда не знає SQL
- Потрібні вбудовані валідації та hooks
- MySQL/MariaDB проект (краща підтримка)

**НЕ використовуй для нових проектів:**
- TypeScript підтримка слабка
- Занадто багато overhead
- ESM проблеми
- Краще обрати Drizzle або Prisma
