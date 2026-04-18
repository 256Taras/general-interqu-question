# Accessibility (a11y) для Senior Full-Stack інженерів

## Що таке a11y і чому це важливо?

A11y — скорочення від "accessibility" (буква 'a', 11 літер, буква 'y'). Це практика розробки продуктів, доступних для всіх людей, незалежно від їхніх фізичних чи когнітивних можливостей.

Чому це критично:
- **1 мільярд людей** світу мають якусь форму інвалідності (ВОЗ)
- **Legal compliance**: WCAG 2.2, ADA (США), EN 301 549 (ЄС) — невиконання грозить судами
- **SEO**: доступні сайти краще ранжируються в Google
- **Business**: розширена аудиторія = більше конверсій
- **Ethical**: моральний обов'язок перед людьми

```html
<!-- Погано: без alt, без семантики -->
<div onclick="navigate('/about')">About Us</div>
<img src="/logo.png">

<!-- Добре: семантична розмітка, alt text -->
<button onclick="navigate('/about')">About Us</button>
<img src="/logo.png" alt="Company logo">
```

---

## WCAG 2.2: чотири принципи POUR

**WCAG** — Web Content Accessibility Guidelines. Стандарт від W3C з три рівні відповідності:

| Рівень | Вимога | Приклад |
|---|---|---|
| A | Базовий | Color contrast 4.5:1 для основного тексту |
| AA | Рекомендований | Color contrast 4.5:1 для всього, 3:1 для більших елементів |
| AAA | Оптимальний | Color contrast 7:1, розширені функції |

### POUR принципи:

**1. Perceivable (Сприймаємо)** — інформація повинна бути доступна для сприйняття
- Alt text для зображень
- Captions для відео
- Достатній color contrast

**2. Operable (Керовано)** — користувачи мають керувати через клавіатуру, не тільки мишею
- Keyboard navigation (Tab, Enter, Escape)
- No keyboard traps
- Достатньо часу для взаємодії

**3. Understandable (Зрозуміло)** — текст і логіка повинні бути простими
- Простої мова
- Передбачувана навігація
- Помилки чітко позначені

**4. Robust (Стійко)** — код повинен бути сумісним з технологіями доступу
- Valid HTML/ARIA
- Тестування з screen readers

```html
<!-- WCAG AA приклад -->
<section aria-labelledby="heading-products">
  <!-- Color contrast: 4.5:1 мінімум -->
  <h2 id="heading-products">Products</h2>
  
  <!-- Доступна форма -->
  <form>
    <label for="search-input">Search products:</label>
    <input id="search-input" type="search" required>
    <button type="submit">Search</button>
  </form>
</section>
```

---

## Semantic HTML: чому `<button>` краще за `<div onClick>`

Semantic HTML автоматично забезпечує доступність без додатків. Screen readers розуміють семантичні елементи.

```html
<!-- Погано: div не має a11y семантики -->
<div class="button" onclick="submitForm()">
  Submit
</div>

<!-- Добре: button має a11y вбудовану -->
<button type="submit">Submit</button>

<!-- Screen reader прочитає "button, Submit" автоматично -->
```

| Елемент | Роль | Keyboard |
|---|---|---|
| `<button>` | button | Enter, Space активує |
| `<a href>` | link | Enter активує |
| `<nav>` | navigation | Автоматично вказує на навігацію |
| `<main>` | main | Основний контент |
| `<article>` | article | Автономна стаття |
| `<form>` | form | Контейнер форми |
| `<h1>-<h6>` | heading | Структура заголовків |

```jsx
// React: semantic компоненти
export function Card({ title, children }) {
  return (
    <article className="card">
      <h2>{title}</h2>
      {children}
    </article>
  );
}

// Гірше: div-based (потрібно ARIA)
export function BadCard({ title, children }) {
  return (
    <div className="card" role="article">
      <div className="title" role="heading" aria-level="2">
        {title}
      </div>
      {children}
    </div>
  );
}
```

---

## ARIA Basics: roles, states, properties

ARIA — **Accessible Rich Internet Applications**. Додатковий HTML-атрибут для передачі інформації доступності, коли semantic HTML недостатньо.

**Перше правило ARIA**: НЕ ВИКОРИСТОВУЙ ARIA, якщо можна semantic HTML.

