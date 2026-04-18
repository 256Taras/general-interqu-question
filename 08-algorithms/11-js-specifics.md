# JavaScript-специфічні нюанси для алгоритмічних співбесід

## Вступ: JS на алгоритмічній співбесіді -- це компроміс

JavaScript -- **не найкраща мова** для алгоритмічних інтерв'ю. Для порівняння:
- **C++** -- швидкий, явні типи (`int`, `long long`), чітко видно integer overflow, зручні контейнери STL.
- **Python** -- arbitrary-precision integers, мінімум boilerplate, швидко писати.
- **Java** -- суворі типи, зручні `HashMap`/`TreeMap`/`ArrayDeque`, чітка поведінка.
- **JavaScript** -- один тип `Number` (float під капотом), немає int/long, немає TreeMap/SortedSet, слабша поведінка при великих числах.

Чому тоді писати на JS?
1. Якщо ти щоденно пишеш Node.js -- ти **впевнено** володієш мовою, не губишся в синтаксисі.
2. Інтерв'юер майже завжди дозволяє "на мові, яку знаєш найкраще".
3. JS достатній для 95% задач з LeetCode. Питання не в мові, а в умінні думати.

**Що треба знати, щоб не спіткнутися:**
- Числа -- float, а не int. Overflow проявляється не так, як у C++.
- Немає вбудованого сортованого контейнера (`TreeMap`, `SortedSet`) -- для задач з "знайти найближче менше" треба писати самому або емулювати.
- Рекурсія без tail call optimization -- при великій глибині зламає стек.
- Стандартні методи масиву мають неочевидну складність (`shift`, `unshift`, `splice` -- O(n)).
- Об'єкт != Map. На алгоритмах **завжди Map**.
- Immutable строки -- `str += x` у циклі -- O(n²).

Цей файл -- про всі ці пастки і як їх обходити.

---

## Числа в JavaScript: один тип на всі випадки

У JS немає окремих `int`, `long`, `float`, `double`. Все -- один тип `Number`, який під капотом є **64-бітним float (IEEE 754)**.

Це має три наслідки.

### Наслідок 1: Проблеми точності при арифметиці з дробами

```javascript
// Класичний сюрприз для новачків (і не тільки)
console.log(0.1 + 0.2);          // 0.30000000000000004
console.log(0.1 + 0.2 === 0.3);  // false

console.log(0.3 - 0.2);          // 0.09999999999999998
console.log(0.3 - 0.2 === 0.1);  // false
```

Це не баг JS -- це IEEE 754. Те саме у Python, Java (для `double`), C++ (для `double`).

**На алгоритмах це рідко проблема**, бо більшість задач з LeetCode оперують цілими числами. Але треба пам'ятати:

```javascript
// ❌ Небезпечно: порівнювати float напряму
if (result === 0.3) { ... }

// ✅ Безпечно: порівнювати з epsilon
const EPSILON = 1e-9;
if (Math.abs(result - 0.3) < EPSILON) { ... }
```

### Наслідок 2: Safe integer range -- 2⁵³-1, не 2⁶³-1

```javascript
console.log(Number.MAX_SAFE_INTEGER);  // 9007199254740991  (2^53 - 1)
console.log(Number.MIN_SAFE_INTEGER);  // -9007199254740991
```

Ціле число більше цього -- **не гарантовано точне**.

```javascript
const big = Number.MAX_SAFE_INTEGER;
console.log(big + 1);      // 9007199254740992  (ще коректно)
console.log(big + 2);      // 9007199254740992  (!!! -- втрата точності)
console.log(big + 1 === big + 2);  // true  (а мало б бути false)
```

У C++/Java з `long long` (64-bit) можна оперувати числами до 2⁶³-1 ≈ 9 × 10¹⁸. У JS -- "тільки" до 2⁵³-1 ≈ 9 × 10¹⁵. Різниця 1000×.

**Коли треба більше -- використовуй `BigInt`:**

```javascript
const big = 9007199254740991n;   // суфікс "n" робить BigInt
const bigger = big + 1n;
console.log(bigger);             // 9007199254740992n
console.log(bigger + 10n);       // 9007199254741002n -- точно

// BigInt і Number не змішуються без явної конверсії
// BigInt(5) + 3 → TypeError
console.log(BigInt(5) + 3n);     // 8n
console.log(Number(bigger));     // 9007199254740992 (може загубити точність)
```

**Коли на співбесіді використовувати BigInt:**
- Великі множення -- множення двох n ≤ 10⁹ може дати 10¹⁸, а це вже поза safe integer.
- Хеш-коди для rolling hash / Rabin-Karp: множення beta × hash + char легко вилітає за 2⁵³.
- Задачі з комбінаторикою (факторіали, біноміальні коефіцієнти).
- Modular exponentiation -- `a^b mod p` з великими a, b.

**Застереження:** BigInt повільніший за Number (в 5-50 разів залежно від операції). Не використовуй без потреби.

### Наслідок 3: Integer division -- немає `//`

У Python є `//`, в C++ -- автоматично при діленні `int / int`. У JS `/` завжди float:

```javascript
console.log(7 / 2);              // 3.5, а не 3
console.log(Math.floor(7 / 2));  // 3 -- так і треба робити

// Для додатних чисел і n < 2³¹ є трюк з бітовим OR
console.log((7 / 2) | 0);        // 3 -- швидко, але тільки для 32-bit range
console.log((10000000000 / 3) | 0);  // БАГО! 32-bit overflow, не те що очікуєш

// Для від'ємних чисел Math.floor і |0 дають різні результати
console.log(Math.floor(-7 / 2)); // -4 (округлення вниз)
console.log((-7 / 2) | 0);       // -3 (truncation до нуля)
console.log(Math.trunc(-7 / 2)); // -3 (явний truncation)
```

**Правило:** у проєкті -- `Math.floor` / `Math.trunc`. Оптимізацію `| 0` застосовуй тільки якщо впевнений у діапазоні.

---

## Bitwise operations -- твій друг на деяких задачах

