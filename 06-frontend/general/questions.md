# Загальні Frontend питання для інтерв'ю

## Як працює браузер (Rendering Pipeline)?

Коли браузер отримує HTML, він проходить через ці етапи:

```
HTML → DOM Tree
                 ↘
                   Render Tree → Layout → Paint → Composite
                 ↗
CSS  → CSSOM Tree
```

1. **Parsing HTML** → побудова DOM (Document Object Model)
2. **Parsing CSS** → побудова CSSOM (CSS Object Model)
3. **Render Tree** — об'єднання DOM + CSSOM (тільки видимі елементи)
4. **Layout (Reflow)** — обчислення розмірів і позицій кожного елемента
5. **Paint** — малювання пікселів (кольори, тіні, тексти)
6. **Composite** — складання шарів у фінальне зображення

**JavaScript** може блокувати parsing HTML (якщо без `async`/`defer`), тому скрипти краще ставити внизу `<body>` або з атрибутами `defer`/`async`.

---

## Critical Rendering Path

**Critical Rendering Path** — мінімальний набір кроків для першого рендеру сторінки.

**Оптимізація:**
1. **Мінімізувати критичний CSS** — inline critical CSS у `<head>`
2. **Defer некритичний CSS** — `media="print"` або завантаження через JS
3. **Defer JavaScript** — `<script defer>` або `<script async>`
4. **Зменшити розмір HTML/CSS/JS** — мініфікація, стиснення (gzip, brotli)
5. **Preload важливих ресурсів** — `<link rel="preload">`
6. **Preconnect** — `<link rel="preconnect" href="https://api.example.com">`

**Метрики (Core Web Vitals):**
- **LCP** (Largest Contentful Paint) — час до рендеру найбільшого елемента (< 2.5с)
- **INP** (Interaction to Next Paint) — затримка відповіді на взаємодію (< 200мс)
- **CLS** (Cumulative Layout Shift) — візуальна стабільність (< 0.1)

---

## Reflow vs Repaint

### Reflow (Layout)
Перерахунок розмірів і позицій елементів. **Дорога операція.**

**Що викликає reflow:**
- Зміна розмірів вікна
- Зміна шрифту
- Додавання/видалення елементів
- Зміна CSS: width, height, margin, padding, position
- Читання layout properties: offsetWidth, clientHeight, getBoundingClientRect()

### Repaint
Перемалювання елементів без зміни layout. **Дешевша операція.**

**Що викликає лише repaint:**
- Зміна color, background-color, visibility, box-shadow

**Оптимізація:**
- Групувати DOM зміни (DocumentFragment, або змінювати className замість окремих стилів)
- Використовувати `transform` та `opacity` для анімацій (вони не викликають reflow)
- `will-change: transform` — підказка браузеру для оптимізації
- Уникати читання layout properties у циклах

---

## Event Delegation

Замість прикріплення обробника до кожного елемента — прикріплюємо один до батьківського:

```js
// ❌ Обробник на кожному елементі (1000 обробників)
document.querySelectorAll(".item").forEach(item => {
  item.addEventListener("click", handleClick);
});

// ✅ Event delegation (1 обробник)
document.querySelector(".list").addEventListener("click", (e) => {
  if (e.target.matches(".item")) {
    handleClick(e);
  }
});
```

**Як працює:** події в DOM "спливають" (bubbling) від дочірнього до батьківського елемента. Ми ловимо подію на батьківському рівні і перевіряємо `event.target`.

**Переваги:**
- Менше обробників — менше пам'яті
- Працює з динамічно доданими елементами
- Простіше управляти

---

## Closure, Scope, Hoisting

### Closure
Функція, яка має доступ до змінних зовнішньої функції навіть після її завершення:
```js
function createCounter() {
  let count = 0;
  return {
    increment: () => ++count,
    getCount: () => count,
  };
}
const counter = createCounter();
counter.increment(); // 1
counter.increment(); // 2
```

### Scope
- **Global scope** — доступний всюди
- **Function scope** — `var` обмежений функцією
- **Block scope** — `let`/`const` обмежені блоком `{}`
- **Lexical scope** — функція бачить змінні з місця свого **оголошення**, не виклику

### Hoisting
JavaScript "піднімає" оголошення на початок scope:
```js
console.log(a); // undefined (var hoisted, але не значення)
var a = 5;

console.log(b); // ReferenceError (let/const у temporal dead zone)
let b = 10;

foo(); // працює (function declarations hoisted повністю)
function foo() { return "hello"; }

bar(); // TypeError (тільки var hoisted, не функція)
var bar = function() { return "hello"; };
```

---

## Prototype Chain

JavaScript використовує прототипне наслідування:

```js
const animal = { eat() { return "eating"; } };
const dog = Object.create(animal);
dog.bark = function() { return "woof"; };

dog.bark();  // "woof" — власна властивість
dog.eat();   // "eating" — знайдено через prototype chain
dog.fly();   // undefined → TypeError — не знайдено ніде
```

