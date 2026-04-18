# Графи: структура, обхід, класичні задачі

## Що таке граф? Чому це окрема тема?

**Граф** -- це структура даних, яка моделює **зв'язки** між об'єктами. Формально граф -- це пара `G = (V, E)`, де:
- `V` (vertices) -- множина **вершин** (вузлів).
- `E` (edges) -- множина **ребер**, кожне з яких з'єднує дві вершини.

Дерево, яке ми розбирали раніше -- це окремий випадок графа (зв'язний, ациклічний, з одним коренем). Граф ширший: у ньому можуть бути цикли, кілька компонент зв'язності, орієнтовані ребра, ваги.

```
     Граф:                        Дерево:
        A                             A
       / \                           / \
      B---C                         B   C
       \ /                         /|
        D                         D E
  (є цикл B-C-D)             (без циклів, ієрархія)
```

Граф -- це найбільш універсальна структура для моделювання реальних систем: соцмережі, карти, інтернет, залежності, потоки. Тому задач на графи на співбесідах багато, і знання шаблонів (BFS, DFS, topological sort, Union-Find) дає тобі суперсилу.

---

## Основні терміни

Перед тим як писати код, розберемо словник.

**Вершина (vertex, node)** -- точка у графі. Наприклад, людина у соцмережі, місто на карті, файл у системі залежностей.

**Ребро (edge)** -- зв'язок між двома вершинами. Може бути:
- **Undirected (неорієнтоване)** -- двосторонній зв'язок. Якщо A пов'язано з B, то B пов'язано з A. Приклад: друзі у Facebook.
- **Directed (орієнтоване)** -- односторонній зв'язок. A -> B не означає B -> A. Приклад: підписники у Twitter (я підписаний на тебе, ти на мене -- ні).

```
     Undirected:              Directed:
        A --- B                 A --> B
        |     |                 ^     |
        C --- D                 |     v
                                D <-- C
```

**Weighted (зважений)** -- кожному ребру приписано число (вага). Приклад: відстань між містами, вартість перельоту, пропускна здатність каналу. **Unweighted (незважений)** -- ребра рівноцінні.

```
  Weighted:                Unweighted:
      A --5-- B                A ----- B
      |       |                |       |
      3       2                |       |
      |       |                |       |
      C --8-- D                C ----- D
```

**Cyclic vs acyclic.** Цикл -- це шлях, який починається і закінчується в одній вершині. **DAG** (Directed Acyclic Graph) -- орієнтований граф без циклів -- дуже важливий клас (task dependencies, build systems, git commits).

**Degree (ступінь вершини)** -- скільки ребер у неї входить. В орієнтованому графі розрізняють:
- **In-degree** -- скільки ребер входить.
- **Out-degree** -- скільки ребер виходить.

**Path (шлях)** -- послідовність вершин, з'єднаних ребрами: A -> B -> C -> D.

**Connected component (компонента зв'язності)** -- максимальна підмножина вершин, між якими є шлях. Граф може складатись з кількох компонент:

```
    Компонента 1:      Компонента 2:
      A --- B             E --- F
      |                    |
      C                    G
```

**Dense vs sparse.** Щільний граф -- де ребер приблизно `V²` (повнозв'язний). Розріджений -- ребер порядку `V`. Від цього залежить вибір представлення і алгоритмів.

---

## Реальні приклади: де графи всюди

Щоб зрозуміти, чому графи настільки важливі, подивись, де вони:

1. **Соцмережі.** Facebook: вершини -- люди, ребра -- дружба (undirected). Twitter/Instagram: ребра -- підписки (directed). "Friends of friends", "People you may know" -- це BFS.

2. **Карти та навігація.** Google Maps: вершини -- перехрестя, ребра -- дороги зі зваженням (відстань, час, пробки). Маршрут -- shortest path (Dijkstra, A*).

3. **Залежності задач (DAG).** Task scheduler, build systems (Make, npm, Webpack), Airflow pipelines. Треба виконати задачі у правильному порядку -- **topological sort**. Виявити циклічну залежність -- cycle detection.

4. **Web crawling.** Інтернет -- гігантський directed graph (сторінки -- вершини, гіперпосилання -- ребра). Crawler робить BFS/DFS. PageRank теж працює на графі.

5. **Recommendation engines.** "Користувачі, які купили X, також купили Y". Збудуй bipartite graph (користувачі <-> товари), знаходь схожих користувачів через спільні ребра.

6. **Бізнес-процеси, workflow.** State machine -- вершини -- стани, ребра -- переходи. Orders, refunds, KYC -- typical directed graphs з умовами.

7. **Версійний контроль.** Git commit graph -- DAG. Merge, rebase, cherry-pick -- це операції на цьому графі.

8. **Network routing.** Інтернет-роутери будують таблиці маршрутизації на основі shortest path алгоритмів (OSPF, BGP).

9. **Fraud detection.** Транзакції між рахунками -- граф. Цикли підозрілих переказів -- можливий money laundering.

10. **Knowledge graphs.** Google Knowledge Graph, Wikipedia links, RDF triples -- все це графи.

Коли на співбесіді чуєш "relationships", "connected", "path", "dependency", "network" -- це майже завжди граф.

---

## Як зображати граф у коді? Три основні представлення

Коли ти моделюєш граф, треба вирішити **як зберігати** його у пам'яті. Від цього залежить ефективність операцій.

### 1. Adjacency list (список суміжності) -- найпоширеніше

Для кожної вершини зберігаємо список її сусідів. У JavaScript зручно використовувати `Map<Node, Node[]>` або об'єкт.

```javascript
// Undirected граф:
//     A --- B
//     |     |
//     C --- D

const graph = new Map();
graph.set("A", ["B", "C"]);
graph.set("B", ["A", "D"]);
graph.set("C", ["A", "D"]);
graph.set("D", ["B", "C"]);

// Або для directed графа:
//     A --> B
//     |     |
//     v     v
//     C --> D
const directed = new Map();
directed.set("A", ["B", "C"]);
directed.set("B", ["D"]);
directed.set("C", ["D"]);
directed.set("D", []);
```

**Пам'ять:** O(V + E). Для кожної вершини одна комірка + для кожного ребра один запис у списку (двічі, якщо undirected).

**Операції:**
- Перевірити, чи є ребро A-B: O(degree(A)) -- лінійний пошук у списку. Якщо замість масиву використати `Set`, буде O(1).
- Обійти всіх сусідів вершини: O(degree(V)).
- Обійти весь граф: O(V + E).

**Коли використовувати:** майже завжди. Особливо для розріджених графів (sparse), де `E << V²`. Соцмережі, карти, інтернет -- всі вони sparse.

**JS patterns:**

```javascript
// Додавання ребра в undirected граф
function addEdge(graph, u, v) {
  if (!graph.has(u)) graph.set(u, []);
  if (!graph.has(v)) graph.set(v, []);
  graph.get(u).push(v);
  graph.get(v).push(u);  // двостороннє!
}

// Directed: додаємо тільки одну сторону
function addDirectedEdge(graph, from, to) {
  if (!graph.has(from)) graph.set(from, []);
  if (!graph.has(to)) graph.set(to, []);
  graph.get(from).push(to);
}

// Для зваженого графа: зберігаємо пари [сусід, вага]
const weighted = new Map();
weighted.set("A", [["B", 5], ["C", 3]]);
weighted.set("B", [["A", 5], ["D", 2]]);
// Або об'єктами: { to: "B", weight: 5 }
```

### 2. Adjacency matrix (матриця суміжності)

Двовимірний масив `V × V`, де `matrix[i][j] = 1` (або вага), якщо є ребро i -> j, інакше 0.

```javascript
// Граф: A(0) -- B(1) -- C(2)
//         \___________/ (A-C)

const matrix = [
  //  A  B  C
  [  0, 1, 1 ], // A
  [  1, 0, 1 ], // B
  [  1, 1, 0 ], // C
];

// Перевірка ребра A-B: matrix[0][1] → O(1)
// Додавання ребра: matrix[i][j] = 1 → O(1)
```

**Пам'ять:** O(V²). Для великого sparse графа це катастрофа: граф на мільйон вершин = трильйон комірок.

**Операції:**
- Перевірка ребра: O(1). **Швидше за adjacency list.**
- Обійти всіх сусідів: O(V). **Повільніше за adjacency list**, бо треба пробігти цілий рядок.
- Обійти весь граф: O(V²).

**Коли використовувати:**
- **Щільні графи** (dense), де `E ≈ V²`.
- Коли потрібні часті перевірки "чи є ребро між X і Y" за O(1).
- Невеликий граф (скажімо, до 1000 вершин).
- Алгоритми типу Floyd-Warshall, які природно працюють на матриці.

```
     Adjacency List                 Adjacency Matrix
  A: [B, C]                       A B C D
  B: [A, D]                    A [0 1 1 0]
  C: [A, D]                    B [1 0 0 1]
  D: [B, C]                    C [1 0 0 1]
                               D [0 1 1 0]
  Пам'ять: O(V+E)              Пам'ять: O(V²)
  Пошук ребра: O(deg(V))       Пошук ребра: O(1)
```

### 3. Edge list (список ребер)

Просто масив пар (або трійок для weighted): `[[u, v], [u, v], ...]`.

```javascript
const edges = [
  ["A", "B", 5],
  ["A", "C", 3],
  ["B", "D", 2],
  ["C", "D", 8],
];
```

**Пам'ять:** O(E).

**Коли використовувати:**
- Якщо граф передається саме у такому форматі (типовий вхід у LeetCode задачах).
- **Kruskal algorithm** для MST (сортує всі ребра).
- Коли треба ітерувати всі ребра, але не сусідів конкретної вершини.

Часто ти перетворюєш edge list на adjacency list одразу на вході.

```javascript
function buildAdjList(edges, n) {
  const graph = new Map();
  for (let i = 0; i < n; i++) graph.set(i, []);
  for (const [u, v] of edges) {
    graph.get(u).push(v);
    graph.get(v).push(u);
  }
  return graph;
}
```

---

## BFS: Breadth-First Search (пошук у ширину)

BFS -- це обхід, який досліджує граф **рівнями**. Спочатку всі сусіди стартової вершини, потім сусіди сусідів, і так далі.

Уяви, що ти кидаєш камінь у воду -- хвилі розходяться колами від центру. Це BFS.

```
Старт: A          Рівень 0: A
                  Рівень 1: B, C (сусіди A)
    A             Рівень 2: D, E (сусіди B, C)
   / \
  B   C
  |   |
  D   E

  Порядок BFS: A, B, C, D, E
```

### Ключова ідея: черга (queue)

BFS завжди використовує **FIFO чергу**. Елементи додаються в кінець, вилучаються з початку.

### Шаблонний код BFS

```javascript
function bfs(graph, start) {
  const visited = new Set();      // щоб не повертатися
  const queue = [start];          // починаємо з start
  visited.add(start);
  const order = [];               // порядок відвідування

  while (queue.length > 0) {
    const node = queue.shift();   // УВАГА: shift -- O(n)!
    order.push(node);

    for (const neighbor of graph.get(node) || []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);    // позначаємо ВІДРАЗУ, не коли витягуємо
        queue.push(neighbor);
      }
    }
  }

  return order;
}
```

**Важливо про `shift()`:** у JS `Array.prototype.shift` -- O(n), бо зсуває всі елементи. Для великих графів це б'є по продуктивності. Рішення:
1. Використовувати власну deque (двобічну чергу).
2. Використовувати індексну змінну замість фактичного shift.

```javascript
function bfsOptimized(graph, start) {
  const visited = new Set([start]);
  const queue = [start];
  let head = 0;                   // вказівник на початок

  while (head < queue.length) {
    const node = queue[head++];   // "витягаємо" без зсуву
    for (const neighbor of graph.get(node) || []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push(neighbor);
      }
    }
  }
}
```

### Візуалізація BFS крок за кроком

Граф:
```
     1
    / \
   2   3
  / \   \
 4   5   6
```

Стартуємо з 1:

```
Крок 0: queue = [1], visited = {1}
Крок 1: витягнули 1 → сусіди 2, 3
        queue = [2, 3], visited = {1, 2, 3}
Крок 2: витягнули 2 → сусіди 4, 5
        queue = [3, 4, 5], visited = {1, 2, 3, 4, 5}
Крок 3: витягнули 3 → сусід 6
        queue = [4, 5, 6], visited = {1, 2, 3, 4, 5, 6}
Крок 4: витягли 4, 5, 6 по черзі -- нема нових сусідів
        queue = []
```

Порядок: 1, 2, 3, 4, 5, 6 -- рівень за рівнем.

### Use cases BFS

1. **Shortest path в unweighted графі.** BFS гарантує, що вперше ми досягаємо вершини по найкоротшому шляху (по кількості ребер). Це ключова властивість.

2. **Layer-by-layer обробка.** Знайти всіх друзів до 3-го ступеня (friends of friends of friends).

3. **Level order traversal** дерева (окремий випадок BFS).

4. **Connectivity** -- хто досяжний з вершини X.

5. **Web crawler** -- обходити сторінки рівнями від стартового URL.

### BFS для shortest path (з відстанями)

```javascript
function shortestPath(graph, start, target) {
  if (start === target) return 0;

  const visited = new Set([start]);
  const queue = [[start, 0]];   // [вершина, відстань]

  while (queue.length > 0) {
    const [node, dist] = queue.shift();

    for (const neighbor of graph.get(node) || []) {
      if (neighbor === target) return dist + 1;
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push([neighbor, dist + 1]);
      }
    }
  }

  return -1; // недосяжно
}
```

Або альтернативно -- з окремою мапою відстаней:

```javascript
function shortestPaths(graph, start) {
  const dist = new Map([[start, 0]]);
  const queue = [start];
  let head = 0;

  while (head < queue.length) {
    const node = queue[head++];
    for (const neighbor of graph.get(node) || []) {
      if (!dist.has(neighbor)) {
        dist.set(neighbor, dist.get(node) + 1);
        queue.push(neighbor);
      }
    }
  }
  return dist;
}
```

**Complexity BFS:** O(V + E) time, O(V) space (черга + visited).

---

## DFS: Depth-First Search (пошук у глибину)

DFS -- це обхід, який іде "вглибину" першим. Доходить до тупика, потім повертається і пробує інший шлях.

Уяви, що ти йдеш по лабіринту, завжди обираєш перший доступний коридор, а коли впираєшся в стіну -- повертаєшся до попередньої розвилки і пробуєш інший коридор.

```
Старт: A
                          DFS порядок може бути:
    A                     A → B → D → E → C
   / \                    (пішов глибоко по лівій гілці,
  B   C                    повернувся, пішов у праву)
 / \
D   E
```

### Ключова ідея: стек (stack)

DFS природно реалізується через **рекурсію** (рекурсивний стек викликів -- це стек) або явно через **Stack структуру**.

### Рекурсивний DFS

```javascript
function dfs(graph, start, visited = new Set()) {
  if (visited.has(start)) return;
  visited.add(start);
  console.log(start);   // обробка вершини (pre-order)

  for (const neighbor of graph.get(start) || []) {
    dfs(graph, neighbor, visited);
  }
}
```

Читається майже як природна мова: "відвідай себе, потім для кожного сусіда викличи dfs".

### Ітеративний DFS через явний стек

```javascript
function dfsIterative(graph, start) {
  const visited = new Set();
  const stack = [start];

  while (stack.length > 0) {
    const node = stack.pop();     // LIFO!
    if (visited.has(node)) continue;
    visited.add(node);
    console.log(node);

    for (const neighbor of graph.get(node) || []) {
      if (!visited.has(neighbor)) {
        stack.push(neighbor);
      }
    }
  }
}
```

Різниця між BFS та DFS коду: **`queue.shift()` vs `stack.pop()`**. Більше нічого не змінюється!

```
BFS:        DFS:
queue.shift()   stack.pop()
FIFO            LIFO
→ рівні         → глибина
```

### Візуалізація DFS крок за кроком

Той самий граф:
```
     1
    / \
   2   3
  / \   \
 4   5   6
```

DFS з 1 (рекурсивно):

```
dfs(1): visit 1
  dfs(2): visit 2
    dfs(4): visit 4 (нема сусідів, повернення)
    dfs(5): visit 5 (нема сусідів, повернення)
  dfs(3): visit 3
    dfs(6): visit 6
```

Порядок: 1, 2, 4, 5, 3, 6.

### Use cases DFS

1. **Cycle detection.** Якщо під час DFS натрапив на вершину, яка вже в поточному шляху -- це цикл.

2. **Topological sort** у DAG (через post-order DFS).

3. **Connectivity / connected components.** Викликаєш DFS з кожної невідвіданої вершини -- кількість викликів = кількість компонент.

4. **Pathfinding** (знайти ХОЧ ЯКИЙСЬ шлях; НЕ для shortest).

5. **Backtracking** задачі (N-Queens, Sudoku, permutations) -- це DFS.

6. **Maze solving, labyrinth search.**

### Рекурсивний DFS -- пастка stack overflow

На графі з 1 мільйоном вершин ланцюжкового типу (A -> B -> C -> ... -> Z) рекурсивний DFS зламається через переповнення стеку.

```javascript
// Не працює на довгих ланцюгах у Node.js:
function dfs(graph, v, visited) {
  visited.add(v);
  for (const u of graph.get(v)) if (!visited.has(u)) dfs(graph, u, visited);
}

// Node.js default stack size ≈ 10-20k викликів
```

**Рішення:** використовувати **ітеративний DFS через явний стек**. Або збільшити `--stack-size`, але це hack, не фікс.

**Complexity DFS:** O(V + E) time, O(V) space (стек/рекурсія + visited). Той самий, що і BFS.

---

## Visited set: чому це критично

**Без visited** BFS/DFS на циклічному графі піде у нескінченний цикл.

```
   A --- B
   |     |
   C --- D

DFS(A) без visited:
  visit A → піти до B → піти до A → піти до B → ... ∞
```

Always, always, always тримай `visited` множину. Два типові патерни:

**Паттерн 1: позначити при додаванні у чергу/стек.**
```javascript
visited.add(start);
queue.push(start);
while (queue.length) {
  const node = queue.shift();
  for (const n of graph.get(node)) {
    if (!visited.has(n)) {
      visited.add(n);          // ← одразу позначаємо
      queue.push(n);
    }
  }
}
```
**Перевага:** елемент ніколи не потрапляє у чергу двічі.

**Паттерн 2: позначити при витягуванні.**
```javascript
queue.push(start);
while (queue.length) {
  const node = queue.shift();
  if (visited.has(node)) continue;
  visited.add(node);            // ← позначаємо при обробці
  for (const n of graph.get(node)) {
    if (!visited.has(n)) queue.push(n);
  }
}
```
**Недолік:** елемент може потрапити у чергу кілька разів (за рахунок пам'яті).

Для BFS **завжди перший паттерн** (інакше shortest path може бути неправильним). Для DFS -- обидва ок.

### Альтернативи Set

Для графів, де вершини -- числа 0..n-1, можна використовувати boolean масив:

```javascript
const visited = new Array(n).fill(false);
visited[v] = true;
```

Це швидше за `Set` у JS (доступ за індексом O(1) проти hash lookup).

Для матричних задач (islands) часто модифікують саму матрицю -- замінюють '1' на '0' замість окремого visited. Економія пам'яті, але руйнує вхід (спитай інтерв'юера, чи це ок).

---

## Класичні задачі на графи

Ось топ задач, які майже гарантовано зустрінеш.

### 1. Number of Islands (LeetCode 200)

**Задача:** дана матриця з `'1'` (земля) і `'0'` (вода). Порахувати кількість островів. Острів -- група з'єднаних по горизонталі/вертикалі клітинок землі.

**Ідея:** матриця -- це неявний граф, де сусіди -- 4 суміжні клітинки. Кожен острів -- компонента зв'язності. Пробігаємо матрицю, на кожній '1' запускаємо DFS/BFS, який "затоплює" весь острів (міняє '1' на '0'). Кількість запусків = кількість островів.

```javascript
function numIslands(grid) {
  const rows = grid.length;
  const cols = grid[0].length;
  let count = 0;

  function dfs(r, c) {
    if (r < 0 || c < 0 || r >= rows || c >= cols) return;
    if (grid[r][c] === '0') return;
    grid[r][c] = '0';   // позначаємо як відвідане (топимо)
    dfs(r + 1, c);
    dfs(r - 1, c);
    dfs(r, c + 1);
    dfs(r, c - 1);
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === '1') {
        count++;
        dfs(r, c);
      }
    }
  }

  return count;
}
```

**Complexity:** O(rows × cols) time, O(rows × cols) space (stack у worst case).

**Варіації:** "Max Area of Island", "Surrounded Regions", "Island Perimeter" -- всі на тій самій ідеї.

### 2. Clone Graph (LeetCode 133)

**Задача:** дано вершину графа (з посиланнями на сусідів). Повернути глибоку копію всього графа.

**Ідея:** DFS/BFS з мапою `original → clone`. При відвідуванні вершини:
1. Якщо вже клонували -- повертаємо існуючий клон.
2. Інакше створюємо новий клон, рекурсивно клонуємо сусідів.

```javascript
function cloneGraph(node) {
  if (!node) return null;
  const map = new Map();  // original → clone

  function dfs(original) {
    if (map.has(original)) return map.get(original);
    const clone = { val: original.val, neighbors: [] };
    map.set(original, clone);         // ставимо ПЕРЕД рекурсією,
                                       // щоб уникнути зациклення
    for (const n of original.neighbors) {
      clone.neighbors.push(dfs(n));
    }
    return clone;
  }

  return dfs(node);
}
```

**Ключовий нюанс:** `map.set(original, clone)` має бути **до** рекурсії, інакше на циклічних графах буде infinite recursion.

### 3. Course Schedule (LeetCode 207, 210)

**Задача:** є `n` курсів і список prerequisites `[a, b]` (щоб пройти `a`, треба спочатку `b`). Чи можна пройти всі курси? Якщо так -- в якому порядку?

**Ідея:** це задача про **DAG + topological sort**. Якщо у графі є цикл -- неможливо. Якщо немає -- порядок topological sort дає валідну послідовність.

Реалізація через **Kahn's algorithm** (BFS):

```javascript
function findOrder(numCourses, prerequisites) {
  const graph = Array.from({ length: numCourses }, () => []);
  const inDegree = new Array(numCourses).fill(0);

  // побудова графа: ребро b → a (b перед a)
  for (const [a, b] of prerequisites) {
    graph[b].push(a);
    inDegree[a]++;
  }

  // стартуємо з вершин без prerequisites
  const queue = [];
  for (let i = 0; i < numCourses; i++) {
    if (inDegree[i] === 0) queue.push(i);
  }

  const order = [];
  while (queue.length) {
    const course = queue.shift();
    order.push(course);
    for (const next of graph[course]) {
      if (--inDegree[next] === 0) queue.push(next);
    }
  }

  return order.length === numCourses ? order : [];
}
```

Якщо наприкінці `order.length < numCourses` -- залишились вершини з ненульовим in-degree, отже цикл.

### 4. Word Ladder (LeetCode 127)

**Задача:** дано beginWord, endWord, словник. Трансформуй beginWord в endWord, змінюючи по одній букві за раз, кожне проміжне слово має бути у словнику. Мінімальна кількість кроків.

**Ідея:** це **shortest path в unweighted графі**, де:
- Вершини -- слова.
- Ребра -- між словами, що відрізняються на одну букву.
- BFS знаходить найкоротший шлях.

```javascript
function ladderLength(beginWord, endWord, wordList) {
  const wordSet = new Set(wordList);
  if (!wordSet.has(endWord)) return 0;

  const queue = [[beginWord, 1]];
  const visited = new Set([beginWord]);

  while (queue.length) {
    const [word, len] = queue.shift();
    if (word === endWord) return len;

    // генеруємо всіх сусідів: замінюємо кожну букву на a-z
    for (let i = 0; i < word.length; i++) {
      for (let c = 97; c <= 122; c++) {  // 'a'..'z'
        const next = word.slice(0, i) + String.fromCharCode(c) + word.slice(i + 1);
        if (wordSet.has(next) && !visited.has(next)) {
          visited.add(next);
          queue.push([next, len + 1]);
        }
      }
    }
  }
  return 0;
}
```

Ключова оптимізація -- генерувати сусідів, а не перебирати всі пари слів (останнє -- O(N² × L)).

### 5. Pacific Atlantic Water Flow (LeetCode 417)

**Задача:** матриця висот. Вода тече з клітинки у сусідню, якщо та нижча (або рівна). Знайти всі клітинки, з яких вода досягає обох океанів (Pacific -- верх і лівий край, Atlantic -- низ і правий).

**Ідея:** замість того щоб запускати BFS/DFS з кожної клітинки ("чи досягає обох океанів" -- O((rows×cols)²)), **інвертуємо мислення**. Запускаємо BFS/DFS з країв -- з Pacific і Atlantic окремо -- йдемо "вгору" (від низької до високої). Результат -- перетин множин.

```javascript
function pacificAtlantic(heights) {
  const rows = heights.length;
  const cols = heights[0].length;
  const pacific = new Set();
  const atlantic = new Set();

  function dfs(r, c, visited, prevHeight) {
    const key = r * cols + c;
    if (r < 0 || c < 0 || r >= rows || c >= cols) return;
    if (visited.has(key)) return;
    if (heights[r][c] < prevHeight) return;  // вода "піднімається"
    visited.add(key);
    const h = heights[r][c];
    dfs(r + 1, c, visited, h);
    dfs(r - 1, c, visited, h);
    dfs(r, c + 1, visited, h);
    dfs(r, c - 1, visited, h);
  }

  for (let r = 0; r < rows; r++) {
    dfs(r, 0, pacific, 0);
    dfs(r, cols - 1, atlantic, 0);
  }
  for (let c = 0; c < cols; c++) {
    dfs(0, c, pacific, 0);
    dfs(rows - 1, c, atlantic, 0);
  }

  const result = [];
  for (const key of pacific) {
    if (atlantic.has(key)) {
      result.push([Math.floor(key / cols), key % cols]);
    }
  }
  return result;
}
```

Це класичний приклад **multi-source BFS/DFS** -- запустити пошук одночасно з багатьох стартових точок.

---

## Topological Sort (топологічне сортування)

**Topological sort** -- впорядкування вершин DAG так, що для кожного ребра `u → v` вершина `u` стоїть перед `v`.

**Приклад:** задачі білдера залежностей (npm install, make). Якщо пакет A залежить від B, значить B має бути "перед" A.

```
  A → B → D
  ↓   ↑
  C ──┘

  Можливий topological order: A, C, B, D
  Або: A, B, C (ні! бо B → D, D не серед них)
                правильно: A, C, B, D
```

**Існує тільки для DAG.** Якщо є цикл -- topological sort неможливий.

### Алгоритм Кана (BFS-based)

1. Порахуй in-degree кожної вершини.
2. Поклади у чергу всі вершини з in-degree = 0.
3. Витягуй з черги, додавай у результат, зменшуй in-degree сусідів. Якщо in-degree сусіда став 0 -- додай у чергу.
4. Якщо наприкінці оброблено не всі вершини -- є цикл.

```javascript
function topologicalSort(numNodes, edges) {
  const graph = Array.from({ length: numNodes }, () => []);
  const inDegree = new Array(numNodes).fill(0);

  for (const [u, v] of edges) {
    graph[u].push(v);
    inDegree[v]++;
  }

  const queue = [];
  for (let i = 0; i < numNodes; i++) {
    if (inDegree[i] === 0) queue.push(i);
  }

  const result = [];
  while (queue.length) {
    const node = queue.shift();
    result.push(node);
    for (const next of graph[node]) {
      if (--inDegree[next] === 0) queue.push(next);
    }
  }

  if (result.length !== numNodes) {
    throw new Error("Cycle detected");
  }
  return result;
}
```

### Варіант через DFS (post-order)

Ідея: роби DFS, додавай вершину у результат **коли повертаєшся з неї** (post-order). Наприкінці переверни масив.

```javascript
function topologicalSortDFS(numNodes, edges) {
  const graph = Array.from({ length: numNodes }, () => []);
  for (const [u, v] of edges) graph[u].push(v);

  const visited = new Array(numNodes).fill(0); // 0 = білий, 1 = сірий, 2 = чорний
  const result = [];

  function dfs(node) {
    if (visited[node] === 1) throw new Error("Cycle");
    if (visited[node] === 2) return;
    visited[node] = 1;                // сірий -- в обробці
    for (const next of graph[node]) dfs(next);
    visited[node] = 2;                // чорний -- оброблено
    result.push(node);
  }

  for (let i = 0; i < numNodes; i++) {
    if (visited[i] === 0) dfs(i);
  }
  return result.reverse();
}
```

**Triple coloring** (білий/сірий/чорний) -- стандартний спосіб виявляти цикли у DFS:
- **Білий** -- ще не відвідано.
- **Сірий** -- у поточному шляху рекурсії. Якщо зустрів сіру -- є цикл.
- **Чорний** -- оброблено повністю.

**Use cases topological sort:**
- Build systems, task schedulers (Airflow, Kubernetes jobs).
- Курсові prerequisites.
- Spreadsheet formula evaluation (cell A1 залежить від B1 -- B1 рахуємо першим).
- Symbol table у компіляторах.

---

## Union-Find (Disjoint Set Union, DSU)

**Union-Find** -- структура даних для ефективного керування набором неперетинних множин. Підтримує дві операції:
- `find(x)` -- повертає ідентифікатор множини, якій належить `x`.
- `union(x, y)` -- об'єднує множини, що містять `x` і `y`.

**Навіщо:** відповідати на питання "чи `x` і `y` в одній компоненті зв'язності?" за майже O(1).

### Базова реалізація

```javascript
class UnionFind {
  constructor(n) {
    this.parent = Array.from({ length: n }, (_, i) => i); // кожен сам собі корінь
    this.rank = new Array(n).fill(0);
    this.components = n;
  }

  find(x) {
    if (this.parent[x] !== x) {
      this.parent[x] = this.find(this.parent[x]); // path compression
    }
    return this.parent[x];
  }

  union(x, y) {
    const rootX = this.find(x);
    const rootY = this.find(y);
    if (rootX === rootY) return false; // вже в одній множині

    // union by rank: менше дерево стає дитиною більшого
    if (this.rank[rootX] < this.rank[rootY]) {
      this.parent[rootX] = rootY;
    } else if (this.rank[rootX] > this.rank[rootY]) {
      this.parent[rootY] = rootX;
    } else {
      this.parent[rootY] = rootX;
      this.rank[rootX]++;
    }
    this.components--;
    return true;
  }

  connected(x, y) {
    return this.find(x) === this.find(y);
  }
}
```

### Як це працює (інтуїція)

Кожен елемент має посилання на "батька". Корінь -- той, хто вказує сам на себе. `find(x)` йде по посиланнях до кореня.

```
Початок (n=5):  0 1 2 3 4     кожен сам собі корінь
                ↑ ↑ ↑ ↑ ↑

union(0, 1):    0 ← 1          1 тепер вказує на 0
                ↑
                0

union(2, 3):    0 ← 1   2 ← 3
                ↑       ↑

union(0, 2):    0 ← 1
                ↑
                0 ← 2 ← 3      тепер все одна компонента з 4
                        ↑
                        4 (сам собі корінь, окремий)
```

**Path compression:** при `find(x)` одразу приєднуємо всі проміжні вершини до кореня. Це "стискає" дерево, робить наступні `find` швидшими.

```
  Перед find(3):     Після find(3):
       0                  0
       ↑                 ↑ ↑ ↑
       1                 1 2 3
       ↑
       2
       ↑
       3
```

**Union by rank:** при об'єднанні менше дерево приєднуємо до більшого. Це утримує висоту маленькою.

**Complexity:** амортизовано **O(α(n))** на операцію (α -- обернена функція Акермана, практично константа < 5 для будь-яких реальних n). Тобто майже O(1).

### Use cases Union-Find

1. **Number of Connected Components** (LeetCode 323). Обробили всі ребра union-ом, `components` дає відповідь.

2. **Redundant Connection** (LeetCode 684). Пробігаємо ребра, робимо union. Якщо union повертає false -- це ребро створює цикл.

```javascript
function findRedundantConnection(edges) {
  const uf = new UnionFind(edges.length + 1);
  for (const [u, v] of edges) {
    if (!uf.union(u, v)) return [u, v];
  }
  return [];
}
```

3. **Kruskal's MST** -- сортуємо ребра, додаємо, якщо union ще не об'єднав їх.

4. **Friend circles / account merging** -- групування по транзитивних зв'язках.

5. **Dynamic connectivity** -- онлайн-задачі, де ребра додаються і треба відповідати на connectivity запити.

Коли на співбесіді чуєш "connected components", "grouping", "cycle detection в undirected графі" -- Union-Find має спасти на думку.

---

## Shortest Path: алгоритми

### BFS -- для unweighted графів

Ми це вже розібрали. BFS гарантує, що перший раз, коли ми досягаємо вершини, це найкоротший шлях **по кількості ребер**. Працює тільки якщо всі ребра "коштують" однаково.

**Complexity:** O(V + E).

### Dijkstra -- для weighted графів з невід'ємними вагами

Dijkstra узагальнює BFS на зважений граф. Замість FIFO черги використовує **priority queue (min-heap)** -- завжди витягує вершину з найменшою поточною відстанню.

**Алгоритм словами:**
1. Встанови `dist[start] = 0`, решта = ∞.
2. Поклади `(0, start)` у priority queue.
3. Витягни вершину з найменшою `dist`. Якщо вже оброблена -- пропусти.
4. Для кожного сусіда: якщо `dist[node] + weight < dist[neighbor]` -- оновлюй і додавай у PQ.
5. Коли PQ порожня -- готово.

```javascript
function dijkstra(graph, start) {
  // graph: Map<node, [[neighbor, weight], ...]>
  const dist = new Map();
  for (const node of graph.keys()) dist.set(node, Infinity);
  dist.set(start, 0);

  // Для простоти -- масив-PQ (у реальності використай MinHeap)
  const pq = [[0, start]];   // [distance, node]

  while (pq.length) {
    pq.sort((a, b) => a[0] - b[0]);   // НЕ ефективно! Лише для прикладу.
    const [d, node] = pq.shift();

    if (d > dist.get(node)) continue;   // застарілий запис

    for (const [neighbor, weight] of graph.get(node) || []) {
      const alt = d + weight;
      if (alt < dist.get(neighbor)) {
        dist.set(neighbor, alt);
        pq.push([alt, neighbor]);
      }
    }
  }
  return dist;
}
```

Для production використовуй справжню min-heap (Priority Queue), інакше O(V²) замість O((V+E) log V).

**Візуалізація:**

```
Граф:
     1
  A----B
  |    |
  4    2
  |    |
  C----D
     5

Dijkstra з A:
  dist: {A: 0, B: ∞, C: ∞, D: ∞}
  PQ: [(0, A)]

  Витягли A (d=0):
    B: 0+1 = 1 < ∞ → dist[B]=1, push (1,B)
    C: 0+4 = 4 < ∞ → dist[C]=4, push (4,C)
  dist: {A: 0, B: 1, C: 4, D: ∞}

  Витягли B (d=1):
    D: 1+2 = 3 < ∞ → dist[D]=3, push (3,D)
  dist: {A: 0, B: 1, C: 4, D: 3}

  Витягли D (d=3):
    C: 3+5 = 8 > 4 — не оновлюємо
  dist: {A: 0, B: 1, C: 4, D: 3}

  Витягли C (d=4): нема оновлень.

Результат: A→0, B→1, D→3, C→4
```

**Complexity:** O((V + E) log V) з binary heap.

**Важливо:** Dijkstra **не працює з від'ємними вагами**. Там потрібен Bellman-Ford.

### Bellman-Ford -- з від'ємними вагами

Повільніший, O(V × E), але **підтримує від'ємні ребра** і вміє **детектувати negative cycles**.

Ідея: V-1 разів проходимо всі ребра і релаксуємо (`dist[v] = min(dist[v], dist[u] + w)`). Якщо після V-1 ітерацій все ще можемо релаксувати -- є negative cycle.

Use case: currency arbitrage (від'ємні ваги = виграш при конвертації).

### A* -- з евристикою

Модифікація Dijkstra для швидшого пошуку до конкретної цілі. Використовує **heuristic** `h(node)` -- оцінку відстані від вершини до цілі.

Пріоритет у PQ: `f(n) = dist(start, n) + h(n, goal)`.

Якщо `h` правдива (admissible, не overestimate) -- A* знаходить оптимум швидше Dijkstra.

Use case: pathfinding у іграх (сітки), GPS navigation.

**Короткий порівняльний підсумок:**

| Алгоритм      | Граф             | Складність             | Що знаходить                |
|---------------|------------------|------------------------|-----------------------------|
| BFS           | unweighted       | O(V + E)               | Shortest path by edges      |
| Dijkstra      | non-negative w   | O((V+E) log V)         | Shortest path (single src)  |
| Bellman-Ford  | any weights      | O(V × E)               | Shortest + negative cycles  |
| A*            | weighted + heur. | Залежить від h         | Shortest до конкретної цілі |
| Floyd-Warshall| any weights      | O(V³)                  | All-pairs shortest          |

---

## Minimum Spanning Tree (MST) -- коротко

**Spanning tree** -- підграф, який з'єднує всі вершини і є деревом (V-1 ребер, без циклів). **Minimum ST** -- той з найменшою сумою ваг ребер.

**Use cases:** мережеві кабелі (мінімізувати довжину), cluster analysis, image segmentation.

### Kruskal (через Union-Find)

1. Сортуй всі ребра за вагою.
2. Проходиш по ребрах у зростаючому порядку. Якщо `union(u, v)` успішний -- додаєш ребро у MST.
3. Зупиняєшся, коли в MST V-1 ребер.

```javascript
function kruskal(n, edges) {
  edges.sort((a, b) => a[2] - b[2]);
  const uf = new UnionFind(n);
  const mst = [];
  for (const [u, v, w] of edges) {
    if (uf.union(u, v)) {
      mst.push([u, v, w]);
      if (mst.length === n - 1) break;
    }
  }
  return mst;
}
```

**Complexity:** O(E log E).

### Prim (через priority queue)

Схожий на Dijkstra, але пріоритет -- вага ребра, не кумулятивна відстань. Починаєш з однієї вершини, на кожному кроці додаєш найдешевше ребро, що веде назовні MST.

**Complexity:** O((V + E) log V).

На співбесідах MST питають рідше, ніж shortest path. Але знати, що Kruskal + Union-Find = pattern -- корисно.

---

## Типові пастки та помилки

1. **Забутий visited set.** Найчастіша помилка новачків. Без нього BFS/DFS на графі з циклами зациклюються. Завжди додавай його як першу дію у шаблоні.

2. **DFS для shortest path.** DFS не гарантує найкоротший шлях. Для shortest path в unweighted графі -- **BFS**. В weighted -- **Dijkstra** (або Bellman-Ford, якщо є від'ємні ваги).

3. **Рекурсивний DFS на великому графі.** Node.js має обмежений стек (~10-20k). Для графа з 10⁶ вершин ланцюжкового типу рекурсія впаде. Використовуй ітеративний DFS зі стеком.

4. **Плутанина directed vs undirected.** У шаблонах для undirected ти додаєш ребро в обидві сторони. Якщо працюєш з directed (Course Schedule) -- тільки одну. Помилка тут -- неправильний граф і неправильна відповідь.

5. **`queue.shift()` в BFS -- O(n).** На великих графах це критично повільно. Використовуй index-pointer або власну deque.

6. **Positioning visited у BFS.** Якщо позначати `visited` при **витягуванні** з черги (а не при додаванні), той самий вузол може потрапити в чергу кілька разів, а shortest path може бути неправильним. Завжди позначай **при додаванні** в чергу.

7. **Cycle detection у DAG vs undirected.**
   - У **directed** графі потрібен trick з кольорами (white/gray/black), бо проста перевірка "чи батько" не працює.
   - У **undirected** графі досить пам'ятати parent: якщо бачиш visited, що не parent -- цикл.

8. **Dijkstra з від'ємними вагами.** Не працює. Алгоритм припускає, що "коли вершину витягли з PQ -- її відстань фінальна". З від'ємними вагами це припущення ламається.

9. **Adjacency matrix для sparse графа.** Мільйон вершин? Матриця 10¹² комірок -- неможливо. Використовуй adjacency list.

10. **Не оновлювати in-degree в Kahn's algorithm.** Якщо забув `inDegree[next]--` -- алгоритм зависне або поверне неповний порядок.

11. **Для задач "all paths" -- DFS, не BFS.** BFS ефективний для shortest path, але перерахувати всі шляхи простіше через backtracking DFS.

12. **Path compression в Union-Find без union by rank.** Ок, але дерево може вирости, і find буде повільним у деяких патернах. Додавай обидва оптимізації.

---

## Ключові думки

- Граф = `V + E`. Усі complexity оцінюємо у цих термінах.
- **Adjacency list** -- дефолт для sparse графів. `Map<node, array>` або масив масивів.
- **BFS** -- черга, рівень-за-рівнем, shortest path в unweighted.
- **DFS** -- стек/рекурсія, вглибину, topological sort, cycle detection, backtracking.
- **Visited set** -- обов'язково. Позначай при додаванні в чергу для BFS.
- **Topological sort** -- для DAG. Kahn (BFS + in-degree) або DFS post-order.
- **Union-Find** -- для connected components і cycle detection в undirected графі. O(α(n)) per op.
- **Dijkstra** -- shortest path у weighted з positive ваги, через min-heap.
- **Bellman-Ford, A*, Kruskal, Prim** -- знай, що існують і для чого.

Графи виглядають складно, але 90% задач розв'язуються одним із 3-4 шаблонів: BFS, DFS, topological sort, Union-Find. Розберись з ними -- і більшість графових задач стають комфортними.

У наступному файлі розберемо сортування та бінарний пошук -- фундамент для ще одного великого класу задач.