Бітові оператори `&`, `|`, `^`, `~`, `<<`, `>>`, `>>>` **завжди** працюють з 32-бітними цілими зі знаком (навіть якщо ти передаєш більше число, воно обріжеться до 32 біт).

```javascript
console.log(0xFFFFFFFFFFFF & 0xFF);  // 255 -- обрізається до 32 біт
```

**Практичні прийоми для алгоритмів:**

```javascript
// Перевірка парності -- швидше ніж n % 2
if (n & 1) { /* непарне */ }

// Ділення на 2 націло (для додатних) -- швидше ніж n/2
const half = n >> 1;

// Множення на 2 -- швидше ніж n*2
const doubled = n << 1;

// XOR для задачі "single number"
// Всі елементи зустрічаються двічі, крім одного -- знайти його
function singleNumber(arr) {
  let result = 0;
  for (const x of arr) result ^= x;
  return result;
}
// Працює, бо x ^ x = 0, x ^ 0 = x

// Перевірка, чи n -- степінь двійки
function isPowerOfTwo(n) {
  return n > 0 && (n & (n - 1)) === 0;
}

// Кількість встановлених бітів (popcount)
function popcount(n) {
  let count = 0;
  while (n) {
    n &= n - 1;      // знімає найправіший 1-біт
    count++;
  }
  return count;
}
```

**Пастка:** `>>>` -- unsigned right shift. Потрібен, коли хочеш працювати з числом як з unsigned 32-bit:

```javascript
console.log(-1 >> 1);    // -1 (sign extension зберігає знак)
console.log(-1 >>> 1);   // 2147483647 (всі біти як unsigned)
```

### Division by zero -- не виняток, а Infinity

У C++/Java ділення на нуль -- виняток чи UB. У JS:

```javascript
console.log(10 / 0);     //  Infinity
console.log(-10 / 0);    // -Infinity
console.log(0 / 0);      //  NaN
console.log(10 % 0);     //  NaN
```

**Пастка на співбесіді:** якщо пишеш бінарний пошук або формулу з діленням -- перевір, чи дільник не 0. Код не впаде, а мовчки поверне `Infinity`, і ти полюватимеш на баг півгодини.

```javascript
// ❌ Тихий баг: rate = 0 дасть Infinity, якщо не перевірити
function timeToFill(volume, rate) {
  return volume / rate;
}

// ✅ Явна перевірка
function timeToFill(volume, rate) {
  if (rate === 0) throw new Error('rate must be positive');
  return volume / rate;
}
```

---

## Масиви: звичайні vs типізовані

### Звичайний Array -- універсальний, але "товстий"

```javascript
const arr = [1, 'hello', {x: 1}, null];  // будь-які типи
arr[100] = 'x';                           // масив розріджений
console.log(arr.length);                  // 101
console.log(arr[50]);                     // undefined -- "hole"

// V8 обробляє розріджені масиви як slow objects.
// Не створюй "дір" зумисне, якщо хочеш швидкість.
```

Звичайний Array у V8 внутрішньо зберігається як:
- **Packed** -- щільний, без "дірок" -- швидкий доступ.
- **Holey** -- з "дірками" -- повільніший.
- **SMI-only** -- всі елементи small integers -- найшвидший.
- **Double** -- всі floats -- теж швидкий.
- **Object** -- різні типи -- повільніший.

Щоб масив залишався швидким -- **не змішуй типи**.

### Typed arrays -- коли потрібна швидкість

```javascript
const ints = new Int32Array(1000);       // 1000 нулів, кожен 32-bit int
const bytes = new Uint8Array(256);       // 256 нулів, кожен 8-bit
const floats = new Float64Array(100);    // 100 нулів, кожен 64-bit float

ints[0] = 42;
ints[1] = 'hello';    // "hello" → 0 (тихо конвертується у число)
ints.length;          // 1000 (фіксована!)
ints.push(5);         // ❌ TypeError -- нема push/pop
```

**Коли використовувати Typed Arrays:**
- Великий DP-масив чисел (10⁶+ елементів) -- `Int32Array` може бути в 2-5 разів швидший.
- Sieve of Eratosthenes -- `Uint8Array` як bit vector.
- Hashing / checksum / компресія -- `Uint8Array` для байтів.

**Обмеження:** фіксований розмір, тільки числові типи. Методи `map`, `filter` є, але повертають той самий typed array (не звичайний).

### Створення масиву фіксованого розміру

```javascript
// Найпростіше -- порожні holes (НЕ використовуй для циклів)
const a = new Array(5);
console.log(a);              // [ <5 empty items> ]
a.forEach(x => console.log(x));  // нічого не виведе! forEach пропускає holes

// ✅ З початковим значенням
const b = new Array(5).fill(0);
console.log(b);              // [0, 0, 0, 0, 0]

// ✅ Масив з індексами 0..n-1
const c = Array.from({length: 5}, (_, i) => i);
console.log(c);              // [0, 1, 2, 3, 4]
```

### Пастка: `fill` з посиланням на об'єкт

```javascript
// ❌ ПАСТКА: всі 3 елементи посилаються на ТОЙ САМИЙ масив
const grid = new Array(3).fill([]);
grid[0].push(1);
console.log(grid);  // [[1], [1], [1]] -- не те, що очікували!

// ✅ Правильно: створюй окремі масиви для кожного індексу
const grid = Array.from({length: 3}, () => []);
grid[0].push(1);
console.log(grid);  // [[1], [], []]
```

Те саме з об'єктами:

```javascript
// ❌ Один об'єкт, розшарений
const users = new Array(3).fill({name: ''});
users[0].name = 'Taras';
console.log(users);  // [{name:'Taras'}, {name:'Taras'}, {name:'Taras'}]

// ✅
const users = Array.from({length: 3}, () => ({name: ''}));
```

### 2D-масиви

