# HTML та CSS основи

## Що таке семантичний HTML і чому він важливий?

Семантичний HTML — це використання HTML5 елементів, які описують значення контенту, а не лише його зовнішній вигляд. Замість `<div>` повсюди, ми використовуємо `<header>`, `<nav>`, `<article>`, `<section>`, `<aside>`, `<footer>` та інші елементи.

**Чому важливо:**
- SEO: пошукові системи краще розуміють структуру сторінки
- Доступність (a11y): скрін-рідери можуть правильно навігувати контент
- Підтримуваність: легше зрозуміти код без коментарів
- Семантика: HTML описує не "як виглядає", а "що це означає"

```html
<!-- ❌ Неправильно: всі div-и -->
<div class="header">
  <div class="nav">
    <div class="nav-item"><a href="/home">Домашня сторінка</a></div>
  </div>
</div>
<div class="main-content">
  <div class="article">
    <h1>Заголовок статті</h1>
    <p>Текст статті...</p>
  </div>
  <div class="sidebar">Бічна панель</div>
</div>
<div class="footer">Підвал</div>

<!-- ✅ Правильно: семантичні елементи -->
<header>
  <nav>
    <a href="/home">Домашня сторінка</a>
  </nav>
</header>
<main>
  <article>
    <h1>Заголовок статті</h1>
    <p>Текст статті...</p>
  </article>
  <aside>Бічна панель</aside>
</main>
<footer>Підвал</footer>
```

```html
<!-- Основні семантичні елементи -->
<header>      <!-- Верхня частина сторінки, логотип, назва -->
<nav>         <!-- Навігаційні посилання -->
<main>        <!-- Основний контент сторінки (один на сторінку) -->
<article>     <!-- Самостійна статья, блог-пост, новина -->
<section>     <!-- Логічна секція контенту з заголовком -->
<aside>       <!-- Бічний контент, сиджбар, реклама -->
<footer>      <!-- Нижня частина сторінки -->
```

---

## Що таке CSS Box Model? Як він працює?

CSS Box Model описує, як браузер розраховує ширину та висоту елемента. Кожен елемент складається з чотирьох концентричних прямокутників (від усередини назовні):

```
┌─────────────────────────────────────┐
│          MARGIN (зовні)             │
│  ┌───────────────────────────────┐  │
│  │   BORDER (межа)               │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │ PADDING (внутрішній стиск)│  │  │
│  │  │ ┌───────────────────┐  │  │  │
│  │  │ │   CONTENT         │  │  │  │
│  │  │ │  (текст, картинки)│  │  │  │
│  │  │ └───────────────────┘  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

```css
/* За замовчуванням: box-sizing: content-box */
.box {
  width: 200px;          /* ширина ЛИШЕ контенту */
  padding: 20px;         /* додає 20px з кожного боку */
  border: 5px solid;     /* додає 5px межу */
  margin: 10px;          /* простір ЗЗОВНІ */
}
/* Результат: 200 + 20*2 + 5*2 = 250px фактична ширина */
```

```css
/* Рекомендовано: box-sizing: border-box */
.box {
  box-sizing: border-box;
  width: 200px;      /* ширина = контент + padding + border */
  padding: 20px;
  border: 5px solid;
  margin: 10px;      /* margin НЕ входить у ширину */
}
/* Результат: 200px = контент + padding + border */
```

```javascript
// Рекомендується встановити глобально
const style = `
  * {
    box-sizing: border-box;
  }
`;
```

| Властивість | `content-box` (старо) | `border-box` (сучасно) |
|---|---|---|
| Що включено в width | Лише контент | Контент + padding + border |
| Коли змінювати padding | Змінює фактичну ширину | Ширина не змінюється |
| Рекомендація | Не використовувати | **Використовувати за замовчуванням** |

---

## Яка різниця між display: block, inline, inline-block, flex та grid?

`display` контролює, як елемент розташовується та займає місце на сторінці.

```
┌─────────────────────────────────────┐
│ display: block                       │
│ Займає всю ширину, перехід на новий │
│ рядок, margin/padding працюють       │
└─────────────────────────────────────┘

