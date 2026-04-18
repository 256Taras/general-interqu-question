# Топ алгоритмічних патернів для співбесід (зведений огляд)

## 1. Як користуватися цим файлом

Цей файл -- **довідник-навігатор**, а не підручник. Основна ідея: 80% алгоритмічних задач на співбесідах зводяться до 15-20 патернів. Якщо ти розпізнаєш патерн -- шаблонний код пишеться майже автоматично.

**Коли відкривати цей файл:**
1. **За тиждень до співбесіди** -- перегляд "з висоти пташиного польоту", освіжити пам'ять.
2. **Під час тренування** -- отримав нову задачу, не знаєш з чого почати -- глянь секцію "Як розпізнати патерн за умовою задачі" нижче.
3. **Після розв'язання задачі** -- знайди патерн у цьому файлі, зіставвь зі своїм розв'язком, зрозумій, куди це вписується.

**Як читати секцію патерну:**
- **Коли використовувати** -- тригерні слова в умові. Якщо бачиш ці слова -- спробуй цей патерн першим.
- **Основна ідея** -- коротко, щоб згадати суть за 30 секунд.
- **Шаблонний код** -- generic скелет. На співбесіді адаптуєш під задачу.
- **Приклади задач** -- класичні LeetCode-задачі, на яких треновано цей патерн.
- **Часто разом з** -- додаткові структури даних, які доповнюють патерн.
- **Складність** -- типові time/space.
- **Детальніше** -- посилання на основний файл у цій папці.

**Важливо:** цей файл -- **не заміна** детальних файлів. Він індекс. Якщо у тебе прогалина в темі (наприклад, не впевнений у DFS) -- йди в `06-trees.md` і `07-graphs.md`, вчи з нуля, потім повертайся сюди для консолідації.

**Попередні файли у папці (рекомендований порядок вивчення):**
- [00-intro-big-o.md](00-intro-big-o.md) -- Big O, аналіз складності
- [01-arrays-strings.md](01-arrays-strings.md) -- масиви, рядки, two pointers, sliding window
- [02-hash-maps-sets.md](02-hash-maps-sets.md) -- hash map / set для O(1)
- [03-linked-lists.md](03-linked-lists.md) -- зв'язані списки, Floyd's cycle
- [04-stacks-queues.md](04-stacks-queues.md) -- стеки, черги, monotonic stack
- [05-recursion-backtracking.md](05-recursion-backtracking.md) -- рекурсія, backtracking
- [06-trees.md](06-trees.md) -- дерева, DFS, BFS, Trie
- [07-graphs.md](07-graphs.md) -- графи, Union-Find, Topological Sort
- [08-sorting-searching.md](08-sorting-searching.md) -- сортування, бінарний пошук
- [09-dynamic-programming.md](09-dynamic-programming.md) -- динамічне програмування

---

## 2. Two Pointers

**Коли використовувати:**
- Масив **відсортований** (або можна відсортувати).
- Треба знайти **пару, трійку, набір елементів** з певною сумою/властивістю.
- Перевірка паліндрому.
- Злиття двох відсортованих структур.
- Видалення дублікатів in-place.
- Тригерні слова: "two sum в sorted array", "container with most water", "palindrome", "merge sorted".

**Основна ідея:**

Замість вкладених циклів O(n²) -- два індекси `left` і `right` рухаються назустріч (або в одному напрямку). На кожному кроці приймаємо рішення: куди рухати. Це скорочує простір пошуку лінійно -- O(n).

```
Two Sum у відсортованому масиві:

[1, 3, 5, 7, 9, 11]    target = 14
 ^              ^
 L              R       sum = 1+11 = 12 < 14  -> L++

[1, 3, 5, 7, 9, 11]
    ^           ^
    L           R       sum = 3+11 = 14  -> знайшли!
```

**Шаблонний код:**

```javascript
// Загальний шаблон для two pointers у відсортованому масиві
function twoPointers(arr, target) {
  let left = 0;
  let right = arr.length - 1;

  while (left < right) {
    const current = arr[left] + arr[right];

    if (current === target) {
      return [left, right];        // знайшли
    } else if (current < target) {
      left++;                       // потрібна більша сума -> рухаємо L вправо
    } else {
      right--;                      // потрібна менша сума -> рухаємо R вліво
    }
  }

  return [-1, -1];
}
```

**Приклади задач:**
- **Two Sum II (sorted)** -- знайти пару з сумою = target.
- **3Sum** -- знайти всі трійки з сумою 0 (фіксуємо один елемент, two pointers на решту).
- **Container With Most Water** -- максимальна площа між двома лініями.
- **Valid Palindrome** -- перевірка паліндрому.
- **Remove Duplicates from Sorted Array** -- in-place видалення, two pointers у одному напрямку (slow + fast).

**Часто разом з:**
- Сортування (якщо масив ще не відсортований).
- Hash map (альтернатива для невідсортованих даних).

**Складність:**
- Time: O(n), або O(n²) для 3Sum (зовнішній цикл + two pointers всередині).
- Space: O(1).

**Детальніше:** [01-arrays-strings.md](01-arrays-strings.md)

---

## 3. Sliding Window (fixed & variable)

**Коли використовувати:**
- Треба знайти/обчислити щось **у підмасиві/підрядку** (contiguous subarray).
- "Розмір вікна K" -- fixed window.
- "Найдовший/найкоротший підрядок з умовою X" -- variable window.
- Тригерні слова: "subarray", "substring", "contiguous", "window", "longest", "shortest".

**Основна ідея:**

Замість пересчитувати всю суму/статистику підмасиву заново при кожному зсуві -- підтримуй вікно `[left, right]` і **інкрементально** оновлюй стан: додаєш елемент справа (`right++`), видаляєш зліва (`left++`).

**Fixed window (розмір K):**
```
Знайти max sum підмасиву розміру 3:

[2, 1, 5, 1, 3, 2]
 [---]                sum = 2+1+5 = 8
    [---]             sum - 2 + 1 = 7
       [---]          sum - 1 + 3 = 9  <- max
          [---]       sum - 5 + 2 = 6
```

**Variable window (розмір змінюється):**
```
Найдовший підрядок без повторень символів:

"a b c a b c b b"
 L R                   вікно "a",    довжина 1
 L   R                 вікно "ab",   довжина 2
 L     R               вікно "abc",  довжина 3  <- max
   L   R               'a' дубль -> зсуваємо L доки не приберемо дубль
   L     R             вікно "bca",  довжина 3
   ...
```

**Шаблонний код:**

```javascript
// Fixed-size sliding window
function fixedWindow(arr, k) {
  let windowSum = 0;
  for (let i = 0; i < k; i++) windowSum += arr[i];    // перше вікно

  let maxSum = windowSum;
  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i] - arr[i - k];                  // додав новий, прибрав старий
    maxSum = Math.max(maxSum, windowSum);
  }
  return maxSum;
}

// Variable-size sliding window (найдовший підрядок з умовою)
function variableWindow(s) {
  const seen = new Map();         // символ -> остання позиція
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    // якщо умова порушена -- зсуваємо left
    if (seen.has(c) && seen.get(c) >= left) {
      left = seen.get(c) + 1;
    }
    seen.set(c, right);
    maxLen = Math.max(maxLen, right - left + 1);
  }
  return maxLen;
}
```

**Приклади задач:**
- **Maximum Sum Subarray of Size K** -- класичний fixed.
- **Longest Substring Without Repeating Characters** -- variable + hash set.
- **Minimum Window Substring** -- variable + hash map з підрахунком.
- **Permutation in String** -- fixed + частотна мапа.
- **Longest Repeating Character Replacement** -- variable + лічильник max частоти.

**Часто разом з:**
- Hash map / Set (для підрахунку частот у вікні).
- Deque (для sliding window max -- дивись патерн 9).

**Складність:**
- Time: O(n) -- кожен елемент заходить і виходить з вікна раз.
- Space: O(k) або O(алфавіт) для допоміжної структури.