```javascript
const m = 3, n = 4;

// ❌ Все посилається на один рядок
const bad = new Array(m).fill(new Array(n).fill(0));
bad[0][0] = 5;
console.log(bad);   // [[5,0,0,0], [5,0,0,0], [5,0,0,0]]

// ✅ Правильно
const grid = Array.from({length: m}, () => new Array(n).fill(0));
grid[0][0] = 5;
console.log(grid);  // [[5,0,0,0], [0,0,0,0], [0,0,0,0]]

// Альтернатива з циклом (зрозуміліше)
const grid = [];
for (let i = 0; i < m; i++) {
  grid.push(new Array(n).fill(0));
}
```

Візуалізація "розшареного" рядка:

```
new Array(3).fill([0,0])  →   індекс 0  ──┐
                              індекс 1  ──┼──→ [0, 0]   (один масив!)
                              індекс 2  ──┘

Array.from({length:3}, () => [0,0])   →   індекс 0  →  [0, 0]
                                          індекс 1  →  [0, 0]  (різні масиви)
                                          індекс 2  →  [0, 0]
```

---

## Copy vs Reference: мовчазні пастки

### Основне правило

- **Примітиви** (`number`, `string`, `boolean`, `null`, `undefined`, `symbol`, `bigint`) -- передаються **за значенням**.
- **Об'єкти** (включно з масивами і функціями) -- **за посиланням**.

```javascript
function increment(x) { x++; }
let n = 5;
increment(n);
console.log(n);      // 5 (примітив)

function pushToArr(arr) { arr.push(42); }
let a = [1, 2, 3];
pushToArr(a);
console.log(a);      // [1, 2, 3, 42] (об'єкт, змінено in-place)
```

### Shallow copy -- O(n), не O(1)

```javascript
const arr = [1, 2, 3];
const copy = [...arr];      // O(n) -- копіює кожен елемент
const clone = arr.slice();  // O(n)
const clone2 = [...arr];    // O(n)

const obj = {a: 1, b: 2};
const objCopy = {...obj};   // O(кількість ключів)
```

**Що означає "shallow":** копіюється верхній рівень, вкладені об'єкти/масиви залишаються по посиланню.

```javascript
const original = {name: 'Taras', tags: ['js', 'node']};
const shallow = {...original};
shallow.name = 'Ivan';        // не впливає на original
shallow.tags.push('algo');    // ВПЛИВАЄ на original.tags!

console.log(original.tags);   // ['js', 'node', 'algo']  ❗
```

### Deep copy

```javascript
// Варіант 1: structuredClone (Node 17+)
const deep = structuredClone(original);

// Варіант 2: JSON -- швидкий, але обмежений
const deep2 = JSON.parse(JSON.stringify(original));
// Втрачає: Date → string, Map → {}, Set → {}, функції, Symbol, undefined,
// BigInt → throws, циклічні посилання → throws.

// Варіант 3: рекурсивна функція -- якщо хочеш повний контроль
function deepClone(x) {
  if (x === null || typeof x !== 'object') return x;
  if (Array.isArray(x)) return x.map(deepClone);
  const out = {};
  for (const k of Object.keys(x)) out[k] = deepClone(x[k]);
  return out;
}
```

**Для алгоритмів** здебільшого достатньо shallow copy (`[...arr]` чи `arr.slice()`). Deep copy потрібен рідко -- зазвичай коли ти береш input зі складних об'єктів.

### Класична пастка backtracking

```javascript
// Задача: згенерувати всі підмножини масиву
function subsets(nums) {
  const result = [];
  const current = [];

  function dfs(i) {
    if (i === nums.length) {
      // ❌ ПАСТКА: push посилання на current
      result.push(current);
      return;
    }
    dfs(i + 1);                // без nums[i]
    current.push(nums[i]);
    dfs(i + 1);                // з nums[i]
    current.pop();
  }

  dfs(0);
  return result;
}

console.log(subsets([1, 2]));
// Очікуєш: [[], [2], [1], [1,2]]
// Отримаєш: [[], [], [], []]  ← всі посилаються на той самий current,
//                               який у кінці порожній
```

**Правильно:**

```javascript
function dfs(i) {
  if (i === nums.length) {
    result.push([...current]);  // ✅ копія на момент збереження
    return;
  }
  // ...
}
```

Це -- **одна з топ-3 пасток на алгоритмічній секції**. Будь уважний.

---

## Sort quirks: кілька міц, що тебе обов'язково спробують

### Пастка 1: Sort без comparator -- лексикографічний

```javascript
[10, 2, 1].sort();                    // [1, 10, 2]  ❗ не [1, 2, 10]
['banana', 'apple'].sort();            // ['apple', 'banana'] (рядки -- ok)
[1, '10', 2].sort();                   // [1, '10', 2]
```

Причина: `sort` без аргументу конвертує елементи у рядки і сортує лексикографічно. "10" < "2", бо '1' < '2'.

**Правило:** для чисел **завжди** передавай comparator.

```javascript
[10, 2, 1].sort((a, b) => a - b);     // [1, 2, 10]   ↑ за зростанням
[10, 2, 1].sort((a, b) => b - a);     // [10, 2, 1]   ↓ за спаданням
```

### Пастка 2: `a - b` для дуже великих чисел

У JS `a - b` повертає float. Для чисел поза safe integer range можливі проблеми:

```javascript
const arr = [Number.MAX_SAFE_INTEGER, Number.MAX_SAFE_INTEGER - 1];
arr.sort((a, b) => a - b);   // працює: різниця -1

// Але якщо числа дуже різні та великі -- теоретично можуть бути проблеми.
// Стандартний workaround -- явна перевірка:
arr.sort((a, b) => {
  if (a < b) return -1;
  if (a > b) return 1;
  return 0;
});
```

Для практики LeetCode це майже не трапляється. Але **для BigInt -- `a - b` не працює**, бо comparator має повертати Number:

```javascript
const bigs = [10n, 2n, 5n];

// ❌ TypeError: не можна віднімати BigInt і повертати Number
bigs.sort((a, b) => a - b);

// ✅
bigs.sort((a, b) => (a < b ? -1 : a > b ? 1 : 0));
```

### Пастка 3: Sort мутує, не повертає копію

