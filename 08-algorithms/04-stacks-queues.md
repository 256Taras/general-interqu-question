# Stacks & Queues: стеки та черги

## Що таке stack (LIFO)?

**Stack (стек)** -- це структура даних, де елементи додаються і видаляються **з одного кінця**. Працює за принципом **LIFO**: **L**ast **I**n, **F**irst **O**ut -- "останній зайшов, перший вийшов".

Аналогія: стопка тарілок. Коли миєш посуд, нову тарілку кладеш зверху; коли береш -- теж зверху. Нижні тарілки лежать, поки не прибереш усі верхні.

```
Операції стеку:

push(A):        push(B):        push(C):        pop() → C:
┌───┐           ┌───┐           ┌───┐           ┌───┐
│   │           │   │           │ C │ ◀── top   │   │
├───┤           ├───┤           ├───┤           ├───┤
│   │           │ B │ ◀── top   │ B │           │ B │ ◀── top
├───┤           ├───┤           ├───┤           ├───┤
│ A │ ◀── top   │ A │           │ A │           │ A │
└───┘           └───┘           └───┘           └───┘
```

Базові операції:
- **push(x)** -- додати на верх. O(1).
- **pop()** -- зняти з верху і повернути. O(1).
- **peek() / top()** -- подивитись на верх, не знімаючи. O(1).
- **isEmpty()** -- чи порожній. O(1).

### Реальні приклади

1. **Call stack** у будь-якій мові програмування. Кожен виклик функції "push-иться" в стек. Коли функція повертається -- pop. `stack overflow` -- це буквально переповнення цього стеку.

2. **Undo/Redo** в редакторах. Кожна дія -> push у undo-стек. `Ctrl+Z` -> pop з undo, push у redo. `Ctrl+Y` -> навпаки.

3. **Парсинг дужок.** Коли бачиш `(`, push. Коли `)` -- pop і перевір, чи пара збігається.

4. **Backtracking.** DFS, розв'язки N-queens, sudoku -- все це стек (явний або через рекурсію).

5. **Browser history.** Кнопка "назад" -- класичний стек.

---

## Що таке queue (FIFO)?

**Queue (черга)** -- елементи додаються з одного боку ("задній"/rear), а видаляються з іншого ("передній"/front). Принцип **FIFO**: **F**irst **I**n, **F**irst **O**ut -- "перший зайшов, перший вийшов".

Аналогія: черга в магазині. Хто прийшов першим -- того першим і обслуговують.

```
front                                           rear
  │                                               │
  ▼                                               ▼
 [A] ─── [B] ─── [C] ─── [D]
  ↑                       ↑
  dequeue()              enqueue(x)
  звідси виймаємо        сюди додаємо
```

Базові операції:
- **enqueue(x)** -- додати в хвіст. O(1).
- **dequeue()** -- видалити з голови, повернути. O(1).
- **peek() / front()** -- подивитись на голову. O(1).
- **isEmpty()** -- O(1).

### Реальні приклади

1. **BFS (breadth-first search)** у графах і деревах. Використовує чергу для обходу "по рівнях".

2. **Task queue / Job queue.** Redis + Bull, RabbitMQ, SQS -- усі це черги. Продюсер кладе задачу, консюмер бере.

3. **Throttling / Rate limiting.** Токен-черга для обмеження кількості запитів.

4. **Event loop в Node.js.** Macrotask queue, microtask queue -- усе це черги. `setTimeout` кидає в macrotask queue, `Promise.then` -- у microtask.

5. **Print queue, message queue** -- будь-яка обробка "в порядку надходження".

---

## Реалізація на JS

### Stack через масив

Масив у JS ідеально підходить для стеку: `push` і `pop` -- **O(1) amortized**.

```javascript
class Stack {
  constructor() {
    this.items = [];
  }

  push(x) {
    this.items.push(x);
  }

  pop() {
    return this.items.pop();   // undefined якщо порожньо
  }

  peek() {
    return this.items[this.items.length - 1];
  }

  isEmpty() {
    return this.items.length === 0;
  }

  get size() {
    return this.items.length;
  }
}

// Використання:
const s = new Stack();
s.push(1); s.push(2); s.push(3);
s.pop();          // 3
s.peek();         // 2
```

На практиці в алгоритмічних задачах часто пишуть просто:
```javascript
const stack = [];
stack.push(x);        // як push у стек
stack.pop();          // як pop
stack[stack.length - 1];  // peek
```

