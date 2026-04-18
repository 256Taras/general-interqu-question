# Дерева (Trees)

## Що таке дерево

**Дерево** -- це ієрархічна структура даних, що складається з вузлів (nodes). Один вузол оголошений кореневим (root), а кожен вузол має посилання на свої "діти" (children). Відмінність від графа: у дереві немає циклів і кожен non-root вузол має рівно одного "батька".

Уяви генеалогічне дерево або структуру папок -- це все дерева.

```
                [ 1 ]            ← root (корінь)
               /  |  \
           [ 2 ][ 3 ][ 4 ]       ← children 1-го
           / \       |
        [ 5 ][ 6 ] [ 7 ]         ← глибше
              |
            [ 8 ]                ← leaf (листок)
```

### Базова термінологія (запам'ятай)

- **Node (вузол)** -- елемент дерева, що тримає значення і посилання на дітей.
- **Root (корінь)** -- єдиний вузол без батька. Початок дерева.
- **Leaf (листок)** -- вузол без дітей.
- **Parent (батько)** -- вузол, що вказує на інший. На схемі вище `1` -- батько `2`, `3`, `4`.
- **Child (дитина)** -- вузол, на який вказує інший. `2` -- дитина `1`.
- **Sibling (сусід)** -- вузли зі спільним батьком. `2`, `3`, `4` -- siblings.
- **Ancestor (предок)** -- будь-який вузол вище по шляху до root. Предки `8`: `6`, `2`, `1`.
- **Descendant (нащадок)** -- будь-який вузол нижче. Нащадки `2`: `5`, `6`, `8`.
- **Depth (глибина)** вузла -- відстань від root. Root має depth 0, його діти -- 1, і так далі.
- **Height (висота)** вузла -- відстань від вузла до найглибшого листка у його піддереві. Height дерева = height root-а.
- **Subtree (піддерево)** -- вузол разом з усіма своїми нащадками утворює піддерево.

```
            [ A ]        depth 0, height 2
           /     \
         [ B ]   [ C ]   depth 1
         / \
      [ D ] [ E ]        depth 2 (leaves)
```

Height дерева = 2 (шлях A → B → D або A → B → E).

---

## Чому дерева важливі

Дерева -- повсюди у computer science:

- **DOM** (HTML) -- дерево елементів. `document.body.children[0]` -- типовий обхід.
- **File system** -- папки і файли. `/Users/mak/Documents/...` -- шлях у дереві.
- **Databases** -- індекси у PostgreSQL, MySQL будуються на **B-tree** (а не на бінарному дереві). Це спеціальна багатодорожня структура для швидкого пошуку на диску.
- **Парсинг** -- Abstract Syntax Tree (AST). Коли Babel чи TypeScript компілятор читає твій код, він будує AST.
- **JSON, XML, YAML** -- це дерева.
- **Routing** у Express/Fastify -- часто trie-структура.
- **React Fiber** -- внутрішній стан React-а зберігається як дерево fiber-вузлів.
- **Heap** (пріоритетна черга) -- реалізується як бінарне дерево.
- **Git** -- commits утворюють граф, але його подерева та дерева файлів (tree objects) -- саме дерева.

Розуміти дерева -- значить розуміти половину backend інженерії.

---

## Binary Tree

**Бінарне дерево** -- дерево, у якому кожен вузол має **щонайбільше двох** дітей, званих зазвичай `left` і `right`.

```
            [ 1 ]
           /     \
        [ 2 ]   [ 3 ]
        /   \       \
     [ 4 ] [ 5 ]   [ 6 ]
```

Клас на JS:

```javascript
class TreeNode {
  constructor(val, left = null, right = null) {
    this.val = val;
    this.left = left;    // дитина ліворуч або null
    this.right = right;  // дитина праворуч або null
  }
}

// Побудова дерева з прикладу вище
const root = new TreeNode(1,
  new TreeNode(2, new TreeNode(4), new TreeNode(5)),
  new TreeNode(3, null, new TreeNode(6))
);
```

**Чому саме бінарне?** Багато задач природно мають два напрямки -- менше/більше, так/ні, ліво/право. Бінарні дерева легші за реалізацію та аналіз, ніж n-арні. Більшість класичних алгоритмів (BST, heap, segment tree) -- на бінарних деревах.

**Спеціальні види бінарних дерев:**
- **Full** -- кожен вузол має 0 або 2 дітей.
- **Complete** -- всі рівні заповнені зліва направо (у останньому -- можуть бути прогалини справа).
- **Perfect** -- повністю заповнене, всі leaves на одному рівні.
- **Balanced** -- висота піддерев відрізняється не більш ніж на 1 у кожному вузлі.