```javascript
const arr = [3, 1, 2];
const sorted = arr.sort((a, b) => a - b);
console.log(arr);      // [1, 2, 3]  ← оригінал змінено!
console.log(sorted === arr);  // true

// Якщо потрібна копія:
const sorted = [...arr].sort((a, b) => a - b);
// або ES2023+:
const sorted = arr.toSorted((a, b) => a - b);  // не мутує
```

### Stable sort (ES2019+)

З ES2019 `Array.prototype.sort` **гарантовано stable** в усіх сучасних JS-двигунах. Це означає: якщо два елементи мають однаковий ключ сортування -- їхній взаємний порядок збережеться.

```javascript
const people = [
  {name: 'Taras', age: 30},
  {name: 'Ivan', age: 25},
  {name: 'Olha', age: 30},
];
people.sort((a, b) => a.age - b.age);
// [{Ivan,25}, {Taras,30}, {Olha,30}]  ← Taras перед Olha, бо так було спочатку
```

Сортувати за кількома ключами:

```javascript
people.sort((a, b) => {
  if (a.age !== b.age) return a.age - b.age;     // перший ключ
  return a.name.localeCompare(b.name);           // другий ключ
});
```

---

## Рядки: immutable і з одним сюрпризом

### Immutability

```javascript
let s = 'hello';
s[0] = 'H';         // не кидає помилку (у non-strict mode), але нічого не робить
console.log(s);     // 'hello'

// Щоб "змінити" -- створи новий рядок
s = 'H' + s.slice(1);     // 'Hello'
```

### Concatenation у циклі -- O(n²)

```javascript
// ❌ O(n²) -- кожен += копіює весь рядок
let result = '';
for (let i = 0; i < 100_000; i++) result += 'x';
// Для n=10⁵ це вже сповільнить Node помітно.
// Для n=10⁶ майже гарантовано TLE на LeetCode.

// ✅ O(n) -- масив + join
const parts = [];
for (let i = 0; i < 100_000; i++) parts.push('x');
const result = parts.join('');
```

**Чому join швидкий:** двигун знає фінальну довжину (сума довжин) і копіює кожен фрагмент рівно один раз.

### charCodeAt і fromCharCode

```javascript
'a'.charCodeAt(0);            // 97
'A'.charCodeAt(0);            // 65
'0'.charCodeAt(0);            // 48
'z'.charCodeAt(0) - 'a'.charCodeAt(0);  // 25

String.fromCharCode(97);      // 'a'
String.fromCharCode(65);      // 'A'
```

**Класичний трюк:** мапити літеру в індекс масиву [0..25]:

```javascript
function countLetters(s) {
  const counts = new Int32Array(26);
  const A = 'a'.charCodeAt(0);
  for (let i = 0; i < s.length; i++) {
    counts[s.charCodeAt(i) - A]++;
  }
  return counts;
}
```

Це швидше ніж Map для обмеженого алфавіту.

### Unicode: довжина рядка != кількість символів

```javascript
const s = 'Їжак';
console.log(s.length);        // 4  (ok -- всі UTF-16 code units по 1 слову)

const emoji = '👨‍👩‍👧';
console.log(emoji.length);    // 8  (!!! -- ZWJ + surrogate pairs)
console.log([...emoji].length);  // 5 (code points)

const smile = '😀';
console.log(smile.length);    // 2  (surrogate pair)
console.log(smile[0]);        // '\uD83D' -- половина символу
console.log([...smile][0]);   // '😀'
```

**Правило:** для задач з emoji / non-ASCII використовуй `Array.from(str)` або `[...str]`, щоб отримати справжні code points.

Для ASCII-only задач (LeetCode здебільшого) `str.length` і `str[i]` працюють коректно.

### Розбивка на символи

```javascript
const s = 'abc';
s.split('');                  // ['a', 'b', 'c']  -- ⚠ ламається на emoji
[...s];                       // ['a', 'b', 'c']  -- коректно для Unicode
Array.from(s);                // те саме
```

---

## Map vs Object: чому на співбесіді завжди Map

### Чому не Object

```javascript
const obj = {};
obj.toString;                 // [Function] -- успадковано з Object.prototype!
'toString' in obj;            // true
obj.hasOwnProperty('a');      // false, але ім'я вже зайнято
```

1. **Прототипне забруднення.** Ключі `toString`, `constructor`, `__proto__`, `hasOwnProperty` існують "за замовчуванням".
2. **Тільки string/Symbol ключі.** `obj[1] = 'x'; obj['1']` -- те саме.
3. **Немає `.size`** -- `Object.keys(obj).length` -- це O(n).
4. **Не ітерується напряму.** Треба `Object.keys` / `Object.entries` -- додаткове O(n).
5. **Порядок ключів складний.** Integer-keys ідуть перші у числовому порядку, потім string-keys у порядку додавання.

### Map -- те, що треба

```javascript
const m = new Map();
m.set('a', 1);
m.set(42, 'answer');
m.set({obj: 1}, 'by reference');   // будь-який тип ключа!
m.set([1, 2], 'array as key');

m.get('a');                        // 1
m.has(42);                         // true
m.size;                            // 4 -- O(1)
m.delete('a');

for (const [k, v] of m) { }        // ітерується напряму
for (const k of m.keys()) { }
for (const v of m.values()) { }

// Чиста мапа без prototype
// (хоча Map і так не має "забруднених" ключів)
```

**Особливість:** для об'єктних ключів Map використовує reference equality, не глибоке порівняння.

```javascript
const m = new Map();
m.set([1, 2], 'a');
m.get([1, 2]);                     // undefined -- інший масив!

const key = [1, 2];
m.set(key, 'b');
m.get(key);                        // 'b'
```

### Коли все ж Object доречний

- У hot path, коли ключі точно string і критичний час мікрооптимізації. Але V8 оптимізує Map добре, різниця зазвичай невелика.
- Для маленьких "конфігів" у 3-5 ключів. Тут різниця нульова, читабельність Object вища.

**Правило для LeetCode:** завжди Map. Без винятків.

### Object як "чиста" мапа

