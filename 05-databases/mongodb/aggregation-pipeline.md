# MongoDB Aggregation Pipeline — Питання для інтерв'ю

## Основні стадії

Pipeline — послідовність стадій, кожна трансформує дані:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },          // фільтр (як WHERE)
  { $group: {                                     // групування (як GROUP BY)
    _id: "$customer_id",
    totalSpent: { $sum: "$amount" },
    orderCount: { $count: {} }
  }},
  { $sort: { totalSpent: -1 } },                 // сортування
  { $limit: 10 }                                  // ліміт
]);
```

---

## Ключові стадії

### $match — фільтрація
```javascript
{ $match: { status: "active", age: { $gte: 18 } } }
// Завжди ставити на початок — використовує індекси
```

### $group — агрегація
```javascript
{ $group: {
  _id: "$department",         // group by department
  avgSalary: { $avg: "$salary" },
  maxSalary: { $max: "$salary" },
  employees: { $push: "$name" },  // масив імен
  count: { $sum: 1 }
}}
```

### $project — вибір/трансформація полів
```javascript
{ $project: {
  fullName: { $concat: ["$firstName", " ", "$lastName"] },
  year: { $year: "$createdAt" },
  isActive: 1,  // включити
  password: 0   // виключити
}}
```

### $lookup — JOIN з іншою колекцією
```javascript
{ $lookup: {
  from: "orders",
  localField: "_id",
  foreignField: "user_id",
  as: "userOrders"
}}
```

### $unwind — розгортання масиву
```javascript
// { tags: ["a", "b", "c"] }
{ $unwind: "$tags" }
// → { tags: "a" }, { tags: "b" }, { tags: "c" }
```

### $facet — кілька pipeline паралельно
```javascript
{ $facet: {
  totalCount: [{ $count: "count" }],
  data: [{ $skip: 20 }, { $limit: 10 }],
  priceRange: [
    { $group: { _id: null, min: { $min: "$price" }, max: { $max: "$price" } }}
  ]
}}
```

### $bucket — розподіл по діапазонах
```javascript
{ $bucket: {
  groupBy: "$age",
  boundaries: [0, 18, 30, 50, 100],
  default: "Other",
  output: { count: { $sum: 1 } }
}}
```

---

## Оптимізація Aggregation Pipeline

1. **$match першим** — зменшує кількість документів якнайшвидше
2. **$match перед $lookup** — менше JOIN-ів
3. **$project перед $group** — менше даних для обробки
4. **Індекси** — $match і $sort на початку використовують індекси
5. **allowDiskUse: true** — для великих datasets (коли > 100MB RAM)

```javascript
db.collection.aggregate(pipeline, { allowDiskUse: true });
```