**Ланцюжок пошуку:**
```
dog → dog.__proto__ (animal) → animal.__proto__ (Object.prototype) → null
```

**class** у JavaScript — синтаксичний цукор над прототипами:
```js
class Animal {
  eat() { return "eating"; }
}
class Dog extends Animal {
  bark() { return "woof"; }
}
// Dog.prototype.__proto__ === Animal.prototype
```

---

## Promise, async/await — під капотом

### Promise
```js
const promise = new Promise((resolve, reject) => {
  // async operation
  setTimeout(() => resolve("done"), 1000);
});

promise
  .then(result => console.log(result))
  .catch(error => console.error(error))
  .finally(() => console.log("cleanup"));
```

**Стани:** pending → fulfilled (resolved) / rejected
**Promise.all** — всі повинні resolved
**Promise.allSettled** — чекає на всі (resolved або rejected)
**Promise.race** — перший settled
**Promise.any** — перший fulfilled

### async/await
Синтаксичний цукор над Promise:
```js
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    const user = await response.json();
    return user;
  } catch (error) {
    console.error("Failed:", error);
  }
}
```

**Під капотом:** async function повертає Promise, await "призупиняє" виконання і додає continuation як microtask.

---

## Web Workers та Service Workers

### Web Workers
Виконання JavaScript у окремому потоці (не блокує UI):
```js
// main.js
const worker = new Worker("worker.js");
worker.postMessage({ data: largeArray });
worker.onmessage = (e) => console.log(e.data);

// worker.js
self.onmessage = (e) => {
  const result = heavyComputation(e.data);
  self.postMessage(result);
};
```
**Обмеження:** немає доступу до DOM, window, document.

### Service Workers
Proxy між додатком і мережею (працює у фоні):
- Кешування ресурсів (offline support)
- Push notifications
- Background sync

```js
// Реєстрація
navigator.serviceWorker.register("/sw.js");

// sw.js
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => cached || fetch(event.request))
  );
});
```

---

## HTTP/2 vs HTTP/3

### HTTP/2
- **Multiplexing** — кілька запитів через одне TCP з'єднання
- **Header compression** (HPACK)
- **Server Push** — сервер може відправити ресурси до запиту
- **Binary protocol** (замість текстового HTTP/1.1)
- **Проблема:** Head-of-Line blocking на рівні TCP

### HTTP/3
- Побудований на **QUIC** (UDP замість TCP)
- Вирішує Head-of-Line blocking
- Швидше з'єднання (0-RTT)
- Вбудоване шифрування (TLS 1.3)
- Краще працює при нестабільному з'єднанні (мобільні мережі)

---

## WebSocket vs Server-Sent Events