Якщо все-таки треба Object:

```javascript
const m = Object.create(null);     // без прототипу
m.toString;                        // undefined -- чисто
m.key = 'value';
```

---

## Set: "хеш-таблиця без значень"

```javascript
const s = new Set();
s.add(1);
s.add(2);
s.add(1);                          // ігнорується -- дублікат
s.size;                            // 2
s.has(1);                          // true
s.delete(1);

// Створення з масиву
const unique = new Set([1, 2, 2, 3, 3, 3]);    // Set {1, 2, 3}
const uniqueArr = [...new Set(arr)];           // unique як array

// Перетин
function intersect(a, b) {
  const sb = new Set(b);
  return a.filter(x => sb.has(x));
}
```

### Set і об'єкти -- reference equality

```javascript
const s = new Set();
s.add({x: 1});
s.has({x: 1});                     // false!  різні об'єкти

const o = {x: 1};
s.add(o);
s.has(o);                          // true
```

**Висновок:** якщо треба унікальність за "значенням" об'єкта -- клади в Set серіалізований ключ (наприклад, `JSON.stringify`), або використовуй Map з ключем-рядком.

```javascript
// Унікальність пар (x, y)
const seen = new Set();
for (const [x, y] of pairs) {
  const key = `${x}:${y}`;
  if (seen.has(key)) continue;
  seen.add(key);
  // обробка
}
```

---

## Рекурсія: стек закінчиться швидше, ніж у Python

### Обмеження стеку

V8 (Node) має ліміт стеку приблизно **10 000 -- 15 000 кадрів** для типових рекурсивних функцій. Для порівняння: Python 1000 за замовчуванням (але можна змінити), C++ залежить від системи (зазвичай 1MB = сотні тисяч кадрів).

```javascript
function recurse(n) {
  if (n === 0) return;
  recurse(n - 1);
}

recurse(10_000);      // працює
recurse(100_000);     // RangeError: Maximum call stack size exceeded
```

### Немає TCO (tail call optimization)

Хоча ES2015 специфікація передбачила TCO, **жоден з основних двигунів (V8, SpiderMonkey) не реалізував його**. Тому "хвостова рекурсія" у JS не допомагає:

```javascript
function sum(arr, i = 0, acc = 0) {
  if (i === arr.length) return acc;
  return sum(arr, i + 1, acc + arr[i]);   // "хвостовий" виклик -- все одно зростає стек
}

sum(new Array(100_000).fill(1));          // RangeError
```

### Обхідні шляхи

**Варіант 1: ітеративний підхід.**

```javascript
function sum(arr) {
  let acc = 0;
  for (const x of arr) acc += x;
  return acc;
}
```

**Варіант 2: explicit stack.**

Це особливо актуально для DFS:

```javascript
// Замість рекурсивного DFS:
function dfsRec(node) {
  if (!node) return;
  visit(node);
  dfsRec(node.left);
  dfsRec(node.right);
}

// Ітеративний DFS з явним стеком:
function dfs(root) {
  const stack = [root];
  while (stack.length) {
    const node = stack.pop();
    if (!node) continue;
    visit(node);
    stack.push(node.right);
    stack.push(node.left);       // push right first, then left
                                 // → left buде pop-нутий першим
  }
}
```

**Варіант 3: збільшення стеку.**

```bash
node --stack-size=16000 script.js    # не рекомендую на співбесіді
```

Технічно працює, але інтерв'юер оцінить це як "не вмію позбутися рекурсії". Плюс на реальному production-сервері такого не зробиш.

### Візуалізація глибини стеку

```
Рекурсія:
    ┌──────────────┐  ← кожен виклик = кадр стеку (~200 байт)
    │ sum(arr,99)  │
    ├──────────────┤
    │ sum(arr,98)  │
    ├──────────────┤  при 10000 викликів → ~2 MB стеку → близько ліміту
    │    ...       │
    ├──────────────┤
    │ sum(arr,0)   │
    └──────────────┘

Ітерація:
    ┌──────────────┐
    │ sum(arr)     │  один кадр незалежно від розміру
    └──────────────┘
```

---

## Math: корисне і пастки

### Найчастіше вживане

```javascript
Math.floor(7.8);              //  7     -- вниз
Math.ceil(7.2);               //  8     -- вгору
Math.round(7.5);              //  8     -- у найближче
Math.trunc(-7.8);             // -7     -- обрізання до 0
Math.abs(-5);                 //  5
Math.sign(-3);                // -1 | 0 | 1
Math.pow(2, 10);              // 1024   (або 2 ** 10)
Math.sqrt(16);                // 4
Math.log2(8);                 // 3
Math.log10(1000);             // 3
Math.min(3, 1, 2);            // 1
Math.max(3, 1, 2);            // 3

Number.MAX_SAFE_INTEGER;      // 2^53 - 1
Number.MIN_SAFE_INTEGER;      // -(2^53 - 1)
Number.EPSILON;               // ≈ 2.22e-16
Infinity; -Infinity; NaN;
```

### Пастка: `Math.min(...arr)` на великих масивах

```javascript
const big = new Array(1_000_000).fill(0).map((_, i) => i);

// ❌ RangeError: занадто багато аргументів (спред у функцію має ліміт)
const min = Math.min(...big);

// ✅ звичайний reduce / цикл
let min = Infinity;
for (const x of big) if (x < min) min = x;

// або reduce
const min = big.reduce((m, x) => x < m ? x : m, Infinity);
```

Ліміт argument count залежить від двигуна, зазвичай 10k-100k. Безпечніше -- цикл.

### Min/max на порожньому масиві

```javascript
Math.min();                   //  Infinity
Math.max();                   // -Infinity
Math.min(...[]);              //  Infinity

// Тому для шукання min у можливо порожньому масиві:
const min = arr.length ? Math.min(...arr) : null;
```

---

## Ітератори: for...of vs for...in

Це -- найчастіше плутання для початківців.

### for...in -- ітерує КЛЮЧІ (включно з прототипними!)

