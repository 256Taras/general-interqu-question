# Рекурсія та Backtracking

## Що таке рекурсія (з самого нуля)

**Рекурсія** -- це коли функція викликає сама себе. Ось і все визначення. Але за цим простим реченням ховається один із найскладніших для початківців інструментів програмування.

Почнемо з життєвого прикладу. Уяви, що ти стоїш в черзі і хочеш дізнатися, скільки людей перед тобою. Замість того, щоб виходити з черги і перераховувати всіх, ти запитуєш у людини перед собою: "Скільки людей перед тобою?". Вона у свою чергу запитує у людини перед нею. І так далі, аж поки хтось на початку черги не скаже "переді мною нікого". Тоді відповідь "покотиться" назад: "0", "1", "2", "3"... і ти отримаєш "n".

Це і є рекурсія: **велику задачу зводимо до точно такої ж задачі меншого розміру**, і так поки не дійдемо до тривіального випадку, на який знаємо відповідь.

```javascript
// Обчислення n! (факторіал): n! = n × (n-1) × (n-2) × ... × 1
function factorial(n) {
  // Тривіальний випадок -- базовий кейс. Знаємо відповідь без обчислень.
  if (n <= 1) return 1;

  // Зводимо задачу до меншої: припускаємо, що factorial(n-1) ми вже якось обчислили.
  return n * factorial(n - 1);
}

factorial(5); // 120
```

Що тут відбулося? `factorial(5)` не знає відповіді на `5!` напряму, але знає формулу `5! = 5 × 4!`. Отже, він викликає `factorial(4)`. Той, знову ж таки, не знає `4!` прямо, але знає, що `4! = 4 × 3!`. І так далі.

---

## Як працює call stack (дуже важливо зрозуміти це зараз)

Щоб рекурсія перестала здаватися магією, треба зрозуміти, як працює **call stack** (стек викликів) -- внутрішній механізм двигуна JS.

Щоразу, коли функція викликає іншу функцію, двигун JS "ставить" новий запис у стек -- це називається **stack frame**. Frame містить:
- Локальні змінні функції.
- Місце, куди повернутися після виконання.
- Проміжні результати.

Коли функція завершується (виконує `return`), її frame видаляється зі стеку (pop), і керування повертається до попередньої функції.

Розглянемо виконання `factorial(3)` покроково:

```
Крок 1: викликали factorial(3)
┌─────────────────────────┐
│ factorial(3)            │  n = 3, чекає результат factorial(2)
└─────────────────────────┘

Крок 2: factorial(3) викликає factorial(2)
┌─────────────────────────┐
│ factorial(2)            │  n = 2, чекає результат factorial(1)
├─────────────────────────┤
│ factorial(3)            │  n = 3, чекає factorial(2)
└─────────────────────────┘

Крок 3: factorial(2) викликає factorial(1)
┌─────────────────────────┐
│ factorial(1)            │  n = 1, повертає 1 одразу
├─────────────────────────┤
│ factorial(2)            │  n = 2, чекає factorial(1)
├─────────────────────────┤
│ factorial(3)            │  n = 3, чекає factorial(2)
└─────────────────────────┘

Крок 4: factorial(1) повертає 1, frame видаляється
┌─────────────────────────┐
│ factorial(2)            │  тепер може порахувати 2 * 1 = 2
├─────────────────────────┤
│ factorial(3)            │  n = 3, чекає factorial(2)
└─────────────────────────┘

Крок 5: factorial(2) повертає 2, frame видаляється
┌─────────────────────────┐
│ factorial(3)            │  тепер може порахувати 3 * 2 = 6
└─────────────────────────┘

Крок 6: factorial(3) повертає 6, стек порожній
```

**Ключове спостереження:** рекурсія -- це "відкладене обчислення". Коли ми кажемо `return n * factorial(n - 1)`, множення на `n` чекає, поки `factorial(n-1)` відпрацює. Ось чому відповіді "котяться" назад знизу вгору по стеку.

---

## Базові правила рекурсії: base case і recursive case

Будь-яка коректна рекурсивна функція має **рівно дві частини**:

**1. Base case (базовий випадок)** -- коли відповідь тривіальна і рекурсія зупиняється.
**2. Recursive case (рекурсивний випадок)** -- коли ми зводимо задачу до меншої і викликаємо себе.

