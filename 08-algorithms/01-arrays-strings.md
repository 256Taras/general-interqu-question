# Масиви і рядки: найчастіші задачі на співбесідах

Масиви та рядки -- це ~60% усіх алгоритмічних задач на співбесідах. Вони виглядають просто ("просто масив чисел"), але саме тут найбільше можливостей показати, чи вмієш ти думати про Big O, пам'ять, in-place операції.

Перш ніж братися за конкретні задачі, треба зрозуміти **як масив живе в пам'яті**. Без цього багато речей здаватимуться магією.

---

## Як масив зберігається в пам'яті (і чому це важливо)

Масив у класичному розумінні -- це **неперервний блок пам'яті**, де елементи лежать один за одним, і кожен займає фіксовану кількість байтів.

```
Масив [10, 20, 30, 40, 50] в пам'яті:

адреса:   1000    1008    1016    1024    1032
         ┌────┬──┬────┬──┬────┬──┬────┬──┬────┐
         │ 10 │  │ 20 │  │ 30 │  │ 40 │  │ 50 │
         └────┴──┴────┴──┴────┴──┴────┴──┴────┘
           [0]     [1]     [2]     [3]     [4]
```

Кожен елемент -- 8 байтів (якщо це, наприклад, число). Щоб дістати `arr[3]`, комп'ютер рахує:
```
адреса arr[3] = базова_адреса + 3 × 8 = 1000 + 24 = 1024
```

Це **одна арифметична операція**. Тому `arr[i]` -- це **O(1)**. Не важливо, чи `i = 0`, чи `i = 1_000_000` -- час однаковий.

### Чому `push` швидкий, а `unshift` повільний

`push` додає в **кінець**. Якщо є вільне місце після останнього елемента -- просто кладемо значення:

```
До:    [10, 20, 30, _, _]
       arr.push(40)
Після: [10, 20, 30, 40, _]
```

Одна операція -- O(1).

`unshift` додає на **початок**. А на початку немає місця -- там уже живе `arr[0]`. Треба зсунути **всі** елементи вправо:

```
До:    [10, 20, 30, 40]
       arr.unshift(5)

Крок 1: зсунути 40 на позицію [4]:   [10, 20, 30, 40, 40]
Крок 2: зсунути 30 на позицію [3]:   [10, 20, 30, 30, 40]
Крок 3: зсунути 20 на позицію [2]:   [10, 20, 20, 30, 40]
Крок 4: зсунути 10 на позицію [1]:   [10, 10, 20, 30, 40]
Крок 5: покласти 5 на позицію [0]:   [ 5, 10, 20, 30, 40]
```

Для масиву розміром n треба зробити n зсувів -- це **O(n)**.

### JavaScript-нюанс

В JS масив -- це не чистий C-шний масив. V8 (двигун Chrome/Node.js) використовує різні внутрішні представлення:
- Якщо всі елементи -- маленькі цілі числа (SMI), JS використовує справжній неперервний масив.
- Якщо змішані типи -- використовує "packed" representation.
- Якщо є діри (`arr[100] = 1` на порожньому масиві) -- "holey" або навіть hash map.

Для **алгоритмічних задач** працює стара добра модель "неперервний блок пам'яті". І складності з таблиці у попередньому файлі справедливі.

### Практичний висновок

Коли бачиш задачу з масивом -- одразу думай:
- **Доступ за індексом** -- безкоштовний (O(1)).
- **Додати/видалити в кінці** -- дешево (O(1) amortized).
- **Додати/видалити на початку** -- дорого (O(n)).
- **Вставити посередині** -- дорого (O(n), бо треба зсувати).
- **Пошук елемента** -- O(n), якщо масив не відсортований.

Багато оптимізацій у задачах базуються на цих правилах.

---

## Патерн 1: Two Pointers (два вказівники)

Це **найпоширеніший патерн** для задач на масиви і рядки. Якщо вивчиш лише один патерн -- вивчи цей.

### Що це таке

Замість одного циклу з `i` ми використовуємо **два індекси** (вказівники), які рухаються незалежно. Зазвичай:
- **Left/right** -- один з початку, другий з кінця, рухаються назустріч.
- **Slow/fast** -- обидва з початку, рухаються з різною швидкістю.

### Коли застосовувати