---

## Обходи дерева (Tree Traversals)

Обхід -- процес відвідування кожного вузла рівно один раз. Є дві великі групи:

1. **Depth-First Search (DFS)** -- "йдемо вглиб" перед тим як розглядати сусідів.
2. **Breadth-First Search (BFS)** -- "йдемо по рівнях" зверху вниз.

DFS поділяється на три варіанти залежно від того, **коли** ми обробляємо поточний вузол відносно його дітей: **pre-order**, **in-order**, **post-order**.

Використаємо одне й те саме дерево для всіх прикладів:

```
            [ 1 ]
           /     \
        [ 2 ]   [ 3 ]
        /   \       \
     [ 4 ] [ 5 ]   [ 6 ]
```

### Pre-order DFS (N → L → R)

"Спочатку вузол, потім ліве піддерево, потім праве."

**Порядок виведення: 1, 2, 4, 5, 3, 6**

Покрокова робота:

```
Крок 1: відвідуємо 1 → вивели 1
        далі → у ліве піддерево (2)

Крок 2: у вузлі 2 → вивели 2
        далі → у ліве піддерево (4)

Крок 3: у вузлі 4 → вивели 4
        лівий=null, правий=null → повертаємось у 2

Крок 4: у 2 → обходимо праве піддерево (5)
        у вузлі 5 → вивели 5
        дітей нема → повертаємось у 2, звідти у 1

Крок 5: у 1 → обходимо праве піддерево (3)
        у вузлі 3 → вивели 3
        лівий=null → перескакуємо, правий=6

Крок 6: у вузлі 6 → вивели 6
        дітей нема → повертаємось у 3, потім у 1

Кінець. Результат: 1, 2, 4, 5, 3, 6
```

Реалізація:

```javascript
// Рекурсивно
function preorder(node, result = []) {
  if (node === null) return result;
  result.push(node.val);           // спочатку вузол
  preorder(node.left, result);     // потім ліво
  preorder(node.right, result);    // потім право
  return result;
}

// Ітеративно через явний стек
function preorderIter(root) {
  if (!root) return [];
  const result = [];
  const stack = [root];

  while (stack.length) {
    const node = stack.pop();
    result.push(node.val);
    // Важливо: спочатку push правого, потім лівого.
    // Бо stack -- LIFO, і ми хочемо, щоб лівий обробився першим.
    if (node.right) stack.push(node.right);
    if (node.left) stack.push(node.left);
  }
  return result;
}
```

**Коли використовувати:** створення копії дерева, серіалізація (зберігаєш у порядку, який легко відновити), виведення ієрархії.

### In-order DFS (L → N → R)

"Ліве піддерево, потім вузол, потім праве."

**Порядок виведення: 4, 2, 5, 1, 3, 6**

Крок за кроком:

```
Починаємо з 1. Маємо перейти у ліве піддерево (2).
  У 2. Маємо перейти у ліве піддерево (4).
    У 4. Ліве=null → нічого не робимо.
         Виводимо 4.
         Праве=null → повертаємось у 2.
  У 2. Повернулись з лівого.
       Виводимо 2.
       Йдемо у праве піддерево (5).
    У 5. Ліве=null.
         Виводимо 5.
         Праве=null → повертаємось у 2, звідти у 1.
У 1. Повернулись з лівого.
     Виводимо 1.
     Йдемо у праве піддерево (3).
  У 3. Ліве=null.
       Виводимо 3.
       Праве=6 → йдемо у 6.
    У 6. Ліве=null.
         Виводимо 6.
         Праве=null → повертаємось у 3, у 1.

Результат: 4, 2, 5, 1, 3, 6
```

```javascript
function inorder(node, result = []) {
  if (node === null) return result;
  inorder(node.left, result);      // спочатку ліво
  result.push(node.val);           // потім вузол
  inorder(node.right, result);     // потім право
  return result;
}

// Ітеративно -- трохи трюкувато, бо треба спочатку дійти до самого лівого листка
function inorderIter(root) {
  const result = [];
  const stack = [];
  let curr = root;

  while (curr !== null || stack.length) {
    // Доходимо до самого лівого
    while (curr !== null) {
      stack.push(curr);
      curr = curr.left;
    }
    // Тепер виводимо
    curr = stack.pop();
    result.push(curr.val);
    // Переходимо у праве піддерево
    curr = curr.right;
  }
  return result;
}
```

