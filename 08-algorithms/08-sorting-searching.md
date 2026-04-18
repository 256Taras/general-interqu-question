# Сортування та пошук: алгоритми, які мусиш знати

## Чому треба знати сортування, якщо є `Array.prototype.sort`?

Справедливе запитання. У 99% production коду ти просто викликаєш `arr.sort((a, b) => a - b)` і не думаєш, що там всередині. Проте на співбесіді знання сортувань перевіряють з кількох причин:

1. **Властивості алгоритмів впливають на вибір.** Stable чи ні? In-place чи ні? Comparison-based чи ні? Для деяких задач ці властивості критичні. Наприклад, sort by second key потребує **stable** сортування.

2. **Merge sort і quicksort -- фундамент для іншого.** Merge sort -> merge intervals, external sorting, count inversions. Quicksort -> QuickSelect (k-th largest за O(n)).

3. **Розуміння trade-offs.** Чому V8 використовує TimSort? Чому Quicksort -- дефолт у C? Це розмови про inputs: випадкові дані, майже відсортовані, з багатьма дублікатами.

4. **Бінарний пошук всюди.** Тема "search" -- одна з найчастіших на співбесідах. Треба вміти його писати без помилок у мінус-варіантах: lower_bound, upper_bound, binary search on answer.

5. **Big O свідомість.** Знати, що `arr.sort()` -- це O(n log n), а не "магія" -- основа для оцінки твого коду.

Мета цього файлу: дати тобі **тверде розуміння** властивостей алгоритмів, код основних з них, і **майстерне** володіння бінарним пошуком.

---

## Порівняння алгоритмів сортування

Запам'ятай цю таблицю. Вона покриває 90% питань на співбесіді.

| Алгоритм        | Best       | Average    | Worst      | Space    | Stable | In-place |
|-----------------|------------|------------|------------|----------|--------|----------|
| Bubble sort     | O(n)       | O(n²)      | O(n²)      | O(1)     | Так    | Так      |
| Selection sort  | O(n²)      | O(n²)      | O(n²)      | O(1)     | Ні     | Так      |
| Insertion sort  | O(n)       | O(n²)      | O(n²)      | O(1)     | Так    | Так      |
| Merge sort      | O(n log n) | O(n log n) | O(n log n) | O(n)     | Так    | Ні       |
| Quicksort       | O(n log n) | O(n log n) | O(n²)      | O(log n) | Ні     | Так      |
| Heap sort       | O(n log n) | O(n log n) | O(n log n) | O(1)     | Ні     | Так      |
| TimSort (V8)    | O(n)       | O(n log n) | O(n log n) | O(n)     | Так    | Ні       |
| Counting sort   | O(n + k)   | O(n + k)   | O(n + k)   | O(k)     | Так    | Ні       |
| Radix sort      | O(n × d)   | O(n × d)   | O(n × d)   | O(n + k) | Так    | Ні       |