┌──────────┐ ┌──────────┐
│  inline  │ │ inline   │  На одному рядку, margin-left/-right
└──────────┘ └──────────┘  працюють, margin-top/-bottom ні

┌──────────┐ ┌──────────┐
│ inline-  │ │ inline-  │  Як inline, але margin/padding
│ block    │ │ block    │  працюють повністю
└──────────┘ └──────────┘
```

```html
<!-- Демонстрація -->
<div class="container">
  <div class="block">Block: займає всю ширину</div>
  <div class="block">Новий рядок автоматично</div>
</div>

<style>
  .block {
    display: block;
    width: 100%;  /* займає всю доступну ширину */
    margin-bottom: 20px;
  }
</style>
```

```css
/* display: flex - однвимірна компоновка (рядок або стовпець) */
.flex-container {
  display: flex;
  justify-content: space-between;  /* розподіл по горизонталі */
  align-items: center;             /* вирівнювання по вертикалі */
  gap: 10px;                       /* проміжок між дітьми */
}

.flex-item {
  flex: 1;         /* рівний розподіл місця */
  /* або: flex-grow: 1; flex-shrink: 1; flex-basis: 0; */
}
```

```css
/* display: grid - двовимірна компоновка (рядки + стовпці) */
.grid-container {
  display: grid;
  grid-template-columns: 200px 1fr 200px;  /* три стовпці */
  grid-template-rows: auto 1fr auto;       /* три рядки */
  gap: 20px;
}

.grid-item {
  /* можна розташовувати на конкретні комірки */
  grid-column: 1 / 3;  /* від стовпця 1 до 3 */
  grid-row: 2;         /* в рядку 2 */
}
```

| Коли використовувати | Приклад |
|---|---|
| **block** | Параграфи, заголовки, розділи |
| **inline** | Посилання у тексті, спану слів |
| **inline-block** | Кнопки, іконки (рідко) |
| **flex** | Навігація, центрування, однорядкові компоненти |
| **grid** | Макети сторінки, таблиці даних, картки |

---

## Як працює CSS Position?

`position` контролює, як елемент позиціонується відносно свого нормального потоку.

```
┌────────────────────────────────────────┐
│ static (за замовчуванням)              │
│ Слідує нормальному потоку, top/left    │
│ ігноруються, z-index не працює         │
└────────────────────────────────────────┘
```

```css
/* relative: зміщує елемент від його нормального положення */
.relative {
  position: relative;
  top: 20px;    /* зміщення від верхнього краю */
  left: 20px;   /* але місце у потоку залишається! */
}
```

```css
/* absolute: виймає елемент з потоку, позиціонує щодо найближчого
   батька з position !== static */
.container {
  position: relative;  /* створює контекст позиціонування */
}

.absolutely-positioned {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);  /* центрування */
  /* Займає 0 місця у нормальному потоці */
}
```

```css
/* fixed: позиціонується щодо viewport (вікна браузера) */
.sticky-header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 100;  /* щоб бути на верхньому шарі */
  /* Залишається на місці навіть при скролінгу */
}
```

```css
/* sticky: гібрид relative та fixed */
.sticky-nav {
  position: sticky;
  top: 0;  /* прикріплюється до верхньої частини viewport */
  /* Слідує потоку, поки контейнер видимий,
     потім "липнути" до верхньої частини */
}
```

| Position | Контекст | Займає місце | Стану скролінгу |
|---|---|---|---|
| **static** | document | Так | - |
| **relative** | своє місце | Так | - |
| **absolute** | батько (relative) | Ні | - |
| **fixed** | viewport | Ні | Ігнорує скролінг |
| **sticky** | батько + viewport | Таки (поки видимо) | Слідує при скролінгу |

---

## Flexbox: як працюють axes, justify-content, align-items?

Flexbox організує дітей лінійно вздовж **main axis** (основна вісь) та **cross axis** (поперечна вісь).

```
  Row (flex-direction: row) — за замовчуванням
  ─────────────────────────────────────►
  │           main axis (горизонталь)
  │
  │ cross axis (вертикаль)
  ▼

  Column (flex-direction: column)
  │
  │ main axis (вертикаль)
  ▼
  ───────── cross axis (горизонталь)────────►