**Коли використовувати:** у BST in-order дає відсортований порядок -- це ключова властивість. Якщо бачиш "повернути значення BST у відсортованому порядку" -- це in-order.

### Post-order DFS (L → R → N)

"Ліве піддерево, потім праве, потім вузол."

**Порядок виведення: 4, 5, 2, 6, 3, 1**

```
У 1. Йдемо у ліве (2).
  У 2. Йдемо у ліве (4).
    У 4. Ліве=null, Праве=null.
         Виводимо 4.
  У 2. Йдемо у праве (5).
    У 5. Ліве=null, Праве=null.
         Виводимо 5.
  У 2. Обидва піддерева оброблені.
       Виводимо 2.
У 1. Йдемо у праве (3).
  У 3. Ліве=null.
       Йдемо у праве (6).
    У 6. Виводимо 6.
  У 3. Обидва оброблені.
       Виводимо 3.
У 1. Виводимо 1.

Результат: 4, 5, 2, 6, 3, 1
```

```javascript
function postorder(node, result = []) {
  if (node === null) return result;
  postorder(node.left, result);
  postorder(node.right, result);
  result.push(node.val);           // вузол -- останнім
  return result;
}
```

**Коли використовувати:** видалення дерева (звільняєш дітей перед батьком), обчислення розмірів/висот піддерев (треба знати дітей, щоб порахувати батька), evaluation виразних дерев.

### Порівняння трьох DFS-обходів

```
            [ 1 ]
           /     \
        [ 2 ]   [ 3 ]
        /   \       \
     [ 4 ] [ 5 ]   [ 6 ]

Pre-order  (N-L-R): 1, 2, 4, 5, 3, 6
In-order   (L-N-R): 4, 2, 5, 1, 3, 6
Post-order (L-R-N): 4, 5, 2, 6, 3, 1
```

Мнемоніка: у назві обходу "N" (node) ставиться на місце, де обробляється вузол: pre = до дітей, in = між дітьми, post = після.

### BFS (Level-order traversal)

"Обходимо шар за шаром. Спочатку root, потім всі його діти, потім онуки, і так далі."

**Порядок виведення: 1, 2, 3, 4, 5, 6**

```
            [ 1 ]              ← Рівень 0: виводимо 1
           /     \
        [ 2 ]   [ 3 ]          ← Рівень 1: виводимо 2, 3
        /   \       \
     [ 4 ] [ 5 ]   [ 6 ]       ← Рівень 2: виводимо 4, 5, 6
```

Реалізація через **чергу (queue)**:

```javascript
function bfs(root) {
  if (!root) return [];
  const result = [];
  const queue = [root];             // у JS для черги просто масив + shift, або краще клас Deque

  while (queue.length) {
    const node = queue.shift();     // FIFO -- перший зайшов, перший вийшов
    result.push(node.val);
    if (node.left)  queue.push(node.left);
    if (node.right) queue.push(node.right);
  }
  return result;
}
```

**Увага:** `queue.shift()` у JS -- O(n) через зсув масиву. Для великих дерев це перетворює BFS на O(n²). Для продакшену використовуй indexed queue (масив + змінна `head`) або готову бібліотеку. Для інтерв'ю приймається `shift`, але проговори обмеження.

Щоб обходити **по рівнях** (знати, які вузли на одному рівні):

```javascript
function bfsByLevels(root) {
  if (!root) return [];
  const result = [];
  const queue = [root];

  while (queue.length) {
    const levelSize = queue.length;       // скільки вузлів на цьому рівні
    const level = [];
    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();
      level.push(node.val);
      if (node.left)  queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}

// Для нашого дерева: [[1], [2,3], [4,5,6]]
```

Трюк "зняти розмір рівня до початку ітерації" -- must-know для задач на рівні.

**Коли використовувати BFS:**
- Найкоротший шлях (у unweighted графі).
- Обхід по рівнях.
- "Найближчий" елемент до root з якоюсь властивістю.
- Обчислення min depth.

---

## Binary Search Tree (BST)

**BST** -- це бінарне дерево з особливим правилом:

> Для кожного вузла: всі значення в лівому піддереві **менші** за значення вузла, а всі в правому -- **більші**.

Приклад BST:

```
              [ 8 ]
             /     \
          [ 3 ]   [ 10 ]
          /   \        \
       [ 1 ] [ 6 ]    [ 14 ]
              / \       /
           [ 4 ][ 7 ] [ 13 ]
```

Перевір: 3 < 8 ✓, 10 > 8 ✓, 1 < 3 < 6 ✓, 4 < 6 < 7 ✓, 13 < 14 ✓. Все в порядку.