**Пояснення колонок:**
- **Stable** -- чи зберігає відносний порядок рівних елементів. Критично для sort-by-key.
- **In-place** -- чи сортує без додаткового масиву (тільки O(1) або O(log n) пам'яті).
- **Comparison-based** -- чи використовує тільки порівняння (a < b). Comparison-based не може бути швидше за O(n log n). Counting/radix -- non-comparison, тому обходять цей ліміт.

**Що важливо для співбесід:**
- `O(n log n)` -- золотий стандарт для sort.
- **Stable + O(n log n)** -- merge sort або TimSort.
- **In-place + O(n log n)** (average) -- quicksort або heap sort.
- Знати, що `arr.sort()` в JS -- **stable** з ES2019, TimSort-подібний.

---

## Bubble sort

**Ідея:** пробігаєш масивом, порівнюєш сусідні пари, міняєш якщо не в порядку. Повторюєш, доки не буде жодного swap.

Назва -- бо великі елементи "спливають" як бульбашки.

```javascript
function bubbleSort(arr) {
  const n = arr.length;
  for (let i = 0; i < n - 1; i++) {
    let swapped = false;
    // Після кожного зовнішнього проходу найбільший елемент
    // "спливає" в кінець → n - 1 - i
    for (let j = 0; j < n - 1 - i; j++) {
      if (arr[j] > arr[j + 1]) {
        [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
        swapped = true;
      }
    }
    if (!swapped) break;  // якщо не було swap — вже відсортовано
  }
  return arr;
}
```

### Візуалізація

```
Start: [5, 3, 8, 4, 2]

Pass 1:
  5 > 3 → swap → [3, 5, 8, 4, 2]
  5 < 8 → нічого
  8 > 4 → swap → [3, 5, 4, 8, 2]
  8 > 2 → swap → [3, 5, 4, 2, 8]    ← 8 на місці

Pass 2:
  3 < 5 → нічого
  5 > 4 → swap → [3, 4, 5, 2, 8]
  5 > 2 → swap → [3, 4, 2, 5, 8]    ← 5 на місці

Pass 3:
  3 < 4 → нічого
  4 > 2 → swap → [3, 2, 4, 5, 8]    ← 4 на місці

Pass 4:
  3 > 2 → swap → [2, 3, 4, 5, 8]    ← 3 на місці

Готово.
```

**Коли використовувати bubble sort:** майже ніколи. O(n²) на середніх розмірах -- занадто повільно. Єдине виправдання -- навчальне або дуже маленький масив (< 10 елементів), де прекмана реалізація важливіша за performance.

---

## Selection sort

**Ідея:** на кожному кроці знайди мінімум з ще не відсортованих, поклади на початок.

```javascript
function selectionSort(arr) {
  const n = arr.length;
  for (let i = 0; i < n - 1; i++) {
    let minIdx = i;
    for (let j = i + 1; j < n; j++) {
      if (arr[j] < arr[minIdx]) minIdx = j;
    }
    if (minIdx !== i) {
      [arr[i], arr[minIdx]] = [arr[minIdx], arr[i]];
    }
  }
  return arr;
}
```

**Властивості:**
- Завжди O(n²), навіть якщо масив вже відсортований.
- **Нестабільне** (може поміняти порядок рівних елементів при swap через весь масив).
- In-place.
- Мінімальна кількість swap-ів (≤ n-1), що буває корисно, якщо swap коштовний.

---

## Insertion sort

**Ідея:** для кожного елемента знайди йому правильне місце серед вже відсортованих зліва. Як сортування карт у руці.

```javascript
function insertionSort(arr) {
  for (let i = 1; i < arr.length; i++) {
    const key = arr[i];
    let j = i - 1;
    // Зсуваємо всі більші елементи вправо
    while (j >= 0 && arr[j] > key) {
      arr[j + 1] = arr[j];
      j--;
    }
    arr[j + 1] = key;
  }
  return arr;
}
```

### Візуалізація

```
Start: [5, 3, 8, 4, 2]

i=1, key=3:  [_, 5, 8, 4, 2] → [3, 5, 8, 4, 2]
i=2, key=8:  [3, 5, _, 4, 2] → [3, 5, 8, 4, 2]  (8 вже на місці)
i=3, key=4:  [3, 5, 8, _, 2] → [3, 4, 5, 8, 2]
i=4, key=2:  [3, 4, 5, 8, _] → [2, 3, 4, 5, 8]
```

**Коли insertion sort дійсно хороший:**
- **Майже відсортовані масиви** -- працює за O(n) (best case).
- **Дуже маленькі масиви** (≤ 16 елементів) -- overhead merge/quick sort не окупається. Саме тому TimSort (V8) використовує insertion sort для маленьких chunk-ів.
- Online algorithm -- можна сортувати потік даних на льоту.

**Властивості:** stable, in-place, adaptive (швидкий на майже відсортованих).

---

## Merge sort

**Merge sort** -- класичний приклад **divide-and-conquer**. Основа багатьох алгоритмів.

**Ідея:**
1. Розділити масив навпіл.
2. Рекурсивно відсортувати кожну половину.
3. Злити відсортовані половини в один відсортований масив.

### ASCII-діаграма поділу

```
                  [5, 3, 8, 4, 2, 7, 1, 6]
                        /          \
                 [5, 3, 8, 4]   [2, 7, 1, 6]
                   /    \         /    \
                [5,3]  [8,4]   [2,7]  [1,6]
                 /\    /\      /\      /\
                [5][3][8][4]  [2][7]  [1][6]
                 \/    \/      \/      \/
                [3,5] [4,8]   [2,7]   [1,6]      ← merge
                   \   /         \   /
                [3,4,5,8]      [1,2,6,7]         ← merge
                       \        /
                 [1, 2, 3, 4, 5, 6, 7, 8]        ← merge
```

### Код merge sort

```javascript
function mergeSort(arr) {
  if (arr.length <= 1) return arr;

  const mid = Math.floor(arr.length / 2);
  const left = mergeSort(arr.slice(0, mid));
  const right = mergeSort(arr.slice(mid));

  return merge(left, right);
}

function merge(left, right) {
  const result = [];
  let i = 0, j = 0;

  // Порівнюємо перші елементи, беремо менший
  while (i < left.length && j < right.length) {
    if (left[i] <= right[j]) {
      result.push(left[i++]);   // <=, а не <, щоб зберегти stability
    } else {
      result.push(right[j++]);
    }
  }

  // Додаємо залишки
  while (i < left.length) result.push(left[i++]);
  while (j < right.length) result.push(right[j++]);

  return result;
}
```

### Візуалізація merge

Злиття `[3, 5]` і `[4, 8]`:
```
  left:  [3, 5]    right: [4, 8]    result: []
  i=0               j=0

  3 <= 4 → result.push(3), i=1
  result: [3]

  5 > 4 → result.push(4), j=1
  result: [3, 4]

  5 <= 8 → result.push(5), i=2 (кінець left)
  result: [3, 4, 5]

  залишок right: 8 → result.push(8)
  result: [3, 4, 5, 8]
```

### Аналіз merge sort

**Time complexity:** T(n) = 2 × T(n/2) + O(n). За master theorem -- **O(n log n)**.
- Рівнів рекурсії: log₂(n).
- На кожному рівні: сумарна робота O(n) (merge).

**Space complexity:** O(n) -- допоміжні масиви + O(log n) стек рекурсії.

**Stable:** так, бо в merge ми вибираємо зліва при рівності (`<=`).

**In-place:** ні -- потрібні додаткові масиви. Існує in-place варіант, але він складний і повільніший на практиці.

### Коли використовувати merge sort

1. **Великі набори, потрібна стабільність.** Наприклад, сортувати список об'єктів за двома полями.
2. **External sorting** -- дані не влазять у пам'ять. Merge працює природно з потоками на диску.
3. **Linked lists.** Merge sort працює без random access, на відміну від quicksort.
4. **Count inversions** (Merge Sort можна модифікувати, щоб рахувати, скільки пар `i < j`, але `arr[i] > arr[j]`).

---

## Quicksort

**Quicksort** -- in-place divide-and-conquer. Дефолтний сорт у багатьох standard libraries (C's qsort, Java для примітивів).

**Ідея:**
1. Обери **pivot** (опорний елемент).
2. Розподіли масив: менші за pivot зліва, більші справа (**partitioning**).
3. Рекурсивно відсортуй обидві частини.

### Візуалізація

```
[5, 3, 8, 4, 2, 7, 1, 6]     pivot = 6 (останній)

Після partition:
[5, 3, 4, 2, 1] [6] [8, 7]
   < 6            > 6

Рекурсія на [5, 3, 4, 2, 1] з pivot=1:
[] [1] [5, 3, 4, 2]                ← pivot 1, менших немає

Рекурсія на [5, 3, 4, 2] з pivot=2:
[] [2] [5, 3, 4]                   ← ...

І так далі, поки не розіб'ємо на одиничні елементи.
```

### Код quicksort (Lomuto partition)

```javascript
function quickSort(arr, left = 0, right = arr.length - 1) {
  if (left >= right) return arr;

  const pivotIdx = partition(arr, left, right);
  quickSort(arr, left, pivotIdx - 1);
  quickSort(arr, pivotIdx + 1, right);

  return arr;
}

function partition(arr, left, right) {
  const pivot = arr[right];   // беремо останній як pivot
  let i = left - 1;

  for (let j = left; j < right; j++) {
    if (arr[j] <= pivot) {
      i++;
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
  }
  // Ставимо pivot на правильне місце
  [arr[i + 1], arr[right]] = [arr[right], arr[i + 1]];
  return i + 1;
}
```

### Аналіз quicksort

**Average:** O(n log n) -- при збалансованих partition.

**Worst case:** O(n²) -- коли pivot завжди найменший або найбільший. Приклади: вже відсортований масив + вибір останнього як pivot.

**Space:** O(log n) середньо (рекурсія), O(n) worst case.

**Stable:** ні.

**In-place:** так.

### Стратегії вибору pivot

1. **Перший/останній елемент.** Простий, але катастрофічний на відсортованих входах.
2. **Середній елемент.** Трохи краще.
3. **Випадковий (randomized quicksort).** Статистично уникає worst case.
4. **Median of three** -- медіана з first, middle, last. Класична стратегія.

```javascript
// Randomized pivot
function partition(arr, left, right) {
  const randomIdx = left + Math.floor(Math.random() * (right - left + 1));
  [arr[randomIdx], arr[right]] = [arr[right], arr[randomIdx]];
  // ... далі стандартна Lomuto partition
}
```

### Quicksort vs Merge sort

| Критерій           | Quicksort        | Merge sort       |
|--------------------|------------------|------------------|
| Average time       | O(n log n)       | O(n log n)       |
| Worst time         | O(n²)            | O(n log n)       |
| Space              | O(log n)         | O(n)             |
| Stable             | Ні               | Так              |
| In-place           | Так              | Ні               |
| Random access      | Потрібен         | Не потрібен      |
| Cache-friendly     | Більш            | Менш             |

**Висновок:** у пам'яті quicksort кращий. У гарантіях -- merge sort. Тому багато real-world sort-ів -- гібриди (див. TimSort).

### QuickSelect -- k-th largest/smallest за O(n) average

Варіація quicksort: шукаємо k-й за порядком елемент, **не сортуючи весь масив**.

```javascript
function quickSelect(arr, k) {
  // Знаходимо k-й найменший (0-indexed)
  let left = 0, right = arr.length - 1;

  while (left <= right) {
    const pivotIdx = partition(arr, left, right);
    if (pivotIdx === k) return arr[pivotIdx];
    if (pivotIdx < k) left = pivotIdx + 1;
    else right = pivotIdx - 1;
  }
}
```

Замість рекурсивно сортувати обидві сторони -- йдемо тільки в ту, де знаходиться k. Average **O(n)**.

Корисно для "kth largest element", "top K elements" -- часті задачі співбесід.

---

## Heap sort (коротко)

**Ідея:** побудувати **max-heap** з масиву, потім витягувати максимум і класти в кінець.

```javascript
function heapSort(arr) {
  const n = arr.length;

  // Build max-heap
  for (let i = Math.floor(n / 2) - 1; i >= 0; i--) heapify(arr, n, i);

  // Витягуємо по одному
  for (let i = n - 1; i > 0; i--) {
    [arr[0], arr[i]] = [arr[i], arr[0]];  // max → в кінець
    heapify(arr, i, 0);                    // відновлюємо heap
  }
  return arr;
}

function heapify(arr, n, i) {
  let largest = i;
  const l = 2 * i + 1, r = 2 * i + 2;
  if (l < n && arr[l] > arr[largest]) largest = l;
  if (r < n && arr[r] > arr[largest]) largest = r;
  if (largest !== i) {
    [arr[i], arr[largest]] = [arr[largest], arr[i]];
    heapify(arr, n, largest);
  }
}
```

**Властивості:** O(n log n) завжди, O(1) space, **не stable**, in-place.

**Коли використовувати:** коли потрібна гарантована O(n log n) + O(1) space. Рідко у production, частіше в embedded. Також heap-структура -- основа **priority queue**, яку використовують Dijkstra, A*, event schedulers.

---

## Counting sort і radix sort (non-comparison)

Comparison-based сортування не може бути швидше за O(n log n). Але якщо ми **не порівнюємо елементи**, а використовуємо їх значення як індекси -- можна швидше.

### Counting sort

**Ідея:** порахуй, скільки разів зустрічається кожне значення. Пройди по підрахунках у порядку, записуй.

```javascript
function countingSort(arr, max) {
  const count = new Array(max + 1).fill(0);
  for (const x of arr) count[x]++;

  const result = [];
  for (let i = 0; i <= max; i++) {
    while (count[i]-- > 0) result.push(i);
  }
  return result;
}
```

**Complexity:** O(n + k), де k -- діапазон значень.

**Коли працює:**
- Цілі числа у невеликому діапазоні (0..10⁶, наприклад).
- Категорійні дані (grades, enum values).

**Коли НЕ працює:**
- Великий діапазон (числа до 10¹⁸) -- O(k) пам'яті вбиває.
- Дійсні числа -- немає природних індексів.

### Radix sort

**Ідея:** сортуй числа по цифрах -- від найменш значущої до найбільш значущої (LSD) або навпаки. На кожному розряді використовуй stable counting sort.

**Complexity:** O(n × d), де d -- кількість розрядів. Для 32-bit int-ів d = 10 (якщо базою 10) -- фактично лінійно.

**Коли використовувати:** сортування 32-bit/64-bit integers, коротких рядків.

**Коли не використовувати:** комплексні comparison (по кастомному ключу) -- radix не знає як порівнювати.

---

## V8 sort: TimSort

З 2018 року V8 (Node.js, Chrome) використовує **TimSort** для `Array.prototype.sort`.

**TimSort** -- гібрид merge sort + insertion sort, створений Tim Peters для Python. Оптимізується на **real-world даних**, які часто частково відсортовані.

**Ключові ідеї:**
1. Розбиває масив на **runs** -- вже відсортовані (зростаюче або спадаюче) підмасиви.
2. Маленькі runs довго-сортує **insertion sort**.
3. Зливає runs через **merge** (як у merge sort).
4. Стратегічно вибирає, які runs зливати (стек з інваріантами).

**Властивості TimSort:**
- **Best case:** O(n) -- коли масив вже відсортований.
- **Worst/average:** O(n log n).
- **Stable.**
- Не in-place (O(n) space).

**Важливо для JS:**
- До ES2019 `Array.prototype.sort` **не гарантував stability** (в Chrome був нестабільний для > 10 елементів). З ES2019 -- stable обов'язково.
- TimSort чудово справляється з частково відсортованими даними. Якщо ти сортуєш майже відсортований масив -- це майже O(n).

---

## Кастомне сортування в JS: шаблони і пастки

### `arr.sort()` без comparator -- ПАСТКА

```javascript
[1, 2, 10, 21].sort();
// Очікування: [1, 2, 10, 21]
// Реальність: [1, 10, 2, 21]  ← ЛЕКСИКОГРАФІЧНЕ!
```

Дефолт `sort()` перетворює елементи на рядки і порівнює лексикографічно. `"10" < "2"`, бо `'1' < '2'`.

**Завжди** передавай comparator для числового сортування:
```javascript
[1, 2, 10, 21].sort((a, b) => a - b);  // ✅ [1, 2, 10, 21]
```

### Числове сортування

```javascript
arr.sort((a, b) => a - b);   // зростаюче
arr.sort((a, b) => b - a);   // спадаюче
```

Формула: `a - b` зростаюче (бо якщо a < b -> від'ємне -> a перед b).

**Пастка:** для великих чисел `a - b` може переповнитись. Для Number.MAX_SAFE_INTEGER це не проблема, але для BigInt не працює. Безпечніше:
```javascript
arr.sort((a, b) => a < b ? -1 : a > b ? 1 : 0);
```

### Сортування рядків

```javascript
arr.sort();                  // лексикографічне (ok для simple ASCII)
arr.sort((a, b) => a.localeCompare(b));  // з урахуванням locale (ua, en, тощо)
```

### Сортування за ключем

```javascript
users.sort((a, b) => a.age - b.age);               // за age
users.sort((a, b) => a.name.localeCompare(b.name)); // за name
```

### Stable sort і second-key ordering

Stability важлива, коли сортуєш за кількома критеріями послідовно.

```javascript
// Сортуємо спочатку за age, потім за name (зберігаючи age-порядок при однаковому name)
users.sort((a, b) => a.age - b.age);      // вторинний ключ ПЕРШИМ
users.sort((a, b) => a.name.localeCompare(b.name));  // первинний ключ ДРУГИМ
// Завдяки stability other equal elements зберігають порядок з попереднього сорту
```

Альтернатива -- composite comparator:
```javascript
users.sort((a, b) => {
  const byName = a.name.localeCompare(b.name);
  if (byName !== 0) return byName;
  return a.age - b.age;
});
```

### `sort()` модифікує масив на місці

```javascript
const arr = [3, 1, 2];
const sorted = arr.sort((a, b) => a - b);
// arr === sorted → true (той самий масив!)
// arr = [1, 2, 3]
```

Якщо тобі потрібна копія -- `[...arr].sort(...)` або `arr.slice().sort(...)`.

### ES2023: `toSorted()`

З ES2023 є immutable-варіант:
```javascript
const sorted = arr.toSorted((a, b) => a - b);  // нова копія, arr не змінюється
```

Якщо підтримує таргет -- зручно і безпечно.

---

## Binary Search: детально

**Бінарний пошук** -- пошук у **відсортованому** масиві за O(log n). Один з найважливіших алгоритмів для співбесід.

### Базова ідея

```
Ми шукаємо 7 у масиві [1, 3, 5, 7, 9, 11, 13, 15]:

  l                             r
  1, 3, 5, 7, 9, 11, 13, 15
           m
  mid=3, arr[mid]=7 → ЗНАЙШЛИ!

Якби шукали 11:
  mid=3, arr[3]=7 < 11 → l = mid+1

        l                       r
  1, 3, 5, 7, 9, 11, 13, 15
                 m
  mid=5, arr[5]=11 → ЗНАЙШЛИ!
```

На кожному кроці відкидаємо половину. За log₂(n) кроків -- знаходимо.

### Базовий template

```javascript
function binarySearch(arr, target) {
  let left = 0, right = arr.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
  }
  return -1;
}
```

**Ключові точки:**
- `right = arr.length - 1` (не `.length`).
- Умова `while (left <= right)`.
- `left = mid + 1`, `right = mid - 1` -- **завжди** ++1/--1, інакше infinite loop.
- `Math.floor((left + right) / 2)` -- заокруглення вниз.

### Off-by-one: дві версії template

**Template A: closed interval [left, right]**
```javascript
while (left <= right) {
  ...
  left = mid + 1;
  right = mid - 1;
}
```
Виходимо, коли `left > right`.

**Template B: half-open interval [left, right)**
```javascript
right = arr.length;   // ! 
while (left < right) {
  ...
  left = mid + 1;
  right = mid;         // ! не mid - 1
}
```
Виходимо, коли `left === right`. Після циклу `left` -- позиція вставки.

Обирай **одну** версію і дотримуйся її. Інтервенція різних версій -- головна причина багів.

### Binary search overflow у JS

У C/Java `mid = (l + r) / 2` може переповнитись. Безпечно: `mid = l + (r - l) / 2`.

У JS `Number.MAX_SAFE_INTEGER = 2^53 - 1` -- для масивів це не проблема (індекси набагато менші). Але згадати про overflow -- добра практика.

### Перший/останній match (lower_bound / upper_bound)

Якщо у масиві є дублікати `[1, 2, 2, 2, 3]`, і ми шукаємо `2`:
- Базовий binary search знайде **якийсь** індекс, не обов'язково перший.

**Lower bound** (перший індекс, де `arr[i] >= target`):

```javascript
function lowerBound(arr, target) {
  let left = 0, right = arr.length;
  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    if (arr[mid] < target) left = mid + 1;
    else right = mid;
  }
  return left;
}
```

**Upper bound** (перший індекс, де `arr[i] > target`):
```javascript
function upperBound(arr, target) {
  let left = 0, right = arr.length;
  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    if (arr[mid] <= target) left = mid + 1;
    else right = mid;
  }
  return left;
}
```

**Кількість входжень `target`:** `upperBound(arr, target) - lowerBound(arr, target)`.

### Візуалізація lower_bound

```
arr = [1, 2, 2, 2, 3], target = 2

l=0, r=5:   mid=2, arr[2]=2, не < 2 → r=2
l=0, r=2:   mid=1, arr[1]=2, не < 2 → r=1
l=0, r=1:   mid=0, arr[0]=1, < 2 → l=1
l=1, r=1:   exit

result: 1 (перший 2 на індексі 1) ✓
```

### Binary search on answer (пошук відповіді бінарним пошуком)

Це **потужний patterns**. Коли відповідь -- число, яке:
- Можна обмежити знизу і зверху.
- Існує функція `valid(x)` -- монотонна (якщо x працює, то x+1 теж працює; або навпаки).

Ти робиш бінарний пошук на **значенні відповіді**, а не на індексах.

**Приклад: Koko Eating Bananas (LeetCode 875).**

Koko їсть банани зі швидкістю k бананів на годину. Треба з'їсти всі piles за h годин. Знайти мінімальне k.

- Нижня межа: `left = 1`.
- Верхня межа: `right = max(piles)` (при такій швидкості з'їсть найбільшу купку за годину).
- `valid(k)` = "Koko встигне за h годин при швидкості k". Монотонна: якщо працює k, працює і k+1.

```javascript
function minEatingSpeed(piles, h) {
  let left = 1, right = Math.max(...piles);

  const canEat = (k) => {
    let hours = 0;
    for (const pile of piles) hours += Math.ceil(pile / k);
    return hours <= h;
  };

  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    if (canEat(mid)) right = mid;       // можемо менше → шукаємо зліва
    else left = mid + 1;
  }
  return left;
}
```

**Приклад: Split Array Largest Sum (LeetCode 410).**

Розділити масив на m підмасивів так, щоб мінімізувати найбільшу суму підмасиву.

- `left = max(arr)` (мінімум -- принаймні найбільший елемент).
- `right = sum(arr)` (максимум -- якщо один підмасив).
- `valid(maxSum)` = "чи можна розділити на ≤ m підмасивів з сумою ≤ maxSum".

Ідея та сама: бінарний пошук на значенні відповіді.

**Загальний patterns шаблон:**
```javascript
let left = MIN, right = MAX;
while (left < right) {
  const mid = Math.floor((left + right) / 2);
  if (feasible(mid)) right = mid;       // шукаємо мінімальний feasible
  else left = mid + 1;
}
return left;
```

### Пошук у rotated sorted array (LeetCode 33)

Масив був відсортований, потім поділили в невідомій точці і поміняли місцями. Наприклад: `[4, 5, 6, 7, 0, 1, 2]`. Знайти target.

**Ідея:** на кожному кроці одна з половин **відсортована**. Перевіряєш, в якій половині target, і стрибаєш туди.

```javascript
function searchRotated(arr, target) {
  let left = 0, right = arr.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    if (arr[mid] === target) return mid;

    // Ліва половина відсортована?
    if (arr[left] <= arr[mid]) {
      if (target >= arr[left] && target < arr[mid]) right = mid - 1;
      else left = mid + 1;
    } else {
      // Права половина відсортована
      if (target > arr[mid] && target <= arr[right]) left = mid + 1;
      else right = mid - 1;
    }
  }
  return -1;
}
```

### Типові пастки бінарного пошуку

1. **Off-by-one.** Плутанина між `< length` і `<= length - 1`, між `mid` і `mid - 1`. Обирай template, дотримуйся.

2. **Infinite loop.** Якщо `mid = (l + r) / 2`, і `l = mid` (замість `l = mid + 1`) -- в певних випадках `mid == l`, цикл зависає. Приклад:
   ```
   l=0, r=1, mid=0, l=mid=0 → l=0 назавжди
   ```
   Правило: принаймні одна з меж має зсуватись на `±1`.

3. **Overflow.** У JS не проблема, але писати `l + (r - l) / 2` замість `(l + r) / 2` -- good practice.

4. **Забули перевірити межі.** Якщо target менший за всі або більший -- `lowerBound` поверне 0 або n. Треба завжди перевірити `arr[result] === target` після.

5. **Масив не відсортований.** Binary search працює **тільки на відсортованих**. Якщо не впевнений -- перевір або сортуй першим. О(n log n) на sort + O(log n) на search краще за O(n) тільки для багаторазових пошуків.

6. **Ранній `return mid`.** У варіації "перший match" -- не повертай одразу, бо правіше може бути ще один такий target.

---

## Пошук у 2D matrix

Коли матриця має спеціальну структуру (рядки/стовпці відсортовані) -- можна пошукати швидше за O(m × n).

### Варіант 1: Кожен рядок відсортований, і перший елемент рядка > останній попереднього

Насправді це один великий відсортований масив. Бінарний пошук за O(log(m × n)):

```javascript
function searchMatrix(matrix, target) {
  const m = matrix.length, n = matrix[0].length;
  let left = 0, right = m * n - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);
    const val = matrix[Math.floor(mid / n)][mid % n];

    if (val === target) return true;
    if (val < target) left = mid + 1;
    else right = mid - 1;
  }
  return false;
}
```

### Варіант 2: Кожен рядок відсортований, кожен стовпець відсортований (але не глобально)

LeetCode 240. Приклад:
```
[1,  4,  7, 11]
[2,  5,  8, 12]
[3,  6,  9, 16]
[10, 13, 14, 17]
```

**Трюк:** старт з **верхнього правого** (або нижнього лівого) кута.
- Якщо `curr > target` → зсунься вліво (всі знизу теж більші).
- Якщо `curr < target` → зсунься вниз (всі зліва теж менші).

```javascript
function searchMatrix(matrix, target) {
  let r = 0, c = matrix[0].length - 1;
  while (r < matrix.length && c >= 0) {
    if (matrix[r][c] === target) return true;
    if (matrix[r][c] > target) c--;
    else r++;
  }
  return false;
}
```

Complexity: O(m + n) -- за один прохід. Не чистий binary, але лінійне зниження.

---

## Типові пастки та помилки

1. **`arr.sort()` без comparator.** Лексикографічне сортування, а не числове.

2. **`arr.sort()` модифікує оригінал.** Якщо не хочеш -- `[...arr].sort()`.

3. **Binary search на невідсортованому масиві.** Результат undefined. Перевір input.

4. **Off-by-one у binary search.** Обирай один template і дотримуйся.

5. **`mid = (l + r) / 2` overflow.** У JS не проблема, але прийнято писати `l + Math.floor((r - l) / 2)`.

6. **Infinite loop** -- коли зсуваєш межу на `mid` замість `mid ± 1`, і `mid == l` в якійсь ітерації.

7. **Stable вимоги не дотримані.** `quickSort` -- НЕ stable. Якщо сортування за другим ключем через послідовні sort-и -- використовуй stable алгоритм.

8. **O(n²) quicksort на відсортованих даних.** Пастка при вивченні. Якщо pivot -- перший/останній елемент, на вже відсортованому масиві отримаєш worst case. Randomized pivot або median-of-three.

9. **Counting sort на великому діапазоні.** `max = 10⁹` → масив на мільярд комірок. Використовуй counting sort тільки при малих значеннях.

10. **Merge sort on linked lists vs arrays.** Для масивів -- merge sort O(n) space. Для linked lists -- O(1) space (просто перемикаємо next-посилання).

11. **Binary search без перевірки меж.** `lowerBound` може повернути `arr.length` (позицію в кінці). Не забудь перевірити результат.

12. **Порівняння BigInt/Date з `a - b`.** Для Date працює, бо `-` повертає number. Для BigInt -- помилка типу. Використовуй `a < b ? -1 : ...`.

---

## Ключові думки

- Знай таблицю **Best/Avg/Worst, Space, Stable** для основних алгоритмів.
- **TimSort** -- дефолт у V8. Stable з ES2019. Оптимізована для частково відсортованих даних.
- **Merge sort** -- O(n log n), stable, O(n) space. Дивиз and conquer, основа для external sort і count inversions.
- **Quicksort** -- O(n log n) average, O(n²) worst, in-place, не stable. Randomized pivot уникає worst case. **QuickSelect** -- O(n) для k-th.
- **Counting/radix sort** -- non-comparison, < O(n log n) для integers у малому діапазоні.
- `arr.sort((a, b) => a - b)` -- завжди передавай comparator для чисел!
- Stable sort -- критично для multi-key ordering.
- **Binary search** -- O(log n). Closed vs half-open interval, lower/upper bound, binary search on answer, rotated array.
- Пастки: off-by-one, infinite loop, лексикографічне sort, unsorted input.
- На співбесіді проговорюй stable/in-place/worst-case -- це показує зрілість.

Сортування й бінарний пошук -- це "knife skills" алгоритміста. Вони з'являються у більшості задач так чи інакше: як preprocessing, як пошук у відповіді, як компонент гібридного розв'язку. Добре володіння цими темами знімає величезний пласт стресу на співбесіді.