**Детальніше:** [01-arrays-strings.md](01-arrays-strings.md)

---

## 4. Prefix Sum / Hash Map for Subarrays

**Коли використовувати:**
- Задачі на **сумму підмасиву**, коли sliding window не підходить (наприклад, у масиві є **від'ємні числа**).
- Треба відповідати на багато запитів "сума від i до j" -- префіксна сума дає O(1) на запит.
- "Скільки підмасивів має суму K" -- prefix sum + hash map.
- Тригерні слова: "subarray sum", "range sum", "equals K", "divisible by K".

**Основна ідея:**

`prefix[i] = arr[0] + arr[1] + ... + arr[i-1]`. Тоді сума підмасиву `[i..j]` = `prefix[j+1] - prefix[i]`.

Для пошуку підмасивів з сумою K: якщо `prefix[j] - prefix[i] = K`, то `prefix[i] = prefix[j] - K`. Тримаємо мапу побачених prefix sums -- шукаємо `current - K` у мапі.

```
arr:     [1, 2, 3, -2, 5]
prefix:  [0, 1, 3, 6,  4, 9]    (prefix[0] = 0 -- порожня сума)

Сума [1..3] = prefix[4] - prefix[1] = 4 - 1 = 3
```

**Шаблонний код:**

```javascript
// 1. Побудова prefix sum -- range queries за O(1)
function buildPrefix(arr) {
  const prefix = [0];
  for (const x of arr) prefix.push(prefix[prefix.length - 1] + x);
  return prefix;
}
function rangeSum(prefix, i, j) {   // сума arr[i..j] включно
  return prefix[j + 1] - prefix[i];
}

// 2. Кількість підмасивів з сумою K (працює з від'ємними!)
function subarraySumEqualsK(arr, k) {
  const counts = new Map();
  counts.set(0, 1);                 // порожній префікс зустрічався раз
  let sum = 0;
  let result = 0;

  for (const x of arr) {
    sum += x;
    // скільки разів раніше бачили prefix = sum - k?
    result += counts.get(sum - k) || 0;
    counts.set(sum, (counts.get(sum) || 0) + 1);
  }
  return result;
}
```

**Приклади задач:**
- **Subarray Sum Equals K** -- prefix sum + map.
- **Continuous Subarray Sum** (divisible by K) -- prefix sum mod K + map.
- **Range Sum Query - Immutable** -- просто prefix.
- **Contiguous Array** (0/1 balanced) -- заміни 0 на -1, шукай підмасив з сумою 0.
- **Maximum Size Subarray Sum Equals K** -- map зі значеннями = перший індекс.

**Часто разом з:**
- Hash map (`prefix -> count` або `prefix -> first index`).
- 2D prefix sum для матриць.

**Складність:**
- Time: O(n). Space: O(n).

**Детальніше:** [01-arrays-strings.md](01-arrays-strings.md), [02-hash-maps-sets.md](02-hash-maps-sets.md)

---

## 5. Hash Map / Set for O(1) lookup

**Коли використовувати:**
- Треба **швидко перевіряти наявність** елемента.
- Треба **групувати** елементи за ключем (анаграми, частоти).
- Треба **рахувати частоти**.
- Тригерні слова: "two sum", "contains duplicate", "group anagrams", "frequency".

**Основна ідея:**

Обмін пам'яті на час. Замість O(n) пошуку в масиві -- O(1) через hash. Часто перетворює O(n²) задачі на O(n).

**Шаблонний код:**

```javascript
// Two Sum за O(n)
function twoSum(arr, target) {
  const seen = new Map();                 // value -> index
  for (let i = 0; i < arr.length; i++) {
    const need = target - arr[i];
    if (seen.has(need)) return [seen.get(need), i];
    seen.set(arr[i], i);
  }
  return [];
}

// Група анаграм
function groupAnagrams(strs) {
  const groups = new Map();
  for (const s of strs) {
    const key = [...s].sort().join("");   // канонічний ключ
    if (!groups.has(key)) groups.set(key, []);
    groups.get(key).push(s);
  }
  return [...groups.values()];
}

// Частоти
function frequency(arr) {
  const freq = new Map();
  for (const x of arr) freq.set(x, (freq.get(x) || 0) + 1);
  return freq;
}
```

**Приклади задач:**
- **Two Sum** -- класика.
- **Contains Duplicate** -- Set.
- **Group Anagrams** -- map за канонічним ключем.
- **Valid Anagram** -- частотна мапа.
- **Longest Consecutive Sequence** -- Set + перевірка "чи я початок послідовності".
- **First Unique Character** -- частотна мапа.

**Часто разом з:**
- Sliding Window (часто всередині вікна потрібна мапа).
- Prefix Sum (pattern 4).

**Складність:**
- Time: O(n) avg. Space: O(n).

**Детальніше:** [02-hash-maps-sets.md](02-hash-maps-sets.md)

---

## 6. Fast & Slow Pointers (Floyd's Cycle Detection)

**Коли використовувати:**
- Зв'язані списки: **детектування циклу**, середина списку, перетин списків.
- Масиви, де елемент вказує на інший індекс: "find the duplicate", "happy number".
- Тригерні слова: "cycle", "middle of list", "loop", "linked list".

**Основна ідея:**

Два вказівники рухаються з різною швидкістю -- `slow` на 1 крок, `fast` на 2. Якщо є цикл, `fast` наздожене `slow` всередині циклу (math theorem). Якщо циклу немає, `fast` досягне кінця.

```
Детектування циклу:

1 -> 2 -> 3 -> 4 -> 5
          ^         |
          +---------+

slow: 1 -> 2 -> 3 -> 4 -> 5 -> 3 -> 4 -> 5 -> 3
fast: 1 -> 3 -> 5 -> 4 -> 3 -> 5 -> 4 -> 3 (зустрілись)
```

**Шаблонний код:**

```javascript
// Детектування циклу
function hasCycle(head) {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow === fast) return true;
  }
  return false;
}

// Середина списку
function middleNode(head) {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
  }
  return slow;      // коли fast вийшов за край, slow на середині
}

// Знайти початок циклу (Floyd's algorithm phase 2)
function cycleStart(head) {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow === fast) {
      let p = head;
      while (p !== slow) { p = p.next; slow = slow.next; }
      return p;
    }
  }
  return null;
}
```

**Приклади задач:**
- **Linked List Cycle** -- has cycle?
- **Linked List Cycle II** -- де початок циклу.
- **Middle of the Linked List** -- середина.
- **Happy Number** -- детектування циклу в послідовності sum of squared digits.
- **Find the Duplicate Number** -- масив як "linked list" через індекси.

**Часто разом з:**
- Reverse Linked List (pattern 7) -- для palindrome linked list.

**Складність:**
- Time: O(n). Space: O(1) -- головна перевага перед hash set.

**Детальніше:** [03-linked-lists.md](03-linked-lists.md)

---

## 7. Reverse Linked List (in-place)

**Коли використовувати:**
- Треба обернути список чи його частину.
- Palindrome linked list (реверс другої половини).
- Reorder list.
- Reverse k-групами.
- Тригерні слова: "reverse", "reorder", "k-group".

**Основна ідея:**

Три вказівники -- `prev`, `curr`, `next`. Ідемо вперед, перенаправляючи `curr.next` на `prev`. In-place, O(1) space.

```
До:    1 -> 2 -> 3 -> 4 -> null
Після: null <- 1 <- 2 <- 3 <- 4

Крок за кроком:
prev=null, curr=1
  next=curr.next (2)
  curr.next=prev (null)    ->  null <- 1    2 -> 3 -> 4
  prev=curr (1), curr=next (2)

prev=1, curr=2
  next=3
  curr.next=prev (1)       ->  null <- 1 <- 2    3 -> 4
  ...
```

**Шаблонний код:**

```javascript
function reverseList(head) {
  let prev = null;
  let curr = head;
  while (curr) {
    const next = curr.next;
    curr.next = prev;        // розвертаємо вказівник
    prev = curr;
    curr = next;
  }
  return prev;               // новий head
}

// Рекурсивний варіант
function reverseRec(head) {
  if (!head || !head.next) return head;
  const newHead = reverseRec(head.next);
  head.next.next = head;
  head.next = null;
  return newHead;
}
```

**Приклади задач:**
- **Reverse Linked List** -- база.
- **Reverse Linked List II** -- реверс діапазону [m..n].
- **Reverse Nodes in k-Group** -- реверс групами.
- **Palindrome Linked List** -- знайти середину, реверснути другу половину, порівняти.
- **Reorder List** -- middle + reverse + merge.

**Часто разом з:**
- Fast & Slow Pointers (pattern 6) для знаходження середини.
- Dummy node (щоб спростити обробку head).

**Складність:**
- Time: O(n). Space: O(1) ітеративно, O(n) рекурсивно (stack).

**Детальніше:** [03-linked-lists.md](03-linked-lists.md)

---

## 8. Monotonic Stack

**Коли використовувати:**
- Треба знайти **наступний більший/менший елемент** для кожної позиції.
- Задачі "скільки днів до вищої температури", "largest rectangle in histogram".
- Тригерні слова: "next greater", "next smaller", "previous greater", "daily temperatures", "largest rectangle", "stock span".

**Основна ідея:**

Стек, у якому елементи (зазвичай індекси) підтримують монотонний порядок (спадний або зростаючий). Коли приходить новий елемент, що порушує монотонність -- виштовхуємо зі стеку, і для кожного виштовхнутого це був момент "знайшли наступний більший/менший".

```
Next Greater Element:  [2, 1, 2, 4, 3]

i=0 (2): stack=[0]            answer=[?, ?, ?, ?, ?]
i=1 (1): 1 < 2, stack=[0,1]
i=2 (2): 2 > 1 -> pop 1, ans[1]=2
         2 == 2, stack=[0,2]
i=3 (4): 4 > 2 -> pop 2, ans[2]=4
         4 > 2 -> pop 0, ans[0]=4
         stack=[3]
i=4 (3): 3 < 4, stack=[3,4]

те, що залишилось у стеку -> -1 (немає greater)
answer = [4, 2, 4, -1, -1]
```

**Шаблонний код:**

```javascript
// Next greater element для кожної позиції
function nextGreater(arr) {
  const result = new Array(arr.length).fill(-1);
  const stack = [];                         // стек індексів, спадний за значенням

  for (let i = 0; i < arr.length; i++) {
    while (stack.length && arr[stack[stack.length - 1]] < arr[i]) {
      const idx = stack.pop();
      result[idx] = arr[i];                 // для idx знайшли next greater
    }
    stack.push(i);
  }
  return result;
}
```

**Приклади задач:**
- **Next Greater Element I/II** -- класика.
- **Daily Temperatures** -- кількість днів до вищої температури.
- **Largest Rectangle in Histogram** -- класична складна, monotonic stack.
- **Trapping Rain Water** -- можна monotonic stack або two pointers.
- **Stock Span** -- previous greater.
- **Sum of Subarray Minimums** -- monotonic stack + математика.

**Часто разом з:**
- Масиви індексів (щоб обчислити відстань).

**Складність:**
- Time: O(n) -- кожен елемент push і pop раз.
- Space: O(n).

**Детальніше:** [04-stacks-queues.md](04-stacks-queues.md)

---

## 9. Queue / Deque (Sliding Window Maximum)

**Коли використовувати:**
- Потрібен **max/min у sliding window**.
- BFS (звичайна Queue) -- дивись pattern 11.
- Задачі з шаблоном "обробка в порядку FIFO".
- Тригерні слова: "sliding window max", "shortest path (unweighted)", "level by level".

**Основна ідея (deque для sliding window max):**

Тримаємо deque індексів, де відповідні значення -- спадні. Front deque завжди -- max поточного вікна. Коли додаємо новий елемент -- виштовхуємо з хвоста всі менші (вони вже не будуть max). Коли індекс з голови виходить за ліву межу вікна -- викидаємо його.

```
arr=[1,3,-1,-3,5,3,6,7], k=3

вікно [1,3,-1]:   deque=[1(3), 2(-1)]          max=3
вікно [3,-1,-3]:  deque=[1(3), 2(-1), 3(-3)]   max=3
вікно [-1,-3,5]:  pop 3,-3,-1;  deque=[4(5)]   max=5
вікно [-3,5,3]:   deque=[4(5), 5(3)]           max=5
...
```

**Шаблонний код:**

```javascript
function slidingWindowMax(arr, k) {
  const deque = [];               // індекси; arr[deque] -- спадний
  const result = [];

  for (let i = 0; i < arr.length; i++) {
    // викинути з голови індекси, що вийшли за межі вікна
    if (deque.length && deque[0] <= i - k) deque.shift();

    // викинути з хвоста всі менші за поточний
    while (deque.length && arr[deque[deque.length - 1]] < arr[i]) {
      deque.pop();
    }
    deque.push(i);

    if (i >= k - 1) result.push(arr[deque[0]]);
  }
  return result;
}
```

**Примітка:** у JavaScript `Array.shift()` -- O(n). Для справжньої O(n) реалізації використовуй двосторонню чергу через linked list або два стеки.

**Приклади задач:**
- **Sliding Window Maximum** -- класика.
- **Shortest Subarray with Sum at Least K** -- deque + prefix sum.
- **Constrained Subsequence Sum** -- DP + deque.
- **Queue using Stacks / Stack using Queues** -- розминка.

**Часто разом з:**
- Sliding Window (pattern 3).
- BFS (pattern 11).

**Складність:**
- Time: O(n). Space: O(k).

**Детальніше:** [04-stacks-queues.md](04-stacks-queues.md)

---

## 10. DFS (Depth-First Search)

**Коли використовувати:**
- Обхід **дерева** (pre/in/post order).
- Обхід **графа** -- перевірка зв'язності, пошук циклу, знаходження всіх шляхів.
- Задачі "чи існує шлях", "перерахувати всі шляхи".
- Backtracking (pattern 14) -- це по суті DFS з відкочуванням.
- Тригерні слова: "traverse tree", "all paths", "connected components", "islands".

**Основна ідея:**

Йдемо вглиб якомога далі, потім повертаємось і йдемо в іншу гілку. Природно реалізується рекурсивно; ітеративно -- через стек.

```
Дерево:        1
              / \
             2   3
            / \   \
           4   5   6

Preorder (корінь, лівий, правий): 1, 2, 4, 5, 3, 6
Inorder  (лівий, корінь, правий): 4, 2, 5, 1, 3, 6
Postorder(лівий, правий, корінь): 4, 5, 2, 6, 3, 1
```

**Шаблонний код:**

```javascript
// DFS по дереву (рекурсивно)
function dfsTree(node) {
  if (!node) return;
  // preorder: обробка ДО спуску
  dfsTree(node.left);
  // inorder: обробка МІЖ
  dfsTree(node.right);
  // postorder: обробка ПІСЛЯ
}

// DFS по графу (з visited, щоб не ходити по колу)
function dfsGraph(start, graph) {
  const visited = new Set();
  function dfs(node) {
    if (visited.has(node)) return;
    visited.add(node);
    // обробка node
    for (const neighbor of graph.get(node) || []) {
      dfs(neighbor);
    }
  }
  dfs(start);
}

// DFS ітеративно через стек
function dfsIterative(root) {
  if (!root) return;
  const stack = [root];
  while (stack.length) {
    const node = stack.pop();
    // обробка
    if (node.right) stack.push(node.right);   // push right перший,
    if (node.left) stack.push(node.left);     // щоб left обробився перший
  }
}
```

**Приклади задач:**
- **Maximum Depth of Binary Tree** -- post-order DFS.
- **Validate BST** -- in-order DFS (має бути зростаючим).
- **Path Sum / Path Sum II** -- DFS з підрахунком.
- **Number of Islands** -- DFS на матриці (grid).
- **Clone Graph** -- DFS + map оригінал->копія.
- **Course Schedule** -- DFS для виявлення циклу в directed graph.

**Часто разом з:**
- Set (visited).
- Stack (для ітеративного).
- Backtracking (повернення стану після рекурсивного виклику).

**Складність:**
- Time: O(V + E) для графа, O(n) для дерева.
- Space: O(h) для дерева (висота), O(V) для графа.

**Детальніше:** [06-trees.md](06-trees.md), [07-graphs.md](07-graphs.md)

---

## 11. BFS (Breadth-First Search)

**Коли використовувати:**
- **Найкоротший шлях у unweighted графі**.
- Обхід **по рівнях** (level-order) у дереві.
- "Мінімальна кількість кроків/ходів".
- Тригерні слова: "shortest path", "minimum steps", "level order", "nearest".

**Основна ідея:**

Queue (FIFO). Спочатку обробляємо всі вузли на рівні d, потім усі на d+1. Перший раз, коли ми досягли цілі -- це найкоротший шлях (кожен крок = 1 ребро).

```
Рівні дерева:
Level 0:     1
Level 1:    2 3
Level 2:   4 5 6

BFS order: 1, 2, 3, 4, 5, 6
```

**Шаблонний код:**

```javascript
// BFS по рівнях (дерево)
function levelOrder(root) {
  if (!root) return [];
  const queue = [root];
  const result = [];

  while (queue.length) {
    const levelSize = queue.length;        // зафіксували розмір рівня
    const level = [];
    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();
      level.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}

// BFS найкоротший шлях у графі (unweighted)
function shortestPath(start, target, graph) {
  const visited = new Set([start]);
  const queue = [[start, 0]];            // [node, distance]

  while (queue.length) {
    const [node, dist] = queue.shift();
    if (node === target) return dist;
    for (const neighbor of graph.get(node) || []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push([neighbor, dist + 1]);
      }
    }
  }
  return -1;
}
```

**Примітка:** `queue.shift()` у JS -- O(n). Для справжньої O(V+E) BFS -- або використовуй індекс замість shift, або структуру Deque.

**Приклади задач:**
- **Binary Tree Level Order Traversal** -- класика.
- **Binary Tree Right Side View** -- останній елемент кожного рівня.
- **Word Ladder** -- shortest transformation.
- **Rotting Oranges** -- multi-source BFS.
- **01 Matrix** -- найближча 0 для кожної клітинки, BFS з усіх 0.
- **Shortest Path in Binary Matrix** -- BFS по grid.

**Часто разом з:**
- Set (visited).
- Deque для multi-source або 0-1 BFS.

**Складність:**
- Time: O(V + E). Space: O(V).

**Детальніше:** [06-trees.md](06-trees.md), [07-graphs.md](07-graphs.md)

---

## 12. Binary Search (у відсортованому масиві)

**Коли використовувати:**
- Масив **відсортований**.
- Шукаємо конкретне значення, перший/останній елемент з умовою.
- Тригерні слова: "sorted array", "find target", "first/last position", "insert position".

**Основна ідея:**

На кожному кроці ділимо простір пошуку навпіл. O(log n). Ключ -- коректно обрати межі (inclusive vs exclusive) і оновлення `left/right`.

```
[1, 3, 5, 7, 9, 11, 13]   target = 9

L=0, R=6, mid=3, arr[3]=7 < 9  ->  L=4
L=4, R=6, mid=5, arr[5]=11 > 9 ->  R=4
L=4, R=4, mid=4, arr[4]=9 == 9 -> return 4
```

**Шаблонний код:**

```javascript
// Класичний бінарний пошук
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

// Перша позиція, де predicate(mid) = true (lower bound)
function lowerBound(arr, predicate) {
  let left = 0, right = arr.length;     // right exclusive
  while (left < right) {
    const mid = Math.floor((left + right) / 2);
    if (predicate(arr[mid])) right = mid;
    else left = mid + 1;
  }
  return left;                          // може == arr.length якщо не знайдено
}
```

**Приклади задач:**
- **Binary Search** -- база.
- **First Bad Version** -- lower bound.
- **Search Insert Position** -- lower bound.
- **Find First and Last Position of Element** -- два бінарних пошуки.
- **Search in Rotated Sorted Array** -- модифікована бінарка.
- **Find Peak Element** -- бінарка на "склоні".

**Часто разом з:**
- Sort (якщо вхід не відсортований).

**Складність:**
- Time: O(log n). Space: O(1).

**Детальніше:** [08-sorting-searching.md](08-sorting-searching.md)

---

## 13. Binary Search on Answer

**Коли використовувати:**
- Задача "знайти мінімальне/максимальне значення X, при якому виконується умова Y".
- Якщо умова Y -- **монотонна** за X (якщо працює для X, то і для X+1; або якщо не працює для X, то і для X-1).
- Тригерні слова: "minimum capacity", "maximum value", "split into K parts", "find the smallest value such that...".

**Основна ідея:**

Не шукаємо в масиві -- шукаємо в **діапазоні можливих відповідей**. На кожному кроці робимо перевірку `feasible(mid)`. Якщо здійсненно -- рухаємо межу, щоб знайти оптимум.

```
Приклад: мінімальна вантажопідйомність корабля, щоб перевезти за D днів.

weights = [1,2,3,4,5,6,7,8,9,10], D = 5

Діапазон відповідей: [max(weights)=10 .. sum(weights)=55]
feasible(capacity) -- чи можна перевезти за D днів з такою капа?

binary search у [10..55] на min capacity, де feasible(capacity) = true
```

**Шаблонний код:**

```javascript
function binarySearchOnAnswer(low, high, feasible) {
  while (low < high) {
    const mid = Math.floor((low + high) / 2);
    if (feasible(mid)) high = mid;      // шукаємо мінімальне -> стискаємо згори
    else low = mid + 1;
  }
  return low;
}

// Приклад: мін. швидкість поглинання бананів за H годин
function minEatingSpeed(piles, H) {
  const feasible = (k) => {
    let hours = 0;
    for (const p of piles) hours += Math.ceil(p / k);
    return hours <= H;
  };
  let low = 1, high = Math.max(...piles);
  return binarySearchOnAnswer(low, high, feasible);
}
```

**Приклади задач:**
- **Koko Eating Bananas** -- мін. швидкість.
- **Capacity to Ship Packages Within D Days** -- мін. вантажопідйомність.
- **Split Array Largest Sum** -- мінімізувати максимальну суму після поділу.
- **Find the Smallest Divisor Given a Threshold**.
- **Median of Two Sorted Arrays** (складна) -- бінарка на позиції розділу.

**Часто разом з:**
- Greedy всередині `feasible` (симуляція).

**Складність:**
- Time: O(log(range) * cost_of_feasible).
- Space: O(1).

**Детальніше:** [08-sorting-searching.md](08-sorting-searching.md)

---

## 14. Backtracking (permutations, subsets, combinations)

**Коли використовувати:**
- Треба згенерувати **всі можливі** комбінації / перестановки / підмножини / розбиття.
- Constraint satisfaction (N-Queens, Sudoku).
- "Знайти будь-яке / всі рішення", коли простір рішень -- дерево.
- Тригерні слова: "all permutations", "all subsets", "all combinations", "generate", "palindrome partitioning", "word break all".

**Основна ідея:**

DFS по дереву рішень. На кожному кроці:
1. **Вибираємо** один варіант.
2. **Рекурсивно** розв'язуємо залишок.
3. **Відкочуємося** (undo) -- прибираємо вибір, пробуємо наступний.

```
Всі підмножини [1,2,3]:

              []
         /    |    \
       [1]   [2]   [3]
      /   \    \
   [1,2] [1,3] [2,3]
     |
  [1,2,3]
```

**Шаблонний код:**

```javascript
// Підмножини
function subsets(nums) {
  const result = [];
  const current = [];

  function backtrack(start) {
    result.push([...current]);                 // кожна вершина -- валідна підмножина
    for (let i = start; i < nums.length; i++) {
      current.push(nums[i]);                   // вибір
      backtrack(i + 1);                        // рекурсія
      current.pop();                           // відкат
    }
  }
  backtrack(0);
  return result;
}

// Перестановки
function permutations(nums) {
  const result = [];
  const current = [];
  const used = new Array(nums.length).fill(false);

  function backtrack() {
    if (current.length === nums.length) {
      result.push([...current]);
      return;
    }
    for (let i = 0; i < nums.length; i++) {
      if (used[i]) continue;
      used[i] = true;
      current.push(nums[i]);
      backtrack();
      current.pop();
      used[i] = false;
    }
  }
  backtrack();
  return result;
}
```

**Приклади задач:**
- **Subsets / Subsets II** (з дублями).
- **Permutations / Permutations II**.
- **Combinations / Combination Sum / Combination Sum II**.
- **Palindrome Partitioning**.
- **Word Search** -- backtracking по grid.
- **N-Queens**.
- **Sudoku Solver**.
- **Generate Parentheses**.

**Часто разом з:**
- DFS.
- Pruning (відсікання поганих гілок).
- Сортування (для обробки дублікатів).

**Складність:**
- Time: O(n! * n) для перестановок, O(2ⁿ * n) для підмножин, варіює.
- Space: O(n) глибина рекурсії.

**Детальніше:** [05-recursion-backtracking.md](05-recursion-backtracking.md)

---

## 15. Dynamic Programming 1D

**Коли використовувати:**
- Задача має **оптимальну підструктуру** (оптимум цілого будується з оптимумів підзадач) і **перекриваючі підзадачі**.
- Стан можна описати **одним індексом** (найчастіше "результат для перших i елементів" або "результат, що закінчується в i").
- Тригерні слова: "minimum cost", "maximum value", "number of ways", "can we achieve", "longest increasing".

**Основна ідея:**

`dp[i]` -- відповідь для префікса або для позиції `i`. Переходи: `dp[i]` виражається через `dp[i-1]`, `dp[i-2]`, ... Часто можна оптимізувати пам'ять до O(1) -- тримати лише останні K значень.

```
Climbing Stairs: dp[i] = dp[i-1] + dp[i-2]

i:     0  1  2  3  4  5
dp[i]: 1  1  2  3  5  8    (Фібоначчі)
```

**Шаблонний код:**

```javascript
// Climbing Stairs -- з масивом
function climbStairs(n) {
  const dp = new Array(n + 1);
  dp[0] = 1; dp[1] = 1;
  for (let i = 2; i <= n; i++) {
    dp[i] = dp[i - 1] + dp[i - 2];
  }
  return dp[n];
}

// Те саме з O(1) space
function climbStairsOpt(n) {
  let a = 1, b = 1;
  for (let i = 2; i <= n; i++) {
    [a, b] = [b, a + b];
  }
  return b;
}

// House Robber: dp[i] = max(dp[i-1], dp[i-2] + nums[i])
function rob(nums) {
  let prev2 = 0, prev1 = 0;
  for (const x of nums) {
    [prev2, prev1] = [prev1, Math.max(prev1, prev2 + x)];
  }
  return prev1;
}
```

**Приклади задач:**
- **Climbing Stairs / Fibonacci** -- база.
- **House Robber** -- вибір з двох попередніх.
- **Maximum Subarray (Kadane)** -- `dp[i] = max(nums[i], dp[i-1] + nums[i])`.
- **Coin Change** -- `dp[amount]` = min coins.
- **Longest Increasing Subsequence** (O(n²) DP або O(n log n) з бінарним пошуком).
- **Decode Ways** -- кількість способів розшифрувати рядок.
- **Word Break** -- чи можна розбити на слова зі словника.

**Часто разом з:**
- Memoization (top-down) або tabulation (bottom-up).
- Space optimization (rolling variables).

**Складність:**
- Time: O(n) або O(n²). Space: O(n) -> часто O(1).

**Детальніше:** [09-dynamic-programming.md](09-dynamic-programming.md)

---

## 16. Dynamic Programming 2D

**Коли використовувати:**
- Стан залежить **від двох параметрів**: два рядки, матриця (рядок + стовпець), діапазон `[i..j]`, "i-й елемент, j-та кількість вибрано".
- Тригерні слова: "longest common subsequence", "edit distance", "unique paths", "knapsack", "palindrome substring", "matrix".

**Основна ідея:**

`dp[i][j]` -- оптимум для підзадачі з двома параметрами. Переходи зазвичай від сусідніх клітинок: `dp[i-1][j]`, `dp[i][j-1]`, `dp[i-1][j-1]`.

```
Longest Common Subsequence ("abc", "ac"):

       ""  a   c
   ""  0   0   0
   a   0   1   1
   b   0   1   1
   c   0   1   2     <- LCS = 2
```

**Шаблонний код:**

```javascript
// Longest Common Subsequence
function lcs(a, b) {
  const m = a.length, n = b.length;
  const dp = Array.from({ length: m + 1 }, () => new Array(n + 1).fill(0));

  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      if (a[i - 1] === b[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1] + 1;
      } else {
        dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
      }
    }
  }
  return dp[m][n];
}

// Unique Paths (m x n grid, тільки вправо/вниз)
function uniquePaths(m, n) {
  const dp = Array.from({ length: m }, () => new Array(n).fill(1));
  for (let i = 1; i < m; i++) {
    for (let j = 1; j < n; j++) {
      dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
    }
  }
  return dp[m - 1][n - 1];
}
```

**Приклади задач:**
- **Longest Common Subsequence**.
- **Edit Distance (Levenshtein)** -- три операції, мінімум.
- **Unique Paths / Unique Paths II** (з перешкодами).
- **Minimum Path Sum** (grid).
- **0/1 Knapsack** -- класика.
- **Longest Palindromic Substring** -- `dp[i][j]` = чи `s[i..j]` паліндром.
- **Interleaving String**.
- **Regular Expression Matching**.

**Часто разом з:**
- Space optimization: часто достатньо лише попереднього рядка -> O(n) замість O(m*n).

**Складність:**
- Time: O(m*n). Space: O(m*n) -> часто O(n).

**Детальніше:** [09-dynamic-programming.md](09-dynamic-programming.md)

---

## 17. Top K / Heap

**Коли використовувати:**
- Треба **K найбільших / найменших / найчастіших** елементів.
- Merge K sorted lists/arrays.
- Median of data stream.
- Scheduler із пріоритетом.
- Тригерні слова: "top K", "K largest", "K smallest", "K closest", "merge K", "K most frequent".

**Основна ідея:**

Heap (priority queue) -- структура, де insert і extract-min/max -- O(log n).

**Ключовий трюк для top K:** тримай **min-heap розміром K** для K найбільших. Коли новий елемент > top -- пробний, pop старий top. У кінці в heap -- K найбільших.

```
Топ 3 найбільших у [3,1,5,12,2,11]:

крок 1: heap=[3]
крок 2: heap=[1,3]
крок 3: heap=[1,3,5]            (розмір K = 3)
крок 4: 12 > top(1): pop, push -> heap=[3,12,5]
крок 5: 2 < top(3): пропускаємо
крок 6: 11 > top(3): pop, push -> heap=[5,12,11]

top K = [5, 11, 12]
```

**Шаблонний код:**

```javascript
// У JavaScript немає вбудованої heap -- доведеться написати або використати бібліотеку.
// Шаблон із гіпотетичним MinHeap:

function topKLargest(arr, k) {
  const heap = new MinHeap();          // підтримує push, peek, pop, size
  for (const x of arr) {
    heap.push(x);
    if (heap.size() > k) heap.pop();   // тримаємо рівно K
  }
  return heap.toArray();               // K найбільших
}

// K найчастіших елементів
function topKFrequent(arr, k) {
  const freq = new Map();
  for (const x of arr) freq.set(x, (freq.get(x) || 0) + 1);

  const heap = new MinHeap((a, b) => a[1] - b[1]);    // сортування за частотою
  for (const [val, cnt] of freq) {
    heap.push([val, cnt]);
    if (heap.size() > k) heap.pop();
  }
  return heap.toArray().map(([v]) => v);
}
```

**Альтернатива без heap:** для top K можна використати **Quickselect** -- O(n) в середньому. Але на співбесіді частіше вимагають heap-рішення за O(n log k).

**Приклади задач:**
- **Kth Largest Element in an Array**.
- **Top K Frequent Elements**.
- **K Closest Points to Origin**.
- **Merge K Sorted Lists** -- heap з поточних голів.
- **Find Median from Data Stream** -- два heap (max + min).
- **Task Scheduler**.

**Часто разом з:**
- Hash Map (для підрахунку частот).
- Custom comparator.

**Складність:**
- Time: O(n log k). Space: O(k).

**Детальніше:** [08-sorting-searching.md](08-sorting-searching.md)

---

## 18. Union-Find (Disjoint Set Union, DSU)

**Коли використовувати:**
- **Динамічна** зв'язність: "чи належать ці два елементи до одного компонента?" під час додавання ребер.
- Кількість компонент зв'язності.
- Детектування циклу в **неорієнтованому** графі.
- Kruskal's MST.
- Тригерні слова: "union", "connected components", "redundant connection", "accounts merge", "number of provinces".

**Основна ідея:**

Кожен елемент -- у певному множинному. Дві операції:
- `find(x)` -- який представник множини x?
- `union(x, y)` -- об'єднати дві множини.

З **path compression** + **union by rank** обидві операції -- майже O(1) амортизовано (inverse Ackermann).

```
Початок:  {1} {2} {3} {4} {5}       кожен сам у собі
union(1,2): {1,2} {3} {4} {5}
union(3,4): {1,2} {3,4} {5}
union(2,3): {1,2,3,4} {5}

find(4) = find(3) = find(2) = find(1) = 1
```

**Шаблонний код:**

```javascript
class UnionFind {
  constructor(n) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
    this.count = n;                            // кількість компонент
  }

  find(x) {
    if (this.parent[x] !== x) {
      this.parent[x] = this.find(this.parent[x]);  // path compression
    }
    return this.parent[x];
  }

  union(x, y) {
    const rx = this.find(x), ry = this.find(y);
    if (rx === ry) return false;              // вже в одній множині

    // union by rank: менше дерево чіпляємо до більшого
    if (this.rank[rx] < this.rank[ry]) this.parent[rx] = ry;
    else if (this.rank[rx] > this.rank[ry]) this.parent[ry] = rx;
    else { this.parent[ry] = rx; this.rank[rx]++; }

    this.count--;
    return true;
  }
}
```

**Приклади задач:**
- **Number of Connected Components** (undirected graph).
- **Number of Provinces**.
- **Redundant Connection** -- знайти ребро, що створює цикл.
- **Accounts Merge** -- union емейлів за власниками.
- **Graph Valid Tree** -- n-1 ребер + без циклу.
- **Kruskal's MST**.
- **Most Stones Removed**.

**Часто разом з:**
- Сортування (у Kruskal за вагою).
- Hash map (для mapping довільних ключів на індекси 0..n-1).

**Складність:**
- Time: O(alpha(n)) per operation ~ O(1).
- Space: O(n).

**Детальніше:** [07-graphs.md](07-graphs.md)

---

## 19. Topological Sort

**Коли використовувати:**
- **Directed Acyclic Graph (DAG)**: потрібен порядок, що поважає залежності.
- "Залежності між задачами", "порядок курсів", "послідовність кроків".
- Детектування циклу в **directed** графі.
- Тригерні слова: "order", "prerequisites", "schedule", "dependency", "course schedule".

**Основна ідея:**

Два підходи:

**Kahn's (BFS, in-degrees):**
1. Порахувати in-degree кожної вершини.
2. У queue -- усі з in-degree 0.
3. Виймаємо з queue -> додаємо до результату -> зменшуємо in-degree сусідів -> ті, що стали 0, додаємо в queue.
4. Якщо результат не містить усіх вершин -- у графі є цикл.

**DFS (post-order):**
1. DFS, у стек додаємо вершину після обробки всіх нащадків.
2. Результат -- reverse стеку.

```
Курси:  0 -> 1 -> 3
        0 -> 2 -> 3

in-degree: 0:0, 1:1, 2:1, 3:2
queue=[0] -> pop 0, ans=[0], dec 1,2 -> queue=[1,2]
pop 1, ans=[0,1], dec 3 -> queue=[2]
pop 2, ans=[0,1,2], dec 3 -> queue=[3]
pop 3, ans=[0,1,2,3]
```

**Шаблонний код:**

```javascript
// Kahn's algorithm
function topoSort(n, edges) {
  const graph = Array.from({ length: n }, () => []);
  const inDegree = new Array(n).fill(0);

  for (const [from, to] of edges) {
    graph[from].push(to);
    inDegree[to]++;
  }

  const queue = [];
  for (let i = 0; i < n; i++) if (inDegree[i] === 0) queue.push(i);

  const result = [];
  while (queue.length) {
    const node = queue.shift();
    result.push(node);
    for (const next of graph[node]) {
      if (--inDegree[next] === 0) queue.push(next);
    }
  }

  if (result.length !== n) return [];        // цикл -> немає валідного порядку
  return result;
}
```

**Приклади задач:**
- **Course Schedule** -- чи можна пройти всі курси?
- **Course Schedule II** -- видати порядок.
- **Alien Dictionary** -- порядок літер за словником.
- **Minimum Height Trees** -- trim leaves (варіація).
- **Parallel Courses** -- мінімальна кількість семестрів.

**Часто разом з:**
- BFS (Kahn's) або DFS (post-order).
- Graph adjacency list.

**Складність:**
- Time: O(V + E). Space: O(V + E).

**Детальніше:** [07-graphs.md](07-graphs.md)

---

## 20. Trie (Prefix Tree)

**Коли використовувати:**
- Швидкий пошук **слів за префіксом**.
- Autocomplete, спеллчекер.
- Словниковий пошук у grid (Word Search II).
- Тригерні слова: "prefix", "starts with", "autocomplete", "dictionary", "word break", "search words in a board".

**Основна ідея:**

Дерево, де кожне ребро -- літера. Шлях від кореня до вузла -- префікс. Прапорець `isEnd` відмічає повні слова.

```
Слова: "car", "cat", "cap"

         root
          |
          c
          |
          a
        / | \
       r  t  p
      *   *  *       (* = кінець слова)
```

**Шаблонний код:**

```javascript
class TrieNode {
  constructor() {
    this.children = new Map();
    this.isEnd = false;
  }
}

class Trie {
  constructor() { this.root = new TrieNode(); }

  insert(word) {
    let node = this.root;
    for (const c of word) {
      if (!node.children.has(c)) node.children.set(c, new TrieNode());
      node = node.children.get(c);
    }
    node.isEnd = true;
  }

  search(word) { return this._find(word, true); }
  startsWith(prefix) { return this._find(prefix, false); }

  _find(str, mustBeEnd) {
    let node = this.root;
    for (const c of str) {
      if (!node.children.has(c)) return false;
      node = node.children.get(c);
    }
    return mustBeEnd ? node.isEnd : true;
  }
}
```

**Приклади задач:**
- **Implement Trie** -- база.
- **Design Add and Search Words Data Structure** (з `.` як wildcard).
- **Word Search II** -- grid + trie зі словника, значно швидше за пошук кожного слова окремо.
- **Replace Words** -- autocomplete коренів.
- **Longest Word in Dictionary**.

**Часто разом з:**
- DFS / backtracking (Word Search II).
- Grid traversal.

**Складність:**
- Insert/search: O(L) де L -- довжина слова.
- Space: O(total characters in all words).

**Детальніше:** [06-trees.md](06-trees.md)

---

## 21. Greedy

**Коли використовувати:**
- Можна довести, що **локально оптимальний вибір на кожному кроці веде до глобального оптимуму**.
- Задачі інтервалів, scheduling.
- Коли DP здається занадто важким і можливе "жадібне" рішення.
- Тригерні слова: "minimum number", "maximum", "intervals", "meeting rooms", "jump game".

**Основна ідея:**

На кожному кроці -- робимо вибір, який здається найкращим **зараз**, без повернення і перегляду.

**Небезпека:** не всі задачі розв'язні жадібно. Якщо не впевнений -- треба довести коректність (або підібрати контрприклад). Типово greedy працює при наявності **exchange argument** або **matroid** структури.

**Шаблонний код (різні приклади):**

```javascript
// Activity Selection / Non-overlapping Intervals
function maxMeetings(intervals) {
  intervals.sort((a, b) => a[1] - b[1]);      // за часом закінчення
  let count = 0, end = -Infinity;
  for (const [s, e] of intervals) {
    if (s >= end) { count++; end = e; }       // беремо найраніше закінчення
  }
  return count;
}

// Jump Game: чи можна дістатися кінця?
function canJump(nums) {
  let maxReach = 0;
  for (let i = 0; i < nums.length; i++) {
    if (i > maxReach) return false;
    maxReach = Math.max(maxReach, i + nums[i]);
  }
  return true;
}

// Gas Station
function canCompleteCircuit(gas, cost) {
  let total = 0, tank = 0, start = 0;
  for (let i = 0; i < gas.length; i++) {
    const diff = gas[i] - cost[i];
    total += diff;
    tank += diff;
    if (tank < 0) { start = i + 1; tank = 0; }
  }
  return total >= 0 ? start : -1;
}
```

**Коли жадібне ПРАЦЮЄ (практичні сигнали):**
- Сортування вхідних даних створює структуру.
- Вибір "найбільшого/найменшого/найранішого" очевидно веде вперед.
- У задачі з DP-подібною структурою переходи надто "прості" (одновимірні).

**Коли жадібне НЕ працює -- бери DP:**
- Потрібен глобальний оптимум, а локальний вибір може "зарубати" кращий шлях (Coin Change з довільним набором монет -- greedy не завжди правильний).

**Приклади задач:**
- **Non-overlapping Intervals / Meeting Rooms II**.
- **Jump Game / Jump Game II**.
- **Gas Station**.
- **Partition Labels**.
- **Minimum Number of Arrows to Burst Balloons**.
- **Task Scheduler** (з heap).
- **Huffman Coding**.

**Часто разом з:**
- Sorting (по якомусь критерію).
- Heap (greedy з пріоритетом).

**Складність:**
- Зазвичай O(n log n) через sort, або O(n) після.
- Space: O(1) або O(n).

**Детальніше:** частково у кількох файлах, greedy прошиває багато тем.

---

## 22. Як розпізнати патерн за умовою задачі (тригерна таблиця)

Читаєш умову -- шукаєш ці сигнали. Перше, що резонує -- той патерн і пробуєш першим.

| Ознака в умові                                              | Патерн                               |
|-------------------------------------------------------------|--------------------------------------|
| "Відсортований масив" і пошук target                        | Binary Search                        |
| "Відсортований масив" і пара/трійка                         | Two Pointers                         |
| "Sorted" + merge двох                                       | Two Pointers                         |
| "Пара з сумою" / "Two Sum"                                  | Hash Map (невідсорт.) або Two Pointers (відсорт.) |
| "Трійка з сумою 0"                                          | Sort + Two Pointers                  |
| "Підмасив/підрядок фіксованої довжини K"                    | Fixed Sliding Window                 |
| "Найдовший/найкоротший підрядок з умовою"                   | Variable Sliding Window              |
| "Максимум у sliding window"                                 | Deque (monotonic)                    |
| "Subarray sum equals K" (з від'ємними)                      | Prefix Sum + Hash Map                |
| "Range sum queries"                                         | Prefix Sum                           |
| "Palindrome"                                                | Two Pointers, або DP 2D для substring |
| "Детектування циклу в linked list"                          | Floyd's (fast & slow)                |
| "Середина linked list"                                      | Fast & Slow Pointers                 |
| "Reverse linked list"                                       | Reverse in-place (3 pointers)        |
| "Наступний більший/менший елемент"                          | Monotonic Stack                      |
| "Histogram", "trapping rain water"                          | Monotonic Stack / Two Pointers       |
| "Дужки валідні"                                             | Stack                                |
| "Evaluate expression"                                       | Stack / recursion                    |
| "Level order" у дереві                                      | BFS                                  |
| "All paths" у дереві                                        | DFS + backtracking                   |
| "Validate BST"                                              | DFS in-order                         |
| "Max/min depth of tree"                                     | DFS або BFS                          |
| "Clone graph"                                               | DFS/BFS + Hash Map                   |
| "Number of islands" / grid connected components             | DFS або BFS                          |
| "Shortest path in unweighted graph"                         | BFS                                  |
| "Shortest path in weighted graph"                           | Dijkstra (heap) / Bellman-Ford       |
| "Order of tasks with dependencies"                          | Topological Sort                     |
| "Cycle in directed graph"                                   | DFS з кольорами або Kahn's           |
| "Union / dynamic connectivity"                              | Union-Find                           |
| "Minimum spanning tree"                                     | Union-Find (Kruskal) / heap (Prim)   |
| "All permutations / subsets / combinations"                 | Backtracking                         |
| "N-Queens / Sudoku"                                         | Backtracking + pruning               |
| "Generate parentheses"                                      | Backtracking                         |
| "Оптимальне значення через вибір" (min/max cost/ways)       | DP                                   |
| "Number of ways to..."                                      | DP (часто лічильне)                  |
| "Longest increasing / common subsequence"                   | DP (1D / 2D)                         |
| "Edit distance", "interleaving"                             | DP 2D                                |
| "Knapsack-подібне: вибір з обмеженням"                      | DP (0/1 knapsack)                    |
| "Decode ways", "word break"                                 | DP 1D                                |
| "Top K elements"                                            | Heap (min-heap розміру K)            |
| "K closest / K most frequent / Kth largest"                 | Heap або Quickselect                 |
| "Merge K sorted"                                            | Heap                                 |
| "Median of stream"                                          | Two heaps (max + min)                |
| "Prefix search / autocomplete"                              | Trie                                 |
| "Word search in grid зі словником"                          | Trie + DFS                           |
| "Intervals: overlap / merge / non-overlapping"              | Sort + Greedy / Sweep line           |
| "Meeting rooms"                                             | Sort + Heap або Sweep line           |
| "Jump game"                                                 | Greedy                               |
| "Gas station"                                               | Greedy                               |
| "Мінімальне X, що задовольняє Y (monotonic)"                | Binary Search on Answer              |
| "Find duplicate в масиві 1..n"                              | Floyd's на індексах / Hash Set       |
| "In-place видалення дублів у sorted array"                  | Two Pointers (slow + fast)           |
| "Rotate array/image"                                        | Reverse трюки / транспозиція         |
| "Matrix -- пошук у 2D sorted"                               | Binary Search / staircase (two ptrs) |
| "Найбільший/найменший за частотою"                          | Hash Map + Heap / bucket sort        |

**Підказка:** якщо нічого з таблиці не підходить -- спробуй переформулювати задачу. "Мінімальна кількість змін" часто = DP. "Чи існує послідовність" часто = BFS. "Всі варіанти" = backtracking.

---

## 23. Чекліст перед написанням коду на співбесіді

Коли інтерв'юер дав задачу -- **не хапайся писати код**. Пройдись по цьому чеклісту. Він сильно підвищує ймовірність правильного розв'язку і справляє враження структурного мислення.

### Крок 1: Розуміння задачі (1-2 хвилини)

1. **Перекажи умову своїми словами.**
   - "Отже, у мене масив чисел, і треба повернути..."
   - Це перевіряє твоє розуміння і дає інтерв'юеру шанс виправити тебе.

2. **Запитай про ввід.**
   - Розмір? (10? 10⁴? 10⁶? -- визначає допустиму складність)
   - Тип даних? (тільки позитивні? float? великі числа?)
   - Можуть бути дублікати?
   - Порожній ввід -- валідний?
   - Масив відсортований?

3. **Запитай про вивід.**
   - Повернути індекс, саме значення, чи boolean?
   - Якщо розв'язків кілька -- будь-який, чи всі, чи конкретно перший?

4. **Приведи приклад.**
   - Візьми маленький приклад і пройдись вручну.
   - "Для `[1,2,3]` і `target=5`, відповідь -- `[1,2]`, бо `2+3=5`."

### Крок 2: Edge cases (30 секунд)

Назви вголос:
- Порожній ввід (`[]`, `""`, `null`).
- Один елемент.
- Усі однакові.
- Усі унікальні.
- Від'ємні числа.
- Дуже великий ввід (границя).
- Дублікати (якщо можливі).
- Відсортований / зворотно-відсортований.

Це **не** означає, що треба всі обробити прямо зараз -- але згадати треба.

### Крок 3: Brute force (1 хвилина)

1. **Придумай "в лоб" рішення.**
   - Перебір усіх пар? Рекурсивний перебір?
2. **Оціни складність.**
   - "Це O(n²) time, O(1) space."
3. **Скажи це вголос.**
   - "Найпростіше -- вкладені цикли, O(n²). Чи цього достатньо, чи треба краще?"

Іноді інтерв'юеру brute force -- вже достатньо (наприклад, n ≤ 100). Але зазвичай він скаже "можеш краще?".

### Крок 4: Оптимізація (2-3 хвилини)

1. **Визнач bottleneck** brute force.
   - "Я двічі перевіряю кожну пару -- чи можна запам'ятовувати?"
2. **Пригадай патерни** з тригерної таблиці вище.
   - Hash map для O(1) lookup?
   - Sorting + two pointers?
   - Sliding window?
   - DP замість рекурсії?
3. **Обміняй час на пам'ять** (typical trade-off).
   - Додай Map / Set / heap.
4. **Сформулюй новий підхід і складність.**
   - "Можу за O(n) time, O(n) space через hash map."
5. **Запитай схвалення.**
   - "Це звучить добре? Я пишу?"

### Крок 5: Код (5-10 хвилин)

1. **Спочатку оголоси сигнатуру і структури.**
   ```javascript
   function solve(arr, target) {
     const seen = new Map();
     // ...
   }
   ```
2. **Пиши охайно, проговорюючи вголос.**
   - Інтерв'юер хоче чути хід думки.
3. **Імена змінних -- осмислені.** `left`, `right`, `maxLen`, а не `x`, `y`, `z`.
4. **Обробляй edge cases на початку** (або свідомо скажи, чому не обробляєш).
5. **Не копіпастій** -- на дошці це видно.

### Крок 6: Тестування (2-3 хвилини)

1. **Прогони свій код на прикладі вручну.**
   - Побудуй таблицю стану змінних.
2. **Перевір edge cases.**
   - `[]` -> повертає правильне?
   - Один елемент?
   - Дублікати?
3. **Перевір off-by-one.**
   - Цикл `< length` чи `<= length - 1`?
   - `mid = (L + R) / 2` чи `L + (R - L) / 2` (overflow safety)?
4. **Знайшов баг? Не панікуй.**
   - "О, бачу помилку -- ось тут..." -- спокійно виправ.

### Крок 7: Аналіз і обговорення (1-2 хвилини)

1. **Проговори фінальну складність.**
   - Time: O(?). Space: O(?).
2. **Згадай можливі оптимізації.**
   - "Пам'ять можна скоротити до O(1), якщо..."
3. **Обговори trade-offs.**
   - "Я обрав hash map для O(1) lookup -- це коштує O(n) пам'яті."
4. **Follow-up питання -- будь готовий.**
   - "А якщо масив не вміщається в пам'ять?" -> зовнішнє сортування, streaming.
   - "А якщо multithreading?" -> структури, безпечні до race conditions.

### Золоті правила

- **Ніколи не мовчи.** Проговорюй усе. Навіть "я подумаю 30 секунд -- можна?".
- **Не ображайся на підказки.** Інтерв'юер помічає, як ти реагуєш.
- **Чесно визнавай, коли застряг.** "Чесно, я не бачу оптимізації. Що я пропускаю?" -- краще, ніж мовчання.
- **Не пиши код одразу -- спочатку план.** Перший написаний код без плану -- майже завжди з помилками.
- **Не оптимізуй заздалегідь.** Спершу робоче рішення, потім швидке.
- **Test-driven mindset.** Один мінімальний приклад + один edge case -- і ти побачиш 80% багів.

---

## 24. Фінальна шпаргалка

### Патерни за типом задачі

| Тип задачі                             | Головні патерни                           |
|----------------------------------------|--------------------------------------------|
| Масиви / рядки                         | Two Pointers, Sliding Window, Prefix Sum   |
| Швидкий пошук                          | Hash Map / Set, Binary Search              |
| Linked lists                           | Fast & Slow, Reverse, Dummy node           |
| Stack / queue                          | Monotonic Stack, Deque                     |
| Дерева                                 | DFS, BFS, Trie                             |
| Графи                                  | DFS, BFS, Union-Find, Topo Sort, Dijkstra  |
| Пошук "всіх варіантів"                 | Backtracking                               |
| Оптимум / лічба способів               | DP (1D, 2D)                                |
| Top K / потоки                         | Heap                                       |
| Intervals                              | Sort + Greedy / Sweep line                 |
| Min/max значення з умовою              | Binary Search on Answer                    |

### Типовий набір структур в арсеналі

1. `Map` / `Set` -- O(1) lookup, підрахунок частот.
2. `Array` -- індексований доступ, sliding window, DP табличка.
3. Stack / Queue / Deque -- структурні обходи.
4. Heap -- top K, priority.
5. Trie -- префіксний пошук.
6. Union-Find -- динамічна зв'язність.
7. Лінковані списки -- коли є у вхідних даних.

### Стандартні "трюки" підвищення продуктивності

- **Hash map для O(1) пошуку пари** -> Two Sum.
- **Sort + два вказівники** -> 3Sum, closest pair.
- **Sliding window замість перебору підмасивів** -> O(n) замість O(n²).
- **Prefix sum замість повторних обчислень сум**.
- **Monotonic stack замість "для кожного i шукати next greater" в лоб**.
- **BFS замість DFS** для найкоротшого шляху unweighted.
- **Heap розміру K** замість повного сортування для top K.
- **Binary search on answer** замість перебору можливих відповідей.
- **DP з мемоізацією** замість рекурсії без кешу (`fib` без мемо -- O(2ⁿ)).
- **Bit manipulation / bitmask DP** для підмножин малого розміру (n ≤ 20).

### Коли що НЕ спрацює

- `unshift`, `shift`, `splice` у циклі -> O(n²) замість очікуваного O(n). Уникай.
- `str += x` у циклі -> O(n²). Збирай у масив + `join`.
- `arr.includes(x)` у циклі -> O(n²). Заміни на `Set`.
- Рекурсія Fibonacci без мемо -> O(2ⁿ).
- Greedy там, де потрібна DP (coin change з довільним набором).
- Вкладені цикли по двох масивах, коли один можна перевести в Map.

---

Цей файл -- стискання всієї алгоритмічної підготовки в один довідник. Якщо перед співбесідою у тебе 1 день -- прочитай його від початку до кінця. Якщо 1 тиждень -- розподіли патерни по днях і за кожним проходь 2-3 задачі з LeetCode. Якщо 1 місяць -- вчи послідовно файли 01-09, а цей тримай як карту.

Удачі на співбесідах. Головне -- **думай вголос і не бійся питати**. Інтерв'юер -- союзник, а не екзаменатор.