Обгортати в клас не обов'язково -- масив вже дає потрібне API. Але клас читабельніший у продакшн-коді.

### Queue через масив -- пастка!

Природно хочеться:
```javascript
class NaiveQueue {
  constructor() { this.items = []; }
  enqueue(x) { this.items.push(x); }       // O(1) -- ок
  dequeue() { return this.items.shift(); } // O(n) -- ❌
}
```

Проблема: **`arr.shift()` -- O(n)**, бо треба зсунути всі елементи вліво.

Якщо ти робиш n операцій enqueue + n операцій dequeue, сумарно це **O(n²)**. На n = 100 000 -- сотні мільйонів операцій, таймаут гарантовано.

```
Масив [A, B, C, D, E]

shift():
  Позиція 0 звільняється.
  Треба зсунути B→0, C→1, D→2, E→3.
  Це O(n).
```

На співбесіді це один з головних "червоних прапорців" -- `shift` у гарячому циклі. Питання "як оптимізувати цю чергу?" задають постійно.

### Ефективна черга через два вказівники (масив + head index)

Найпростіший спосіб уникнути `shift` -- не зсувати елементи, а тримати індекс голови:

```javascript
class Queue {
  constructor() {
    this.items = [];
    this.head = 0;      // індекс наступного для dequeue
  }

  enqueue(x) {
    this.items.push(x);
  }

  dequeue() {
    if (this.head >= this.items.length) return undefined;
    const x = this.items[this.head];
    this.items[this.head] = undefined;   // звільнимо посилання для GC
    this.head++;

    // Раз на деякий час "стискаємо" масив, щоб не росла пам'ять
    if (this.head > 50 && this.head * 2 > this.items.length) {
      this.items = this.items.slice(this.head);
      this.head = 0;
    }

    return x;
  }

  peek() {
    return this.items[this.head];
  }

  get size() {
    return this.items.length - this.head;
  }
}
```

`dequeue` тепер **amortized O(1)**. Це найпопулярніший промислової-якості підхід у JS (так робить багато бібліотек).

### Ефективна черга через linked list

Linked list дає O(1) на обох кінцях природно:

```javascript
class Node {
  constructor(value) { this.value = value; this.next = null; }
}

class LinkedQueue {
  constructor() {
    this.head = null;
    this.tail = null;
    this.length = 0;
  }

  enqueue(value) {
    const node = new Node(value);
    if (!this.tail) {
      this.head = node;
      this.tail = node;
    } else {
      this.tail.next = node;
      this.tail = node;
    }
    this.length++;
  }

  dequeue() {
    if (!this.head) return undefined;
    const removed = this.head;
    this.head = this.head.next;
    if (!this.head) this.tail = null;
    this.length--;
    return removed.value;
  }

  peek() {
    return this.head ? this.head.value : undefined;
  }
}
```

`enqueue` і `dequeue` -- обидва справжні O(1). Але пам'яті більше (на кожен вузол +указівник), і немає кеш-локальності масиву.

### Queue через два стеки (класична співбесідна задача)

```javascript
class QueueFromStacks {
  constructor() {
    this.inStack = [];
    this.outStack = [];
  }

  enqueue(x) {
    this.inStack.push(x);
  }

  dequeue() {
    if (this.outStack.length === 0) {
      while (this.inStack.length > 0) {
        this.outStack.push(this.inStack.pop());
      }
    }
    return this.outStack.pop();
  }
}
```

Як це працює:
```
enqueue 1, 2, 3:
  inStack:  [1, 2, 3]        (top=3)
  outStack: []

dequeue():
  -- переливаємо все з in у out (перевертаємо порядок)
  inStack:  []
  outStack: [3, 2, 1]        (top=1)
  -- pop з out → 1 ✔
  outStack: [3, 2]

enqueue 4:
  inStack:  [4]
  outStack: [3, 2]

dequeue():
  outStack не порожній → pop → 2 ✔
```

Кожен елемент "переливається" максимум один раз. **Amortized O(1)** на операцію.

Питання на співбесіді: "чому amortized, а не worst case?". Тому що конкретний `dequeue` може зробити n pushes у outStack. Але кожен елемент переливається лише один раз за все життя, тому в середньому -- O(1).

---

## Deque (double-ended queue)

