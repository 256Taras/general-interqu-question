# Hash Maps і Sets: друга за важливістю тема для алгоритмічних співбесід

## Навіщо взагалі ця тема

Якщо масиви -- це перший інструмент, який витягує кандидат із рюкзака на співбесіді, то hash map -- другий. Приблизно **60-70% задач з категорії "easy/medium" на LeetCode** розв'язуються за допомогою Map або Set. Це не перебільшення -- це спостереження.

Чому? Тому що hash map перетворює **пошук за значенням** з O(n) (лінійного сканування) на **O(1) average**. Усе, що раніше виглядало як "пройди масивом і щось там знайди" -- тепер стає одним рядком: `map.has(x)`.

На практиці це означає, що ціла категорія задач "O(n²) через вкладені цикли" -- знімається простим трюком: запам'ятай те, що вже бачив, у Map або Set.

У цьому файлі розбираємо:
- Як hash map влаштований всередині (потрібно, щоб пояснити колізії і worst case).
- Які операції є в JS і що з них реально вартує уваги.
- 5 канонічних патернів, які покривають левову частку задач.
- Де JS-розробники регулярно ловлять баги.

---

## Що таке hash map під капотом

**Hash map (hash table, асоціативний масив)** -- структура даних, що зберігає пари `(ключ, значення)` з середньою складністю O(1) для операцій `set`, `get`, `has`, `delete`.

Як це взагалі можливо? Адже інтуїтивно, щоб знайти ключ серед мільйона інших -- треба хоч якось їх переглянути.

Трюк у **хеш-функції**.

### Хеш-функція