- Масив **відсортований**.
- Шукаєш **пару** елементів з певною властивістю.
- Треба **модифікувати масив in-place** (без додаткової пам'яті).
- Працюєш з **паліндромом**.
- Треба **видалити дублікати** у відсортованому масиві.

---

### Приклад 1.1: Reverse string in-place

**Задача:** Дано масив символів, розверни його. Не використовуй додаткову пам'ять.

**Наївне рішення:**

```javascript
function reverseNaive(arr) {
  const reversed = [];
  for (let i = arr.length - 1; i >= 0; i--) {
    reversed.push(arr[i]);
  }
  return reversed;
}
// Time: O(n), Space: O(n) -- створили новий масив
```

Працює, але займає додаткову пам'ять. Інтерв'юер попросить "in-place" -- без додаткової пам'яті.

**Two pointers рішення:**

```javascript
function reverse(arr) {
  let left = 0;
  let right = arr.length - 1;

  while (left < right) {
    // swap arr[left] та arr[right]
    [arr[left], arr[right]] = [arr[right], arr[left]];
    left++;
    right--;
  }

  return arr;
}
// Time: O(n), Space: O(1)
```

**Як це працює:**

```
Початок: ['h', 'e', 'l', 'l', 'o']
          L                   R

Крок 1:  ['o', 'e', 'l', 'l', 'h']   swap arr[0] і arr[4]
               L          R

Крок 2:  ['o', 'l', 'l', 'e', 'h']   swap arr[1] і arr[3]
                    L R

Крок 3:  L і R зустрілись -- цикл виходить.
```

**Чому O(n)?** Цикл виконується n/2 разів -- кожна ітерація обробляє два елементи. n/2 = O(n).

**Чому O(1) space?** Не створюємо нового масиву. Лише дві змінні `left` і `right`.

---

### Приклад 1.2: Two Sum у відсортованому масиві

**Задача:** Дано відсортований масив `nums` і число `target`. Знайди два індекси, сума значень на яких дає `target`.

Приклад: `nums = [2, 7, 11, 15], target = 9` → `[0, 1]` (бо `2 + 7 = 9`).

**Наївне рішення: два цикли.**

```javascript
function twoSumNaive(nums, target) {
  for (let i = 0; i < nums.length; i++) {
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] + nums[j] === target) {
        return [i, j];
      }
    }
  }
  return [];
}
// Time: O(n²), Space: O(1)
```

Ми не використали факт, що масив **відсортований**. А це велика підказка.

**Two pointers рішення:**

```javascript
function twoSum(nums, target) {
  let left = 0;
  let right = nums.length - 1;

  while (left < right) {
    const sum = nums[left] + nums[right];

    if (sum === target) {
      return [left, right];
    }

    if (sum < target) {
      // Сума замала -- треба більше. Рухаємо left праворуч (до більших чисел).
      left++;
    } else {
      // Сума завелика -- треба менше. Рухаємо right ліворуч (до менших чисел).
      right--;
    }
  }

  return [];
}
// Time: O(n), Space: O(1)
```

**Чому це працює?** Ключовий інсайт -- **масив відсортований**.
- Якщо `nums[left] + nums[right] < target`, то будь-яка пара з меншим `right` дасть **ще меншу** суму. Тому `right` рухати нема сенсу. Треба збільшити `left`.
- Симетрично для `> target`.

На кожному кроці ми **гарантовано виключаємо** один елемент з розгляду. Це як бінарний пошук по парах. Замість n² пар перевіряємо лише n -- один з двох вказівників зсувається на кожному кроці.

```
nums = [2, 7, 11, 15], target = 9

L                R
[2,  7, 11, 15]      sum = 2+15=17, забагато, R--

L            R
[2,  7, 11, 15]      sum = 2+11=13, забагато, R--

L        R
[2,  7, 11, 15]      sum = 2+7=9, знайшли! return [0, 1]
```

---

### Приклад 1.3: Remove duplicates from sorted array (in-place)

**Задача:** Дано відсортований масив. Видали дублікати in-place. Поверни нову довжину. Перші `k` елементів мають містити унікальні значення.

Приклад: `[1, 1, 2, 2, 3]` → стає `[1, 2, 3, _, _]`, return 3.

**Наївне рішення:**

```javascript
function removeDuplicatesNaive(arr) {
  const unique = [...new Set(arr)];
  // Але нам треба IN-PLACE -- модифікувати arr, не створювати новий.
  for (let i = 0; i < unique.length; i++) {
    arr[i] = unique[i];
  }
  return unique.length;
}
// Space: O(n) через Set -- не те, що просили.
```

**Two pointers (slow/fast) рішення:**

```javascript
function removeDuplicates(arr) {
  if (arr.length === 0) return 0;

  let slow = 0;  // індекс, куди пишемо унікальні елементи

  for (let fast = 1; fast < arr.length; fast++) {
    if (arr[fast] !== arr[slow]) {
      // Знайшли новий унікальний елемент
      slow++;
      arr[slow] = arr[fast];
    }
    // Якщо arr[fast] === arr[slow] -- це дублікат, пропускаємо
  }

  return slow + 1;  // кількість унікальних
}
// Time: O(n), Space: O(1)
```

**Як це працює:**

```
arr = [1, 1, 2, 2, 3]

Початок: slow=0, fast=1
  arr[0]=1, arr[1]=1 → однакові, пропускаємо

slow=0, fast=2
  arr[0]=1, arr[2]=2 → різні! slow++, arr[1]=2
  arr стає [1, 2, 2, 2, 3]

slow=1, fast=3
  arr[1]=2, arr[3]=2 → однакові, пропускаємо

slow=1, fast=4
  arr[1]=2, arr[4]=3 → різні! slow++, arr[2]=3
  arr стає [1, 2, 3, 2, 3]

Повертаємо slow + 1 = 3.
Перші 3 елементи: [1, 2, 3] -- саме те, що треба.
```

**Чому це працює?** `slow` -- це "кордон" унікальних елементів. Усе, що до `slow` включно -- вже унікальне. `fast` сканує далі і коли знаходить новий елемент -- додає його після `slow`.

---

### Приклад 1.4: Is palindrome

**Задача:** Чи є рядок паліндромом (читається однаково з початку і з кінця), ігноруючи не-буквено-цифрові символи і регістр.

Приклад: `"A man, a plan, a canal: Panama"` → `true`.

**Наївне рішення:**

```javascript
function isPalindromeNaive(s) {
  const cleaned = s.toLowerCase().replace(/[^a-z0-9]/g, '');
  const reversed = cleaned.split('').reverse().join('');
  return cleaned === reversed;
}
// Time: O(n), Space: O(n) -- створюємо два нових рядки
```

Працює і з точки зору Big O прийнятно. Але інтерв'юер може попросити O(1) space.

**Two pointers рішення:**

```javascript
function isPalindrome(s) {
  let left = 0;
  let right = s.length - 1;

  while (left < right) {
    // Пропустити не-alphanumeric зліва
    while (left < right && !isAlphanumeric(s[left])) left++;
    // Пропустити не-alphanumeric справа
    while (left < right && !isAlphanumeric(s[right])) right--;

    if (s[left].toLowerCase() !== s[right].toLowerCase()) {
      return false;
    }

    left++;
    right--;
  }

  return true;
}

function isAlphanumeric(ch) {
  return /^[a-z0-9]$/i.test(ch);
}
// Time: O(n), Space: O(1)
```

**Як це працює:** Два вказівники сходяться до центру. На кожному кроці порівнюємо символи. Якщо не збігаються -- це не паліндром.

```
"A man, a plan, a canal: Panama"

L                              R
A man, a plan, a canal: Panama

Пропускаємо пробіли, коми, двокрапки...
Порівнюємо 'a' і 'a' → OK
Далі 'm' і 'm' → OK
...і так далі.
```

---

### Коли two pointers -- правильний вибір

**Сигнали, що треба two pointers:**
1. "Масив відсортований" (або його можна відсортувати).
2. "Знайди пару/трійку".
3. "In-place, без додаткової пам'яті".
4. "Паліндром".
5. "Remove duplicates / remove element".

**Варіанти two pointers:**
- **Opposite ends** (сходяться) -- Two Sum, reverse, palindrome, container with most water.
- **Same direction (slow/fast)** -- remove duplicates, move zeros, cycle detection у linked list.

---

## Патерн 2: Sliding Window (ковзне вікно)

Другий мегапопулярний патерн. Застосовується там, де треба знайти **підмасив або підрядок** з певною властивістю.

### Інтуїція

Уяви, що у тебе є довгий масив і "вікно" -- діапазон `[left, right]`, який ти розсовуєш і стягуєш.

```
Масив: [1, 3, 2, 5, 1, 1, 2, 3]

Вікно розміру 3:
┌─────────┐
[1, 3, 2,] 5, 1, 1, 2, 3       sum = 6

   ┌─────────┐
 1,[3, 2, 5,] 1, 1, 2, 3       sum = 10

      ┌─────────┐
 1, 3,[2, 5, 1,] 1, 2, 3       sum = 8
```

Замість того, щоб на кожному кроці рахувати суму заново (O(k) роботи), ми **віднімаємо той, що виходить з вікна, і додаємо той, що входить** -- O(1) роботи на крок.

---

### Приклад 2.1: Maximum sum of k consecutive elements

**Задача:** Знайди максимальну суму `k` послідовних елементів.

Приклад: `arr = [1, 4, 2, 10, 2, 3, 1, 0, 20], k = 4` → `24` (останні 4: 3+1+0+20).

**Наївне (brute force):**

```javascript
function maxSumNaive(arr, k) {
  let maxSum = -Infinity;

  for (let i = 0; i <= arr.length - k; i++) {
    let sum = 0;
    for (let j = i; j < i + k; j++) {
      sum += arr[j];
    }
    maxSum = Math.max(maxSum, sum);
  }

  return maxSum;
}
// Time: O(n × k), Space: O(1)
```

На кожній позиції `i` ми рахуємо суму `k` елементів. `n × k` операцій.

**Оптимізація через sliding window:**

```javascript
function maxSum(arr, k) {
  if (arr.length < k) return -1;

  // Крок 1: порахувати суму першого вікна
  let windowSum = 0;
  for (let i = 0; i < k; i++) {
    windowSum += arr[i];
  }

  let maxSum = windowSum;

  // Крок 2: рухати вікно
  for (let right = k; right < arr.length; right++) {
    windowSum += arr[right];        // додаємо новий елемент справа
    windowSum -= arr[right - k];    // забираємо старий елемент зліва
    maxSum = Math.max(maxSum, windowSum);
  }

  return maxSum;
}
// Time: O(n), Space: O(1)
```

**Чому це швидше?** У наївному ми для кожного вікна робили `k` додавань. У sliding window ми використовуємо суму **попереднього вікна** і лише "зсуваємо" її: мінус лівий, плюс правий. Дві операції на крок замість k.

```
arr = [1, 4, 2, 10, 2, 3, 1, 0, 20], k = 4

Початкове вікно: [1, 4, 2, 10] → sum = 17

right=4: +arr[4]=2, -arr[0]=1 → sum = 17 + 2 - 1 = 18
       Вікно:  [4, 2, 10, 2]

right=5: +arr[5]=3, -arr[1]=4 → sum = 18 + 3 - 4 = 17
       Вікно:  [2, 10, 2, 3]

right=6: +arr[6]=1, -arr[2]=2 → sum = 17 + 1 - 2 = 16
       Вікно:  [10, 2, 3, 1]

right=7: +arr[7]=0, -arr[3]=10 → sum = 16 + 0 - 10 = 6
       Вікно:  [2, 3, 1, 0]

right=8: +arr[8]=20, -arr[4]=2 → sum = 6 + 20 - 2 = 24   ← max!
       Вікно:  [3, 1, 0, 20]

Return 24.
```

---

### Приклад 2.2: Longest substring without repeating characters

**Задача:** Дано рядок. Знайди довжину найдовшого підрядка без повторюваних символів.

Приклад: `"abcabcbb"` → `3` (`"abc"`).

Це класика. Сюди люблять заганяти на співбесідах.

**Наївне:**

```javascript
function longestUniqueNaive(s) {
  let maxLen = 0;

  for (let i = 0; i < s.length; i++) {
    const seen = new Set();
    for (let j = i; j < s.length; j++) {
      if (seen.has(s[j])) break;
      seen.add(s[j]);
    }
    maxLen = Math.max(maxLen, seen.size);
  }

  return maxLen;
}
// Time: O(n²), Space: O(min(n, alphabet))
```

Для кожного `i` рахуємо довжину унікального префіксу. Це n² у найгіршому випадку.

**Sliding window з Set:**

```javascript
function longestUnique(s) {
  const seen = new Set();
  let left = 0;
  let maxLen = 0;

  for (let right = 0; right < s.length; right++) {
    // Якщо s[right] уже є у вікні -- стягуємо вікно зліва
    while (seen.has(s[right])) {
      seen.delete(s[left]);
      left++;
    }

    seen.add(s[right]);
    maxLen = Math.max(maxLen, right - left + 1);
  }

  return maxLen;
}
// Time: O(n), Space: O(min(n, alphabet))
```

**Як це працює:** вікно `[left, right]` завжди містить **унікальні** символи.
- `right` рухаємо вправо і додаємо символ.
- Якщо символ уже є -- стягуємо `left`, прибираючи символи зліва, поки дублікат не зникне.

```
s = "abcabcbb"

right=0, s[0]='a': вікно [a], maxLen=1
right=1, s[1]='b': вікно [a,b], maxLen=2
right=2, s[2]='c': вікно [a,b,c], maxLen=3
right=3, s[3]='a': 'a' уже є!
   стягуємо: видаляємо s[0]='a', left=1
   вікно [b,c,a], maxLen=3
right=4, s[4]='b': 'b' уже є!
   стягуємо: видаляємо s[1]='b', left=2
   вікно [c,a,b], maxLen=3
right=5, s[5]='c': 'c' уже є!
   стягуємо: видаляємо s[2]='c', left=3
   вікно [a,b,c], maxLen=3
right=6, s[6]='b': 'b' уже є!
   стягуємо: видаляємо s[3]='a', left=4
            видаляємо s[4]='b', left=5
   вікно [c,b], maxLen=3
right=7, s[7]='b': 'b' уже є!
   стягуємо: видаляємо s[5]='c', left=6
            видаляємо s[6]='b', left=7
   вікно [b], maxLen=3

Return 3.
```

**Чому це O(n), а не O(n²)?** Здається, що через внутрішній `while` це n². Але `left` рухається тільки вперед і загалом може пройти максимум n разів. Тобто і `right`, і `left` разом роблять не більше 2n кроків. Це **амортизована** O(n).

---

### Приклад 2.3: Minimum window substring

**Задача:** Дано рядки `s` і `t`. Знайди найменший підрядок `s`, який містить усі символи `t` (з урахуванням кількості).

Приклад: `s = "ADOBECODEBANC", t = "ABC"` → `"BANC"`.

Це складніша вправа, але принцип той самий.

```javascript
function minWindow(s, t) {
  if (t.length > s.length) return "";

  // Порахуємо, скільки разів кожен символ потрібен у t
  const need = new Map();
  for (const ch of t) {
    need.set(ch, (need.get(ch) || 0) + 1);
  }

  let have = new Map();
  let satisfied = 0;             // скільки унікальних символів з t покрито повністю
  const required = need.size;

  let left = 0;
  let minLen = Infinity;
  let result = [0, 0];

  for (let right = 0; right < s.length; right++) {
    const ch = s[right];
    have.set(ch, (have.get(ch) || 0) + 1);

    // Якщо цей символ нам потрібен і ми досягли потрібної кількості
    if (need.has(ch) && have.get(ch) === need.get(ch)) {
      satisfied++;
    }

    // Стягуємо вікно зліва, поки умова виконується
    while (satisfied === required) {
      if (right - left + 1 < minLen) {
        minLen = right - left + 1;
        result = [left, right];
      }

      const leftCh = s[left];
      have.set(leftCh, have.get(leftCh) - 1);
      if (need.has(leftCh) && have.get(leftCh) < need.get(leftCh)) {
        satisfied--;
      }
      left++;
    }
  }

  return minLen === Infinity ? "" : s.slice(result[0], result[1] + 1);
}
// Time: O(n + m), Space: O(alphabet)
```

**Патерн загальний:**
1. Розширюй вікно (`right++`) поки умова не виконується.
2. Як тільки умова виконується -- стягуй (`left++`) поки можеш, фіксуючи мінімальну/максимальну відповідь.

---

### Коли sliding window -- правильний вибір

**Сигнали:**
1. "Знайди підрядок/підмасив з властивістю X".
2. "Найдовший / найкоротший / максимальний".
3. "Послідовні елементи".
4. Вхід -- один масив або рядок.

**Два типи вікна:**
- **Фіксованого розміру** (maximum sum of k elements) -- вікно завжди розміром k.
- **Змінного розміру** (longest substring, min window) -- розсовуємо і стягуємо залежно від умови.

---

## Патерн 3: Prefix Sum (префіксна сума)

Коли потрібно багато разів питати "яка сума елементів від i до j?" -- prefix sum рятує.

### Інтуїція

Створюємо допоміжний масив `prefix`, де `prefix[i] = arr[0] + arr[1] + ... + arr[i-1]`.

```
arr:    [3,  1,  4,  1,  5,  9,  2,  6]
prefix: [0,  3,  4,  8,  9, 14, 23, 25, 31]
         ↑                   ↑
       пусто                 3+1+4+1+5 = 14
       (зручно для формули)
```

Сума від `i` до `j` (включно) = `prefix[j+1] - prefix[i]`.

Наприклад, сума `arr[2..5]` = `4 + 1 + 5 + 9 = 19`. Перевіримо: `prefix[6] - prefix[2] = 23 - 4 = 19`. OK.

---

### Приклад 3.1: Range sum queries

**Задача:** Дано масив. Відповідай на багато запитів виду "сума від i до j".

**Наївне:**

```javascript
function rangeSumNaive(arr, queries) {
  return queries.map(([i, j]) => {
    let sum = 0;
    for (let k = i; k <= j; k++) sum += arr[k];
    return sum;
  });
}
// Time: O(q × n) -- для q запитів по n елементів
```

**Prefix sum:**

```javascript
class PrefixSum {
  constructor(arr) {
    this.prefix = [0];
    for (let i = 0; i < arr.length; i++) {
      this.prefix.push(this.prefix[i] + arr[i]);
    }
    // Препроцесинг: O(n)
  }

  query(i, j) {
    return this.prefix[j + 1] - this.prefix[i];
    // Кожен запит: O(1)!
  }
}

// Використання:
const ps = new PrefixSum([3, 1, 4, 1, 5, 9, 2, 6]);
ps.query(2, 5); // 19
ps.query(0, 3); // 9
// Time: O(n) preprocess + O(1) per query
```

**Чому це класно?** Якщо у нас 10⁶ запитів -- наївно це 10⁶ × n. Prefix sum -- 10⁶ × 1 + n на preprocessing.

---

### Приклад 3.2: Subarray sum equals K

**Задача:** Скільки є неперервних підмасивів із сумою дорівнює `k`?

Приклад: `arr = [1, 1, 1], k = 2` → `2` (підмасиви `[0..1]` і `[1..2]`).

**Наївне O(n²):**

```javascript
function subarraySumNaive(arr, k) {
  let count = 0;
  for (let i = 0; i < arr.length; i++) {
    let sum = 0;
    for (let j = i; j < arr.length; j++) {
      sum += arr[j];
      if (sum === k) count++;
    }
  }
  return count;
}
// Time: O(n²), Space: O(1)
```

**Оптимізація з prefix sum + Map:**

Ідея: `sum(i..j) === k` еквівалентно `prefix[j+1] - prefix[i] === k`, тобто `prefix[i] === prefix[j+1] - k`.

Ідемо по масиву, тримаємо поточну `prefixSum` і в `Map` кількість кожної попередньої prefixSum. Для поточної `prefixSum` питаємо, чи була раніше `prefixSum - k`.

```javascript
function subarraySum(arr, k) {
  const prefixCount = new Map();
  prefixCount.set(0, 1);  // порожній префікс має суму 0

  let count = 0;
  let prefixSum = 0;

  for (const num of arr) {
    prefixSum += num;
    // Скільки разів ми бачили prefixSum - k раніше?
    // Стільки підмасивів закінчуються тут із сумою k.
    if (prefixCount.has(prefixSum - k)) {
      count += prefixCount.get(prefixSum - k);
    }
    prefixCount.set(prefixSum, (prefixCount.get(prefixSum) || 0) + 1);
  }

  return count;
}
// Time: O(n), Space: O(n)
```

**Чому так?** Якщо ми в позиції j і `prefixSum[j+1] - prefixSum[i] === k` для якогось `i <= j`, то підмасив `[i..j]` має суму k. Map зберігає, скільки разів кожна prefix sum зустрічалася, щоб ми могли порахувати всі відповідні `i` в O(1).

---

### Коли prefix sum -- правильний вибір

**Сигнали:**
1. "Сума підмасиву/діапазону".
2. "Скільки підмасивів із властивістю X (по сумі)".
3. Багато запитів на один і той самий масив.

**Варіації:**
- Prefix sum 2D -- для матриць.
- Prefix XOR -- те саме, але з XOR замість суми. Трюк для задач на "XOR підмасиву".
- Prefix product -- рідше, треба пам'ятати про нулі.

---

## Робота з рядками

У JavaScript рядки мають особливість -- вони **immutable**. Це впливає на Big O в неочевидних місцях.

### Чому `str += x` в циклі -- O(n²)

Коли ти робиш `str += "a"`, JS **не змінює** оригінальний рядок. Він створює **новий** рядок, копіюючи всі символи з старого + новий символ.

```javascript
let str = "hello";
str += "!";
// Під капотом:
// 1. Виділити пам'ять для нового рядка довжиною 6.
// 2. Скопіювати "hello" (5 байтів).
// 3. Додати "!".
// 4. str тепер вказує на новий рядок. Старий -- garbage.
```

У циклі:

```javascript
let result = "";
for (let i = 0; i < n; i++) {
  result += "x";   // створюємо рядок довжини 1, потім 2, потім 3...
}
// Робота: 1 + 2 + 3 + ... + n = n(n+1)/2 = O(n²)
```

**Правильно:**

```javascript
const parts = [];
for (let i = 0; i < n; i++) {
  parts.push("x");
}
const result = parts.join("");
// Робота: O(n) на push + O(n) на join = O(n)
```

V8 у сучасних версіях уміє оптимізувати деякі випадки `+=`, але **покладатися на це не варто**. На співбесіді завжди використовуй масив + join.

---

### Anagram check: sort vs counting

**Задача:** Два рядки -- анаграми (з тих самих букв), чи ні?

Приклад: `"listen"` і `"silent"` → `true`.

**Варіант 1: sort.**

```javascript
function isAnagramSort(a, b) {
  if (a.length !== b.length) return false;

  const sorted = s => s.split('').sort().join('');
  return sorted(a) === sorted(b);
}
// Time: O(n log n) -- сортування
// Space: O(n) -- через split
```

Працює. Простий код. Але можна швидше.

**Варіант 2: counting.**

```javascript
function isAnagram(a, b) {
  if (a.length !== b.length) return false;

  const count = new Map();
  for (const ch of a) {
    count.set(ch, (count.get(ch) || 0) + 1);
  }
  for (const ch of b) {
    if (!count.has(ch)) return false;
    count.set(ch, count.get(ch) - 1);
    if (count.get(ch) === 0) count.delete(ch);
  }
  return count.size === 0;
}
// Time: O(n), Space: O(alphabet) -- максимум 26 букв для ASCII
```

**Чому counting швидший?** Sort робить log-n роботи на порівняннях. Counting -- просто рахує частоту кожного символу за один прохід.

**Коли все ж використовувати sort?**
- Коли `n` маленьке і код простіший має значення.
- Коли алфавіт дуже великий (unicode), і Map буде такий самий як sort.

---

### Reverse words

**Задача:** Дано рядок `"the sky is blue"`. Поверни `"blue is sky the"`.

**Простий варіант через split/reverse:**

```javascript
function reverseWords(s) {
  return s.trim().split(/\s+/).reverse().join(' ');
}
// Time: O(n), Space: O(n)
```

**In-place варіант (для низькорівневого інтерв'ю):**

Крок 1: розвернути весь рядок. Крок 2: розвернути кожне слово.

```javascript
function reverseWordsInPlace(arr) {
  // arr -- масив символів

  // Крок 1: розвернути весь масив
  reverseRange(arr, 0, arr.length - 1);

  // Крок 2: розвернути кожне слово
  let start = 0;
  for (let end = 0; end <= arr.length; end++) {
    if (end === arr.length || arr[end] === ' ') {
      reverseRange(arr, start, end - 1);
      start = end + 1;
    }
  }
}

function reverseRange(arr, left, right) {
  while (left < right) {
    [arr[left], arr[right]] = [arr[right], arr[left]];
    left++;
    right--;
  }
}

// Приклад:
// "the sky is blue"
// Крок 1:        "eulb si yks eht"
// Крок 2:        "blue is sky the"
```

Це класичний трюк, який варто знати.

---

### String matching: що треба знати

Якщо треба перевірити, чи рядок `pattern` є підрядком `text`:

**Наївний варіант O(n × m):**

```javascript
function contains(text, pattern) {
  for (let i = 0; i <= text.length - pattern.length; i++) {
    let match = true;
    for (let j = 0; j < pattern.length; j++) {
      if (text[i + j] !== pattern[j]) {
        match = false;
        break;
      }
    }
    if (match) return i;
  }
  return -1;
}
// Time: O(n × m), де n = text.length, m = pattern.length
```

**Швидкі алгоритми -- O(n + m):**
- **KMP (Knuth-Morris-Pratt)** -- використовує таблицю префіксів для пропуску порівнянь.
- **Boyer-Moore** -- порівнює з кінця шаблону, уміє пропускати великі шматки.
- **Rabin-Karp** -- хешує вікна, підходить для пошуку **багатьох** шаблонів.

**На співбесіді:**
- Знати, що вони існують і O(n + m).
- Не треба вміти писати KMP з пам'яті (якщо ти не на Google/Meta senior+).
- Часто інтерв'юер прийме `text.indexOf(pattern)` (V8 використовує оптимізований алгоритм).

---

## In-place модифікації масивів

Часто задачі просять "зроби in-place, O(1) space". Ось типові техніки.

### Чому `splice` -- не in-place в сенсі O(1)

```javascript
arr.splice(i, 1);  // видалити 1 елемент на позиції i
```

Хоч `splice` модифікує той самий масив (не створює новий), він **зсуває всі елементи після i** -- це **O(n)** роботи.

Якщо робити `splice` у циклі -- маєш O(n²).

```javascript
// ❌ O(n²)
function removeEvens(arr) {
  for (let i = 0; i < arr.length; i++) {
    if (arr[i] % 2 === 0) {
      arr.splice(i, 1);
      i--;
    }
  }
}

// ✅ O(n) через two pointers
function removeEvens(arr) {
  let slow = 0;
  for (let fast = 0; fast < arr.length; fast++) {
    if (arr[fast] % 2 !== 0) {
      arr[slow] = arr[fast];
      slow++;
    }
  }
  arr.length = slow;  // "обрізати" масив
}
```

### Move zeros до кінця

**Задача:** Зберігаючи порядок ненульових, перенеси всі нулі в кінець.

Приклад: `[0, 1, 0, 3, 12]` → `[1, 3, 12, 0, 0]`.

```javascript
function moveZeros(arr) {
  let slow = 0;

  // Крок 1: копіюємо всі ненульові на початок
  for (let fast = 0; fast < arr.length; fast++) {
    if (arr[fast] !== 0) {
      arr[slow] = arr[fast];
      slow++;
    }
  }

  // Крок 2: заповнюємо хвіст нулями
  for (let i = slow; i < arr.length; i++) {
    arr[i] = 0;
  }
}
// Time: O(n), Space: O(1)
```

### Rotate array (циклічний зсув)

**Задача:** Зсунь масив праворуч на `k`.

Приклад: `[1,2,3,4,5,6,7], k = 3` → `[5,6,7,1,2,3,4]`.

**Наївне O(n × k):**

```javascript
function rotateNaive(arr, k) {
  k %= arr.length;
  for (let i = 0; i < k; i++) {
    arr.unshift(arr.pop());   // O(n) × k разів
  }
}
```

**Трюк із трьома реверсами O(n), O(1) space:**

```javascript
function rotate(arr, k) {
  k %= arr.length;

  reverse(arr, 0, arr.length - 1);         // розвернути весь
  reverse(arr, 0, k - 1);                   // розвернути перші k
  reverse(arr, k, arr.length - 1);          // розвернути решту
}

function reverse(arr, left, right) {
  while (left < right) {
    [arr[left], arr[right]] = [arr[right], arr[left]];
    left++;
    right--;
  }
}

// Приклад для [1,2,3,4,5,6,7], k = 3:
// Після reverse всього:         [7,6,5,4,3,2,1]
// Після reverse перших 3:       [5,6,7,4,3,2,1]
// Після reverse решти:          [5,6,7,1,2,3,4]  ✓
```

Це красивий трюк. Його варто знати.

---

## Типові задачі: від brute force до оптимуму

### Задача 1: Two Sum (найпопулярніша задача співбесід)

**Задача:** Дано масив (не відсортований!) і target. Знайди індекси двох чисел, що дають суму = target.

**Brute force:**

```javascript
function twoSumBrute(nums, target) {
  for (let i = 0; i < nums.length; i++) {
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] + nums[j] === target) return [i, j];
    }
  }
  return [];
}
// Time: O(n²), Space: O(1)
```

**Оптимум через Map:**

```javascript
function twoSum(nums, target) {
  const seen = new Map();   // value -> index

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];

    if (seen.has(complement)) {
      return [seen.get(complement), i];
    }

    seen.set(nums[i], i);
  }

  return [];
}
// Time: O(n), Space: O(n)
```

**Чому це швидше?** Замість шукати пару подвійним циклом, ми **запам'ятовуємо** всі побачені числа у Map. Для кожного `nums[i]` питаємо Map "чи було колись `target - nums[i]`?" -- це O(1).

**Trade-off:** виграли час (n² → n), заплатили пам'яттю (O(1) → O(n)).

---

### Задача 2: Best Time to Buy and Sell Stock

**Задача:** Дано масив цін акції по днях. Вибери один день для купівлі та один (пізніший) для продажу. Максимізуй прибуток.

Приклад: `[7,1,5,3,6,4]` → `5` (купити за 1, продати за 6).

**Brute force:**

```javascript
function maxProfitBrute(prices) {
  let maxProfit = 0;
  for (let i = 0; i < prices.length; i++) {
    for (let j = i + 1; j < prices.length; j++) {
      maxProfit = Math.max(maxProfit, prices[j] - prices[i]);
    }
  }
  return maxProfit;
}
// Time: O(n²), Space: O(1)
```

**Оптимум -- одним проходом:**

```javascript
function maxProfit(prices) {
  let minPrice = Infinity;
  let maxProfit = 0;

  for (const price of prices) {
    if (price < minPrice) {
      minPrice = price;
    } else if (price - minPrice > maxProfit) {
      maxProfit = price - minPrice;
    }
  }

  return maxProfit;
}
// Time: O(n), Space: O(1)
```

**Чому це працює?** У кожній точці `i` ми знаємо **мінімальну ціну** серед `prices[0..i]`. Максимальний прибуток від продажу саме в день `i` -- це `prices[i] - minSoFar`. Ітеруємо, беремо максимум.

Це **не two pointers, не sliding window**, а просто "тримай running min і running result". Цей патерн теж дуже частий -- варто запам'ятати.

---

### Задача 3: Contains Duplicate

**Задача:** Чи містить масив дублікати?

**Brute force:**

```javascript
function containsDuplicateBrute(nums) {
  for (let i = 0; i < nums.length; i++) {
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] === nums[j]) return true;
    }
  }
  return false;
}
// Time: O(n²), Space: O(1)
```

**Sort-based:**

```javascript
function containsDuplicateSort(nums) {
  nums.sort((a, b) => a - b);
  for (let i = 1; i < nums.length; i++) {
    if (nums[i] === nums[i - 1]) return true;
  }
  return false;
}
// Time: O(n log n), Space: O(1) (модифікує вхід)
```

**Set-based:**

```javascript
function containsDuplicate(nums) {
  const seen = new Set();
  for (const x of nums) {
    if (seen.has(x)) return true;
    seen.add(x);
  }
  return false;
}
// Time: O(n), Space: O(n)

// Однорядковий варіант:
function containsDuplicate(nums) {
  return new Set(nums).size !== nums.length;
}
```

**Який обрати?** Залежить від обмежень:
- Якщо пам'яті мало -- sort.
- Якщо швидкість важливіша -- Set.
- На співбесіді проговорюй обидва варіанти і trade-off.

---

### Задача 4: Product of Array Except Self (без ділення)

**Задача:** Повернути масив, де `output[i] = добуток всіх елементів, крім nums[i]`. Не можна використовувати ділення. O(n) time.

Приклад: `[1, 2, 3, 4]` → `[24, 12, 8, 6]`.

**Brute force O(n²):**

```javascript
function productExceptSelfBrute(nums) {
  const result = [];
  for (let i = 0; i < nums.length; i++) {
    let product = 1;
    for (let j = 0; j < nums.length; j++) {
      if (j !== i) product *= nums[j];
    }
    result.push(product);
  }
  return result;
}
```

**Trick: "prefix product" і "suffix product".**

Для позиції `i`:
- Добуток всіх елементів **лівіше** `i` -- назвемо `leftProduct[i]`.
- Добуток всіх елементів **правіше** `i` -- назвемо `rightProduct[i]`.
- Відповідь `output[i] = leftProduct[i] * rightProduct[i]`.

```javascript
function productExceptSelf(nums) {
  const n = nums.length;
  const result = new Array(n).fill(1);

  // Прохід зліва направо: result[i] = добуток усіх лівіше i
  let leftProduct = 1;
  for (let i = 0; i < n; i++) {
    result[i] = leftProduct;
    leftProduct *= nums[i];
  }

  // Прохід справа наліво: домножаємо на добуток усіх правіше i
  let rightProduct = 1;
  for (let i = n - 1; i >= 0; i--) {
    result[i] *= rightProduct;
    rightProduct *= nums[i];
  }

  return result;
}
// Time: O(n), Space: O(1) (не рахуючи output, як зазвичай домовляються)
```

**Розбір для `[1, 2, 3, 4]`:**

```
Прохід 1 (зліва):
  i=0: result[0] = 1 (нічого лівіше)
       leftProduct = 1 * 1 = 1
  i=1: result[1] = 1
       leftProduct = 1 * 2 = 2
  i=2: result[2] = 2
       leftProduct = 2 * 3 = 6
  i=3: result[3] = 6
       leftProduct = 6 * 4 = 24

Після проходу 1: result = [1, 1, 2, 6]

Прохід 2 (справа):
  i=3: result[3] = 6 * 1 = 6
       rightProduct = 1 * 4 = 4
  i=2: result[2] = 2 * 4 = 8
       rightProduct = 4 * 3 = 12
  i=1: result[1] = 1 * 12 = 12
       rightProduct = 12 * 2 = 24
  i=0: result[0] = 1 * 24 = 24
       rightProduct = 24 * 1 = 24

Результат: [24, 12, 8, 6]  ✓
```

**Чому це красиво?** Два лінійних проходи з двох напрямків -- і все зроблено без ділення.

---

## Типові пастки

### Пастка 1: `unshift`/`shift` у циклі

```javascript
// ❌ O(n²) через unshift
function reverse(arr) {
  const result = [];
  for (const x of arr) {
    result.unshift(x);  // кожен unshift -- O(n)!
  }
  return result;
}

// ✅ O(n)
function reverse(arr) {
  const result = [];
  for (let i = arr.length - 1; i >= 0; i--) {
    result.push(arr[i]);
  }
  return result;
}
```

**Правило:** ніколи не використовуй `unshift` і `shift` в циклі. Якщо потрібна швидка черга з обох кінців -- використовуй deque (можна реалізувати на Map + two pointers на індекси), або просто думай інакше.

### Пастка 2: Створення нового масиву щоразу (spread в циклі)

```javascript
// ❌ O(n²)
function addAll(arr, items) {
  for (const item of items) {
    arr = [...arr, item];    // кожен spread -- O(n)
  }
  return arr;
}

// ✅ O(n)
function addAll(arr, items) {
  const result = [...arr];   // один раз
  for (const item of items) {
    result.push(item);         // O(1) amortized
  }
  return result;
}
```

Spread оператор -- красивий, але в циклі він робить з O(n) задачі O(n²).

### Пастка 3: Concat рядка в циклі

```javascript
// ❌ O(n²) через immutability
function build(n) {
  let result = "";
  for (let i = 0; i < n; i++) {
    result += String(i);
  }
  return result;
}

// ✅ O(n)
function build(n) {
  const parts = [];
  for (let i = 0; i < n; i++) {
    parts.push(String(i));
  }
  return parts.join("");
}
```

### Пастка 4: Off-by-one errors

Вони -- улюблена причина багів у масивах. Типові випадки:

```javascript
// Чи треба right - 1 чи right?
for (let i = 0; i < arr.length; i++) { ... }        // включно 0, невключно arr.length
for (let i = 0; i <= arr.length - 1; i++) { ... }  // те саме
for (let i = 1; i < arr.length; i++) { ... }        // пропустили arr[0]

// Довжина вікна
const windowSize = right - left + 1;  // включаючи обидві межі
const halfSize = right - left;         // НЕ включаючи right

// Substring / slice
arr.slice(i, j);   // від i включно до j НЕвключно -- довжина j - i
```

**Порада:** пиши на папері/у голові маленький приклад (довжина 3-4) і перевіряй індекси перед submit.

### Пастка 5: Мутація під час ітерації

```javascript
// ❌ Можна пропустити елементи
for (let i = 0; i < arr.length; i++) {
  if (arr[i] === 0) arr.splice(i, 1);
  // Після splice наступний елемент опиняється на поточному i,
  // але i++ перескакує через нього.
}

// ✅ Ітеруй з кінця, або використовуй фільтр
for (let i = arr.length - 1; i >= 0; i--) {
  if (arr[i] === 0) arr.splice(i, 1);
}
// Або:
arr = arr.filter(x => x !== 0);
// Або two pointers (див. moveZeros вище).
```

### Пастка 6: Sparse arrays

```javascript
const arr = new Array(5);   // [<5 empty items>]
arr.length;                  // 5
arr.map(x => x);             // [<5 empty items>] -- map пропускає empty!
arr.forEach(x => console.log(x));  // нічого не виведе

const arr2 = new Array(5).fill(0);  // [0, 0, 0, 0, 0]
// Тепер map, forEach працюють нормально.
```

**Порада:** коли створюєш масив фіксованого розміру, **завжди** використовуй `.fill(value)` або `Array.from({length: n}, ...)`.

---

## Шпаргалка: як підходити до задачі на масиви/рядки

1. **Прочитати задачу 2 рази, написати приклади на папері.**
2. **Проговорити обмеження:** розмір входу, допустимий Big O, чи можна модифікувати вхід, чи є додаткові структури.
3. **Спробувати наївне рішення.** Для масивів це зазвичай O(n²) з двома циклами. Порахувати Big O.
4. **Подумати про патерн:**
   - Відсортований масив → бінарний пошук, two pointers.
   - Неперервний підмасив → sliding window, prefix sum.
   - Пошук пари з властивістю → Map для O(1) lookup.
   - In-place → two pointers (slow/fast).
   - Рядок → не забудь про immutability.
5. **Написати код, перевірити на прикладах.**
6. **Граничні випадки:** порожній вхід, один елемент, усі однакові, від'ємні, дуже великі числа.

---

## Ключові думки

- Масив -- неперервний блок пам'яті. Доступ O(1), вставка/видалення в середині або на початку -- O(n).
- **Three patterns to rule them all:** two pointers, sliding window, prefix sum. 80% задач розв'язуються одним з цих.
- **Map і Set** -- найкращі друзі для перетворення O(n²) на O(n).
- Рядки immutable -- уникай `+=` у циклі, використовуй `array + join`.
- In-place модифікації -- це two pointers, а не `splice`. `splice` сам по собі O(n).
- На співбесіді **проговорюй Big O голосно** на кожному кроці. Інтерв'юер оцінює мислення, не тільки результат.

У наступному файлі розглянемо hashmap-based задачі детальніше -- як і коли використовувати Map/Set для агресивної оптимізації.