### ARIA атрибути:

**Roles**: дефінують що це за елемент
```html
<!-- Змінюємо роль div на button -->
<div role="button" onclick="deleteItem()" tabindex="0">
  Delete
</div>

<!-- ЛУЧ: просто використай <button> -->
<button type="button" onclick="deleteItem()">
  Delete
</button>
```

**States**: поточний стан елемента
```html
<!-- aria-checked: чи checked checkbox -->
<div role="checkbox" aria-checked="true">Accept terms</div>

<!-- aria-expanded: чи розгорнуто элемент -->
<button aria-expanded="false" aria-controls="menu">
  Menu
</button>

<!-- aria-disabled: чи заблоковано (крім form elements) -->
<div role="button" aria-disabled="true">Save</div>
```

**Properties**: додаткова інформація
```html
<!-- aria-label: назва елемента (якщо видимого label немає) -->
<button aria-label="Close dialog">×</button>

<!-- aria-required: поле обов'язкове -->
<input aria-required="true">

<!-- aria-valuemin/max/now: для слайдерів, прогресс-барів -->
<div role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="65">
  65% завантажено
</div>
```

```jsx
// React приклад: dropdown з ARIA
export function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className="dropdown">
      {/* Button з ARIA state */}
      <button
        aria-expanded={isOpen}
        aria-haspopup="menu"
        onClick={() => setIsOpen(!isOpen)}
      >
        Options
      </button>

      {/* Menu list з ARIA role */}
      {isOpen && (
        <ul role="menu" aria-label="Options menu">
          <li role="menuitem"><a href="/edit">Edit</a></li>
          <li role="menuitem"><a href="/delete">Delete</a></li>
        </ul>
      )}
    </div>
  );
}
```

---

## aria-label vs aria-labelledby vs aria-describedby

Три способи пов'язати текст з елементом:

| Атрибут | Використання | Приклад |
|---|---|---|
| `aria-label` | Пряма назва без видимого text | `<button aria-label="Close">×</button>` |
| `aria-labelledby` | ID іншого елемента, що служить label | `<button aria-labelledby="my-label">` + `<span id="my-label">` |
| `aria-describedby` | Додатковий опис (не замість label) | Input + hint text |

```html
<!-- Приклад 1: aria-label для icon button -->
<button aria-label="Close dialog" class="close-btn">
  <!-- видимо лише icon -->
  <svg>...</svg>
</button>

<!-- Приклад 2: aria-labelledby для heading -->
<h2 id="dialog-title">Confirm Delete</h2>
<div role="dialog" aria-labelledby="dialog-title">
  <!-- Screen reader прочитає "dialog, Confirm Delete" -->
  Are you sure?
</div>

<!-- Приклад 3: aria-describedby для hint текста -->
<label for="password">Password:</label>
<input
  id="password"
  type="password"
  aria-describedby="pwd-hint"
>
<small id="pwd-hint">
  Мінімум 8 символів, 1 число
</small>
```

```jsx
// React: три атрибути разом
export function FormField() {
  return (
    <div>
      <label id="email-label" htmlFor="email-input">
        Email Address
      </label>
      <input
        id="email-input"
        aria-labelledby="email-label"
        aria-describedby="email-hint"
        type="email"
      />
      <small id="email-hint">
        Ми не будемо дільтися вашим email
      </small>
    </div>
  );
}
```

---

## aria-live regions для динамічного контенту

`aria-live` повідомляє screen reader про оновлення контенту без перезавантаження сторінки.

| Значення | Поведінка |
|---|---|
| `polite` | Чекає, поки користувач закінчить, потім оголошує |
| `assertive` | Одразу переривує і оголошує |
| `off` | Не оголошує (за замовчуванням) |

```html
<!-- Polite: оновлення результатів пошуку -->
<div aria-live="polite" aria-label="search results">
  <p>Found 5 results</p>
</div>

<!-- Assertive: критична помилка -->
<div aria-live="assertive" role="alert">
  <strong>Error: Connection failed!</strong>
</div>

<!-- aria-atomic: оголосити весь блок чи лише змінену частину -->
<div aria-live="polite" aria-atomic="true">
  Cart: <strong>3 items</strong>
</div>
```