Хеш-функція -- це функція `hash(key) -> integer`, яка перетворює ключ (рядок, число, об'єкт) у ціле число. Властивості ідеальної хеш-функції:
1. **Детермінована:** той самий ключ завжди дає той самий хеш.
2. **Швидка:** O(1) або O(k), де k -- довжина ключа.
3. **Рівномірна:** різні ключі дають різні хеші (ідеально), або принаймні рівномірно розподілені.

Приклад простої (поганої, навчальної) хеш-функції:

```javascript
// Навчальна хеш-функція для рядків. НЕ використовуй у продакшені.
function hash(key, bucketCount) {
  let sum = 0;
  for (let i = 0; i < key.length; i++) {
    sum += key.charCodeAt(i);     // додаємо коди символів
  }
  return sum % bucketCount;        // бакет = хеш mod кількість бакетів
}

hash("abc", 8);   // (97+98+99) % 8 = 294 % 8 = 6
hash("cba", 8);   // те саме: 6 -- колізія!
```

### Бакети

Під капотом hash map -- це **масив фіксованого розміру**, де кожна клітинка називається **бакет (bucket)**. Хеш-функція перетворює ключ на індекс бакета.

```
Масив з 8 бакетами:

  index:   0     1     2     3     4     5     6     7
          ┌───┬─────┬─────┬─────┬─────┬─────┬─────┬───┐
  buckets:│   │     │     │     │     │     │     │   │
          └───┴─────┴─────┴─────┴─────┴─────┴─────┴───┘

Вставляємо ("name" -> "Taras"):
  hash("name") % 8 = 3
                              ↓
          ┌───┬─────┬─────┬─────────────┬─────┬─────┬─────┬───┐
  buckets:│   │     │     │("name","T") │     │     │     │   │
          └───┴─────┴─────┴─────────────┴─────┴─────┴─────┴───┘

Вставляємо ("age" -> 30):
  hash("age") % 8 = 1
                ↓
          ┌───┬──────────┬─────┬─────────────┬─────┬─────┬─────┬───┐
  buckets:│   │("age",30)│     │("name","T") │     │     │     │   │
          └───┴──────────┴─────┴─────────────┴─────┴─────┴─────┴───┘
```

Тепер, щоб дістати значення `map.get("name")`:
1. Обчислили `hash("name") % 8 = 3`.
2. Пішли у бакет 3.
3. Повернули значення.

Це O(1) -- не залежить від того, скільки пар у мапі.

### Колізії

**Колізія** -- коли два різні ключі дають однаковий хеш (і, відповідно, один бакет). Це неминуче, бо бакетів скінченна кількість, а ключів -- нескінченно.

Дві основні стратегії розв'язання колізій:

**Стратегія 1: Chaining (ланцюжки).**

У кожному бакеті зберігаємо список (linked list або масив) пар. Якщо в бакет попадає ще одна пара -- додаємо її в кінець списку.

```
Припустимо, hash("name") = hash("mean") = 3 (обидва -- сума 97+109+101+110=417, %8=1... ок, уявімо, що 3)

          ┌───┬──────────┬─────┬──────────────────────────────────┬───┬───┬───┬───┐
  buckets:│   │("age",30)│     │ [("name","T") → ("mean","avg")]  │   │   │   │   │
          └───┴──────────┴─────┴──────────────────────────────────┴───┴───┴───┴───┘
                                         ↑
                              бакет -- список з двох пар
```

Пошук `get("mean")`:
1. Хеш = 3.
2. Зайшли у бакет 3.
3. Пройшли список, порівнюючи ключі: `"name" !== "mean"` → далі. `"mean" === "mean"` → знайшли.

Якщо бакет містить 2 елементи -- це 2 порівняння замість 1. Якщо 10 -- 10. У **найгіршому випадку** (всі n ключів потрапили в один бакет) пошук стає **O(n)**.

**Стратегія 2: Open addressing (лінійне/квадратичне пробування).**

Якщо бакет зайнятий -- шукаємо наступний вільний. Наприклад, лінійне: бакет+1, бакет+2, бакет+3...

```
Вставляємо "mean" при зайнятому бакеті 3:
  hash = 3 → зайнято. Пробуємо 4 → вільно. Вставляємо туди.

          ┌───┬──────────┬─────┬─────────────┬──────────────┬───┬───┬───┐
  buckets:│   │("age",30)│     │("name","T") │("mean","avg")│   │   │   │
          └───┴──────────┴─────┴─────────────┴──────────────┴───┴───┴───┘
                                      3              4 (зайняли через колізію)
```

Open addressing економить пам'ять (не потрібні додаткові лінкед лісти), але складніший у реалізації, особливо для видалення.

V8 (двигун Node.js/Chrome) використовує **chaining** для `Map` і свого роду open addressing у внутрішніх об'єктах.

### Чому O(1) average, але O(n) worst case

- **Average case O(1):** якщо хеш-функція рівномірна і load factor (кількість елементів / кількість бакетів) тримається розумним (наприклад, <0.75), середня довжина ланцюжка в бакеті -- константа.
- **Worst case O(n):** якщо всі ключі потрапляють в один бакет (погана хеш-функція або спеціально підібрані ключі), операції стають лінійними -- треба переглянути весь ланцюжок.
- **Resize (rehash):** коли load factor перевищує поріг, hash map подвоює кількість бакетів і **переносить усі елементи**. Це O(n). Але amortized -- O(1) на операцію (як `Array.push`).

```
Візуалізація resize:

До: 4 бакети, 4 елементи (load = 1.0)
┌───────┬───────┬───────┬───────┐
│ a→b→c │  d    │       │       │  ← перевантажений бакет
└───────┴───────┴───────┴───────┘

Resize: виділяємо 8 бакетів, перераховуємо хеші:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ a │ b │ d │   │ c │   │   │   │  ← розподілилися рівномірно
└───┴───┴───┴───┴───┴───┴───┴───┘
```

---

## Чому hash map -- це серце алгоритмічних співбесід

Головна магія hash map у тому, що він **перетворює пошук за значенням з O(n) на O(1)**.

Типовий наївний алгоритм:
```javascript
// Знайти, чи є в масиві arr елемент x
for (let i = 0; i < arr.length; i++) {
  if (arr[i] === x) return true;
}
```
Це O(n).

Із Set:
```javascript
const set = new Set(arr);    // O(n) one-time
set.has(x);                  // O(1) per query
```

Якщо ми робимо багато запитів до одного набору даних -- Set/Map радикально знижує складність.

**Типовий патерн оптимізації O(n²) → O(n):**

```javascript
// ❌ Наївно: O(n²). Для кожного x перевіряємо, чи є інший y, що x+y=target.
function twoSumNaive(arr, target) {
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {
      if (arr[i] + arr[j] === target) return [i, j];
    }
  }
  return null;
}

// ✅ З Map: O(n). Ідемо одним проходом, запам'ятовуючи бачене.
function twoSumFast(arr, target) {
  const seen = new Map();                // значення -> індекс
  for (let i = 0; i < arr.length; i++) {
    const complement = target - arr[i];  // те, що потрібно знайти
    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }
    seen.set(arr[i], i);                  // запам'ятали поточний
  }
  return null;
}
```

**Загальна формула:** якщо задача має вигляд "для кожного елемента знайти відповідного партнера / подібного / комплементарного" -- з 95% імовірністю це Map/Set.

---

## JavaScript: Map, Set, Object, WeakMap, WeakSet

### Map

Створена для використання як hash map. Ключі -- будь-що (об'єкти, примітиви), зберігає порядок вставки, має чіткий API.

```javascript
const map = new Map();

map.set("name", "Taras");
map.set(42, "число як ключ");
map.set({ id: 1 }, "об'єкт як ключ");

map.get("name");          // "Taras"
map.has(42);              // true
map.delete("name");       // true
map.size;                  // 2

// Ітерація зберігає порядок вставки
for (const [k, v] of map) {
  console.log(k, v);
}

// Конструктор приймає ітерабельне з пар
const m2 = new Map([["a", 1], ["b", 2]]);

// Корисний трюк: set повертає сам map -- можна чейнити
map.set("x", 1).set("y", 2).set("z", 3);
```

### Set

Як Map, але зберігає лише **значення** (без асоційованих пар). Всі значення унікальні.

```javascript
const set = new Set();

set.add(1);
set.add(2);
set.add(1);       // ігнорується -- вже є
set.size;          // 2

set.has(2);        // true
set.delete(1);

// Зручний спосіб зняти дублікати з масиву
const arr = [1, 2, 2, 3, 3, 3];
const unique = [...new Set(arr)];  // [1, 2, 3]
```

### Object

Теж hash map, але з історичним багажем:
- Ключі -- лише `string` або `Symbol` (числові ключі автоматично стають рядками).
- Має **prototype chain**: `obj["toString"]` поверне функцію, навіть якщо ти нічого не клав.
- Немає `size` -- треба `Object.keys(obj).length`, що O(n).
- Ітерація через `for...in` може захопити успадковані ключі.

```javascript
const obj = {};

obj[1] = "a";
obj["1"] = "b";   // перезаписує попереднє -- ключі однакові (рядок "1")!
obj.toString;      // функція з prototype -- а ми нічого не клали

// "Колізія" з prototype
const counts = {};
counts["hasOwnProperty"] = 1;    // перезаписує метод у межах цього об'єкта
counts.hasOwnProperty("x");       // TypeError! hasOwnProperty більше не функція
```

### WeakMap, WeakSet

Як Map/Set, але **ключі не заважають garbage collection**.

```javascript
const cache = new WeakMap();

let user = { id: 1 };
cache.set(user, "деякі дані");

user = null;      // об'єкт більше ніхто не тримає, крім WeakMap
// ↓ GC може прибрати об'єкт і запис із cache автоматично
```

Обмеження:
- Ключі -- тільки об'єкти (не примітиви).
- Не можна ітерувати (`for...of`, `.keys()`, `.size` -- нема).
- Немає `clear()`.

**Коли використовувати WeakMap/WeakSet:**
- **Приватні дані для об'єктів**, не хочеш тримати їх у полі (щоб не заважати GC).
- **Кеш, пов'язаний із життєвим циклом об'єкта** -- коли об'єкт зник, запис у кеші зникне сам.
- **Мітки (is-visited, is-processed)** для об'єктів у рекурсивних обходах, де не хочеш держати сильну референцію.

На алгоритмічних співбесідах WeakMap/WeakSet зустрічаються рідко. Але знати їх треба -- це рівень senior.

### Таблиця операцій і Big O

| Операція               | Map       | Set       | Object       | WeakMap   | WeakSet   |
|------------------------|-----------|-----------|--------------|-----------|-----------|
| Вставка                | O(1) amort.| O(1) amort.| O(1) amort. | O(1)      | O(1)      |
| Отримання / has        | O(1)      | O(1)      | O(1)         | O(1)      | O(1)      |
| Видалення              | O(1)      | O(1)      | O(1)         | O(1)      | O(1)      |
| Розмір                 | O(1)      | O(1)      | O(n) (keys) | немає     | немає     |
| Ітерація всього        | O(n)      | O(n)      | O(n)         | немає     | немає     |
| Ключі                  | будь-які  | будь-які  | string/Symbol| object    | object    |
| Зберігає порядок       | так       | так       | так (з ES2015)| ні      | ні        |
| Заважає GC             | так       | так       | так          | **ні**    | **ні**    |

На практиці `Map`/`Set` на ~10-50% повільніші за `Object` в мікробенчмарках (через overhead V8 оптимізацій для об'єктів як hidden classes). Але це рідко має значення -- і для алгоритмічних задач завжди Map/Set.

---

## Pattern 1: Counting / Frequency Map

**Ідея:** проходимо масив/рядок і рахуємо, скільки разів кожен елемент зустрічається.

```javascript
function countFreq(arr) {
  const freq = new Map();
  for (const x of arr) {
    freq.set(x, (freq.get(x) || 0) + 1);
  }
  return freq;
}

countFreq(["a", "b", "a", "c", "b", "a"]);
// Map { "a" => 3, "b" => 2, "c" => 1 }
```

O(n) time, O(k) space, де k -- кількість унікальних елементів.

### Приклад 1: Anagram check

Два рядки -- анаграми, якщо містять ті самі символи з тією самою частотою.

```javascript
// ❌ Наївно: O(n log n) через сортування
function isAnagramNaive(a, b) {
  if (a.length !== b.length) return false;
  return [...a].sort().join("") === [...b].sort().join("");
}

// ✅ O(n) через частотний словник
function isAnagram(a, b) {
  if (a.length !== b.length) return false;

  const freq = new Map();
  for (const ch of a) freq.set(ch, (freq.get(ch) || 0) + 1);

  for (const ch of b) {
    if (!freq.has(ch)) return false;
    const left = freq.get(ch) - 1;
    if (left < 0) return false;         // у b більше цього символу, ніж у a
    freq.set(ch, left);
  }
  return true;
}

isAnagram("listen", "silent");   // true
isAnagram("hello", "world");     // false
```

### Приклад 2: First non-repeating character

Знайти перший символ у рядку, який з'являється лише раз.

```javascript
function firstUnique(str) {
  const freq = new Map();
  for (const ch of str) freq.set(ch, (freq.get(ch) || 0) + 1);

  for (let i = 0; i < str.length; i++) {
    if (freq.get(str[i]) === 1) return i;
  }
  return -1;
}

firstUnique("leetcode");        // 0 ("l")
firstUnique("loveleetcode");    // 2 ("v")
```

Два проходи, кожен O(n) → O(n) total.

### Приклад 3: Top K frequent elements

Повернути k елементів, що зустрічаються найчастіше.

```javascript
function topKFrequent(nums, k) {
  // 1. Рахуємо частоти: O(n)
  const freq = new Map();
  for (const x of nums) freq.set(x, (freq.get(x) || 0) + 1);

  // 2. Bucket sort по частоті: O(n)
  //    buckets[i] -- масив елементів, що зустрілись рівно i разів
  const buckets = Array.from({ length: nums.length + 1 }, () => []);
  for (const [num, count] of freq) {
    buckets[count].push(num);
  }

  // 3. Збираємо k найчастіших, ідучи від найбільшого до найменшого: O(n)
  const result = [];
  for (let i = buckets.length - 1; i >= 0 && result.length < k; i--) {
    for (const num of buckets[i]) {
      result.push(num);
      if (result.length === k) break;
    }
  }
  return result;
}

topKFrequent([1, 1, 1, 2, 2, 3], 2);   // [1, 2]
```

Загальна складність: O(n). Це краще за наївне O(n log n) через сортування.

**Альтернатива через Min-Heap:** O(n log k). Коли k << n, це буває вигідніше, ніж bucket sort.

---

## Pattern 2: Complement Lookup

**Ідея:** для кожного елемента питаємо "чи є в мапі той, який його доповнює до потрібної властивості". Замість шукати пару перебором.

### Класика: Two Sum

Знайти індекси двох чисел у масиві, сума яких дорівнює `target`.

```javascript
// ❌ O(n²): всі пари
function twoSumNaive(nums, target) {
  for (let i = 0; i < nums.length; i++) {
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] + nums[j] === target) return [i, j];
    }
  }
  return null;
}

// ✅ O(n): один прохід + мапа
function twoSum(nums, target) {
  const seen = new Map();                 // значення -> індекс
  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];  // яке число нам треба
    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }
    seen.set(nums[i], i);
  }
  return null;
}

twoSum([2, 7, 11, 15], 9);   // [0, 1], бо 2+7=9
```

**Чому це працює.** Ми йдемо зліва направо. Для поточного `nums[i]`, якщо колись раніше бачили `target - nums[i]` -- це наша пара. Якщо ні -- кладемо в мапу поточний, щоб майбутні елементи могли знайти нас.

**Ключова думка:** trade time for space. Витратили O(n) пам'яті, зекономили O(n) часу в найгіршому випадку.

### 3Sum (варіація)

Знайти всі унікальні трійки, сума яких = 0. Тут Map теж допомагає, але канонічне рішення -- через сортування + two pointers (буде в наступному файлі). Згадка тут -- щоб знав, що чистий hash map не завжди оптимальний.

---

## Pattern 3: Prefix Sum + Map

Надзвичайно потужний патерн для задач на підмасиви.

**Prefix sum:** `prefix[i]` = сума елементів `arr[0..i-1]`.

```
arr:     [2, 4, 1, 3, 5]
prefix:  [0, 2, 6, 7, 10, 15]
             ↑  ↑  ↑   ↑   ↑
             після 0 елементів: 0
             після 1 (тобто 2): 2
             після 2 (2+4): 6
             після 3 (2+4+1): 7
             ...

Сума arr[l..r] = prefix[r+1] - prefix[l].
Напр., сума arr[1..3] = prefix[4] - prefix[1] = 10 - 2 = 8. (4+1+3=8 ✓)
```

### Subarray Sum Equals K

Дано масив чисел `nums` і число `k`. Скільки є **неперервних підмасивів**, сума яких = `k`?

```javascript
// ❌ O(n²): всі підмасиви
function subarraySumNaive(nums, k) {
  let count = 0;
  for (let i = 0; i < nums.length; i++) {
    let sum = 0;
    for (let j = i; j < nums.length; j++) {
      sum += nums[j];
      if (sum === k) count++;
    }
  }
  return count;
}

// ✅ O(n): prefix sum + map частот
function subarraySum(nums, k) {
  const prefixCount = new Map();
  prefixCount.set(0, 1);    // "порожній префікс" -- 1 спосіб отримати суму 0

  let sum = 0;
  let count = 0;

  for (const x of nums) {
    sum += x;                            // поточний prefix sum
    // Якщо десь раніше був prefix = sum - k, то
    // від того місця до поточного -- підмасив суми k.
    if (prefixCount.has(sum - k)) {
      count += prefixCount.get(sum - k);
    }
    prefixCount.set(sum, (prefixCount.get(sum) || 0) + 1);
  }
  return count;
}

subarraySum([1, 1, 1], 2);         // 2 (підмасиви [0..1] і [1..2])
subarraySum([1, 2, 3], 3);         // 2 ([0..1] і [2..2])
```

**Чому це працює.** Якщо `sum(0..i) - sum(0..j) = k`, то `sum(j+1..i) = k`. Тобто для кожного поточного `sum`, нас цікавить, скільки префіксів мають значення `sum - k`. Тримаємо кількість кожного префіксу в мапі -- і відповідь одразу.

**Ключова деталь:** `prefixCount.set(0, 1)` -- це для випадків, коли весь префікс від початку дорівнює `k` (не треба ніяких відкидань).

---

## Pattern 4: Hash Set для швидкого has-check

Коли нам потрібен лише факт "чи бачили ми це раніше" (без значень) -- Set.

### Contains Duplicate

Чи є в масиві хоча б один дубль?

```javascript
// ❌ O(n²)
function containsDuplicateNaive(nums) {
  for (let i = 0; i < nums.length; i++) {
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] === nums[j]) return true;
    }
  }
  return false;
}

// ✅ O(n)
function containsDuplicate(nums) {
  const seen = new Set();
  for (const x of nums) {
    if (seen.has(x)) return true;
    seen.add(x);
  }
  return false;
}

// 🧙 Трюк-однорядковий (елегантно, але менш читаємо):
const containsDup = (nums) => new Set(nums).size !== nums.length;
```

### Intersection of Two Arrays

Знайти елементи, що є в обох масивах.

```javascript
// ❌ O(n × m)
function intersectionNaive(a, b) {
  return a.filter(x => b.includes(x));
}

// ✅ O(n + m)
function intersection(a, b) {
  const setB = new Set(b);
  const result = new Set();
  for (const x of a) {
    if (setB.has(x)) result.add(x);
  }
  return [...result];
}

intersection([1, 2, 2, 3], [2, 3, 4]);   // [2, 3]
```

### Longest Consecutive Sequence (класика)

Дано невідсортований масив чисел. Знайти **довжину** найдовшої послідовності послідовних цілих (у будь-якому порядку в масиві).

```
Вхід:  [100, 4, 200, 1, 3, 2]
Вихід: 4   (послідовність 1, 2, 3, 4)
```

Наївне рішення -- відсортувати: O(n log n). А можна O(n)!

```javascript
function longestConsecutive(nums) {
  const set = new Set(nums);
  let longest = 0;

  for (const x of set) {
    // Починаємо відлік ЛИШЕ з початку послідовності.
    // Якщо x-1 є в сеті -- x не початок, хтось інший його порахує.
    if (set.has(x - 1)) continue;

    let current = x;
    let length = 1;
    while (set.has(current + 1)) {       // розширюємо вправо
      current++;
      length++;
    }
    longest = Math.max(longest, length);
  }
  return longest;
}

longestConsecutive([100, 4, 200, 1, 3, 2]);   // 4
```

**Чому O(n), а не O(n²)?** Виглядає так, ніби зовнішній цикл × внутрішній while = O(n²). Але:
- Внутрішній while **виконується лише тоді, коли `x` -- початок послідовності**.
- Кожне число "розширюється" рівно один раз за всю роботу.
- Тому сума ітерацій усіх внутрішніх while -- O(n).

Це **amortized analysis** у дії. На співбесіді обов'язково озвуч це пояснення -- воно показує глибину.

### Happy Number

Число "щасливе", якщо повторне виконання "сума квадратів цифр" зрештою дає 1. Інакше -- цикл, і число не щасливе.

```javascript
function isHappy(n) {
  const seen = new Set();

  const sumSq = (num) => {
    let sum = 0;
    while (num > 0) {
      const d = num % 10;
      sum += d * d;
      num = Math.floor(num / 10);
    }
    return sum;
  };

  while (n !== 1 && !seen.has(n)) {
    seen.add(n);
    n = sumSq(n);
  }
  return n === 1;
}

isHappy(19);    // true (1²+9²=82, 8²+2²=68, 6²+8²=100, 1²+0²+0²=1)
isHappy(2);     // false (цикл)
```

Set тут -- детектор циклу. Якщо вже бачили число -- значить зациклились.

---

## Pattern 5: Grouping (групування)

**Ідея:** усі елементи з однаковим ключем кладемо в один "bucket". Ключ -- результат якоїсь нормалізації.

### Group Anagrams

Згрупувати рядки, які є анаграмами один одного.

```
Вхід:  ["eat", "tea", "tan", "ate", "nat", "bat"]
Вихід: [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

**Трюк:** два рядки -- анаграми ⇔ мають однаковий відсортований вигляд. Використовуємо це як ключ.

```javascript
function groupAnagrams(strs) {
  const groups = new Map();

  for (const s of strs) {
    const key = [...s].sort().join("");        // нормалізований ключ
    if (!groups.has(key)) groups.set(key, []);
    groups.get(key).push(s);
  }

  return [...groups.values()];
}

groupAnagrams(["eat", "tea", "tan", "ate", "nat", "bat"]);
// [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

Складність: O(n × k log k), де n -- кількість рядків, k -- середня довжина.

**Ще швидша нормалізація** (якщо рядки з ASCII-букв): замість сортування -- частотний підпис.

```javascript
function groupAnagramsFaster(strs) {
  const groups = new Map();

  for (const s of strs) {
    const counts = new Array(26).fill(0);
    for (const ch of s) counts[ch.charCodeAt(0) - 97]++;
    const key = counts.join(",");          // "1,0,0,...,1"
    if (!groups.has(key)) groups.set(key, []);
    groups.get(key).push(s);
  }
  return [...groups.values()];
}
```

O(n × k) -- краще на довгих рядках.

**Узагальнення pattern'у:** якщо задача каже "згрупуй за X" -- X і є ключем Map. Розраховуй X, кидай у бакет.

---

## Коли Map, а коли Object

На співбесіді: **завжди Map**. Причини:
1. Ключами можуть бути не тільки рядки/Symbols: об'єкти, числа, навіть функції.
2. Немає prototype pollution (`obj["toString"]` -- завжди функція, навіть якщо `toString` -- легітимний ключ у твоєму домені).
3. `.size` -- O(1). Для об'єкта `Object.keys(obj).length` -- O(n).
4. Ітерація через `for...of` дає пари, зрозуміла і без `Object.entries`.
5. Зберігає порядок вставки формально за специфікацією (в об'єкта це теж є з ES2015, але з квирками для числових ключів).

Приклад, де Object ламається:

```javascript
// Числові ключі стають рядками
const obj = {};
obj[1] = "a";
obj["1"] = "b";
console.log(obj);            // { "1": "b" } -- один ключ!

// У Map це різні ключі
const map = new Map();
map.set(1, "a");
map.set("1", "b");
console.log(map.size);       // 2

// Prototype chain
const counts = {};
"toString" in counts;        // true! навіть без вставки
// А в Map:
const counts2 = new Map();
counts2.has("toString");     // false ✓
```

У продакшн-коді, де ключі завжди рядки, об'єкти теж нормально (зокрема, їх легко серіалізувати в JSON). Але для алгоритмів -- Map.

---

## Collisions і worst case

Коли всі ключі "злі" -- вони потрапляють в один бакет -- hash map деградує до лінкед ліста, операції стають O(n).

### Hash flooding / DoS attack

Це не теоретична загроза. У 2011 році було публічно показано, що HTTP-сервери (PHP, ASP.NET, Java, Python) можна покласти, надсилаючи POST-запит із тисячами параметрів, ключі яких підібрані так, щоб колідувати. Парсинг форм викликав O(n²) замість O(n) -- і сервер "вмирав".

**Реакція мов:**
- **Node.js / V8:** для рядкових ключів використовує **рандомізовану хеш-функцію** -- seed генерується при старті процесу. Атакувальник не може наперед знати, які ключі колідуватимуть.
- **Python** те саме з Python 3.3+ (`PYTHONHASHSEED`).
- **Java** для HashMap робить додатковий трюк: якщо бакет має багато елементів -- перетворює його на **балансоване дерево** (red-black tree). Worst case стає O(log n), а не O(n).

### Практичні висновки для JS

- У звичайному код-ревью не треба думати про колізії -- двигун про це подбав.
- Єдине реальне місце, де це може стрельнути -- якщо приймаєш ключі від користувача (наприклад, як назви полів в API) і складаєш їх у `Map`/`Object`. Краще обмежити кількість ключів.
- На співбесіді достатньо знати: **average O(1), worst O(n), V8 рандомізує hash seed**.

---

## Hashing custom objects

Проблема: в JS два "однакові" об'єкти -- різні ключі.

```javascript
const map = new Map();
map.set({ x: 1, y: 2 }, "A");
map.get({ x: 1, y: 2 });      // undefined! Це ІНШИЙ об'єкт.
```

Причина: Map порівнює ключі через **SameValueZero** (як `===` для об'єктів -- за референсом).

### Рішення 1: Composite key через серіалізацію

Перетворюємо об'єкт у рядок і використовуємо його як ключ.

```javascript
const map = new Map();

const key1 = JSON.stringify({ x: 1, y: 2 });
map.set(key1, "A");

const key2 = JSON.stringify({ x: 1, y: 2 });
map.get(key2);         // "A" ✓
```

**Пастки `JSON.stringify`:**
```javascript
JSON.stringify({ a: 1, b: 2 });       // '{"a":1,"b":2}'
JSON.stringify({ b: 2, a: 1 });       // '{"b":2,"a":1}' -- ІНШИЙ рядок!

// Порядок полів впливає на ключ. Рішення -- сортувати ключі:
function stableKey(obj) {
  const keys = Object.keys(obj).sort();
  const sorted = {};
  for (const k of keys) sorted[k] = obj[k];
  return JSON.stringify(sorted);
}
```

Для простих композитних ключів часто достатньо конкатенації:

```javascript
// Задача: порахувати позиції на координатній сітці
const grid = new Map();

const key = (x, y) => `${x},${y}`;         // простий composite key
grid.set(key(3, 5), "ладья");
grid.has(key(3, 5));                        // true
```

### Рішення 2: Об'єкт як ключ напряму

Якщо ключ -- той самий референс об'єкта (напр., у тебе є канонічний об'єкт user, і ти асоціюєш із ним дані) -- Map підтримує це нативно:

```javascript
const user1 = { id: 1 };
const user2 = { id: 2 };

const scores = new Map();
scores.set(user1, 100);
scores.set(user2, 85);

scores.get(user1);      // 100
```

Це **не працює з Object** -- там ключі конвертуються в рядки.

### Рішення 3: Двовимірна Map (для 2D композитних ключів)

```javascript
// Замість `${x},${y}` -- вкладені мапи
const grid = new Map();

function setCell(x, y, value) {
  if (!grid.has(x)) grid.set(x, new Map());
  grid.get(x).set(y, value);
}

function getCell(x, y) {
  return grid.get(x)?.get(y);
}

setCell(3, 5, "ладья");
getCell(3, 5);     // "ладья"
```

Вибір між підходами -- trade-off між простотою (stringify) і продуктивністю (nested Map швидше на частих оновленнях, без парсингу рядка).

---

## Типові пастки (перелік, який потрібно знати напам'ять)

### Пастка 1: Object замість Map при числових ключах

```javascript
// ❌
const counts = {};
counts[1] = "a";
counts["1"] = "b";        // перезаписано! ключі злилися

// Ще гірше -- великі числа
counts[1000000000000000000] = "x";   // стає рядком "1e+21"
```

Для числових ключів -- завжди Map.

### Пастка 2: `map.get(key) || 0` ламається, коли 0 -- легітимне значення

```javascript
const map = new Map();
map.set("score", 0);

// ❌ Хочемо: "якщо є значення -- використай його, інакше 100"
const score = map.get("score") || 100;     // 100 -- БАГ, насправді там 0

// ✅ Варіанти правильно:
const score2 = map.get("score") ?? 100;    // nullish coalescing
const score3 = map.has("score") ? map.get("score") : 100;
```

Те саме для `""`, `false`, `null` -- усі falsy значення збивають `||`.

### Пастка 3: `{...obj}` як "copy before modify" = O(n)

```javascript
// ❌ Immutable-стиль у reduce виглядає елегантно, але це O(n²) загалом!
const freq = arr.reduce((acc, x) => ({ ...acc, [x]: (acc[x] || 0) + 1 }), {});
// Кожна ітерація копіює весь acc -- n × n = n²

// ✅ Mutate -- нормально для локальної змінної
const freq2 = {};
for (const x of arr) freq2[x] = (freq2[x] || 0) + 1;
```

Таке саме правило для `[...arr, x]` у циклі -- O(n²). Це часта "синдромна" помилка Node.js-розробників, які звикли до Redux-стилю.

### Пастка 4: Ітерація Map + мутація

```javascript
const map = new Map([["a", 1], ["b", 2], ["c", 3]]);

// ❌ Спочатку подумай, чи безпечно
for (const [k, v] of map) {
  if (v < 3) map.delete(k);      // технічно дозволено в JS, але легко заплутатись
}
```

Видалення під час ітерації в Map дозволене специфікацією (на відміну від деяких мов), але краще зібрати ключі окремо і видаляти після. Читабельніше.

### Пастка 5: Забутий default при підрахунку

```javascript
// ❌ NaN у підсумку
for (const x of arr) freq.set(x, freq.get(x) + 1);        // при першій зустрічі: undefined + 1 = NaN

// ✅
for (const x of arr) freq.set(x, (freq.get(x) || 0) + 1); // при першій зустрічі: 0 + 1 = 1
```

### Пастка 6: Set і NaN

```javascript
const set = new Set();
set.add(NaN);
set.has(NaN);    // true -- Set спеціально обробляє NaN (SameValueZero)
```

Це правильно для Set (і Map), хоча NaN !== NaN у звичайному `===`. Приємна деталь, яку не всі знають.

### Пастка 7: Довільний об'єкт як ключ Object

```javascript
const obj = {};
const key = { id: 1 };

obj[key] = "value";
// key невидимо конвертований: "[object Object]"
// Будь-який інший об'єкт буде мати той самий ключ!
obj[{ id: 2 }];           // "value" -- бо теж "[object Object]"
```

Якщо ключ -- об'єкт, тільки Map (або WeakMap).

---

## Чекліст перед тим як писати код із hash map

1. **Ключ -- що це?** Рядок / число / об'єкт / композит? Якщо не рядок → Map.
2. **Мапимо в що?** Частоту? Індекс? Список? Інший об'єкт?
3. **Чи треба порядок?** Map зберігає порядок вставки. Set -- теж.
4. **Default value при першому доступі?** `map.get(k) ?? 0`, а не `|| 0`.
5. **Чи треба забути пари, коли зник ключ-об'єкт?** → WeakMap.
6. **Скільки унікальних ключів максимум?** Це твоє space complexity.

---

## Шпаргалка патернів

| Патерн                   | Коли використати                                   | Приклад задачі                    |
|--------------------------|----------------------------------------------------|------------------------------------|
| Frequency Map            | "Скільки разів кожен X?"                           | Anagram, Top K, Majority Element  |
| Complement Lookup        | "Знайти пару з певною властивістю"                 | Two Sum, Pairs with target sum    |
| Prefix Sum + Map         | "Підмасив із сумою K"                              | Subarray Sum Equals K             |
| Hash Set для has-check   | "Чи бачили X раніше / чи є X у наборі"             | Contains Duplicate, Longest Seq   |
| Grouping                 | "Згрупуй елементи за нормалізованим ключем"        | Group Anagrams, Group by category |

---

## Ключові думки

- Hash map -- **обмін пам'яті на швидкість**. Усе, що було O(n²) через пошук, стає O(n) через запам'ятовування.
- У JS для алгоритмів **завжди Map, завжди Set**. Object для продакшн-даних, де ключі -- передбачувані рядки.
- `(map.get(k) || 0) + 1` -- стандартна ідіома для Frequency Map. Але ??, не ||, коли 0/false/"" -- легітимні значення.
- Prefix Sum + Map -- must-know комбінація. Відкриває цілий клас задач на підмасиви.
- Longest Consecutive Sequence -- канонічна задача. Якщо витягнеш її -- виглядаєш впевнено.
- Average O(1), worst O(n) через колізії. Для джуна достатньо знати average. Для senior -- треба вміти пояснити chaining і чому V8 рандомізує seed.
- Composite key через `JSON.stringify` або `${x},${y}` -- стандартний трюк, коли ключ -- об'єкт або пара.

У наступному файлі -- масиви і рядки з two-pointers і sliding window.