### WebSocket
- **Двосторонній** зв'язок (клієнт ↔ сервер)
- Окремий протокол (ws://)
- Підтримує бінарні дані
- **Коли:** чати, ігри, collaborative editing

### Server-Sent Events (SSE)
- **Односторонній** (сервер → клієнт)
- Звичайний HTTP
- Автоматичне перепідключення
- Тільки текстові дані
- **Коли:** нотифікації, live feeds, прогрес операцій

```js
// SSE — простіше і достатньо для багатьох випадків
const source = new EventSource("/api/events");
source.onmessage = (event) => {
  console.log(event.data);
};
```

---

## Web Performance — Core Web Vitals

### LCP (Largest Contentful Paint) < 2.5с
Оптимізація:
- Preload головного зображення / шрифтів
- SSR/SSG для швидшого першого рендеру
- CDN для статичних ресурсів
- Оптимізувати зображення (WebP, AVIF, srcset)

### INP (Interaction to Next Paint) < 200мс
Оптимізація:
- Розбити довгі задачі (`requestIdleCallback`, `scheduler.yield()`)
- Web Workers для важких обчислень
- Debounce/throttle обробників подій

### CLS (Cumulative Layout Shift) < 0.1
Оптимізація:
- Задавати width/height для зображень та відео
- Не вставляти контент над існуючим
- Використовувати `font-display: swap` з fallback шрифтами
- Резервувати місце для динамічного контенту

---

## CORS — як працює?

**CORS** (Cross-Origin Resource Sharing) — механізм, який дозволяє серверу вказати, з яких origin-ів дозволені запити.

**Simple requests** (GET, POST з простими headers):
```
Browser → Server: GET /api/data
                  Origin: https://frontend.com

Server → Browser: Access-Control-Allow-Origin: https://frontend.com
```

**Preflight requests** (PUT, DELETE, custom headers):
```
Browser → Server: OPTIONS /api/data
                  Origin: https://frontend.com
                  Access-Control-Request-Method: DELETE

Server → Browser: Access-Control-Allow-Origin: https://frontend.com
                  Access-Control-Allow-Methods: DELETE
                  Access-Control-Max-Age: 86400

Browser → Server: DELETE /api/data  (actual request)
```

**Ключові headers:**
- `Access-Control-Allow-Origin` — дозволені origin-и
- `Access-Control-Allow-Methods` — дозволені HTTP методи
- `Access-Control-Allow-Headers` — дозволені headers
- `Access-Control-Allow-Credentials: true` — для cookies

---

## Cookies vs localStorage vs sessionStorage

| | Cookies | localStorage | sessionStorage |
|---|---|---|---|
| **Розмір** | ~4KB | ~5-10MB | ~5-10MB |
| **Термін** | Задається (expires/max-age) | Безстроково | До закриття табу |
| **Відправляється з запитом** | Так (автоматично) | Ні | Ні |
| **Доступ** | Сервер + клієнт | Тільки клієнт | Тільки клієнт |
| **Безпека** | HttpOnly, Secure, SameSite | Вразливий до XSS | Вразливий до XSS |

**Коли що:**
- **Cookies** — auth tokens (з HttpOnly + Secure), CSRF tokens
- **localStorage** — налаштування UI, кешовані дані, draft-и
- **sessionStorage** — одноразові дані (wizard state, form progress)

---

## Security (XSS, CSRF, CSP)

### XSS (Cross-Site Scripting)
Вставка зловмисного скрипта на сторінку:
- **Stored XSS** — скрипт збережений у БД
- **Reflected XSS** — скрипт у URL параметрах
- **DOM-based XSS** — скрипт маніпулює DOM

**Захист:** санітизація input, escaping output, Content Security Policy, `textContent` замість `innerHTML`.

### CSRF (Cross-Site Request Forgery)
Зловмисний сайт робить запит від імені авторизованого користувача.

**Захист:** CSRF tokens, SameSite cookies, перевірка Origin header.

### CSP (Content Security Policy)
```
Content-Security-Policy: default-src 'self'; script-src 'self' cdn.example.com; style-src 'self' 'unsafe-inline'
```
Обмежує джерела завантаження ресурсів — захист від XSS.

---

## CSS Specificity

Порядок пріоритету (від найвищого):
1. `!important` (уникати)
2. Inline styles (`style="..."`) — specificity: 1000
3. ID selectors (`#header`) — specificity: 100
4. Class, attribute, pseudo-class (`.btn`, `[type="text"]`, `:hover`) — specificity: 10
5. Element, pseudo-element (`div`, `::before`) — specificity: 1
6. Universal (`*`) — specificity: 0

```css
/* Specificity: 0-1-2-1 = 121 */
div.container .item:hover { color: red; }

/* Specificity: 0-1-0-0 = 100 */
#header { color: blue; }

/* #header переможе бо 100 < 121 — НІ, 100 > 21 (без ID)
   Порівняння йде по категоріям: ID > Class > Element */
```

---

## Flexbox vs Grid

### Flexbox — одновимірний layout (рядок АБО колонка)
```css
.container {
  display: flex;
  justify-content: space-between; /* головна вісь */
  align-items: center;           /* поперечна вісь */
  flex-wrap: wrap;
  gap: 16px;
}
.item {
  flex: 1; /* grow: 1, shrink: 1, basis: 0% */
}
```
**Коли:** навігація, toolbar, карточки в ряд, центрування.

### Grid — двовимірний layout (рядки І колонки)
```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: auto 1fr auto;
  gap: 16px;
}
.header { grid-column: 1 / -1; } /* на всю ширину */
```
**Коли:** загальний layout сторінки, складні сітки, dashboard.

**Правило:** Flexbox для компонентів, Grid для layout.

---

## Module Bundlers (Webpack, Vite, esbuild)

### Webpack
- Найстаріший і найпотужніший
- Величезна екосистема плагінів
- Складна конфігурація
- Code splitting, HMR, tree shaking
- **Коли:** великі enterprise проекти з особливими потребами

### Vite
- Використовує esbuild для dev (надшвидкий)
- Rollup для production build
- Мінімальна конфігурація
- Native ESM у development
- **Коли:** нові проекти (React, Vue, Svelte) — стандарт де-факто

### esbuild
- Написаний на Go — в 10-100x швидший за Webpack
- Мінімальний API
- Використовується іншими інструментами (Vite, tsup)
- **Коли:** бібліотеки, швидкий build без складної конфігурації

---

## Accessibility (a11y) основи

**Чому важливо:** юридичні вимоги (WCAG), більша аудиторія, кращий UX для всіх.

**Основні принципи:**
1. **Semantic HTML** — `<button>` замість `<div onClick>`, `<nav>`, `<main>`, `<article>`
2. **ARIA attributes** — `aria-label`, `aria-hidden`, `role` (коли semantic HTML недостатньо)
3. **Keyboard navigation** — все повинно працювати з клавіатури (Tab, Enter, Escape)
4. **Focus management** — видимий focus indicator, логічний tab order
5. **Alt text** для зображень
6. **Color contrast** — мінімум 4.5:1 для тексту
7. **Form labels** — кожен input повинен мати label
8. **Error messages** — зрозумілі, пов'язані з полем через `aria-describedby`
9. **Skip navigation** — посилання для пропуску навігації

```html
<button aria-label="Закрити модальне вікно" onClick={close}>
  <Icon name="x" aria-hidden="true" />
</button>
```