```jsx
// React приклад: live notifications
export function NotificationCenter() {
  const [notification, setNotification] = useState("");

  const handleDelete = async (id) => {
    await deleteItem(id);
    // Screen reader одразу оголосить
    setNotification("Item deleted successfully");
    setTimeout(() => setNotification(""), 3000);
  };

  return (
    <div
      aria-live="polite"
      aria-label="Notifications"
      role="status"
    >
      {notification}
    </div>
  );
}
```

---

## Keyboard navigation: tabindex, focus management, skip links

Користувачі, які не можуть користуватися мишею, покладаються на клавіатуру. Tab навігація обов'язкова.

```html
<!-- tabindex значення -->
<button tabindex="0">Always focusable (default)</button>
<button tabindex="1">Focus order (избегай позитивных значений!)</button>
<button tabindex="-1">Focusable only via JS, не через Tab</button>

<!-- Skip link: дозволяє пропустити навігацію -->
<a href="#main-content" class="skip-link">
  Skip to main content
</a>

<header>
  <nav><!-- довга навігація --></nav>
</header>

<main id="main-content">
  <!-- основний контент -->
</main>
```

```css
/* Приховати skip link, але зберегти для screen readers */
.skip-link {
  position: absolute;
  left: -9999px;
  z-index: 999;
}

.skip-link:focus {
  left: 0;
  top: 0;
  /* Стає видимою при фокусі */
}
```

```jsx
// React: focus management
import { useRef, useEffect } from 'react';

export function Modal({ isOpen, onClose }) {
  const contentRef = useRef(null);
  const previousFocusRef = useRef(null);

  useEffect(() => {
    if (isOpen) {
      // Зберегти попередній фокус
      previousFocusRef.current = document.activeElement;
      // Перемістити фокус на модальний контент
      contentRef.current?.focus();
    } else {
      // Повернути фокус
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return (
    <div role="dialog" aria-modal="true" ref={contentRef} tabIndex={-1}>
      <button onClick={onClose}>Close</button>
    </div>
  );
}
```

---

## Focus trap у модалках: правильна реалізація

Коли модальне вікно відкрито, фокус не повинен "втікати" до фонового контенту. Це веб-стандарт і best practice для a11y.

```jsx
// React Hook для focus trap
export function useFocusTrap(isActive) {
  const containerRef = useRef(null);

  useEffect(() => {
    if (!isActive || !containerRef.current) return;

    // Знайти все фокусне
    const focusableElements = containerRef.current.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    const firstElement = focusableElements[0];
    const lastElement = focusableElements[focusableElements.length - 1];

    const handleKeyDown = (e) => {
      if (e.key !== 'Tab') return;

      // Перехопити Tab в кінці, перемістити на початок
      if (e.shiftKey && document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      }
      // Перехопити Tab на початку, перемістити в кінець
      else if (!e.shiftKey && document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    };

    document.addEventListener('keydown', handleKeyDown);
    firstElement?.focus();

    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isActive]);

  return containerRef;
}

// Використання
export function Modal({ isOpen, onClose }) {
  const focusTrapRef = useFocusTrap(isOpen);

  if (!isOpen) return null;

  return (
    <div
      ref={focusTrapRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <h2 id="modal-title">Confirm Action</h2>
      <p>Are you sure?</p>
      <button onClick={onClose}>Cancel</button>
      <button onClick={onClose}>Confirm</button>
    </div>
  );
}
```

---

## Color contrast: вимоги WCAG, інструменти

Color contrast — співвідношення яскравості тексту до фону. WCAG вимагає:

- **AA**: 4.5:1 для звичайного тексту, 3:1 для великого тексту (18pt+, bold 14pt+)
- **AAA**: 7:1 для звичайного, 4.5:1 для великого

```html
<!-- Недостатньо (2:1) -->
<p style="color: #999; background: white;">Low contrast text</p>

<!-- Достатньо для AA (7:1) -->
<p style="color: #000; background: white;">Good contrast text</p>

<!-- Перевіряй на disabled buttons! -->
<button disabled style="color: #999;">Disabled button</button>
<!-- Користувачі з low vision не зможуть прочитати -->
```