```javascript
const arr = [10, 20, 30];
arr.extra = 'hi';

for (const key in arr) {
  console.log(key);
}
// '0', '1', '2', 'extra'   ← ключі як рядки, ще й додаткові властивості!
```

**Не використовуй `for...in` для масивів. Ніколи.**

Для об'єктів треба перевіряти `hasOwnProperty`:

```javascript
for (const key in obj) {
  if (!Object.hasOwn(obj, key)) continue;   // ES2022
  // або obj.hasOwnProperty(key), але це теж ризик
}
```

Краще одразу:

```javascript
for (const key of Object.keys(obj)) { }
for (const [k, v] of Object.entries(obj)) { }
```

### for...of -- ітерує ЗНАЧЕННЯ (через ітератор)

```javascript
const arr = [10, 20, 30];
for (const x of arr) {
  console.log(x);            // 10, 20, 30
}

const s = new Set([1, 2, 3]);
for (const x of s) { }       // 1, 2, 3

const m = new Map([['a', 1], ['b', 2]]);
for (const [k, v] of m) { }

for (const ch of 'hello') {
  console.log(ch);           // 'h', 'e', 'l', 'l', 'o'
}
```

**Правило:** в алгоритмах -- тільки `for...of` або класичний `for (let i = 0; i < n; i++)`. Ніколи `for...in`.

### Generators -- для lazy sequences

```javascript
function* range(start, end) {
  for (let i = start; i < end; i++) yield i;
}

for (const x of range(0, 5)) console.log(x);   // 0 1 2 3 4

// Lazy -- не створює масив
let sum = 0;
for (const x of range(0, 1_000_000)) sum += x;  // O(1) space
```

На LeetCode generators рідко потрібні, але для memory-sensitive задач -- корисні.

---

## Асимптотика стандартних методів: шпаргалка

### Array

| Операція                            | Big O             | Коментар                              |
|--------------------------------------|-------------------|----------------------------------------|
| `arr[i]` (read/write)                | O(1)              |                                        |
| `arr.length`                         | O(1)              |                                        |
| `arr.push(x)`                        | O(1) amort.       | Іноді O(n) при realloc                 |
| `arr.pop()`                          | O(1)              |                                        |
| `arr.shift()`                        | **O(n)**          | Зсуває всі елементи вліво              |
| `arr.unshift(x)`                     | **O(n)**          | Зсуває всі елементи вправо             |
| `arr.splice(i, delCnt, ...items)`    | **O(n)**          | Залежить від позиції                   |
| `arr.slice(a, b)`                    | O(b - a)          | Копіює піддіапазон                     |
| `arr.concat(other)`                  | O(n + m)          |                                        |
| `[...arr]`, `Array.from(arr)`        | O(n)              | Shallow copy                           |
| `arr.indexOf(x)`, `arr.lastIndexOf`  | O(n)              | Лінійний                               |
| `arr.includes(x)`                    | O(n)              | Лінійний                               |
| `arr.find(fn)`, `arr.findIndex(fn)`  | O(n)              |                                        |
| `arr.some(fn)`, `arr.every(fn)`      | O(n)              | Може завершитися раніше                |
| `arr.filter(fn)`                     | O(n)              |                                        |
| `arr.map(fn)`                        | O(n)              |                                        |
| `arr.reduce(fn, init)`               | O(n)              |                                        |
| `arr.forEach(fn)`                    | O(n)              |                                        |
| `arr.sort(cmp)`                      | O(n log n)        | Зазвичай TimSort у V8                  |
| `arr.reverse()`                      | O(n)              |                                        |
| `arr.join(sep)`                      | O(n)              |                                        |
| `arr.flat(depth)`                    | O(n)              |                                        |
| `arr.flatMap(fn)`                    | O(n)              |                                        |
| `new Array(n).fill(x)`               | O(n)              |                                        |

### Map, Set

| Операція                 | Big O (avg)       | Worst case                           |
|--------------------------|-------------------|---------------------------------------|
| `map.get(key)`           | O(1)              | O(n) при поганій хеш-функції         |
| `map.set(key, val)`      | O(1) amort.       | O(n) при rehash                      |
| `map.has(key)`           | O(1)              |                                       |
| `map.delete(key)`        | O(1)              |                                       |
| `map.size`               | O(1)              |                                       |
| `for (... of map)`       | O(n)              |                                       |
| `set.add(x)`             | O(1) amort.       |                                       |
| `set.has(x)`             | O(1)              |                                       |
| `set.delete(x)`          | O(1)              |                                       |
| `new Set(arr)`           | O(n)              |                                       |

### String

| Операція                 | Big O             | Коментар                              |
|--------------------------|-------------------|----------------------------------------|
| `str[i]`                 | O(1)              |                                        |
| `str.length`             | O(1)              |                                        |
| `str.charCodeAt(i)`      | O(1)              |                                        |
| `str.slice(a, b)`        | O(b - a)          |                                        |
| `str.substring(a, b)`    | O(b - a)          |                                        |
| `str.indexOf(sub)`       | O(n × m)          | Гірший випадок                         |
| `str.includes(sub)`      | O(n × m)          |                                        |
| `str.split(sep)`         | O(n)              |                                        |
| `str.replace(sub, new)`  | O(n)              |                                        |
| `str1 + str2`            | O(n + m)          |                                        |
| `str += x` (у циклі)     | O(total²)         | Уникай!                                |

### Object

| Операція                 | Big O (avg)       | Коментар                              |
|--------------------------|-------------------|----------------------------------------|
| `obj[key]`, `obj.key`    | O(1) avg          | Повільніше за Map                     |
| `obj[key] = val`         | O(1) avg          |                                        |
| `delete obj[key]`        | O(1) avg          | Деоптимізує об'єкт -- уникай          |
| `key in obj`             | O(1) avg          | Включно з прототипом                  |
| `Object.keys(obj)`       | O(n)              |                                        |
| `Object.values(obj)`     | O(n)              |                                        |
| `Object.entries(obj)`    | O(n)              |                                        |

---

## Особливості Node.js runtime