```javascript
function factorial(n) {
  if (n <= 1) return 1;           // ← BASE CASE: відповідь без рекурсії
  return n * factorial(n - 1);    // ← RECURSIVE CASE: крок до base case
}
```

**Чому без base case буде stack overflow?**

Уяви функцію без base case:

```javascript
function broken(n) {
  return broken(n - 1);   // ніколи не зупиниться!
}
broken(5); // RangeError: Maximum call stack size exceeded
```

Стек буде зростати нескінченно, бо кожен новий виклик додає frame, але жоден не завершується. У V8 (двигун Node.js) ліміт стеку -- приблизно **10 000 - 15 000 frames** залежно від розміру кожного frame. Коли ліміт перевищено -- `RangeError: Maximum call stack size exceeded`.

**Правило:** recursive case завжди має наближати нас до base case. Якщо ти викликаєш `broken(n - 1)`, переконайся, що є умова для малих `n`, що зупинить рекурсію.

Класична помилка -- неправильна умова:

```javascript
// ❌ Поганий base case: для парних n все працює, для непарних -- нескінченно
function halveUntilZero(n) {
  if (n === 0) return 0;
  return halveUntilZero(n - 2);
}
halveUntilZero(5); // Stack overflow! бо 5 → 3 → 1 → -1 → -3 → ...

// ✅ Правильно
function halveUntilZeroOrNegative(n) {
  if (n <= 0) return 0;
  return halveUntilZeroOrNegative(n - 2);
}
```

Base case має покривати **всі** можливі шляхи виходу.

---

## Як думати рекурсивно: leap of faith

Найскладніше для початківців -- навчитися думати рекурсивно. Є трюк, який змінює все. Називається **"leap of faith"** (стрибок довіри).

**Ідея:** коли пишеш рекурсивну функцію, **не треба** в голові розгортати всі виклики. Замість цього:

1. Уяви, що функція вже працює правильно для менших входів.
2. Запитай себе: "Якщо я **довіряю**, що `f(n-1)` повертає правильну відповідь, як мені побудувати відповідь для `n`?"
3. Додай base case для найменшого випадку.

Приклад. Треба написати `sum(arr)` -- сума елементів масиву.

**Думай так:**
- "Якщо я можу викликати `sum(arr.slice(1))` і довіряю, що воно поверне суму всіх елементів крім першого, то відповідь для всього масиву -- це `arr[0] + sum(arr.slice(1))`."
- "Коли зупинитися? Коли масив порожній, сума -- 0."

```javascript
function sum(arr) {
  if (arr.length === 0) return 0;            // base case
  return arr[0] + sum(arr.slice(1));         // leap of faith: довіряємо, що sum хвоста працює
}
```

Не треба в голові моделювати `sum([1,2,3])` → `1 + sum([2,3])` → `1 + 2 + sum([3])` → ... -- це швидко виносить мозок. Довіряй, що менша версія функції працює, і будуй на цьому.

**Цей принцип -- ключ до рекурсії.** Якщо ти досі намагаєшся "прокрутити" всі виклики подумки -- ти переключаєшся у неправильний режим мислення. Призупинись і запитай себе: "яку меншу задачу я можу довірити функції?"

---

## Рекурсія vs ітерація: коли що краще

Кожна рекурсивна функція може бути переписана як ітеративна (з циклом), і навпаки. Ось приклад:

```javascript
// Рекурсивно
function factorialRec(n) {
  if (n <= 1) return 1;
  return n * factorialRec(n - 1);
}

// Ітеративно
function factorialIter(n) {
  let result = 1;
  for (let i = 2; i <= n; i++) result *= i;
  return result;
}
```