**Інструменти перевірки:**
- **WebAIM Contrast Checker**: webaim.org/resources/contrastchecker/
- **Polypane**: polypane.app (CSS variable support)
- **Chrome DevTools**: Elements → Computed → color contrast info
- **axe DevTools**: browser extension, автоматична перевірка

```jsx
// React: динамічна перевірка contrast (дуже базово)
function getContrastRatio(hex1, hex2) {
  // Спрощена версія
  // Реально: використовуй WCAG формулу з L (luminance)
  // https://www.w3.org/TR/WCAG20/#relativeluminancedef
  return Math.random() * 10; // Заглушка :)
}

export function ColorPicker({ color, onColorChange }) {
  const contrast = getContrastRatio(color, '#ffffff');

  return (
    <div>
      <input
        type="color"
        value={color}
        onChange={(e) => onColorChange(e.target.value)}
      />
      <p role="status">
        Contrast ratio: {contrast.toFixed(2)}:1
        {contrast >= 4.5 ? '✓ Pass AA' : '✗ Fails AA'}
      </p>
    </div>
  );
}
```

---

## Форми: labels, error messages, required fields

Доступні форми мають явні labels, лаконічні помилки, й позначені обов'язкові поля.

```html
<!-- Погано: немає label -->
<input type="email" placeholder="Enter email">

<!-- Добре: label + for атрибут -->
<label for="email-input">Email Address:</label>
<input id="email-input" type="email" aria-required="true">

<!-- Обов'язкові поля позначити явно -->
<label for="name">Name <span aria-label="required">*</span></label>
<input id="name-input" type="text" required>

<!-- Помилки: aria-invalid + aria-describedby -->
<label for="password-input">Password:</label>
<input
  id="password-input"
  type="password"
  aria-invalid="true"
  aria-describedby="password-error"
>
<small id="password-error" role="alert">
  Password must be at least 8 characters
</small>
```

```jsx
// React приклад: доступна форма
export function LoginForm() {
  const [formData, setFormData] = useState({ email: '', password: '' });
  const [errors, setErrors] = useState({});

  const handleSubmit = (e) => {
    e.preventDefault();
    const newErrors = {};
    
    if (!formData.email) {
      newErrors.email = 'Email is required';
    }
    if (formData.password.length < 8) {
      newErrors.password = 'Password must be at least 8 characters';
    }

    setErrors(newErrors);
  };

  return (
    <form onSubmit={handleSubmit} aria-label="Login form">
      {/* Email field */}
      <div className="form-group">
        <label htmlFor="email">
          Email Address
          <span aria-label="required">*</span>
        </label>
        <input
          id="email"
          type="email"
          value={formData.email}
          onChange={(e) => setFormData({ ...formData, email: e.target.value })}
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : undefined}
        />
        {errors.email && (
          <small id="email-error" role="alert">
            {errors.email}
          </small>
        )}
      </div>

      {/* Password field */}
      <div className="form-group">
        <label htmlFor="password">
          Password
          <span aria-label="required">*</span>
        </label>
        <input
          id="password"
          type="password"
          value={formData.password}
          onChange={(e) => setFormData({ ...formData, password: e.target.value })}
          aria-invalid={!!errors.password}
          aria-describedby={errors.password ? 'password-error' : 'password-hint'}
        />
        {errors.password && (
          <small id="password-error" role="alert">
            {errors.password}
          </small>
        )}
        {!errors.password && (
          <small id="password-hint">
            Мінімум 8 символів, 1 число, 1 літера
          </small>
        )}
      </div>

      <button type="submit">Log In</button>
    </form>
  );
}
```

---

## Alt text для зображень: добрі і погані приклади

Alt text — замінюючий текст коли зображення не завантажується або для screen readers.