**Deque** (вимовляється "дек") -- гібрид стеку і черги: можна додавати і видаляти з **обох кінців** за O(1).

```
   front                   rear
     │                       │
     ▼                       ▼
     ◀── [A] [B] [C] [D] ──▶
   addFront                addRear
   removeFront             removeRear
```

Операції: `addFront`, `addRear`, `removeFront`, `removeRear`, `peekFront`, `peekRear`.

В JS **немає вбудованого Deque**. Реалізують:
- Через doubly linked list (природно O(1) на обох кінцях).
- Через circular buffer / масив з двома індексами.

Для алгоритмічних задач часто достатньо масиву з вмінням "ігнорувати" один кінець, але "справжній" deque потрібен для:

1. **Sliding Window Maximum** (побачимо далі).
2. **Undo/Redo з обмеженим розміром** (коли старіші дії треба викидати з іншого кінця).
3. **Work-stealing scheduler** (один тред бере з одного кінця, інший з іншого).
4. **Monotonic deque** в алгоритмах (різновид monotonic stack, але з двох кінців).

Мінімальна реалізація через doubly linked list:

```javascript
class DequeNode {
  constructor(value) {
    this.value = value;
    this.next = null;
    this.prev = null;
  }
}

class Deque {
  constructor() {
    this.head = null;
    this.tail = null;
    this.length = 0;
  }

  addRear(value) {
    const node = new DequeNode(value);
    if (!this.tail) {
      this.head = node;
      this.tail = node;
    } else {
      node.prev = this.tail;
      this.tail.next = node;
      this.tail = node;
    }
    this.length++;
  }

  addFront(value) {
    const node = new DequeNode(value);
    if (!this.head) {
      this.head = node;
      this.tail = node;
    } else {
      node.next = this.head;
      this.head.prev = node;
      this.head = node;
    }
    this.length++;
  }

  removeRear() {
    if (!this.tail) return undefined;
    const v = this.tail.value;
    this.tail = this.tail.prev;
    if (this.tail) this.tail.next = null;
    else this.head = null;
    this.length--;
    return v;
  }

  removeFront() {
    if (!this.head) return undefined;
    const v = this.head.value;
    this.head = this.head.next;
    if (this.head) this.head.prev = null;
    else this.tail = null;
    this.length--;
    return v;
  }
}
```

Усі операції O(1).

---

## Priority Queue / Heap (короткий ввід)

**Priority Queue** -- черга, де елементи виходять не в порядку додавання, а за **пріоритетом** (наприклад, найменше число першим).

Реалізується через **heap (купу)** -- спеціальне бінарне дерево, де батько завжди ≤ (min-heap) або ≥ (max-heap) за дітей.

```
Min-heap:           Max-heap:
    1                   9
   / \                 / \
  3   5               7   8
 / \                 / \
4   6               3   2
```

Операції:
- **insert(x)** -- O(log n).
- **extractMin/Max()** -- O(log n).
- **peek()** -- O(1).

### Use cases

1. **Dijkstra's algorithm** -- знаходження найкоротшого шляху.
2. **A\* search**.
3. **Top-K елементів** -- "знайти k найбільших у потоці" розв'язується через heap розміру k за O(n log k).
4. **Event simulation** -- подія з найближчим часом виконання йде першою.
5. **Task scheduling** за пріоритетом.
6. **Median maintenance** -- два хіпи (min + max).