**Чому BST кайф?** Пошук, вставка, видалення -- **O(log n)** у середньому (якщо дерево збалансоване). А in-order обхід дає відсортовану послідовність: `1, 3, 4, 6, 7, 8, 10, 13, 14`.

### Пошук у BST

```javascript
function search(root, target) {
  if (root === null) return null;            // не знайшли
  if (root.val === target) return root;      // знайшли!
  if (target < root.val) return search(root.left, target);
  return search(root.right, target);
}
```

Логіка як у бінарному пошуку: якщо `target < node.val`, то він може бути тільки ліворуч. Кожен крок відкидаємо половину дерева -- звідси O(log n) в середньому.

### Вставка у BST

```javascript
function insert(root, val) {
  if (root === null) return new TreeNode(val);
  if (val < root.val) {
    root.left = insert(root.left, val);
  } else if (val > root.val) {
    root.right = insert(root.right, val);
  }
  // Якщо val === root.val -- ігноруємо (залежить від політики: чи дозволено дублікати)
  return root;
}
```

Новий вузол завжди вставляється як листок. Йдемо вниз, поки не знайдемо `null`-слот, і туди вставляємо.

### Видалення у BST (найскладніше)

Три випадки:

**Випадок 1:** вузол -- листок. Просто видаляємо.

**Випадок 2:** вузол має одну дитину. Замінюємо вузол його дитиною.

**Випадок 3:** вузол має двох дітей. Це найскладніше:
- Знаходимо **inorder successor** (найменший у правому піддереві) або **inorder predecessor** (найбільший у лівому).
- Копіюємо його значення у поточний вузол.
- Видаляємо цей successor (він уже легко видалиться -- не має лівої дитини за побудовою).

```javascript
function remove(root, val) {
  if (!root) return null;

  if (val < root.val) {
    root.left = remove(root.left, val);
  } else if (val > root.val) {
    root.right = remove(root.right, val);
  } else {
    // Знайшли вузол для видалення
    if (!root.left)  return root.right;   // 0 або 1 дитина (права)
    if (!root.right) return root.left;    // 1 дитина (ліва)

    // 2 дітей: знаходимо successor
    let successor = root.right;
    while (successor.left) successor = successor.left;
    root.val = successor.val;
    root.right = remove(root.right, successor.val);
  }
  return root;
}
```

### Чому середня глибина log(n), а найгірша n

Збалансоване BST на `n` вузлах має висоту ~`log₂(n)`. Наприклад, для 1 мільйона вузлів -- висота ~20. Пошук -- 20 кроків.

Але якщо ти вставиш у BST вже відсортовані числа `1, 2, 3, 4, 5, 6, 7`, отримаєш:

```
[ 1 ]
     \
    [ 2 ]
         \
        [ 3 ]
             \
            [ 4 ]
                 \
                ...
```

Це по суті **linked list**. Висота = n, пошук = O(n). Катастрофа.

Тому у продакшені використовують **self-balancing BSTs**: AVL, Red-Black, B-tree.

---

## Balanced trees (коротко)

### AVL Tree

Винайдено Адельсоном-Вельським і Ландісом (1962). Для кожного вузла: різниця висот лівого і правого піддерев не перевищує 1.

- Після вставки/видалення, якщо дерево розбалансувалось -- виконуються **ротації** (left rotation, right rotation, або їх комбінації).
- Висота гарантовано ~1.44 × log(n) -- дуже строго збалансоване.
- Дуже швидкий пошук. Вставка/видалення трохи повільніше через часті ротації.

### Red-Black Tree

Винайдено Рудольфом Байером. Менш строго збалансоване за AVL (висота до 2 × log(n)), але з меншими витратами на ротації.

Правила (5 штук, для загальної ерудиції):
1. Кожен вузол або red, або black.
2. Root -- black.
3. Всі leaves (null-вузли) -- black.
4. Якщо вузол red, обидва його діти -- black.
5. Від будь-якого вузла до його потомків-leaves міститься однакова кількість black-вузлів.

Ці правила разом гарантують приблизну балансованість.

**Де використовується:**
- **C++ `std::map`, `std::set`** -- red-black tree.
- **Java `TreeMap`, `TreeSet`** -- red-black.
- **Linux kernel** (Completely Fair Scheduler) -- red-black.

### B-Tree (та B+Tree)

Не бінарне! Кожен вузол може мати **багато** дітей (наприклад, до 500). Використовується у базах даних та файлових системах.