### V8: JIT, hidden classes, inline caching

Коротко про те, чому V8 швидкий:

1. **JIT** -- компілює JS у машинний код на льоту, переоптимізує гарячі функції.
2. **Hidden classes** -- V8 групує об'єкти з однаковим "shape" (ті самі ключі в тому самому порядку) у приховані класи, що дає швидкий доступ.
3. **Inline caching** -- при виклику методу V8 кешує, де цей метод знаходиться, і не робить dynamic lookup щоразу.

**Висновок для алгоритмів:**

```javascript
// ❌ Об'єкти з різним shape -- V8 не може оптимізувати
const users = [];
users.push({name: 'A'});
users.push({name: 'B', age: 30});       // інший shape
users.push({age: 30, name: 'C'});       // ще інший shape (порядок ключів!)

// ✅ Однаковий shape -- V8 оптимізує
users.push({name: 'A', age: null});
users.push({name: 'B', age: 30});
users.push({name: 'C', age: 40});
```

На LeetCode це рідко критично, але знати варто.

### Garbage collection

JS має GC, який **сам** збирає невикористану пам'ять. Але:

```javascript
const cache = new Map();

function solve(input) {
  // ...
  cache.set(input, result);       // cache росте, нічого не збирається
}
```

Якщо тримаєш посилання на великий об'єкт/масив -- GC його не збере. У LeetCode це рідко проблема (тест закінчується -- пам'ять звільняється), але у production завжди питай себе: коли цей об'єкт може бути звільнений?

Є `WeakMap` / `WeakSet` -- дозволяють GC збирати ключі, якщо на них немає інших посилань. Рідко потрібно на співбесіді.

### Arrow vs regular functions -- з точки зору алгоритму байдуже

```javascript
arr.map(x => x * 2);
arr.map(function(x) { return x * 2; });
```

Швидкість однакова. Єдина різниця:
- Arrow lexically binds `this` -- важливо всередині методів класу.
- Regular function має власний `this`, `arguments`.

На алгоритмічних задачах клас/this майже не вживаються, тому -- пиши як зручніше.

---

## Типові пастки на співбесіді з JS

### Пастка 1: parseInt без radix

```javascript
parseInt('10');         // 10 -- здається, що все ok
parseInt('08');         // 8 (у старих двигунах міг бути 0 -- octal!)
parseInt('0x10');       // 16 (шістнадцятковий!)

// ✅ Завжди явно:
parseInt('10', 10);     // 10
parseInt('0x10', 10);   // 0 (стоп на 'x')

// Для чистих чисел простіше:
Number('10');           // 10
+'10';                  // 10
```

### Пастка 2: == vs ===

```javascript
0 == '';                // true  ❗
0 == '0';               // true  ❗
null == undefined;      // true
null == 0;              // false
'' == 0;                // true
'0' == 0;               // true
NaN == NaN;             // false ❗

// ✅ Завжди ===, крім одного випадку:
if (x == null) { }      // перевірка одночасно на null і undefined
// (або явно: if (x === null || x === undefined))
```

### Пастка 3: null vs undefined

```javascript
let a;                  // undefined -- змінна не ініціалізована
const b = null;         // null -- явне "нічого"

function foo(x) {
  if (x == null) { }    // ✅ покриває обидва
}

obj.missingKey;         // undefined
```

**У коді LeetCode:**
- `null` -- "явно немає" (наприклад, `node.left === null`).
- `undefined` -- "ще не присвоєно" або "ключа немає".

### Пастка 4: Float comparison

```javascript
// ❌
if (a === b) { }                    // для float небезпечно

// ✅
if (Math.abs(a - b) < 1e-9) { }
```

### Пастка 5: Забутий return у callback

```javascript
// ❌ ПАСТКА: reduce без return → acc стає undefined на другій ітерації
const sum = arr.reduce((acc, x) => { acc + x }, 0);
// Результат: undefined

// ✅
const sum = arr.reduce((acc, x) => acc + x, 0);
// або з фігурними:
const sum = arr.reduce((acc, x) => { return acc + x; }, 0);
```

Та сама пастка з sort:

```javascript
arr.sort((a, b) => { a - b });    // ❌ без return → comparator повертає undefined → sort глючить
arr.sort((a, b) => a - b);        // ✅
```

### Пастка 6: Деструктуризація з default values

```javascript
function foo({a = 10, b = 20} = {}) {
  return a + b;
}

foo();                  // 30 (всі дефолти)
foo({a: 5});            // 25

// Default спрацьовує тільки для undefined, не для null!
foo({a: null});         // null + 20 = '20null'  ❗ (або NaN залежно від контексту)
foo({a: 0});            // 0 + 20 = 20 (0 -- не undefined, дефолт не спрацьовує)
```

### Пастка 7: `this` у методах класу

```javascript
class Solver {
  constructor() {
    this.count = 0;
  }

  add() {
    this.count++;
  }
}

const s = new Solver();
const fn = s.add;
fn();                   // ❌ TypeError: Cannot read properties of undefined

// ✅ Варіант 1: bind
const fn = s.add.bind(s);

// ✅ Варіант 2: arrow-функція як властивість
class Solver {
  count = 0;
  add = () => { this.count++; };    // lexical this
}
```

На алгоритмічній співбесіді класи рідко потрібні. Коли треба ООП (наприклад, design Twitter) -- завжди arrow-методи або bind.

---

## Шаблони та snippets для співбесіди

### 1. Створення 2D масиву

```javascript
// m рядків × n стовпців, ініціалізовано нулями
const dp = Array.from({length: m}, () => new Array(n).fill(0));

// з default значенням іншого типу
const visited = Array.from({length: m}, () => new Array(n).fill(false));
```

### 2. Частотний counter через Map

```javascript
function frequencyMap(arr) {
  const map = new Map();
  for (const x of arr) {
    map.set(x, (map.get(x) ?? 0) + 1);
  }
  return map;
}

// Топ-K найчастіших
function topK(arr, k) {
  const freq = frequencyMap(arr);
  return [...freq.entries()]
    .sort((a, b) => b[1] - a[1])
    .slice(0, k)
    .map(([val]) => val);
}
```

