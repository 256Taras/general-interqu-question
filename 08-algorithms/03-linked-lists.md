# Linked Lists: зв'язані списки від нуля

## Що таке linked list?

**Linked list (зв'язаний список)** -- це структура даних, де елементи (вузли) зберігаються НЕ у неперервному блоці пам'яті, а розкидані по пам'яті і пов'язані між собою через посилання (pointers).

Кожен вузол (node) містить:
1. **Значення** (value, data) -- те, що ми зберігаємо.
2. **Посилання на наступний вузол** (next).

```
  head
   │
   ▼
 ┌─────┬──────┐    ┌─────┬──────┐    ┌─────┬──────┐    ┌─────┬──────┐
 │  1  │ next─┼───▶│  2  │ next─┼───▶│  3  │ next─┼───▶│  4  │ null │
 └─────┴──────┘    └─────┴──────┘    └─────┴──────┘    └─────┴──────┘
```

Останній вузол вказує на `null` -- це ознака кінця списку. `head` -- це посилання на перший вузол; сам список в JS зазвичай зберігається саме як `head`.

---

## Порівняння з масивом

Масив -- це **неперервний блок пам'яті**. Якщо ми кажемо "масив з 5 чисел", двигун виділяє один шматок пам'яті достатнього розміру, і елементи лежать поруч.

```
Масив в пам'яті:
  [ 10 ][ 20 ][ 30 ][ 40 ][ 50 ]
   ^
   базова адреса, наприклад 0x1000

Щоб дістати arr[3]:
  адреса = базова + 3 × розмір_елемента
  → O(1), бо це просто арифметика
```

Linked list -- це **ланцюг вузлів**, розкиданих по пам'яті. Кожен вузол окремо виділений (`new Node(...)`), а зв'язок -- через `.next`.

```
Linked list в пам'яті (вузли можуть бути де завгодно):

  0x1000: Node(10) next=0x3400
  0x3400: Node(20) next=0x2100
  0x2100: Node(30) next=0x7700
  0x7700: Node(40) next=null

Щоб дістати 3-й елемент:
  треба пройти: head → node1 → node2 → node3
  → O(n)
```

### Коли що краще?

| Операція                          | Array                | Linked List      |
|-----------------------------------|----------------------|------------------|
| Доступ за індексом `arr[i]`       | **O(1)**             | O(n)             |
| Пошук значення                    | O(n)                 | O(n)             |
| Вставка в початок                 | O(n) (зсув всього)   | **O(1)**         |
| Видалення з початку               | O(n)                 | **O(1)**         |
| Вставка в кінець                  | O(1) amortized       | O(1)* або O(n)   |
| Вставка в середину (при наявності посилання на вузол) | O(n) | **O(1)** |
| Пам'ять                           | Компактно            | +указівники      |

*O(1) якщо зберігаємо `tail`, інакше O(n) -- треба пройти до кінця.

**Коли linked list краще:**
- Часта вставка/видалення в початок або середину (коли вже маємо посилання на вузол).
- Розмір невідомий і сильно змінюється (не потрібна реалокація блоків).
- Імплементація черг, стеків, LRU-кешу, undo-історії.

**Коли масив краще:**
- Треба швидкий random access (`arr[42]`).
- Важлива кеш-локальність (CPU любить неперервну пам'ять).
- Знаєш розмір наперед, робиш багато обходів підряд.

**У реальному Node.js** масив майже завжди виграє. Linked list корисний як **концепція** для задач та як фундамент для складніших структур (hash map з chaining, adjacency list у графах, LRU cache).

---

## Singly Linked List vs Doubly Linked List

### Singly (однобічний)

Кожен вузол має тільки `next`. Можна рухатись ТІЛЬКИ вперед.

```
 head
  │
  ▼
┌───┬──┐   ┌───┬──┐   ┌───┬──┐
│ A │ ─┼─▶│ B │ ─┼─▶│ C │ ▪ │   ▪ = null
└───┴──┘   └───┴──┘   └───┴──┘
```

**Плюси:** менше пам'яті, простіша реалізація.
**Мінуси:** не можна повернутись назад. Видалення вузла вимагає знати попередній вузол.

### Doubly (двобічний)

Кожен вузол має `next` І `prev`. Можна рухатись в обидві сторони.

```
         head                                            tail
          │                                                │
          ▼                                                ▼
 null ◀─┬───┬──┐   ┌──┬───┬──┐   ┌──┬───┬──┐   ┌──┬───┬─┐─▶ null
        │ A │ ─┼─▶│─▶│ B │ ─┼─▶│─▶│ C │ ─┼─▶│─▶│ D │   │
        └───┴──┘  └──┴───┴──┘  └──┴───┴──┘  └──┴───┴───┘
            ◀──────────     ◀──────────     ◀──────────
                 prev            prev             prev
```

**Плюси:** обхід в обидві сторони, видалення вузла за O(1) без знання попередника.
**Мінуси:** більше пам'яті (+1 поле на вузол), більше коду (треба оновлювати і `next`, і `prev`).

**Реальний приклад:** у Node.js стандартної linked list немає, але `Map` всередині V8 використовує зв'язаний список insert-order. Undo/redo стеки, кеші (LRU), плейлисти -- класичні use cases для doubly linked list.

---

## Базова реалізація на JS (Singly Linked List)

Починаємо з класу `Node`.

```javascript
class Node {
  constructor(value) {
    this.value = value;
    this.next = null;      // за замовчуванням -- кінець списку
  }
}
```

Тепер сам список. Зберігатимемо `head`, `tail` і `length` -- це стандартний підхід, що дає O(1) додавання в кінець і O(1) `.size`.

```javascript
class LinkedList {
  constructor() {
    this.head = null;
    this.tail = null;
    this.length = 0;
  }
}
```

### push -- додати в кінець

```javascript
push(value) {
  const node = new Node(value);

  if (!this.head) {
    // Порожній список -- новий вузол стає і головою, і хвостом
    this.head = node;
    this.tail = node;
  } else {
    // Неочевидно: чому просто не this.tail = node?
    // Тому що нам ТРЕБА щоб попередній хвіст вказував на новий вузол.
    this.tail.next = node;
    this.tail = node;
  }

  this.length++;
  return this;
}
```

Діаграма:
```
До push(D):
  head ─▶ [A] ─▶ [B] ─▶ [C] ◀─ tail

Після push(D):
  head ─▶ [A] ─▶ [B] ─▶ [C] ─▶ [D] ◀─ tail
```

**Big O:** O(1), бо маємо прямий доступ до `tail`.

### pop -- видалити з кінця

Тут хитрість: навіть маючи `tail`, ми не знаємо, хто ПОПЕРЕДНИК хвоста (у singly linked list `prev` немає). Доведеться пройти список, щоб знайти передостанній вузол.

```javascript
pop() {
  if (!this.head) return undefined;

  // Єдиний вузол -- очищуємо список
  if (this.length === 1) {
    const only = this.head;
    this.head = null;
    this.tail = null;
    this.length = 0;
    return only;
  }

  // Знаходимо передостанній вузол
  let current = this.head;
  while (current.next !== this.tail) {
    current = current.next;
  }

  const removed = this.tail;
  current.next = null;       // обірвати зв'язок
  this.tail = current;       // новий хвіст
  this.length--;

  return removed;
}
```

**Big O:** O(n), бо треба пройти до передостаннього. Це головна причина, чому для частих pop-ів використовують **doubly** linked list (там pop = O(1)).

### unshift -- додати в початок

```javascript
unshift(value) {
  const node = new Node(value);

  if (!this.head) {
    this.head = node;
    this.tail = node;
  } else {
    node.next = this.head;   // новий вузол вказує на старий head
    this.head = node;         // новий вузол стає головою
  }

  this.length++;
  return this;
}
```

```
До unshift(X):
  head ─▶ [A] ─▶ [B] ─▶ [C]

Після unshift(X):
  head ─▶ [X] ─▶ [A] ─▶ [B] ─▶ [C]
```

**Big O:** O(1). Це одна з головних переваг linked list над масивом (`arr.unshift` -- O(n)!).

### shift -- видалити з початку

```javascript
shift() {
  if (!this.head) return undefined;

  const removed = this.head;
  this.head = this.head.next;
  this.length--;

  if (this.length === 0) {
    // Видалили єдиний вузол -- треба обнулити і tail
    this.tail = null;
  }

  return removed;
}
```

**Big O:** O(1). Теж сильно швидше за `arr.shift` -- O(n).

### get -- отримати вузол за індексом

```javascript
get(index) {
  if (index < 0 || index >= this.length) return null;

  let current = this.head;
  let i = 0;
  while (i < index) {
    current = current.next;
    i++;
  }
  return current;
}
```

**Big O:** O(n). Це ключовий недолік linked list -- немає random access.

### insert -- вставити за індексом

```javascript
insert(index, value) {
  if (index < 0 || index > this.length) return false;

  // Спеціальні випадки: на краях -- O(1) через unshift/push
  if (index === 0) return !!this.unshift(value);
  if (index === this.length) return !!this.push(value);

  const prev = this.get(index - 1);   // O(n)
  const node = new Node(value);
  node.next = prev.next;
  prev.next = node;

  this.length++;
  return true;
}
```

Діаграма вставки в середину:
```
Було (вставляємо X між B і C):
  [A] ─▶ [B] ─▶ [C] ─▶ [D]
          │
          prev

Створюємо X.next = prev.next:
  [A] ─▶ [B] ─▶ [C] ─▶ [D]
                ▲
          [X] ──┘

Переставляємо prev.next = X:
  [A] ─▶ [B] ─▶ [X] ─▶ [C] ─▶ [D]
```

**Big O:** O(n) через пошук `prev`. Якби в нас уже був вказівник на попередній вузол (часта ситуація в алгоритмах) -- саме переставляння посилань це O(1).

### remove -- видалити за індексом

```javascript
remove(index) {
  if (index < 0 || index >= this.length) return undefined;

  if (index === 0) return this.shift();
  if (index === this.length - 1) return this.pop();

  const prev = this.get(index - 1);   // O(n)
  const removed = prev.next;
  prev.next = removed.next;

  this.length--;
  return removed;
}
```

Діаграма:
```
Було:
  [A] ─▶ [B] ─▶ [C] ─▶ [D]
          │      │
          prev   removed

Перекидаємо prev.next на removed.next:
  [A] ─▶ [B] ─────────▶ [D]
                 [C]  (залишився ізольованим -- GC прибере)
```

**Big O:** O(n).

### Швидкий підсумок складності (Singly)

| Операція | Big O |
|----------|-------|
| push     | O(1)  |
| pop      | **O(n)** (!) |
| unshift  | O(1)  |
| shift    | O(1)  |
| get(i)   | O(n)  |
| insert   | O(n)  |
| remove   | O(n)  |
| search   | O(n)  |

### Doubly Linked List -- ключові відмінності

Вузол має ще `prev`:
```javascript
class DoublyNode {
  constructor(value) {
    this.value = value;
    this.next = null;
    this.prev = null;
  }
}
```

`pop` стає O(1):
```javascript
pop() {
  if (!this.tail) return undefined;

  const removed = this.tail;
  if (this.length === 1) {
    this.head = null;
    this.tail = null;
  } else {
    this.tail = removed.prev;   // є прямий доступ до prev!
    this.tail.next = null;
    removed.prev = null;         // розірвати, щоб GC спрацював
  }

  this.length--;
  return removed;
}
```

---

## Pattern 1: Two Pointers / Fast and Slow

Найпотужніший pattern для задач на linked list. Ідея: тримаємо **два вказівники**, рухаємо з різною швидкістю або на різних позиціях.

### Приклад A: знайти середину списку

Наївно: пройтись, порахувати довжину n, пройтись ще раз до n/2. Два проходи.

Elegant: **slow крокує по 1, fast крокує по 2**. Коли fast дійде до кінця -- slow буде посередині.

```javascript
function middleNode(head) {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow.next;
    fast = fast.next.next;
  }

  return slow;
}
```

Візуалізація на `[1, 2, 3, 4, 5]`:
```
Крок 0:  [1] [2] [3] [4] [5]
          ↑
        slow, fast

Крок 1:  [1] [2] [3] [4] [5]
              ↑       ↑
            slow     fast

Крок 2:  [1] [2] [3] [4] [5]
                  ↑       ↑
                slow     fast  (fast.next = null → стоп)

Повертаємо slow → вузол 3. ✔
```

На парному списку `[1, 2, 3, 4]`:
```
Крок 0: slow=1, fast=1
Крок 1: slow=2, fast=3
Крок 2: slow=3, fast=null (fast.next.next був би null.next → стоп раніше)
Повертаємо 3 (для парного списку це "другий із двох середніх" -- стандартна умова).
```

**Big O:** O(n) time, O(1) space. Один прохід.

### Приклад B: знайти n-й з кінця

Задача: повернути n-й вузол з кінця (наприклад, 2-й з кінця = передостанній).

Наївно: знайти довжину, потім пройти length - n. Два проходи.

Elegant: **fast стартує на n вузлів попереду slow**. Коли fast дійде до кінця -- slow стоїть на n-му з кінця.

```javascript
function nthFromEnd(head, n) {
  let slow = head;
  let fast = head;

  // Просуваємо fast на n кроків вперед
  for (let i = 0; i < n; i++) {
    if (!fast) return null;    // список коротший за n
    fast = fast.next;
  }

  // Тепер рухаємо обидва, поки fast не дійде до null
  while (fast !== null) {
    slow = slow.next;
    fast = fast.next;
  }

  return slow;
}
```

Візуалізація на `[1, 2, 3, 4, 5]`, n=2:
```
Старт:         [1] [2] [3] [4] [5]
                ↑
              slow=fast

Після просування fast на 2:
                [1] [2] [3] [4] [5]
                ↑       ↑
              slow    fast

Рухаємо обидва:
                [1] [2] [3] [4] [5]
                    ↑       ↑
                  slow    fast

                [1] [2] [3] [4] [5]
                        ↑       ↑
                      slow    fast

                [1] [2] [3] [4] [5]
                            ↑       ↑
                          slow    fast=null → стоп

Повертаємо slow → вузол 4, що є 2-м з кінця. ✔
```

**Big O:** O(n), **один прохід**.

### Приклад C: детекція циклу (Floyd's Tortoise and Hare)

Задача: перевірити, чи в списку є цикл (якийсь вузол вказує назад на попередній).

```
head ─▶ [1] ─▶ [2] ─▶ [3] ─▶ [4] ─▶ [5]
                       ▲              │
                       └──────────────┘   ← цикл!
```

**Ідея:** slow по 1, fast по 2. Якщо циклу немає -- fast досягне `null`. Якщо цикл є -- fast колись "наздожене" slow (як у круглій біговій доріжці).

```javascript
function hasCycle(head) {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow.next;
    fast = fast.next.next;

    if (slow === fast) return true;   // зустрілись у циклі
  }

  return false;   // fast дійшов до null → циклу немає
}
```

**Чому це працює?** Уяви коло. Якщо один біжить по 1 м/с, а інший по 2 м/с -- другий щосекунди скорочує дистанцію між ними на 1 м. Рано чи пізно він наздожене першого. У списку те саме: кожну ітерацію fast наближається до slow на 1 крок (відносно), тому через ≤ довжину циклу ітерацій вони зустрінуться.

**Big O:** O(n) time, O(1) space. Альтернатива -- Set з відвіданими вузлами -- теж O(n) time, але O(n) space.

### Bonus: знайти ПОЧАТОК циклу

Коли виявили зустріч, є елегантний трюк: переносимо один з вказівників на head і рухаємо обидва по 1 кроку. Місце зустрічі -- початок циклу.

```javascript
function detectCycleStart(head) {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow === fast) {
      // Фаза 2: знайти початок
      let ptr = head;
      while (ptr !== slow) {
        ptr = ptr.next;
        slow = slow.next;
      }
      return ptr;
    }
  }

  return null;
}
```

**Чому працює?** Нехай L -- відстань від head до початку циклу, C -- довжина циклу, x -- позиція зустрічі всередині циклу. Можна показати математично, що `L ≡ C - x (mod C)`. Переміщення обох на 1 з точки head і точки зустрічі призводить до зустрічі рівно на початку циклу. На співбесіді не обов'язково пам'ятати доказ -- достатньо знати факт.

---

## Pattern 2: Reverse a Linked List (найпопулярніша задача)

Задача: перевернути список, `[1,2,3,4]` -> `[4,3,2,1]`.

### Ітеративно (класика)

Ідея: йдемо по списку і на кожному кроці перевертаємо стрілку поточного вузла назад.

```javascript
function reverse(head) {
  let prev = null;
  let current = head;

  while (current !== null) {
    const next = current.next;   // запам'ятовуємо наступний ДО перевороту
    current.next = prev;          // перевертаємо стрілку
    prev = current;               // зсуваємо prev
    current = next;               // зсуваємо current
  }

  return prev;   // коли current = null, prev -- новий head
}
```

Покрокова ASCII-діаграма на `[1, 2, 3]`:

```
Початок:
  head ─▶ [1] ─▶ [2] ─▶ [3] ─▶ null
          ↑
       current
  prev = null

─── Крок 1 ───
  next = [2]                      (зберегли)
  [1].next = null                 (перевернули)
  prev = [1]
  current = [2]

  null ◀─ [1]       [2] ─▶ [3] ─▶ null
           ↑         ↑
          prev    current

─── Крок 2 ───
  next = [3]
  [2].next = [1]                  (перевернули)
  prev = [2]
  current = [3]

  null ◀─ [1] ◀─ [2]       [3] ─▶ null
                  ↑         ↑
                prev      current

─── Крок 3 ───
  next = null
  [3].next = [2]
  prev = [3]
  current = null

  null ◀─ [1] ◀─ [2] ◀─ [3]
                         ↑
                       prev      (current=null → вихід з циклу)

Повертаємо prev → новий head = [3]. ✔
```

**Big O:** O(n) time, O(1) space.

### Рекурсивно

```javascript
function reverseRecursive(head) {
  // База: порожній список або єдиний вузол
  if (head === null || head.next === null) return head;

  // Розвертаємо "хвіст" (все після head)
  const newHead = reverseRecursive(head.next);

  // Тепер head.next -- останній вузол реверснутого хвоста.
  // Треба щоб він вказував назад на head.
  head.next.next = head;
  head.next = null;        // head стає новим хвостом

  return newHead;
}
```

На `[1, 2, 3]`:
```
reverseRecursive([1,2,3])
  reverseRecursive([2,3])
    reverseRecursive([3])
      return [3]                      ← база
    head=[2], head.next=[3]
    [3].next = [2]  →  [3] ─▶ [2]
    [2].next = null →  [3] ─▶ [2] ─▶ null
    return [3]
  head=[1], head.next=[2]
  [2].next = [1]  →  [3] ─▶ [2] ─▶ [1]
  [1].next = null →  [3] ─▶ [2] ─▶ [1] ─▶ null
  return [3]
```

**Big O:** O(n) time, **O(n) space** (стек рекурсії). На довгих списках можна отримати stack overflow -- в Node.js ліміт ~10k-15k викликів. Ітеративна версія безпечніша.

---

## Pattern 3: Merge Two Sorted Lists

Задача: маємо два відсортовані списки, об'єднати в один відсортований.

```
L1:  1 → 3 → 5
L2:  2 → 4 → 6
────────────────
Out: 1 → 2 → 3 → 4 → 5 → 6
```

**Ідея:** два вказівники, беремо менший з голів, "зшиваємо" в новий список.

Тут зручно використати **dummy node** (див. наступну секцію) -- це сильно спрощує код.

```javascript
function mergeTwoLists(l1, l2) {
  const dummy = new Node(0);   // фіктивний вузол-заглушка
  let tail = dummy;

  while (l1 !== null && l2 !== null) {
    if (l1.value <= l2.value) {
      tail.next = l1;
      l1 = l1.next;
    } else {
      tail.next = l2;
      l2 = l2.next;
    }
    tail = tail.next;
  }

  // Прикріпити залишок (один зі списків ще не вичерпаний)
  tail.next = l1 || l2;

  return dummy.next;   // пропускаємо dummy
}
```

Візуалізація одного кроку:
```
L1: [1] ─▶ [3] ─▶ [5]
L2: [2] ─▶ [4] ─▶ [6]

dummy ─▶ null            tail = dummy

Крок 1: 1 < 2, беремо з L1
dummy ─▶ [1] ─▶ ?        tail = [1], L1 = [3]...

Крок 2: 3 > 2, беремо з L2
dummy ─▶ [1] ─▶ [2] ─▶ ? tail = [2], L2 = [4]...

... і так далі
```

**Big O:** O(n + m) time, O(1) space (не створюємо нових вузлів -- реюзаємо існуючі).

---

## Pattern 4: Remove Nth Node From End (один прохід)

Задача: видалити n-й вузол з кінця. Приклад: `[1,2,3,4,5]`, n=2 → `[1,2,3,5]`.

Наївне рішення -- два проходи (порахувати довжину, потім видалити). Одним проходом робимо через fast/slow.

**Ключ:** нам треба знайти **попередник** того, кого видаляємо. Тому fast стартує на n+1 попереду slow. А щоб обробити випадок видалення голови, використовуємо dummy.

```javascript
function removeNthFromEnd(head, n) {
  const dummy = new Node(0);
  dummy.next = head;

  let slow = dummy;
  let fast = dummy;

  // fast робить n+1 крок вперед
  for (let i = 0; i <= n; i++) {
    fast = fast.next;
  }

  // рухаємо обидва, поки fast не стане null
  while (fast !== null) {
    slow = slow.next;
    fast = fast.next;
  }

  // slow тепер перед видаляємим
  slow.next = slow.next.next;

  return dummy.next;
}
```

На `[1,2,3,4,5]`, n=2:
```
dummy ─▶ [1] ─▶ [2] ─▶ [3] ─▶ [4] ─▶ [5] ─▶ null

Після fast += 3 (n+1=3):
  slow = dummy
  fast = [3]

  dummy ─▶ [1] ─▶ [2] ─▶ [3] ─▶ [4] ─▶ [5]
   ↑                      ↑
  slow                   fast

Рухаємо обидва:
  slow = [1], fast = [4]
  slow = [2], fast = [5]
  slow = [3], fast = null → стоп

  dummy ─▶ [1] ─▶ [2] ─▶ [3] ─▶ [4] ─▶ [5]
                          ↑       ▲
                         slow    видаляємо

slow.next = slow.next.next  →  [3].next = [5]

Результат: dummy ─▶ [1] ─▶ [2] ─▶ [3] ─▶ [5]
```

**Big O:** O(n) time, O(1) space, **один прохід**.

---

## Pattern 5: Dummy (Sentinel) Node

**Dummy node** -- це фіктивний вузол перед справжньою головою. Його єдина мета -- спростити код у випадках, коли голова може змінитись.

Без dummy:
```javascript
function removeValue(head, target) {
  // Випадок 1: треба видалити голову (окрема гілка коду)
  while (head !== null && head.value === target) {
    head = head.next;
  }
  if (head === null) return null;

  // Випадок 2: видалити не-голову
  let current = head;
  while (current.next !== null) {
    if (current.next.value === target) {
      current.next = current.next.next;
    } else {
      current = current.next;
    }
  }
  return head;
}
```

З dummy:
```javascript
function removeValue(head, target) {
  const dummy = new Node(0);
  dummy.next = head;

  let current = dummy;
  while (current.next !== null) {
    if (current.next.value === target) {
      current.next = current.next.next;
    } else {
      current = current.next;
    }
  }
  return dummy.next;
}
```

Код коротший на третину, і ми не дублюємо логіку для голови.

**Правило:** як тільки задача "може видалити голову" або "може вставити нову голову" -- одразу заводь dummy. Це економить години дебагу.

---

## Типові пастки

### 1. Null check на `.next`

```javascript
// ❌ Впаде на останньому вузлі, бо current.next = null → null.next undefined
while (current.next.next !== null) { ... }

// ✅ Завжди перевіряй КОЖНУ ланку ланцюга
while (current !== null && current.next !== null) { ... }
```

У Floyd's algorithm: `fast.next.next` вимагає і `fast`, і `fast.next` не null. Інакше -- TypeError.

### 2. Off-by-one у fast/slow

Задача "знайти середину" -- чи повертати перше з двох середніх чи друге? Залежить від умови циклу:

```javascript
// while (fast && fast.next)            → середина = друге з двох на парному
// while (fast.next && fast.next.next)  → середина = перше з двох на парному
```

Різниця тонка, але критична для задач на paliндроми чи split пополам.

### 3. Забути рухати вказівник

```javascript
// ❌ Нескінченний цикл -- current ніколи не змінюється
while (current !== null) {
  if (current.value === x) {
    current.next = current.next.next;
    // забули current = current.next після цього коли НЕ видалили
  }
}

// ✅ Рухаємо тільки якщо не видалили
while (current !== null && current.next !== null) {
  if (current.next.value === x) {
    current.next = current.next.next;   // видалили -- не рухаємо current
  } else {
    current = current.next;
  }
}
```

### 4. Втратити посилання до наступного

```javascript
// ❌ Після current.next = prev ми втрачаємо доступ до оригінального current.next
current.next = prev;
current = current.next;   // це тепер prev, а не оригінальний next!

// ✅ Зберігаємо заздалегідь
const next = current.next;
current.next = prev;
current = next;
```

Це щоденна помилка в задачах на переворот списку.

### 5. Не оновити `tail` при видаленні останнього

У власній імплементації `LinkedList` при `remove(lastIndex)` треба:
- Оновити `tail = newLast`.
- Обнулити `newLast.next = null`.
- Зменшити `length`.

Якщо забути -- `push` потім вставить у "мертву" гілку списку.

### 6. Цикл в списку ламає все

Якщо хтось створив цикл (наприклад, `tail.next = head`) -- `console.log(list)` зависне. Перед будь-якою debug-відладкою на незнайомому вході зроби `hasCycle`.

### 7. Garbage collection і пам'ять

JS сам збирає сміття, але якщо ти зберігаєш десь ще одне посилання на вузол (наприклад, у `Map`), він не видалиться. У singly linked list при видаленні достатньо `prev.next = removed.next`. У doubly краще ще занулити `removed.prev` і `removed.next`, щоб не тримати зайві посилання.

---

## Ключові думки

- Linked list -- це **ланцюг вузлів з посиланнями**, протилежність неперервному масиву.
- **O(1) для вставки/видалення на краях** (якщо є tail). **O(n) для доступу за індексом**.
- Два вказівники (fast/slow) -- основа половини всіх задач на linked list.
- Floyd's Tortoise & Hare для детекції циклу -- must-know.
- **Dummy node** спрощує код, коли голова може змінитись.
- Рекурсивні рішення елегантні, але дають O(n) space і ризик stack overflow.
- У реальному Node.js частіше використовуєш масиви; linked list потрібен для задач, черг, LRU cache, спецструктур.
- Найпоширеніші помилки -- null checks на `.next`, втрата посилань при reverse, забуття оновити `tail`.

У наступному файлі -- стеки та черги, які часто імплементуються саме через linked list.