- Оптимізоване під читання з диска: кожне читання -- дорога операція, тому один "фат" вузол читається разом.
- Висота дуже мала навіть для мільярдів записів.
- PostgreSQL, MySQL InnoDB індекси -- B+tree.

**Для співбесіди:** знати, що існують, коли використовуються. Реалізовувати -- не треба, якщо тебе не питають спеціально.

---

## Класичні задачі на дерева

### 1. Max depth of binary tree

"Знайти максимальну глибину (висоту) дерева."

```javascript
function maxDepth(root) {
  if (!root) return 0;
  return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}
// Time: O(n), Space: O(h), де h -- висота дерева
```

Розбір:
- Base case: порожнє дерево -- глибина 0.
- Recursive: глибина = 1 (сам root) + макс глибина піддерев.

### 2. Min depth of binary tree

"Знайти мінімальну глибину -- шлях від root до найближчого листка."

```javascript
function minDepth(root) {
  if (!root) return 0;

  // Якщо вузол має тільки одну дитину, "порожню" сторону не рахуємо.
  if (!root.left)  return 1 + minDepth(root.right);
  if (!root.right) return 1 + minDepth(root.left);

  return 1 + Math.min(minDepth(root.left), minDepth(root.right));
}
```

**Пастка:** `Math.min(maxDepth(left), maxDepth(right))` для вузла з одним null-дитячим дає 0, що невірно -- "листок" має бути без **обох** дітей. Саме тому окремі гілки для `!root.left` і `!root.right`.

Альтернатива -- BFS: зупиняємось на першому листку. Це навіть ефективніше, бо не обходимо все дерево.

```javascript
function minDepthBFS(root) {
  if (!root) return 0;
  const queue = [[root, 1]];
  while (queue.length) {
    const [node, depth] = queue.shift();
    if (!node.left && !node.right) return depth;  // знайшли листок!
    if (node.left)  queue.push([node.left, depth + 1]);
    if (node.right) queue.push([node.right, depth + 1]);
  }
}
```

### 3. Same tree

"Два дерева однакові, якщо структура і значення збігаються."

```javascript
function isSameTree(p, q) {
  if (!p && !q) return true;           // обидва null -- однакові
  if (!p || !q) return false;          // один null, інший -- ні
  if (p.val !== q.val) return false;   // значення різні
  return isSameTree(p.left, q.left) && isSameTree(p.right, q.right);
}
```

### 4. Symmetric tree

"Дерево симетричне, якщо його дзеркальне відображення дорівнює йому ж."

```
      [ 1 ]
     /     \
   [ 2 ]  [ 2 ]     симетричне
   / \    / \
 [3] [4][4] [3]
```

```javascript
function isSymmetric(root) {
  if (!root) return true;
  return isMirror(root.left, root.right);
}

function isMirror(t1, t2) {
  if (!t1 && !t2) return true;
  if (!t1 || !t2) return false;
  return t1.val === t2.val
      && isMirror(t1.left, t2.right)    // лівий лівий vs правий правий
      && isMirror(t1.right, t2.left);   // лівий правий vs правий лівий
}
```

### 5. Invert binary tree

"Обернути дерево -- обмінюємо left і right у кожному вузлі."

```
Було:                     Стало:
    [ 1 ]                    [ 1 ]
   /     \                  /     \
[ 2 ]   [ 3 ]           [ 3 ]   [ 2 ]
 / \                              / \
[4][5]                         [5][4]
```

```javascript
function invertTree(root) {
  if (!root) return null;
  [root.left, root.right] = [invertTree(root.right), invertTree(root.left)];
  return root;
}
```

Історичне: Макс Говард (автор Homebrew) провалив це завдання на співбесіді в Google і твіт про це став мемом. Тепер питають усюди.

### 6. Validate BST (з min/max границями)

"Дано дерево. Чи є воно валідним BST?"

**Пастка:** просто перевірити `node.left.val < node.val < node.right.val` -- **неправильно**. Треба перевіряти діапазон.

```
      [ 10 ]
      /    \
   [ 5 ]  [ 15 ]
          /    \
       [ 6 ]  [ 20 ]
```

Це не BST! Бо `6 < 10`, але лежить у правому піддереві `10`. Локальна перевірка `15 > 10 і 6 < 15` пройде -- і функція скаже, що BST. Помилка.

Правильний підхід: передавати дозволений діапазон `[min, max]`.

```javascript
function isValidBST(root, min = -Infinity, max = Infinity) {
  if (!root) return true;
  if (root.val <= min || root.val >= max) return false;

  return isValidBST(root.left, min, root.val)
      && isValidBST(root.right, root.val, max);
}
```

