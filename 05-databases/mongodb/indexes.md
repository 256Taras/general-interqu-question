# MongoDB Indexes — Питання для інтерв'ю

## Типи індексів

### Single Field
```javascript
db.users.createIndex({ email: 1 });  // ascending
db.users.createIndex({ age: -1 });   // descending
```

### Compound (складений)
```javascript
db.users.createIndex({ country: 1, city: 1, age: -1 });
// Працює для: country, country+city, country+city+age
// НЕ працює для: city (без country), age (без country+city)
```

### Multikey (для масивів)
```javascript
db.products.createIndex({ tags: 1 });
// Автоматично створюється для кожного елемента масиву
db.products.find({ tags: "electronics" }); // використовує індекс
```

### Text Index
```javascript
db.articles.createIndex({ title: "text", body: "text" });
db.articles.find({ $text: { $search: "mongodb performance" } });
```

### 2dsphere (геодані)
```javascript
db.places.createIndex({ location: "2dsphere" });
db.places.find({
  location: { $near: { $geometry: { type: "Point", coordinates: [30.5, 50.4] }, $maxDistance: 5000 } }
});
```

### Hashed
```javascript
db.users.createIndex({ _id: "hashed" }); // для sharding
```

### TTL (Time-To-Live)
```javascript
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 });
// Документи автоматично видаляються через 1 годину
```

### Partial Index
```javascript
db.orders.createIndex(
  { status: 1 },
  { partialFilterExpression: { status: "active" } }
);
// Індексує тільки active orders — менший розмір
```

### Sparse Index
```javascript
db.users.createIndex({ phone: 1 }, { sparse: true });
// Індексує тільки документи де phone існує
```

---

## ESR Rule (Equality, Sort, Range)

Порядок полів у compound index:
1. **E**quality — поля з точним порівнянням (`status = "active"`)
2. **S**ort — поля сортування (`sort by createdAt`)
3. **R**ange — поля з діапазоном (`age > 18`)

```javascript
// Запит: status = "active", age > 18, sort by createdAt
db.users.createIndex({ status: 1, createdAt: -1, age: 1 }); // ESR
```

---

## explain()

```javascript
db.users.find({ email: "john@example.com" }).explain("executionStats");

// Ключові поля:
// - winningPlan.stage: "IXSCAN" (добре) vs "COLLSCAN" (повне сканування)
// - totalDocsExamined: скільки документів переглянуто
// - totalKeysExamined: скільки ключів індексу переглянуто
// - executionTimeMillis: час виконання

// Ідеально: totalDocsExamined ≈ nReturned (мінімум зайвих)
```
