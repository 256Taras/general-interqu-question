# MongoDB Schema Design — Питання для інтерв'ю

## Embedding vs Referencing

### Embedding (вкладені документи)
```json
{
  "_id": "user1",
  "name": "John",
  "addresses": [
    { "city": "Kyiv", "street": "Khreshchatyk 1" },
    { "city": "Lviv", "street": "Svobody 10" }
  ]
}
```
**Коли:** 1-to-few, дані завжди читаються разом, не потрібні окремо.

### Referencing (посилання)
```json
// users collection
{ "_id": "user1", "name": "John" }

// orders collection
{ "_id": "order1", "user_id": "user1", "total": 99.99 }
```
**Коли:** 1-to-many/many-to-many, дані потрібні окремо, великі або зростаючі масиви.

### Правила вибору
| Фактор | Embedding | Referencing |
| --- | --- | --- |
| Кардинальність | 1-to-few (< 100) | 1-to-many (> 100) |
| Читання | Завжди разом | Часто окремо |
| Оновлення | Рідко | Часто |
| Розмір документа | < 16MB ліміт | Без обмежень |
| Consistency | Атомарне оновлення | Потребує транзакцій |

---

## Schema Design Patterns

### Attribute Pattern
Різні атрибути для різних документів:
```json
{ "name": "Phone", "specs": [
  { "key": "screen", "value": "6.1 inch" },
  { "key": "ram", "value": "8GB" }
]}
```
Замість різних полів — масив key-value. Один індекс на `specs.key` + `specs.value`.

### Bucket Pattern
Групування часових даних:
```json
{
  "sensor_id": "s1",
  "day": "2024-01-15",
  "readings": [
    { "time": "10:00", "temp": 22.5 },
    { "time": "10:05", "temp": 22.7 }
  ],
  "count": 2
}
```
Зменшує кількість документів, покращує write performance.

### Computed Pattern
Зберігаємо обчислене значення:
```json
{
  "product_id": "p1",
  "reviews_count": 150,
  "avg_rating": 4.5
}
```
Замість обчислення кожен раз — оновлюємо при додаванні review.

### Subset Pattern
Зберігаємо тільки останні/популярні:
```json
{
  "product_id": "p1",
  "recent_reviews": [ /* останні 10 */ ],
  "all_reviews_count": 500
}
```
Повний список — в окремій колекції.

### Outlier Pattern
Для документів, що перевищують ліміти:
```json
{
  "celebrity_id": "c1",
  "followers_count": 10000000,
  "has_extras": true,
  "followers_sample": [ /* перші 1000 */ ]
}
// Решта followers — в окремій колекції
```

---

## Денормалізація в MongoDB

MongoDB **заохочує** денормалізацію (на відміну від SQL):
- Дані, що читаються разом — зберігай разом
- Дублювання — нормально, якщо дані рідко змінюються
- Мета: один запит = всі потрібні дані (без JOIN-ів)

**Але:** якщо дублійовані дані часто змінюються — referencing краще (уникає cascade updates).