Для root діапазон -- `(-∞, +∞)`. При спуску в ліве піддерево, верхня границя стає `root.val`. У праве -- нижня границя стає `root.val`.

Альтернатива: in-order обхід. Якщо послідовність строго зростає -- BST. Спокусливо, але треба обережно з пам'яттю.

### 7. Lowest Common Ancestor (LCA)

"Знайти найнижчого спільного предка двох вузлів `p` і `q`."

```
          [ 6 ]
         /     \
      [ 2 ]   [ 8 ]
      / \     / \
   [0] [4] [7] [9]
        / \
      [3][5]

LCA(3, 5) = 4
LCA(3, 8) = 6
LCA(7, 9) = 8
```

**Для BST (спрощений варіант):**

```javascript
function lcaBST(root, p, q) {
  if (p.val < root.val && q.val < root.val) return lcaBST(root.left, p, q);
  if (p.val > root.val && q.val > root.val) return lcaBST(root.right, p, q);
  return root;         // одне ліворуч, інше праворуч (або одне = root) -- root і є LCA
}
```

Логіка: якщо обидва менші за поточний вузол -- LCA десь ліворуч. Якщо обидва більші -- праворуч. Якщо по різні боки -- поточний вузол і є LCA.

**Для загального бінарного дерева:**

```javascript
function lca(root, p, q) {
  if (!root || root === p || root === q) return root;

  const left  = lca(root.left, p, q);
  const right = lca(root.right, p, q);

  if (left && right) return root;          // p і q у різних піддеревах → root -- LCA
  return left || right;                     // обидва у одному піддереві
}
```

Розбір:
- Якщо поточний вузол -- `p` чи `q`, це кандидат на LCA.
- Шукаємо у лівому і правому піддеревах.
- Якщо `p` знайшовся ліворуч, а `q` праворуч -- LCA -- поточний вузол.
- Інакше -- те, яке не null, і є LCA (поверне далі вверх).

### 8. Binary Tree Level Order Traversal

Уже розібрали у BFS вище. Типовий варіант:

```javascript
function levelOrder(root) {
  if (!root) return [];
  const result = [];
  const queue = [root];

  while (queue.length) {
    const levelSize = queue.length;
    const level = [];
    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();
      level.push(node.val);
      if (node.left)  queue.push(node.left);
      if (node.right) queue.push(node.right);
    }
    result.push(level);
  }
  return result;
}
```

Варіанти:
- Zigzag (парні рівні нормально, непарні -- навпаки) -- додай `reverse` або `unshift`.
- Right-side view (перший побачений вузол на кожному рівні зправа) -- беремо останній з кожного рівня.

### 9. Path Sum

"Чи існує шлях від root до листка, сума значень якого дорівнює `target`?"

```javascript
function hasPathSum(root, target) {
  if (!root) return false;
  if (!root.left && !root.right) return root.val === target;   // листок

  const remaining = target - root.val;
  return hasPathSum(root.left, remaining) || hasPathSum(root.right, remaining);
}
```

**Пастка:** перевіряти суму треба ТІЛЬКИ на листках. Якщо пишеш `if (target === root.val) return true` не тільки на листку -- отримаєш false positive, коли сума збіглася в середині дерева.

Варіант `pathSum II` -- повернути ВСІ такі шляхи. Тут уже backtracking:

```javascript
function pathSumAll(root, target) {
  const result = [];
  function dfs(node, remaining, path) {
    if (!node) return;
    path.push(node.val);
    remaining -= node.val;
    if (!node.left && !node.right && remaining === 0) {
      result.push([...path]);
    }
    dfs(node.left, remaining, path);
    dfs(node.right, remaining, path);
    path.pop();                                 // backtrack
  }
  dfs(root, target, []);
  return result;
}
```

### 10. Serialize and Deserialize Binary Tree (коротко -- Hard)

"Серіалізувати дерево у рядок так, щоб з нього можна було відновити точно те саме дерево."

Один з підходів -- **pre-order з маркерами null**:

```javascript
function serialize(root) {
  const result = [];
  function dfs(node) {
    if (!node) { result.push('#'); return; }
    result.push(node.val);
    dfs(node.left);
    dfs(node.right);
  }
  dfs(root);
  return result.join(',');
}

function deserialize(data) {
  const tokens = data.split(',');
  let idx = 0;
  function build() {
    if (tokens[idx] === '#') { idx++; return null; }
    const node = new TreeNode(parseInt(tokens[idx++]));
    node.left = build();
    node.right = build();
    return node;
  }
  return build();
}
```