```html
<!-- Погано: порожній alt -->
<img src="/product.jpg" alt="">
<!-- Screen reader прочитає "image" або пропустить -->

<!-- Погано: надмірно детальний alt -->
<img src="/cat.jpg" alt="A fluffy orange tabby cat with green eyes sitting on a gray couch in a living room">
<!-- Занадто довгий, зашумлює -->

<!-- Добре: стислий опис -->
<img src="/cat.jpg" alt="Orange tabby cat on a gray couch">

<!-- Добре: для функціональних зображень -->
<button>
  <img src="/search-icon.png" alt="Search products">
</button>

<!-- Погано: для декоративних зображень -->
<img src="/decorative-line.png" alt="Decorative horizontal line">
<!-- Не додавай alt для pure-декоративних! -->

<!-- Добре: для декоративних -->
<img src="/decorative-line.png" alt="">
<!-- або просто не вклучай alt -->

<!-- Графіки, діаграми: опиши дані, не стиль -->
<img
  src="/sales-chart.png"
  alt="Q4 sales: Product A 45%, Product B 35%, Product C 20%"
>
```

```jsx
// React: приклади alt text
export function ProductCard({ product }) {
  return (
    <article>
      {/* Потрібен alt: показує контент */}
      <img
        src={product.image}
        alt={product.name}
      />
      
      {/* Потрібен alt: icon у button */}
      <button aria-label="Add to cart">
        <img src="/cart-icon.png" alt="">
      </button>
      
      {/* Можна пропустити alt: чисто декоративне */}
      <span className="decorative">→</span>
    </article>
  );
}

export function AvatarList() {
  return (
    <div className="avatars">
      {/* Групування: один alt для групи */}
      <img src="/user1.jpg" alt="User 1">
      <img src="/user2.jpg" alt="User 2">
      <img src="/user3.jpg" alt="User 3">
      
      {/* Або один alt для всієї групи */}
      <div role="group" aria-label="Team members">
        <img src="/user1.jpg" alt="">
        <img src="/user2.jpg" alt="">
      </div>
    </div>
  );
}
```

---

## Screen readers: NVDA, JAWS, VoiceOver — коротко про тестування

Screen readers — спеціальне ПО, яке читає вголос вміст сторінки. Три найпопулярніші:

| Screen Reader | ОС | Вартість | Тестування |
|---|---|---|---|
| **NVDA** | Windows | Безкоштовна | Найбільше для testing |
| **JAWS** | Windows | $90/рік | Professionalна, дорога |
| **VoiceOver** | Mac/iOS | Вбудована | Cmd+F5 для увімкнення |

**Як тестувати локально:**
1. **Mac**: System Preferences → Accessibility → VoiceOver → Enable (Cmd+F5)
2. **Windows**: Завантажити NVDA з nvaccess.org
3. **Chrome**: Lighthouse audit → Run audit (a11y пункт)

```bash
# Тестування з лін NVDA (Windows/Linux)
# Встановити NVDA, відкрити браузер, натиснути Ctrl+Alt+N

# VoiceOver на Mac
# Cmd+F5 для включення
# VO це Ctrl+Option за замовчуванням
# VO+Right Arrow: прочитати наступний елемент
# VO+Left Arrow: прочитати попередній елемент
```

**Screen reader очікує:**
- Semantic HTML (`<button>`, `<h1>`, `<nav>`)
- ARIA roles/labels коли потрібні
- Логічна TAB порядок
- Опис для зображень (alt)
- Form labels для inputs
- Meaningful link text ("Learn More" погано, "Learn more about our products" добре)

```jsx
// Як screen reader читає цей код
export function BadExample() {
  return (
    <div className="header">
      <div className="title">Welcome</div>
      {/* Screen reader: "div, Welcome" - не зрозуміло, що це заголовок */}
      
      <div onclick="navigate('/about')">Learn More</div>
      {/* Screen reader: "div, Learn More" - не зрозуміло, що це посилання */}
    </div>
  );
}

export function GoodExample() {
  return (
    <header>
      <h1>Welcome</h1>
      {/* Screen reader: "heading level 1, Welcome" */}
      
      <a href="/about">Learn more about our products</a>
      {/* Screen reader: "link, Learn more about our products" */}
    </header>
  );
}
```

---

## axe-core / Lighthouse a11y audit: автоматична перевірка

Автоматична перевірка не ловить 100% помилок, але ловить явні нарушення WCAG.

**axe-core**: JavaScript engine для тестування a11y, під капотом Lighthouse, DevTools.

```bash
# 1. Встановити axe DevTools розширення (Chrome/Firefox)
# 2. Відкрити DevTools → Preferences → Extensions → axe DevTools
# 3. Натиснути "Scan ALL of my page" для повної перевірки

# Сонцеть результати по категоріям:
# - Violations (критичні порушення)
# - Best Practices
# - Needs Review (ручна перевірка)
```