```

```css
.container {
  display: flex;
  flex-direction: row;        /* або column, row-reverse, column-reverse */
  
  /* Розподіл вздовж main axis */
  justify-content: space-between;
  /* можливі значення:
     flex-start (за замовчуванням), flex-end, center,
     space-between, space-around, space-evenly
  */
  
  /* Вирівнювання вздовж cross axis */
  align-items: center;
  /* можливі значення:
     flex-start, flex-end, center, stretch (за замовчуванням), baseline
  */
  
  gap: 20px;  /* проміжок між дітьми */
}

.item {
  flex: 1;              /* flex-grow, flex-shrink, flex-basis */
  /* або окремо:
     flex-grow: 1;      розширюється, якщо є місце
     flex-shrink: 1;    стискується, якщо не хватає місця
     flex-basis: auto;  базова ширина (за замовчуванням: width)
  */
}
```

```html
<!-- Приклад навігації -->
<nav class="navbar">
  <div class="logo">Logo</div>
  <div class="nav-items">
    <a href="#home">Home</a>
    <a href="#about">About</a>
    <a href="#contact">Contact</a>
  </div>
  <div class="user-menu">Профіль</div>
</nav>

<style>
  .navbar {
    display: flex;
    justify-content: space-between;  /* logo + items + menu розподілені */
    align-items: center;             /* вирівнювання по вертикалі */
    padding: 20px;
    background: #333;
  }
  
  .nav-items {
    display: flex;
    gap: 20px;  /* проміжок між посиланнями */
  }
  
  .nav-items a {
    flex: 0 0 auto;  /* не розширюватись, не стискуватись */
  }
</style>
```

---

## CSS Grid: rows, columns, template-areas, Flexbox vs Grid?

CSS Grid організує контент у двовимірну таблицю рядків та стовпців.

```
  1      2      3
┌──────┬──────┬──────┐
│      │      │      │ 1
├──────┼──────┼──────┤
│      │      │      │ 2
├──────┼──────┼──────┤
│      │      │      │ 3
└──────┴──────┴──────┘
```

```css
/* Простий grid */
.grid {
  display: grid;
  grid-template-columns: 1fr 2fr 1fr;  /* 3 стовпці, середній подвійної ширини */
  grid-template-rows: auto 200px auto; /* висоти рядків */
  gap: 20px;  /* проміжок в обох напрямках */
}
```

```css
/* Grid з template-areas (назване розташування) */
.layout {
  display: grid;
  grid-template-columns: 200px 1fr 200px;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header header"
    "sidebar main aside"
    "footer footer footer";
  gap: 20px;
}

header {
  grid-area: header;  /* займає область, названу "header" */
}

.sidebar {
  grid-area: sidebar;
}

main {
  grid-area: main;
}

aside {
  grid-area: aside;
}