Приклад:
```
    1             Serialize: "1,2,#,#,3,#,#"
   / \
  2   3
```

Альтернатива -- BFS-серіалізація, часто зустрічається у LeetCode ("[1,2,3,null,null,4,5]"). Pre-order простіший для розуміння.

Чому hard: треба чітко розрізняти null-вузли, обмінюватися стратегією між serialize/deserialize, і думати про edge cases.

---

## Trie (Prefix Tree)

**Trie** -- спеціальне дерево для ефективного пошуку рядків за префіксом. Кожен вузол представляє символ, шлях від root до вузла -- префікс.

Приклад trie для слів `["cat", "car", "cart", "dog"]`:

```
         [ root ]
         /      \
       c          d
       |          |
       a          o
      / \          \
     t*  r          g*
         |*
         t*

* означає "кінець слова"
```

- Шлях `c → a → t` дає "cat" (кінець слова).
- Шлях `c → a → r` дає "car" (кінець слова).
- Шлях `c → a → r → t` дає "cart".

### Структура

```javascript
class TrieNode {
  constructor() {
    this.children = new Map();     // char -> TrieNode
    this.isEndOfWord = false;
  }
}

class Trie {
  constructor() {
    this.root = new TrieNode();
  }

  insert(word) {
    let node = this.root;
    for (const ch of word) {
      if (!node.children.has(ch)) {
        node.children.set(ch, new TrieNode());
      }
      node = node.children.get(ch);
    }
    node.isEndOfWord = true;
  }

  search(word) {
    const node = this._traverse(word);
    return node !== null && node.isEndOfWord;
  }

  startsWith(prefix) {
    return this._traverse(prefix) !== null;
  }

  _traverse(str) {
    let node = this.root;
    for (const ch of str) {
      if (!node.children.has(ch)) return null;
      node = node.children.get(ch);
    }
    return node;
  }
}

// Використання:
const trie = new Trie();
trie.insert("cat");
trie.insert("car");
trie.insert("cart");
trie.search("cat");        // true
trie.search("ca");         // false (ca -- тільки префікс)
trie.startsWith("ca");     // true
trie.search("carpet");     // false
```

### Autocomplete -- класичне застосування

"Для префіксу повернути всі слова, що починаються з нього."

```javascript
function autocomplete(trie, prefix) {
  const node = trie._traverse(prefix);
  if (!node) return [];

  const result = [];
  function dfs(curr, path) {
    if (curr.isEndOfWord) result.push(path);
    for (const [ch, child] of curr.children) {
      dfs(child, path + ch);
    }
  }
  dfs(node, prefix);
  return result;
}
```

### Коли використовувати Trie

- **Autocomplete / typeahead** -- пошукові рядки, IDE.
- **Spell checker**.
- **IP routing** -- збіг префіксу адреси (використовується в мережевих роутерах).
- **Словникові задачі** на співбесідах -- "знайди всі слова на дошці" (Word Search II).

**Trade-off:**
- Time: вставка/пошук слова довжиною `m` -- O(m). Не залежить від загальної кількості слів!
- Space: O(сумарна довжина всіх слів × розмір алфавіту). Може бути memory-heavy.

Альтернатива для простих випадків -- звичайний Set рядків (O(1) пошук повного слова, але немає швидкого `startsWith`).

---

## Типові пастки

### 1. Не перевіряти null перед доступом до .left / .right

```javascript
// ❌
function maxVal(root) {
  return Math.max(root.val, maxVal(root.left), maxVal(root.right));
  // TypeError, коли root === null
}

// ✅
function maxVal(root) {
  if (!root) return -Infinity;
  return Math.max(root.val, maxVal(root.left), maxVal(root.right));
}
```

**Правило:** будь-яка рекурсивна tree-функція **починається** з `if (!node) return ...;`.

### 2. Validate BST: порівняння тільки з parent замість діапазону

Вже розбирали у задачі про валідацію BST. Ключ -- передавати `min` та `max` уздовж рекурсії.

### 3. BFS без відстеження рівнів, коли треба по рівнях

```javascript
// ❌ Не знаємо, де закінчується рівень
while (queue.length) {
  const node = queue.shift();
  result.push(node.val);
  // результат -- плаский масив, без поділу на рівні
}

// ✅ Фіксуємо розмір рівня ДО початку ітерації
while (queue.length) {
  const size = queue.length;
  const level = [];
  for (let i = 0; i < size; i++) {
    const node = queue.shift();
    level.push(node.val);
    // ...
  }
  result.push(level);
}
```

### 4. Memory для deep recursion