**Коли рекурсія зручніша:**
- Структура даних вже рекурсивна (дерева, графи, вкладені об'єкти).
- Задача природно розбивається на підзадачі (merge sort, quicksort, divide-and-conquer).
- Backtracking (перебір варіантів з відкотом).
- Парсинг (AST, JSON з довільною вкладеністю).

**Коли ітерація краща:**
- Простий лінійний обхід (масив, рядок) -- цикл зрозуміліший.
- Великий розмір входу (>10 000 кроків у глибину) -- ризик stack overflow.
- Продуктивність критична -- виклик функції дорожчий за ітерацію циклу.

### Tail recursion у JavaScript: сумна історія

**Tail call** -- це виклик функції, який є **останньою** дією у функції (нічого не відбувається після повернення). Наприклад:

```javascript
// Це НЕ tail call: після factorial(n-1) ще треба помножити на n
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);   // множення після -- не tail
}

// А це TAIL call: рекурсивний виклик -- остання дія
function factorialTail(n, acc = 1) {
  if (n <= 1) return acc;
  return factorialTail(n - 1, acc * n);   // нічого не відбувається після виклику
}
```

У мовах з **tail call optimization (TCO)** -- наприклад, Scheme, Erlang, Scala -- компілятор розпізнає tail call і переписує його як цикл, не додаючи нового frame у стек. Таким чином, рекурсія не переповнює стек навіть на мільйонах ітерацій.

**У JavaScript TCO формально є у специфікації ES6, але жоден великий двигун (V8, SpiderMonkey) його не реалізував.** Причина -- складнощі з відладкою (stack trace стає нечитабельним). Safari одно-двічі вмикав і вимикав.

**Практично:** у JS глибока рекурсія = ризик stack overflow. Якщо треба оптимізувати -- переписуй вручну на ітерацію або використовуй явний стек (масив як стек).

---

## Класичні приклади рекурсії

### 1. Factorial

```javascript
// n! = n * (n-1) * (n-2) * ... * 1
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}
// Time: O(n), Space: O(n) -- n frames у стеку
```

### 2. Fibonacci (наївний -- НЕ так у продакшені)

```javascript
// F(n) = F(n-1) + F(n-2), F(0) = 0, F(1) = 1
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}
// Time: O(2ⁿ) -- катастрофічно! Space: O(n) -- максимальна глибина стеку
```

Подивись на дерево викликів `fib(5)`:

```
                         fib(5)
                       /        \
                  fib(4)          fib(3)
                 /      \         /    \
             fib(3)    fib(2)  fib(2)  fib(1)
            /    \     /   \    /  \
        fib(2) fib(1) fib(1) fib(0) fib(1) fib(0)
        /   \
    fib(1) fib(0)
```

Помічаєш? `fib(3)` викликається 2 рази, `fib(2)` -- 3 рази, `fib(1)` -- 5 разів. Кожен раз ми перераховуємо ті самі значення. Для `fib(50)` це буде трильйони викликів. Розв'язок -- memoization (див. нижче).

### 3. Power of x (x^n)

```javascript
// Наївно: x^n = x * x * x * ... (n разів). Time: O(n).
function powerNaive(x, n) {
  if (n === 0) return 1;
  return x * powerNaive(x, n - 1);
}

// Оптимізовано: якщо n парне, x^n = (x^(n/2))². Time: O(log n).
function powerFast(x, n) {
  if (n === 0) return 1;
  if (n % 2 === 0) {
    const half = powerFast(x, n / 2);
    return half * half;
  }
  return x * powerFast(x, n - 1);
}

powerFast(2, 10); // 1024
```

Friendly reminder: для від'ємних `n` треба окремий кейс (`1 / powerFast(x, -n)`).

### 4. Sum of array recursively

```javascript
function sumArr(arr, i = 0) {
  if (i === arr.length) return 0;     // base case: пройшли весь масив
  return arr[i] + sumArr(arr, i + 1); // хвіст: сума решти
}
// Time: O(n), Space: O(n) -- n frames
```

Варіант через slice (красивіше, але O(n²) через копіювання масиву):

```javascript
function sumArrSlice(arr) {
  if (arr.length === 0) return 0;
  return arr[0] + sumArrSlice(arr.slice(1));   // slice -- O(n), робиться n разів!
}
// Time: O(n²), Space: O(n²). НЕ роби так на великих входах.
```

### 5. Reverse string recursively

```javascript
function reverse(str) {
  if (str.length <= 1) return str;
  return reverse(str.slice(1)) + str[0];
}

reverse("hello"); // "olleh"
```

Як це працює на `"abc"`:
```
reverse("abc")
  = reverse("bc") + "a"
  = (reverse("c") + "b") + "a"
  = ("c" + "b") + "a"
  = "cb" + "a"
  = "cba"
```

Знову ж, у продакшені просто `str.split("").reverse().join("")`. Рекурсія тут -- для тренування мислення.

---

## Memoization: перетворюємо Fibonacci O(2ⁿ) на O(n)

**Memoization** -- техніка, коли результати функції кешуються за її аргументами, щоб не обчислювати те саме двічі.

```javascript
function fibMemo(n, memo = new Map()) {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n);      // якщо вже рахували -- беремо з кешу

  const result = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
  memo.set(n, result);
  return result;
}

fibMemo(50); // миттєво, замість годин
// Time: O(n), Space: O(n) -- по одному запису в memo на кожне n
```

Порівняння дерев викликів:

```
Без memo -- fib(5):          З memo -- fibMemo(5):
                               
        fib(5)                         fib(5)
        /    \                         /    \
     fib(4)  fib(3)                 fib(4)  [cache: 3]
     /   \    /  \                  /   \
  fib(3) fib(2) fib(2) fib(1)    fib(3) [cache: 2]
  /  \    ...    ...              /  \
...                             fib(2) [cache: 1]
                                 /  \
                             fib(1) fib(0)

Кожен вузол обчислюється         Кожен fib(k) обчислюється
кілька разів → O(2ⁿ)             РІВНО ОДИН РАЗ → O(n)
```

**Memoization -- це основа Dynamic Programming (DP).** DP -- це загальніша техніка: розбити задачу на підзадачі, що перекриваються, і не рахувати їх двічі. Memoization -- "top-down" DP (рекурсивно + кеш). Іноді краще "bottom-up" DP -- ітеративно заповнюємо таблицю від малих підзадач до великих. Деталі -- в іншому файлі по DP.

**Де ще memoization допомагає:**
- Climbing stairs, coin change, edit distance, longest common subsequence -- всі типові DP задачі.
- Будь-яка рекурсія з повторюваними підзадачами.

**Коли memoization НЕ допоможе:**
- Якщо підзадачі не повторюються (наприклад, у merge sort -- кожна підзадача унікальна).
- Якщо простір аргументів величезний і кеш вичерпає пам'ять.

---

## Що таке Backtracking

**Backtracking** -- це техніка розв'язання задач, у яких треба знайти всі (або якийсь конкретний) варіант із множини можливих комбінацій. Метод: пробуємо варіант, рекурсивно йдемо далі; якщо дорога заводить у глухий кут -- **відкочуємось** (backtrack) і пробуємо інший варіант.

Аналогія: ти у лабіринті шукаєш вихід. Йдеш у перший коридор -- тупик. Повертаєшся до розвилки, йдеш у другий. Тупик. Повертаєшся. Пробуєш третій. Так поки не знайдеш.

**Загальний шаблон backtracking:**

```javascript
function backtrack(state, choices) {
  // 1. Чи ми досягли повного розв'язку? (base case)
  if (isSolution(state)) {
    result.push([...state]);   // збираємо копію -- state змінюється!
    return;
  }

  // 2. Перебираємо можливі наступні кроки
  for (const choice of choices) {
    if (!isValid(state, choice)) continue;

    // 3. Робимо вибір: додаємо до state
    state.push(choice);

    // 4. Рекурсивно йдемо далі
    backtrack(state, updatedChoices);

    // 5. ВАЖЛИВО: відкочуємось -- забираємо вибір
    state.pop();
  }
}
```

Ключовий момент -- **рядки 4 та 5 (push + pop)**. Ми модифікуємо `state`, йдемо глибше, і після повернення повертаємо `state` у попередній стан. Це дозволяє не копіювати масив на кожному кроці (економія пам'яті).

---

## Класичні backtracking задачі

### 1. Generate all permutations (всі перестановки)

"Дано масив `[1, 2, 3]`, повернути всі перестановки: `[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,1,2], [3,2,1]`."

```javascript
function permutations(nums) {
  const result = [];
  const used = new Array(nums.length).fill(false);

  function backtrack(current) {
    // base case: зібрали повну перестановку
    if (current.length === nums.length) {
      result.push([...current]);       // копія, бо current далі мутуватиметься
      return;
    }

    for (let i = 0; i < nums.length; i++) {
      if (used[i]) continue;           // це число вже у current

      // вибираємо
      used[i] = true;
      current.push(nums[i]);

      // рекурсія
      backtrack(current);

      // відкочуємось
      current.pop();
      used[i] = false;
    }
  }

  backtrack([]);
  return result;
}

permutations([1, 2, 3]);
// [[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
// Time: O(n! × n), Space: O(n) для стеку + O(n! × n) для результату
```

Дерево рішень для `[1, 2, 3]`:

```
                  []
         /        |        \
       [1]       [2]       [3]
       / \       / \       / \
    [1,2][1,3] [2,1][2,3] [3,1][3,2]
     |    |    |    |     |    |
  [1,2,3][1,3,2][2,1,3][2,3,1][3,1,2][3,2,1]
```

На кожному рівні ми вибираємо одне з **невикористаних** чисел. Після повернення з рекурсії повертаємо це число у "невикористані".

### 2. Generate all subsets (power set)

"Дано `[1, 2, 3]`, повернути всі підмножини: `[], [1], [2], [3], [1,2], [1,3], [2,3], [1,2,3]`." Загалом 2ⁿ підмножин.

```javascript
function subsets(nums) {
  const result = [];

  function backtrack(start, current) {
    result.push([...current]);        // кожен стан -- це валідна підмножина

    for (let i = start; i < nums.length; i++) {
      current.push(nums[i]);          // беремо nums[i]
      backtrack(i + 1, current);      // далі -- тільки більші індекси
      current.pop();                  // відкочуємось
    }
  }

  backtrack(0, []);
  return result;
}

subsets([1, 2, 3]);
// [[],[1],[1,2],[1,2,3],[1,3],[2],[2,3],[3]]
// Time: O(2ⁿ × n), Space: O(n) глибина
```

Дерево рішень:

```
                         []
              /          |          \
            [1]         [2]         [3]
           /   \          \
        [1,2] [1,3]      [2,3]
         /
     [1,2,3]
```

Ключова деталь: параметр `start` гарантує, що ми не повторюємо елементи і не генеруємо дублікати на зразок `[1,2]` і `[2,1]`.

### 3. Combinations (C(n, k))

"Дано `n` і `k`, повернути всі комбінації `k` чисел з `1..n`. Наприклад, `n=4, k=2` → `[[1,2],[1,3],[1,4],[2,3],[2,4],[3,4]]`."

```javascript
function combine(n, k) {
  const result = [];

  function backtrack(start, current) {
    if (current.length === k) {
      result.push([...current]);
      return;
    }

    // Оптимізація: якщо не вистачить чисел, щоб набрати k, -- стоп.
    // Залишилося k - current.length чисел вибрати. Доступно n - start + 1.
    for (let i = start; i <= n - (k - current.length) + 1; i++) {
      current.push(i);
      backtrack(i + 1, current);
      current.pop();
    }
  }

  backtrack(1, []);
  return result;
}

combine(4, 2);
// [[1,2],[1,3],[1,4],[2,3],[2,4],[3,4]]
```

### 4. N-Queens (коротко)

"Розставити N ферзів на дошці N×N так, щоб жоден не атакував інший (не на тому ж рядку, колонці чи діагоналі)."

Підхід:
- Йдемо по рядках згори вниз.
- Для кожного рядка пробуємо поставити ферзя у кожну колонку.
- Перевіряємо, чи не атакується ця позиція вже розставленими ферзями.
- Якщо валідно -- рекурсивно йдемо до наступного рядка. Інакше -- пробуємо наступну колонку.
- Якщо всі N рядків заповнені -- зберігаємо розв'язок.

```javascript
function solveNQueens(n) {
  const result = [];
  const cols = new Set();         // зайняті колонки
  const diag1 = new Set();        // зайняті "/" діагоналі (row + col)
  const diag2 = new Set();        // зайняті "\" діагоналі (row - col)
  const queens = [];              // queens[row] = col

  function backtrack(row) {
    if (row === n) {
      result.push([...queens]);
      return;
    }
    for (let col = 0; col < n; col++) {
      if (cols.has(col) || diag1.has(row + col) || diag2.has(row - col)) continue;

      // вибираємо
      queens.push(col);
      cols.add(col);
      diag1.add(row + col);
      diag2.add(row - col);

      backtrack(row + 1);

      // відкочуємось
      queens.pop();
      cols.delete(col);
      diag1.delete(row + col);
      diag2.delete(row - col);
    }
  }

  backtrack(0);
  return result;
}
```

Ключова техніка: зберігати "зайнятість" у Set-ах, щоб перевірка була O(1). Без цього було б O(n) на перевірку, і загальний час -- ще гірший.

### 5. Sudoku solver (концептуально)

"Дано частково заповнену дошку 9×9, заповнити порожні клітини так, щоб кожен рядок, кожна колонка і кожен 3×3 квадрат містили цифри 1-9 без повторів."

Ідея:
- Знайти наступну порожню клітину.
- Для кожної цифри 1-9 перевірити, чи валідно її поставити (немає повторів у рядку, колонці, 3×3 блоці).
- Якщо валідно -- поставити, рекурсивно спробувати розв'язати решту дошки.
- Якщо рекурсія повернулася без розв'язку -- прибрати цифру, спробувати наступну.
- Якщо всі клітини заповнені -- розв'язано.

Псевдокод:
```
solve(board):
  cell = findEmpty(board)
  if cell is null: return true  // всі заповнені

  for digit 1..9:
    if isValid(board, cell, digit):
      board[cell] = digit        // вибір
      if solve(board): return true
      board[cell] = empty        // backtrack

  return false                    // жодна цифра не підійшла
```

Продуктивність покращується, якщо шукати найбільш "обмежену" клітину (з найменшою кількістю варіантів) -- це знижує гілкування. Називається **MRV heuristic (Minimum Remaining Values)**.

### 6. Word Search у матриці

"Дано сітку літер і слово. Треба перевірити, чи слово можна скласти, йдучи по сусідніх клітинах (вгору/вниз/вліво/вправо), не відвідуючи клітину двічі."

```javascript
function exist(board, word) {
  const rows = board.length;
  const cols = board[0].length;

  function dfs(r, c, i) {
    if (i === word.length) return true;                 // знайшли все слово
    if (r < 0 || r >= rows || c < 0 || c >= cols) return false;
    if (board[r][c] !== word[i]) return false;

    const temp = board[r][c];
    board[r][c] = '#';                                  // маркуємо як відвідану

    const found =
      dfs(r + 1, c, i + 1) ||
      dfs(r - 1, c, i + 1) ||
      dfs(r, c + 1, i + 1) ||
      dfs(r, c - 1, i + 1);

    board[r][c] = temp;                                 // backtrack: повертаємо літеру
    return found;
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (dfs(r, c, 0)) return true;
    }
  }
  return false;
}
```

Техніка **"тимчасової мутації"**: замість окремого `visited` Set-а, ми мутуємо саму клітину (ставимо `'#'`), а на backtrack повертаємо літеру. Економія пам'яті -- не треба створювати додаткову структуру.

`Time: O(m × n × 4^L)`, де `L` -- довжина слова. На кожному кроці 4 напрямки. Хоча на практиці значно менше через ранній відсів.

---

## Типові пастки

### 1. Забутий base case → stack overflow

```javascript
// ❌
function countdown(n) {
  console.log(n);
  countdown(n - 1);          // ніколи не зупиниться
}
```

Завжди перше, що пишеш у рекурсивній функції -- `if (basecase) return ...`. Потім уже думай про recursive case.

### 2. Модифікація входу без undo (в backtracking)

```javascript
// ❌ Забули зробити pop після рекурсії
function permutations(nums) {
  const result = [];
  function backtrack(current) {
    if (current.length === nums.length) {
      result.push([...current]);
      return;
    }
    for (let i = 0; i < nums.length; i++) {
      if (current.includes(nums[i])) continue;
      current.push(nums[i]);
      backtrack(current);
      // ❌ забули current.pop() -- current "протече" у братні гілки
    }
  }
  backtrack([]);
  return result;
}
```

**Золоте правило backtracking:** кожну мутацію `state` після рекурсії обов'язково скасовуй.

### 3. Exponential complexity без memoization

```javascript
// ❌ O(2ⁿ) -- для n=40 працює хвилини
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}

// ✅ O(n) через memo
function fib(n, memo = new Map()) {
  if (n <= 1) return n;
  if (memo.has(n)) return memo.get(n);
  const r = fib(n - 1, memo) + fib(n - 2, memo);
  memo.set(n, r);
  return r;
}
```

Якщо бачиш, що функція викликає себе двічі (чи більше) з перекриваними аргументами -- подумай про memoization.

### 4. Глибока рекурсія в JS (~10-15k frames)

```javascript
// Для великого n -- stack overflow
function sumTo(n) {
  if (n === 0) return 0;
  return n + sumTo(n - 1);
}

sumTo(100_000); // RangeError: Maximum call stack size exceeded
```

**Розв'язки:**
- Переписати як ітеративну функцію з циклом.
- Використати явний стек (масив як структуру `stack.push()` / `stack.pop()`).
- Trampoline (рідко у JS).

Приклад trampoline -- виняток, коли треба зберегти рекурсивний вигляд:

```javascript
function trampoline(fn) {
  return function (...args) {
    let result = fn(...args);
    while (typeof result === 'function') result = result();
    return result;
  };
}

const sumTo = trampoline(function inner(n, acc = 0) {
  if (n === 0) return acc;
  return () => inner(n - 1, acc + n);   // замість рекурсивного виклику -- повертаємо функцію
});

sumTo(1_000_000);  // працює без stack overflow
```

На практиці у продакшн-коді просто пиши цикл.

### 5. Копіювання state замість push/pop (повільно)

```javascript
// ❌ повільно: копія масиву на кожен виклик
function permutations(nums, current = []) {
  if (current.length === nums.length) return [current];
  const result = [];
  for (let i = 0; i < nums.length; i++) {
    if (current.includes(nums[i])) continue;
    result.push(...permutations(nums, [...current, nums[i]]));  // spread O(n)
  }
  return result;
}
```

Для перевірки `includes` -- O(n). Для spread -- O(n). Це робить алгоритм гірше, ніж треба. Краще через push/pop + used-масив, як у прикладі вище.

### 6. Забута копія при збереженні результату

```javascript
// ❌ Зберігаємо посилання на current, а він далі мутується!
function subsets(nums) {
  const result = [];
  function bt(start, current) {
    result.push(current);              // ❌ посилання
    for (let i = start; i < nums.length; i++) {
      current.push(nums[i]);
      bt(i + 1, current);
      current.pop();
    }
  }
  bt(0, []);
  return result;                        // всі елементи = [] (остання версія current)
}

// ✅ Копія через spread
result.push([...current]);
```

Масиви в JS -- посилання. Якщо зберігаєш у результат посилання на `current`, а потім мутуєш `current`, -- збережене теж зміниться.

---

## Як підступатися до рекурсивної задачі на співбесіді

1. **Сформулюй задачу через саму себе.** "Як розв'язання для n залежить від розв'язання для n-1 (або меншої версії)?"
2. **Визнач base case.** Найменший вхід, на якому відповідь тривіальна.
3. **Опиши recursive case.** Leap of faith: довіряй, що менша версія працює.
4. **Оціни складність.** Скільки викликів? Яка робота на виклик? Чи є повторювані підзадачі → memoization?
5. **Подумай про глибину стеку.** Якщо n велике (>10k) -- переписуй на ітерацію.
6. **Для backtracking:** сформулюй state, choices, base case (рішення знайдено), validity check, undo дію.

---

## Ключові думки

- Рекурсія = функція кличе себе на меншому вході, плюс base case.
- Call stack тримає frames; кожен рекурсивний виклик додає frame.
- **Leap of faith**: довіряй, що менша версія функції працює -- і будуй на цьому.
- Base case без якого -- stack overflow.
- JS не має TCO. Глибина рекурсії обмежена ~10k frames.
- Memoization перетворює експоненційну рекурсію на поліноміальну.
- Backtracking = пробуй + рекурсія + undo. Шаблон: choose → explore → unchoose.
- Типові backtracking задачі: permutations, subsets, combinations, N-Queens, Sudoku, word search.
- Після рекурсії -- **обов'язково** undo мутацію state. Завжди.
- Зберігаєш state у результаті -- роби копію (`[...current]`).

Далі -- найбагатша тема алгоритмічних інтерв'ю: **дерева**.