У JS **немає вбудованого heap** (сором'язлива обставина). Пишуть руками або беруть бібліотеку (`heap-js`, `mnemonist`).

Детальніше про heap -- у файлі з сортуваннями (там є heapsort) або в окремому матеріалі з дерев.

---

## Pattern: Monotonic Stack

**Монотонний стек** -- стек, у якому елементи завжди впорядковані (строго зростають або строго спадають зверху вниз). Коли новий елемент порушує порядок, ми **виштовхуємо (pop)** елементи, що не відповідають, перед тим як push-нути новий.

Це один з найчастіших patterns на співбесідах на позиції senior. Розв'язує цілий клас задач "знайти найближчий більший/менший елемент".

**Основна ідея:**
```
Ми йдемо по масиву. Для кожного елемента нам треба знайти наступний
більший (або менший) елемент справа.

Тримаємо стек індексів, де значення ЗРОСТАЮТЬ знизу вгору (нестрого)
АБО СПАДАЮТЬ (залежно від задачі).

Коли приходить новий елемент x:
  - Поки на верхівці стеку елемент МЕНШИЙ за x (для "next greater")
    → pop. Для цього popped елемента наступний більший = x.
  - Push x.
```

### Приклад 1: Next Greater Element

Задача: для кожного елемента масиву знайти наступний більший праворуч. Якщо немає -- -1.

```
arr:    [2, 1, 2, 4, 3, 1]
result: [4, 2, 4, -1, -1, -1]
```

Наївно: для кожного i йдемо j = i+1..n, шукаємо перше arr[j] > arr[i]. O(n²).

Монотонний стек: O(n).

```javascript
function nextGreaterElement(arr) {
  const result = new Array(arr.length).fill(-1);
  const stack = [];   // індекси елементів, що ще чекають на відповідь

  for (let i = 0; i < arr.length; i++) {
    // Поки верх стека має елемент, МЕНШИЙ за поточний -- ми знайшли
    // для нього відповідь
    while (stack.length > 0 && arr[stack[stack.length - 1]] < arr[i]) {
      const idx = stack.pop();
      result[idx] = arr[i];
    }
    stack.push(i);
  }

  return result;
}
```

Прогонимо на `[2, 1, 2, 4, 3, 1]`:
```
i=0, arr[i]=2:  stack порожній     → push 0  → stack:[0]
i=1, arr[i]=1:  arr[0]=2 > 1       → нічого → push 1 → stack:[0,1]
i=2, arr[i]=2:  arr[1]=1 < 2       → pop 1, result[1]=2
                arr[0]=2 < 2? НІ   → стоп    → push 2 → stack:[0,2]
i=3, arr[i]=4:  arr[2]=2 < 4       → pop 2, result[2]=4
                arr[0]=2 < 4       → pop 0, result[0]=4
                стек порожній      → push 3 → stack:[3]
i=4, arr[i]=3:  arr[3]=4 > 3       → нічого → push 4 → stack:[3,4]
i=5, arr[i]=1:  arr[4]=3 > 1       → нічого → push 5 → stack:[3,4,5]

Кінець. Стек:[3,4,5] -- ті, для кого не знайшовся більший, result=-1.

result = [4, 2, 4, -1, -1, -1] ✔
```

**Чому O(n)?** Кожен індекс заходить у стек рівно один раз і виходить максимум один раз. Разом -- 2n операцій.

### Приклад 2: Daily Temperatures

Задача: масив температур за дні. Повернути масив, де на позиції i стоїть кількість днів, які треба чекати до першої теплішої температури. Якщо не буде -- 0.

```
T:      [73, 74, 75, 71, 69, 72, 76, 73]
result: [ 1,  1,  4,  2,  1,  1,  0,  0]
```

Те саме, але зберігаємо різницю індексів:

```javascript
function dailyTemperatures(T) {
  const result = new Array(T.length).fill(0);
  const stack = [];

  for (let i = 0; i < T.length; i++) {
    while (stack.length > 0 && T[stack[stack.length - 1]] < T[i]) {
      const idx = stack.pop();
      result[idx] = i - idx;
    }
    stack.push(i);
  }

  return result;
}
```

**Big O:** O(n) time, O(n) space.

### Приклад 3: Largest Rectangle in Histogram

Це "важка" задача на LeetCode, але через monotonic stack вирішується за O(n). Класика FAANG.

Задача: дано масив висот стовпців гістограми. Знайти площу найбільшого прямокутника, що поміщається всередині.

```
Heights: [2, 1, 5, 6, 2, 3]

          █
          █
    █ █   █
    █ █   █
█   █ █ █ █
█   █ █ █ █
0 1 2 3 4 5

Відповідь: 10 (прямокутник висотою 5 або 6 на двох стовпцях 2-3 → 5*2=10).
Насправді навіть перевіримо: висота 5, ширина 2 → 10. Чи висота 2 (мінімум), ширина 5 → 10. Тобто відповідь = 10.
```

**Ідея:** для кожного стовпця питаємо -- якщо цей стовпець є НАЙКОРОТШИМ у прямокутнику, то як далеко вліво і вправо можна розтягнути прямокутник?

Щоб відповісти швидко, треба для кожного i знайти:
- `left[i]` -- найближчий індекс зліва, де висота менша за arr[i].
- `right[i]` -- те саме справа.

Тоді ширина = `right[i] - left[i] - 1`, площа = `arr[i] * width`.

Обидва `left` і `right` знаходяться монотонним стеком за O(n).

```javascript
function largestRectangle(heights) {
  const n = heights.length;
  const stack = [];
  let maxArea = 0;

  // Трюк: додаємо sentinel 0 у кінець, щоб спорожнити стек
  for (let i = 0; i <= n; i++) {
    const h = i === n ? 0 : heights[i];

    while (stack.length > 0 && heights[stack[stack.length - 1]] > h) {
      const top = stack.pop();
      const height = heights[top];
      const leftBoundary = stack.length === 0 ? -1 : stack[stack.length - 1];
      const width = i - leftBoundary - 1;
      maxArea = Math.max(maxArea, height * width);
    }

    stack.push(i);
  }

  return maxArea;
}
```

**Ідея сенсорної -- "коли елемент виходить зі стеку, ми знаємо його межі":**
- Ліва межа -- попередній елемент у стеку (або -1).
- Права межа -- поточний індекс i.

**Big O:** O(n) time, O(n) space.

Ця задача -- стандартна для перевірки розуміння стеків. Якщо вирішив її -- розумієш monotonic stack добре.

---

## Pattern: Stack для parsing

### Valid Parentheses

Задача: дано рядок з `()[]{}`. Перевірити, чи всі дужки закриті в правильному порядку.

```
"()[]{}"   → true
"(]"       → false
"([{}])"   → true
"([)]"     → false
```

**Ідея:** при відкривальній дужці -- push. При закривальній -- pop і перевірити пару.

```javascript
function isValid(s) {
  const stack = [];
  const pairs = { ')': '(', ']': '[', '}': '{' };

  for (const ch of s) {
    if (ch === '(' || ch === '[' || ch === '{') {
      stack.push(ch);
    } else {
      // Закривальна -- має бути відповідна відкривальна на верху стеку
      if (stack.pop() !== pairs[ch]) return false;
    }
  }

  return stack.length === 0;   // не залишилось невідкритих
}
```

Приклад `"([{}])"`:
```
ch='(': push → stack: [(]
ch='[': push → stack: [(, []
ch='{': push → stack: [(, [, {]
ch='}': pop { == pairs['}'] → ok
ch=']': pop [ == pairs[']'] → ok
ch=')': pop ( == pairs[')'] → ok
Кінець, стек порожній → true ✔
```

**Big O:** O(n) time, O(n) space.

### Evaluate Reverse Polish Notation (RPN)

RPN -- це постфіксний запис, де оператор йде ПІСЛЯ операндів. `"3 4 +"` = 7. `"2 1 + 3 *"` = (2+1)*3 = 9.

```
tokens = ["2", "1", "+", "3", "*"]
→ push 2 → stack: [2]
→ push 1 → stack: [2, 1]
→ + → pop 1, pop 2, push 2+1=3 → stack: [3]
→ push 3 → stack: [3, 3]
→ * → pop 3, pop 3, push 3*3=9 → stack: [9]
Результат: 9
```

```javascript
function evalRPN(tokens) {
  const stack = [];
  const ops = {
    '+': (a, b) => a + b,
    '-': (a, b) => a - b,
    '*': (a, b) => a * b,
    '/': (a, b) => Math.trunc(a / b),   // цілочисельне ділення
  };

  for (const t of tokens) {
    if (t in ops) {
      const b = stack.pop();
      const a = stack.pop();
      stack.push(ops[t](a, b));
    } else {
      stack.push(Number(t));
    }
  }

  return stack[0];
}
```

**Big O:** O(n) time, O(n) space.

**Важливо:** порядок `b = pop; a = pop;` критичний для `-` і `/`. Перший pop -- правий операнд.

### Simplify Path (Unix path)

Задача: канонізувати Unix-шлях. `"/a/./b/../../c/"` → `"/c"`.

Правила:
- `.` -- поточна тека (ігноруємо).
- `..` -- вийти на рівень вище (pop зі стеку).
- Подвоєні `/` -- ігноруємо.

```javascript
function simplifyPath(path) {
  const parts = path.split('/');
  const stack = [];

  for (const p of parts) {
    if (p === '' || p === '.') continue;
    if (p === '..') {
      stack.pop();   // підніматись вище, якщо можна
    } else {
      stack.push(p);
    }
  }

  return '/' + stack.join('/');
}
```

Приклад:
```
"/a/./b/../../c/"
parts = ['', 'a', '.', 'b', '..', '..', 'c', '']

'' skip
'a' push → [a]
'.' skip
'b' push → [a, b]
'..' pop → [a]
'..' pop → []
'c' push → [c]
'' skip

result = "/c"  ✔
```

**Big O:** O(n) time, O(n) space.

---

## Pattern: Queue для BFS

**BFS (breadth-first search)** -- обхід "по рівнях" у дереві чи графі. Черга тримає "фронтир" -- вузли, які ми вже бачили, але ще не обробили.

Псевдокод:
```javascript
function bfs(start) {
  const queue = [start];
  const visited = new Set([start]);

  while (queue.length > 0) {
    const node = queue.shift();        // ❌ O(n)! Використай справжню чергу
    // обробити node
    for (const neighbor of node.neighbors) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor);
      }
    }
  }
}
```

Для великих графів -- обов'язково справжня O(1) черга (індекс голови або linked list).

BFS використовується для:
- Найкоротшого шляху в unweighted графі.
- Level-order traversal дерева.
- Web crawler (відвідуємо сторінки "по рівнях глибини").
- Social network degrees of separation.

Детально про BFS -- у файлі з графами.

---

## Sliding Window Maximum через Deque

Класична задача: даний масив і вікно розміру k. Для кожної позиції вікна вивести максимум у вікні.

```
nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3
вікна:
  [1, 3, -1]              max = 3
     [3, -1, -3]          max = 3
        [-1, -3, 5]       max = 5
            [-3, 5, 3]    max = 5
                [5, 3, 6] max = 6
                   [3, 6, 7] max = 7

result = [3, 3, 5, 5, 6, 7]
```

Наївно: для кожного вікна знаходити max → O(n * k). Для n=10⁵ і k=10⁴ це 10⁹ -- таймаут.

**Deque-рішення -- O(n).**

**Ідея:** тримаємо deque **індексів**, де значення спадають (monotonic decreasing). Голова deque -- завжди індекс максимуму поточного вікна.

Правила:
1. Перед додаванням нового i -- виштовхуємо з хвоста всі індекси, де значення ≤ arr[i] (вони більше не можуть бути максимумом).
2. Якщо голова вилетіла з вікна (індекс < i - k + 1) -- видаляємо її.
3. Коли i ≥ k - 1 -- пишемо результат arr[deque.front()].

```javascript
function maxSlidingWindow(nums, k) {
  const result = [];
  const deque = [];   // зберігає індекси, значення спадають

  for (let i = 0; i < nums.length; i++) {
    // 1. Викинути з хвоста всі менші або рівні
    while (deque.length > 0 && nums[deque[deque.length - 1]] <= nums[i]) {
      deque.pop();
    }
    deque.push(i);

    // 2. Перевірити, чи голова ще у вікні
    if (deque[0] <= i - k) {
      deque.shift();   // у реальному коді -- справжній deque без O(n) shift
    }

    // 3. Записати результат, коли перше вікно сформоване
    if (i >= k - 1) {
      result.push(nums[deque[0]]);
    }
  }

  return result;
}
```

Прохід на `[1, 3, -1, -3, 5, 3, 6, 7]`, k=3:
```
i=0, nums[0]=1:
  deque: [0]  (значення 1)
i=1, nums[1]=3:
  nums[0]=1 ≤ 3 → pop → deque: []
  deque: [1]  (значення 3)
i=2, nums[2]=-1:
  nums[1]=3 > -1 → не чіпаємо
  deque: [1, 2]  (значення 3, -1)
  i=2 ≥ k-1=2 → result = [3]

i=3, nums[3]=-3:
  nums[2]=-1 > -3 → не чіпаємо
  deque: [1, 2, 3]  (значення 3, -1, -3)
  head=1, i-k=0, 1 > 0 → ок
  result = [3, 3]

i=4, nums[4]=5:
  nums[3]=-3 ≤ 5 → pop → [1, 2]
  nums[2]=-1 ≤ 5 → pop → [1]
  nums[1]=3 ≤ 5 → pop → []
  deque: [4]  (значення 5)
  result = [3, 3, 5]

...і так далі.
```

**Big O:** O(n) -- кожен індекс заходить і виходить з deque максимум один раз. Space O(k).

**Важливо:** у коді вище `.shift()` на масиві це O(n). Для справжнього O(n) алгоритму треба:
- Або використати linked list / справжній deque.
- Або тримати індекс голови окремо і не викликати `.shift()`.

На співбесіді можна проговорити: "Я використовую масив як deque для читаемості, але в продакшн-коді взяв би справжню структуру".

---

## Типові пастки

### 1. `arr.shift()` у великій черзі

```javascript
// ❌ O(n²) сумарно при n операціях
const queue = [];
for (let i = 0; i < 100000; i++) queue.push(i);
while (queue.length > 0) {
  const x = queue.shift();   // O(n) щоразу
  process(x);
}
```

Той самий код з індексом голови -- O(n). Різниця у секундах → часи виконання.

### 2. Не перевіряти, чи стек не порожній

```javascript
// ❌ TypeError при "()"
function isValid(s) {
  const stack = [];
  for (const ch of s) {
    if (ch === ')') {
      if (stack.pop() !== '(') return false;   // pop() на порожньому → undefined
    }
    ...
  }
}

// Це насправді не впаде (undefined !== '(' → true, return false).
// Але для більш складних кейсів -- так.
```

Правильно -- явна перевірка:
```javascript
if (stack.length === 0 || stack.pop() !== pairs[ch]) return false;
```

### 3. Забути перевернути порядок операндів у RPN

```javascript
// ❌ для "-" і "/" порядок має значення
const a = stack.pop();
const b = stack.pop();
stack.push(a - b);   // ПОМИЛКА: a -- правий операнд, має бути b - a

// ✅
const b = stack.pop();   // правий
const a = stack.pop();   // лівий
stack.push(a - b);
```

### 4. Плутати stack і queue

Stack -- LIFO. Queue -- FIFO. Використання `arr.push + arr.pop` -- це **stack**. Щоб отримати чергу з масиву, треба `push + shift` (але shift повільний).

```javascript
// ❌ Це стек, не черга
const queue = [];
queue.push(1); queue.push(2); queue.push(3);
queue.pop();  // 3 -- ❌ ми хотіли 1
```

### 5. Monotonic stack -- який порядок?

У задачах "наступний більший" стек тримає зростаючі значення, виштовхуємо коли приходить більший. У задачах "наступний менший" -- навпаки. Легко переплутати. **Правило:** на співбесіді проговори приклад вголос, не намагайся відразу писати код.

### 6. Переповнення стеку в рекурсії

Рекурсія = неявний стек. Node.js ламає стек на ~10-15 тисяч вкладених викликів. Для DFS на глибоких графах/деревах -- перекладай на явний стек.

```javascript
// ❌ на глибині 50000 викине RangeError
function dfs(node) {
  if (!node) return;
  dfs(node.left);
  dfs(node.right);
}

// ✅ ітеративно через явний стек
function dfs(root) {
  const stack = [root];
  while (stack.length > 0) {
    const node = stack.pop();
    if (!node) continue;
    stack.push(node.right);
    stack.push(node.left);
  }
}
```

### 7. Deque через `unshift/shift` -- те саме прокляття

`arr.unshift` теж O(n). Використовувати масив як справжній deque -- помилка. Або пиши linked list, або користуйся двома масивами (по одному на кожен кінець), або тримай індекси голови/хвоста.

---

## Ключові думки

- **Stack = LIFO, Queue = FIFO.** Знай, коли яка потрібна.
- Stack у JS -- це просто `[].push/.pop`, O(1). Черга -- СКЛАДНІШЕ: наївний `shift` -- O(n).
- **Ефективна черга:** індекс голови в масиві або linked list.
- **Dummy node pattern** з linked list, **monotonic stack pattern** зі стеком -- двоє з найпотужніших прийомів.
- Monotonic stack розв'язує "next greater/smaller" сімейство задач за O(n) замість O(n²).
- Валідація дужок, RPN, simplify path, browser history -- стандартні задачі на стек.
- BFS -- завжди черга. DFS -- завжди стек (явний або рекурсивний).
- Sliding Window Maximum через deque -- ілюстрація того, як монотонна структура дає O(n).
- Уникай `arr.shift()` і `arr.unshift()` у гарячих циклах -- класичний performance red flag.
- На співбесіді завжди проговорюй: "який тут Big O?". Якщо використовуєш `shift` -- одразу озвуч, що це проблема і як ти б її вирішив у продакшні.

У наступних файлах -- дерева і графи, де стек і черга стають основою всіх обходів.