### 3. Sort за кількома ключами

```javascript
arr.sort((a, b) => {
  // 1-й ключ: age asc
  if (a.age !== b.age) return a.age - b.age;
  // 2-й ключ: name lexicographically
  if (a.name !== b.name) return a.name < b.name ? -1 : 1;
  // 3-й ключ: id desc
  return b.id - a.id;
});
```

### 4. Binary search template

```javascript
// Класика: знайти індекс target (або -1)
function binarySearch(arr, target) {
  let left = 0, right = arr.length - 1;
  while (left <= right) {
    const mid = left + ((right - left) >> 1);    // безпечно від overflow
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}

// Lower bound: перший індекс, де arr[i] >= target
function lowerBound(arr, target) {
  let left = 0, right = arr.length;          // right ексклюзивний!
  while (left < right) {
    const mid = left + ((right - left) >> 1);
    if (arr[mid] < target) left = mid + 1;
    else right = mid;
  }
  return left;
}

// Upper bound: перший індекс, де arr[i] > target
function upperBound(arr, target) {
  let left = 0, right = arr.length;
  while (left < right) {
    const mid = left + ((right - left) >> 1);
    if (arr[mid] <= target) left = mid + 1;
    else right = mid;
  }
  return left;
}
```

### 5. DFS boilerplate (рекурсивний і ітеративний)

```javascript
// Рекурсивний (обережно з глибиною)
function dfs(node, visited = new Set()) {
  if (!node || visited.has(node)) return;
  visited.add(node);
  visit(node);
  for (const next of node.neighbors) dfs(next, visited);
}

// Ітеративний зі стеком
function dfs(start) {
  const visited = new Set();
  const stack = [start];
  while (stack.length) {
    const node = stack.pop();
    if (visited.has(node)) continue;
    visited.add(node);
    visit(node);
    for (const next of node.neighbors) {
      if (!visited.has(next)) stack.push(next);
    }
  }
}
```

### 6. BFS boilerplate

```javascript
function bfs(start) {
  const visited = new Set([start]);
  const queue = [start];
  let head = 0;                         // "вказівник", щоб не викликати shift (O(n))
  while (head < queue.length) {
    const node = queue[head++];
    visit(node);
    for (const next of node.neighbors) {
      if (!visited.has(next)) {
        visited.add(next);
        queue.push(next);
      }
    }
  }
}
```

**Чому `head++` замість `queue.shift()`:** `shift` -- O(n), тому весь BFS стає O(n²). З `head++` -- справжній O(n).

### 7. BFS за рівнями

```javascript
function bfsByLevel(root) {
  let queue = [root];
  let level = 0;
  while (queue.length) {
    const next = [];
    for (const node of queue) {
      visit(node, level);
      if (node.left) next.push(node.left);
      if (node.right) next.push(node.right);
    }
    queue = next;
    level++;
  }
}
```

### 8. Union-Find (DSU) skeleton

```javascript
class DSU {
  constructor(n) {
    this.parent = Array.from({length: n}, (_, i) => i);
    this.rank = new Array(n).fill(0);
  }
  find(x) {
    if (this.parent[x] !== x) {
      this.parent[x] = this.find(this.parent[x]);    // path compression
    }
    return this.parent[x];
  }
  union(a, b) {
    const ra = this.find(a), rb = this.find(b);
    if (ra === rb) return false;
    if (this.rank[ra] < this.rank[rb]) this.parent[ra] = rb;
    else if (this.rank[ra] > this.rank[rb]) this.parent[rb] = ra;
    else { this.parent[rb] = ra; this.rank[ra]++; }
    return true;
  }
}
```

### 9. Queue на двох стеках (без shift)

```javascript
class Queue {
  constructor() { this.inStack = []; this.outStack = []; }
  push(x) { this.inStack.push(x); }
  pop() {
    if (!this.outStack.length) {
      while (this.inStack.length) this.outStack.push(this.inStack.pop());
    }
    return this.outStack.pop();
  }
  get size() { return this.inStack.length + this.outStack.length; }
}
// Amortized O(1) per op.
```

### 10. Частотна різниця (для anagram-задач)

```javascript
function isAnagram(a, b) {
  if (a.length !== b.length) return false;
  const counts = new Int32Array(26);
  const A = 'a'.charCodeAt(0);
  for (let i = 0; i < a.length; i++) {
    counts[a.charCodeAt(i) - A]++;
    counts[b.charCodeAt(i) - A]--;
  }
  return counts.every(c => c === 0);
}
```

---

## Ключові думки

- **Числа -- float.** Пам'ятай про 2⁵³ межу, точність, `/` завжди float, ділення на 0 → Infinity.
- **Масиви -- універсальні, але `shift`/`unshift` O(n).** Для BFS -- `head++`, не `shift`.
- **`new Array(n).fill([])` -- пастка.** Завжди `Array.from({length: n}, () => [])`.
- **`[...arr]` -- O(n), не O(1).** У циклі це O(n²).
- **Рядки immutable.** Не роби `result += x` у циклі -- масив+join.
- **Map > Object для алгоритмів.** Завжди.
- **Set -- reference equality для об'єктів.** Для "значеннєвих" ключів -- серіалізація.
- **Рекурсія -- до ~10-15k глибини.** Немає TCO.
- **Sort -- завжди з comparator.** `(a, b) => a - b`.
- **`for...of` для значень, `for...in` -- ніколи для масивів.**
- **Bitwise -- 32-bit signed.** Корисні для парності, ділення на 2, XOR-трюків.
- **BigInt -- коли числа > 2⁵³.** Rolling hash, великі множення.
- **Typed arrays** (`Int32Array`, `Uint8Array`) -- коли потрібна реальна швидкість на числових масивах.

JavaScript не ідеальний для алгоритмів, але якщо ти знаєш ці нюанси -- написати чистий O(n) розв'язок на JS не складніше, ніж на Python. Головне -- не спіткнутися на дрібницях.