Дерево може мати висоту `n` у найгіршому випадку (degenerate chain). Рекурсія з'їсть `n` frames. Для великих дерев (100k вузлів) -- stack overflow.

Розв'язок: переписати на ітеративний обхід з явним стеком, або використати Morris traversal (O(1) space, але складний).

### 5. Модифікація дерева, коли цього не хочеться

```javascript
// ❌ Ця функція мутує дерево
function invert(root) {
  if (!root) return null;
  [root.left, root.right] = [root.right, root.left];
  invert(root.left);
  invert(root.right);
  return root;
}
```

Ти повернув дерево, але також зіпсував вхід. Якщо потрібна **чиста** функція -- створюй нові вузли:

```javascript
function invertPure(root) {
  if (!root) return null;
  return new TreeNode(root.val, invertPure(root.right), invertPure(root.left));
}
```

### 6. Рекурсія для серіалізації без правильних null-маркерів

Якщо ти серіалізуєш дерево без маркерів null, різні дерева можуть дати один рядок:

```
[1, 2, 3]    і    [1, 3]
                   /
                  2
```

Без null-маркерів невідомо, хто ліво, хто право. Завжди додавай "#" для null.

### 7. Non-binary дерева -- забуваєш обійти всіх дітей

Для n-арних дерев `node.children` -- масив, а не `left`/`right`.

```javascript
function dfs(node) {
  if (!node) return;
  // робота...
  for (const child of node.children) {   // не забудь про цей цикл
    dfs(child);
  }
}
```

---

## Як підступатися до tree-задач на співбесіді

1. **Зрозумій структуру дерева.** Бінарне? N-арне? BST? З посиланням на parent?
2. **Вирішити DFS чи BFS.**
   - Треба по рівнях / найкоротший шлях → BFS.
   - Треба пройти всі піддерева і зібрати щось знизу → DFS (post-order).
   - Треба глибина / висота / сума → DFS (часто рекурсія).
3. **Визнач, яку інформацію передавати вниз, а яку піднімати.**
   - Приклад "вниз": діапазон `[min, max]` у `isValidBST`.
   - Приклад "вгору": max depth, sum.
4. **Проговори час і пам'ять.**
   - Time зазвичай O(n) -- треба відвідати кожен вузол.
   - Space -- O(h) для DFS рекурсії, O(w) для BFS (w -- макс ширина).
5. **Edge cases:** `root === null`, дерево з одного вузла, повністю лівий/правий "list".

---

## Шпаргалка складностей

| Операція                   | Balanced BST | Unbalanced (worst)  |
|----------------------------|--------------|---------------------|
| Search                     | O(log n)     | O(n)                |
| Insert                     | O(log n)     | O(n)                |
| Delete                     | O(log n)     | O(n)                |
| DFS / BFS обхід            | O(n)         | O(n)                |

| Операція                   | Trie (слово довжини m, алфавіт A) |
|----------------------------|-----------------------------------|
| Insert                     | O(m)                              |
| Search                     | O(m)                              |
| StartsWith                 | O(m)                              |
| Space                      | O(сума довжин слів × A)           |

| Обхід           | Time   | Space (рекурсія)      |
|-----------------|--------|-----------------------|
| Pre/in/post DFS | O(n)   | O(h)                  |
| BFS             | O(n)   | O(w) -- макс ширина   |

---

## Ключові думки

- Дерево -- ієрархічна структура: root, nodes, leaves, parents, children.
- Бінарне дерево -- не більше 2 дітей. Клас `TreeNode { val, left, right }`.
- Три DFS-обходи: pre (N-L-R), in (L-N-R), post (L-R-N). У BST in-order дає сортований порядок.
- BFS (level-order) -- через чергу. Для обходу по рівнях фіксуй `queue.length` до ітерації.
- BST: ліве < вузол < праве. Пошук/вставка O(log n) в середньому, O(n) у worst case.
- Self-balancing trees (AVL, Red-Black, B-tree) уникають worst case. У продакшені всюди.
- Для tree-рекурсії завжди пиши `if (!node) return ...;` першим рядком.
- Validate BST -- через передачу діапазону `[min, max]`, не через локальне порівняння.
- LCA у BST -- порівняння значень; у загальному дереві -- подивись, звідки прийшли `p` і `q`.
- Trie -- ідеальна структура для autocomplete, префіксних пошуків, spell check.
- На співбесіді завжди проговорюй time і space: O(n) time + O(h) space для DFS.

У наступних файлах розберемо графи і dynamic programming -- теми, що глибоко використовують ідеї з рекурсії та обходів.