footer {
  grid-area: footer;
}
```

```css
/* fr (fraction) — розподіл вільного місця */
.responsive-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  /* автоматично створює стовпці мінімум 250px,
     розширюються до рівних долей вільного місця */
  gap: 20px;
}
```

| Характеристика | **Flexbox** | **Grid** |
|---|---|---|
| **Виміри** | 1D (рядок або стовпець) | 2D (рядки + стовпці) |
| **Коли** | Навігація, списки, центрування | Макети сторінок, таблиці |
| **Вирівнювання** | justify-content, align-items | grid-column, grid-row |
| **Адаптивність** | flex-wrap + media queries | repeat(auto-fit) + minmax |

---

## Що таке CSS Specificity? Як рахується порядок правил?

Specificity визначає, яке CSS правило буде застосовано при конфлікті. Чим вище specificity, тим вище пріоритет.

```
Specificity рахується як: (a, b, c)
a = кількість ID селекторів (#id)
b = кількість класів, псевдо-класів (:[^:], .class)
c = кількість елементів та псевдо-елементів (::, div)

Порівняння: 100 > 010 > 001
```

```css
/* Приклади specificity */
div { }                        /* (0, 0, 1) */
.class { }                     /* (0, 1, 0) */
#id { }                        /* (1, 0, 0) */
div.class { }                  /* (0, 1, 1) */
div.class:hover { }            /* (0, 2, 1) */
#id .class div:first-child { } /* (1, 1, 2) */
```

```html
<!-- Приклад конфлікту -->
<button id="btn" class="action">Клік</button>

<style>
  button { color: blue; }         /* (0, 0, 1) */
  .action { color: red; }         /* (0, 1, 0) */
  #btn { color: green; }          /* (1, 0, 0) */
  
  /* Переможець: #btn { color: green; } */
  /* ID має найвищу specificity */
</style>
```

```css
/* !important — наслідує selectivity, але вище за все інше */
.class { color: red !important; }
#id { color: blue; }

/* color буде red, оскільки !important переважає */
/* Тим не менш: УНИКАЙТЕ !important, використовуйте правильну specificity */
```

**Порядок пріоритету (від найнижчого до найвищого):**
1. Browser defaults
2. Елементи (div, p)
3. Класи (.class), псевдо-класи (:hover)
4. ID (#id)
5. Inline style (`style=""`)
6. !important

---

## Як працює z-index та Stacking Context?

`z-index` контролює, який елемент у з 3D-глибині (вглиб/виглиб екрану) розташовується на верхньому шарі.

```
z-index: 3   ┌──────┐         Більший z-index = вище
             │      │  ◄──── видно повністю
             └──────┘

z-index: 2   ┌──────┐
             │      │  ◄──── частково видна
             └──────┘

z-index: 1   ┌──────┐
             │      │  ◄──── позаду
             └──────┘
```

```css
.modal {
  position: absolute;
  z-index: 1000;  /* високе значення = верхній шар */
}

.background {
  position: relative;
  z-index: 1;
}
```

**Стинг контекст (Stacking Context)** — це концепція, яку часто неправильно розуміють. `z-index` працює ЛИШЕ в межах одного stacking context.

```html
<!-- Пастка: z-index не працює як очікується -->
<div class="parent-1">
  <div class="child-1" style="z-index: 1000;">Я високий z-index</div>
</div>

<div class="parent-2">
  <div class="child-2" style="z-index: 1;">Я низький z-index</div>
</div>

<style>
  .parent-2 {
    position: relative;
    z-index: 1;  /* батько-2 має вищий z-index за батька-1 */
  }
  
  /* Результат: child-2 буде ВИЩЕ за child-1,
     оскільки parent-2 > parent-1, незважаючи на z-index дітей */
</style>
```

**Що створює новий Stacking Context:**
- `position: relative/absolute/fixed` + `z-index` (не auto)
- `opacity` < 1
- `transform`, `filter`, `perspective` (не none)
- `will-change` з вищезазначеними властивостями

```css
/* Правильно: встановлювати z-index батькам, а не дітям */
.modal-container {
  position: relative;
  z-index: 100;  /* створює новий stacking context */
}

.modal-content {
  z-index: 1;  /* достатньо, оскільки батько уже зверху */
}
```

---

## Що таке CSS Custom Properties (--var)? Як вони працюють?

CSS Custom Properties (змінні) дозволяють повторно використовувати значення та динамічно змінювати стилі.

```css
/* Оголошення змінних у :root (глобально) */
:root {
  --primary-color: #3498db;
  --secondary-color: #e74c3c;
  --padding-lg: 20px;
  --padding-sm: 10px;
  --border-radius: 8px;
}

/* Використання */
.button {
  background: var(--primary-color);
  padding: var(--padding-lg);
  border-radius: var(--border-radius);
}

.button:hover {
  background: var(--secondary-color);
}
```

```css
/* Scoping: змінні можуть визначатися на будь-якому рівні */
:root {
  --color: blue;  /* глобально */
}

.parent {
  --color: red;  /* тільки для .parent та його дітей */
}

.parent .child {
  /* color буде red, ініціалізовано з батька */
  color: var(--color);
}

.sibling {
  /* color буде blue, прямо з :root */
  color: var(--color);
}
```

```css
/* Fallback (запасне значення) */
.button {
  /* якщо --color не визначена, використовуємо blue */
  color: var(--color, blue);
}
```

```javascript
/* Зміна змінних через JavaScript */
const root = document.documentElement;

// Встановлювання
root.style.setProperty('--primary-color', '#2ecc71');

// Отримання
const primaryColor = getComputedStyle(root).getPropertyValue('--primary-color');
console.log(primaryColor); // #2ecc71

// Реальний приклад: dark mode
document.addEventListener('change', (e) => {
  if (e.target.id === 'dark-mode-toggle') {
    if (e.target.checked) {
      root.style.setProperty('--bg-color', '#1a1a1a');
      root.style.setProperty('--text-color', '#ffffff');
    } else {
      root.style.setProperty('--bg-color', '#ffffff');
      root.style.setProperty('--text-color', '#000000');
    }
  }
});
```

---

## Яка різниця між pseudo-classes та pseudo-elements?

**Pseudo-classes** (`:`) вибирають елементи в особливому стані.  
**Pseudo-elements** (`::`) створюють віртуальні елементи, які не існують у HTML.

```css
/* Pseudo-classes: описують стан */
a:hover { color: red; }        /* при наведенні миші */
input:focus { border: 2px solid blue; }  /* при фокусі */
li:first-child { font-weight: bold; }    /* перший елемент */
p:nth-child(2n) { background: #f0f0f0; } /* парні параграфи */
div:not(.special) { opacity: 0.7; }      /* усе крім .special */
```

```css
/* Pseudo-elements: створюють контент */
p::before {
  content: "→ ";  /* додавляємо контент ПЕРЕД параграфом */
}

p::after {
  content: " ←";  /* додавляємо контент ПІСЛЯ параграфом */
}

p::first-line {
  font-weight: bold;  /* стиль першої лінії параграфу */
}

p::first-letter {
  font-size: 2em;  /* стиль першої літери */
}
```

```html
<!-- Приклад: списки з іконками -->
<ul class="custom-list">
  <li>Перший пункт</li>
  <li>Другий пункт</li>
  <li>Третій пункт</li>
</ul>

<style>
  .custom-list {
    list-style: none;
    padding: 0;
  }
  
  .custom-list li::before {
    content: "✓ ";  /* замість стандартної кулі */
    color: green;
    font-weight: bold;
    margin-right: 10px;
  }
  
  .custom-list li:hover {
    background: #f0f0f0;
  }
</style>
```

| Тип | Синтаксис | Приклад | Що робить |
|---|---|---|---|
| **Pseudo-class** | `:state` | `:hover`, `:focus`, `:nth-child(n)` | Вибирає елементи у спеціальному стані |
| **Pseudo-element** | `::element` | `::before`, `::after`, `::first-line` | Створює віртуальні елементи |

---

## Responsive Design: media queries, mobile-first, viewport meta

Responsive Design дозволяє адаптувати сторінку до різних розмірів экранів.

```html
<!-- Важливо: viewport meta для мобільних пристроїв -->
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<!-- width=device-width: сторінка займає всю ширину пристрою -->
<!-- initial-scale=1.0: без масштабування при завантаженні -->
```

```css
/* Mobile-first підхід: починаємо з мобільних стилів */
/* Потім додаємо стилі для більших екранів */

body {
  font-size: 14px;     /* мобільний розмір */
  padding: 10px;
}

.container {
  width: 100%;  /* займає всю ширину на мобільному */
  display: grid;
  grid-template-columns: 1fr;  /* один стовпець на мобільному */
}

/* Планшет: від 768px */
@media (min-width: 768px) {
  body {
    font-size: 16px;
  }
  
  .container {
    max-width: 750px;
    margin: 0 auto;
    grid-template-columns: 1fr 1fr;  /* два стовпці */
  }
}

/* Десктоп: від 1024px */
@media (min-width: 1024px) {
  body {
    font-size: 18px;
  }
  
  .container {
    max-width: 1200px;
    grid-template-columns: 200px 1fr 200px;  /* три стовпці */
  }
}

/* Великі екрани: від 1440px */
@media (min-width: 1440px) {
  .container {
    max-width: 1400px;
  }
}
```

```css
/* Інші типи media queries */
@media (max-width: 480px) {
  /* на мобільних екранах до 480px */
}

@media (orientation: landscape) {
  /* горизонтальна орієнтація */
}

@media (prefers-color-scheme: dark) {
  /* користувач обраний темну тему в ОС */
  background: #1a1a1a;
  color: #fff;
}

@media (prefers-reduced-motion: reduce) {
  /* користувач просить менше анімацій */
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

**Критичні точки розриву (Breakpoints):**
- Mobile: 0–480px
- Tablet: 481px–768px
- Desktop: 769px–1024px
- Large Desktop: 1025px+

---

## CSS Units: px vs em vs rem vs % vs vh/vw

Різні одиниці вимірювання в CSS мають різні базові значення.

```
┌──────────────────────────────────────────────────────────┐
│ PX (pixels) — абсолютна одиниця                         │
│ 16px завжди = 16 пікселів, незалежно від контексту      │
├──────────────────────────────────────────────────────────┤
│ EM — відносно розміру шрифту батька                      │
│ Якщо батько 16px, то 1em = 16px; 2em = 32px             │
│ ⚠️ Може бути плутаниною при вкладаннях                    │
├──────────────────────────────────────────────────────────┤
│ REM — відносно розміру шрифту <html> (root)             │
│ Завжди те саме значення, навіть при вкладаннях           │
│ ✅ Рекомендується для глобальних розмірів                 │
├──────────────────────────────────────────────────────────┤
│ % — відносно батька                                      │
│ 50% width = половина батька, 100% height = висота батька│
├──────────────────────────────────────────────────────────┤
│ VH/VW — відносно viewport (вікна браузера)              │
│ 100vh = висота вікна, 100vw = ширина вікна              │
└──────────────────────────────────────────────────────────┘
```

```css
/* Рекомендована стратегія */
html {
  font-size: 16px;  /* базовий розмір */
}

body {
  font-size: 1rem;  /* 16px (спадає з html) */
}

h1 {
  font-size: 2rem;  /* 32px */
}

p {
  font-size: 1rem;  /* 16px */
  line-height: 1.5; /* безблогадна одиниця */
  padding: 1rem;    /* 16px */
}

.container {
  width: 90%;           /* 90% батька */
  max-width: 1200px;    /* максимум 1200px */
  margin: 0 auto;
}

.hero {
  height: 100vh;  /* займає всю висоту viewport */
}
```

```css
/* EM: пастка вкладаннях */
.parent {
  font-size: 16px;
}

.child {
  font-size: 1.5em;  /* 16 * 1.5 = 24px */
}

.nested-child {
  font-size: 1.5em;  /* 24 * 1.5 = 36px (!!! каскадне збільшення) */
}

/* Рішення: використовувати REM замість EM */
.nested-child {
  font-size: 1.5rem;  /* завжди 24px (1.5 * 16px з html) */
}
```

| Одиниця | База | Адаптивність | Рекомендація |
|---|---|---|---|
| **px** | Пікселі | Ні | Бордери, маленькі spacing |
| **em** | Батько font-size | Каскадна | Уникайте, брутна вкладання |
| **rem** | HTML font-size | Глобальна | ✅ Для розмірів шрифту та spacing |
| **%** | Батько | Залежить | Ширина контейнерів |
| **vh/vw** | Viewport | Залежить | Full-height sections, responsive |

---

## Типові пастки в CSS

### 1. Margin Collapse (схлопування відступів)

```css
.parent {
  background: blue;
  /* Проблема: margin дитини схлопується з margin батька */
}

.child {
  margin-top: 20px;
  /* Очікуємо 20px відступу всередину батька,
     але margin передається батькові! */
}

/* Рішення 1: Додати padding до батька */
.parent {
  padding-top: 1px;  /* або overflow: auto, display: flex */
}

/* Рішення 2: Використовувати padding замість margin */
.child {
  padding-top: 20px;
}
```

### 2. z-index без position

```css
.box {
  z-index: 100;  /* ❌ НЕ ПРАЦЮЄ! z-index вимагає position */
}

/* Рішення */
.box {
  position: relative;  /* або absolute, fixed */
  z-index: 100;       /* тепер працює */
}
```

### 3. Тисячі z-index значень

```css
.modal { z-index: 9999; }
.dropdown { z-index: 9998; }
.tooltip { z-index: 9997; }

/* ❌ Неправильно: числа напруги, складна підтримка */

/* ✅ Правильно: осмислені значення з документацією */
:root {
  --z-layer-base: 1;
  --z-layer-dropdown: 10;
  --z-layer-modal: 100;
  --z-layer-tooltip: 110;
}

.modal { z-index: var(--z-layer-modal); }
.dropdown { z-index: var(--z-layer-dropdown); }
.tooltip { z-index: var(--z-layer-tooltip); }
```

### 4. Забування box-sizing: border-box

```css
/* ❌ Неправильно: width змінюється при додаванні padding */
.box {
  width: 200px;
  padding: 20px;
  /* фактична ширина: 240px */
}

/* ✅ Правильно: встановіть глобально у * */
* {
  box-sizing: border-box;
}

.box {
  width: 200px;
  padding: 20px;
  /* фактична ширина: 200px */
}
```

### 5. Неправильне використання display: inline для spacing

```html
<span class="icon">🔔</span>
<span class="notification-count">5</span>

<style>
  .notification-count {
    display: inline;
    margin-left: 10px;  /* ❌ НЕ ПРАЦЮЄ! margin-left ігнорується для inline */
  }
  
  /* ✅ Правильно: використовуємо inline-block або margin на батькові */
  .notification-count {
    display: inline-block;
    margin-left: 10px;  /* тепер працює */
  }
</style>
```

### 6. Забування @media query точок розриву

```css
/* ❌ Неправильно: невелика сторінка має CSS для десктопу */
.sidebar { width: 200px; }
.main { width: calc(100% - 200px); }

/* На мобільному це сломається! */

/* ✅ Правильно: mobile-first */
.sidebar { display: none; }  /* сховано на мобільному */
.main { width: 100%; }

@media (min-width: 768px) {
  .sidebar { display: block; width: 200px; }
  .main { width: calc(100% - 200px); }
}
```

---

## Кілька швидких порад

1. **Завжди встановлюйте viewport meta у `<head>`** — без цього мобільне буде масштабувати сторінку неправильно.

2. **Використовуйте flexbox для одновимірних макетів** (навігація, списки) та **grid для двовимірних** (сторінка, таблиці).

3. **REM для масштабованих розмірів**, PX для неможливо масштабованих (бордери, outline).

4. **Старайтеся уникати absolute positioning** — grid або flexbox більш гнучкіші.

5. **CSS Variables (--var) — ваш друг** для теми, spacing, кольорів. Легко змінювати через JS.

6. **Mobile-first: починайте зі стилів для мобільних**, потім розширюйте для більших екранів.

7. **Документуйте z-index значення** у CSS змінних, не розкидайте магічні числа по коді.

8. **Избегайте !important** — це сигнал про проблему з specificity.

9. **Тестуйте на реальних мобільних пристроях** — браузерний девтулс еміляція не завжди точна.

10. **CSS Grid у production готов** — не бійтеся використовувати його для складних макетів.