```bash
# Або через CLI з npm:
npm install -D @axe-core/cli

# Проскануй локальний сервер
axe http://localhost:3000
```

```jsx
// React: axe-core програмно (розробка)
import { axe } from 'jest-axe';

describe('Button accessibility', () => {
  it('should have no a11y violations', async () => {
    const { container } = render(
      <button>Submit Form</button>
    );

    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});
```

**Lighthouse audit:**
1. Відкрити Chrome DevTools → Lighthouse tab
2. Select "Accessibility" checkbox
3. Натиснути "Analyze page load" або "Generate report"
4. Див результат: Passed, Failed, Manual checks

---

## A11y Checklist

Перед puush до production:

- [ ] **Semantic HTML**: `<button>` замість `<div>`, `<nav>` для навігації, `<main>` для основного контенту
- [ ] **Form labels**: кожен `<input>` має `<label>` з `for` атрибутом
- [ ] **Alt text**: всі зображення мають alt (можна пустий для декоративних)
- [ ] **Color contrast**: >= 4.5:1 (AA) для звичайного тексту
- [ ] **Keyboard navigation**: всі інтерактивні елементи доступні через Tab, fokус видимий
- [ ] **ARIA**: використовуй тільки коли semantic HTML недостатньо
- [ ] **aria-label**: для icon buttons, close buttons без видимого тексту
- [ ] **aria-live**: для динамічного контенту (notifications, alerts)
- [ ] **Focus trap**: модальні вікна утримують фокус
- [ ] **Error messages**: aria-invalid + aria-describedby
- [ ] **Skip links**: дозволяють пропустити навігацію
- [ ] **axe-core audit**: 0 violations, пройдено в CI/CD
- [ ] **Screen reader testing**: протестовано з NVDA/VoiceOver/JAWS
- [ ] **Mobile a11y**: VoiceOver на iOS, TalkBack на Android

---

## Типові пастки

**1. Div замість button**
```jsx
// Гаразд? Ні! Клавіатура не працює, немає role
<div onclick="handleClick()">Click me</div>

// Добре
<button onclick="handleClick()">Click me</button>
```

**2. Placeholder замість label**
```jsx
// Гаразд? Ні! Placeholder зникає при вводі
<input placeholder="Email">

// Добре
<label htmlFor="email">Email</label>
<input id="email" type="email">
```

**3. tabindex="1"**
```jsx
// Гаразд? Ні! Нарушає TAB порядок
<button tabindex="1">First</button>
<button tabindex="2">Second</button>

// Добре: default tabindex="0" або без атрибута
<button>First</button>
<button>Second</button>
```

**4. Color тільки для інформації**
```jsx
// Гаразд? Ні! Колірні сліпі не зрозуміють
<p style={{ color: 'red' }}>Error occurred</p>

// Добре: додай текст/icon
<p style={{ color: 'red' }}>
  ✕ Error occurred
</p>
```

**5. ARIA замість semantic**
```jsx
// Гаразд? Ні! Складно, потрібно вручну ARIA
<div role="button" tabindex="0" onclick="...">
  Delete
</div>

// Добре
<button>Delete</button>
```

**6. aria-hidden на видимому контенті**
```jsx
// Гаразд? Ні! Screen reader не прочитає
<button>
  Submit
  <span aria-hidden="true"> →</span>
  {/* arrow не буде озвучено */}
</button>
```

**7. На тестування тільки axe-core**
```jsx
// axe ловить 30% a11y проблем.
// Завжди тестуй і вручну з screen reader!
// Автоматизація != повна a11y
```

---

## Ресурси

- **WCAG 2.2**: https://www.w3.org/WAI/WCAG22/quickref/
- **MDN a11y**: https://developer.mozilla.org/en-US/docs/Web/Accessibility
- **axe DevTools**: https://www.deque.com/axe/devtools/
- **WebAIM**: https://webaim.org/ (контрастність, screen reader гайди)
- **A11y Matters**: https://a11y-matters.com/ (практичні приклади)
- **jest-axe**: https://github.com/nickcolley/jest-axe (тести)
