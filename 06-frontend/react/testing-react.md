# Тестування React

## Яка піраміда тестування для React-додатку?

Піраміда тестування показує розподіл типів тестів за кількістю й глибиною. На底 багато швидких unit-тестів, у середині — інтеграційні, на вершині — кілька E2E-тестів.

```
     E2E (Playwright, Cypress)
         ↑   ~ 5-10 тестів
        ↑↑ 
       ↑ ↑  Integration (~30% тестів)
      ↑   ↑
     ↑─────↑  Unit (~60% тестів)
```

**Unit-тестування** (60%): окремі компоненти, утиліти, хуки без залежностей
```jsx
// Button.test.jsx
import { render, screen } from '@testing-library/react';
import Button from './Button';

describe('Button', () => {
  it('має рендеритися з текстом', () => {
    render(<Button>Натисніть</Button>);
    expect(screen.getByRole('button')).toHaveTextContent('Натисніть');
  });
});
```

**Integration тестування** (30%): компоненти з дітьми, форми, простий API мокування
```jsx
// LoginForm.test.jsx — тестуємо взаємодію 3-4 компонентів
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import LoginForm from './LoginForm';

it('надсилає форму з валідними даними', async () => {
  render(<LoginForm onSubmit={mockSubmit} />);
  
  await userEvent.type(screen.getByLabelText(/імейл/i), 'user@example.com');
  await userEvent.type(screen.getByLabelText(/пароль/i), 'pass123');
  await userEvent.click(screen.getByRole('button', { name: /увійти/i }));
  
  expect(mockSubmit).toHaveBeenCalledWith({ email: 'user@example.com', password: 'pass123' });
});
```

**E2E тестування** (5-10%): повні користувацькі сценарії наскрізь приложення, реальний браузер
```javascript
// auth.spec.js (Playwright)
test('користувач може зареєструватися й увійти', async ({ page }) => {
  await page.goto('http://localhost:3000/signup');
  await page.fill('input[name="email"]', 'new@example.com');
  await page.fill('input[name="password"]', 'SecurePass123');
  await page.click('button:has-text("Зареєстрватися")');
  
  await expect(page).toHaveURL('http://localhost:3000/dashboard');
});
```

---

## Що таке React Testing Library й яка її філософія?

React Testing Library (RTL) — це бібліотека для тестування React-компонентів, яка фокусується на **тестуванні поведінки, а не реалізації**. Замість перевірки внутрішнього стану чи пропсів, ми тестуємо те, що користувач **бачить й робить**.

**Філософія RTL:**
- Тестуй те, як користувач взаємодіє з компонентом
- Уникай деталей реалізації (внутрішній стан, імплементація хуків)
- Пошук елементів за доступністю (role, label, text)
- Покращується рефакторинг — код може змінюватися, тест залишається дійсним

```jsx
// ❌ ПОГАНО: тестуємо реалізацію (внутрішній стан)
import { render } from '@testing-library/react';
import Counter from './Counter';

it('INCREMENT_BUTTON повинен збільшити count на 1', () => {
  const { container } = render(<Counter />);
  // Залежимо від конкретної структури компонента
  expect(container.querySelector('[data-testid="counter-state"]')).toHaveTextContent('0');
});

// ✅ ДОБРЕ: тестуємо поведінку
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Counter from './Counter';

it('кліцяння на кнопку збільшує счетчик', async () => {
  render(<Counter />);
  const incrementBtn = screen.getByRole('button', { name: /збільшити/i });
  
  await userEvent.click(incrementBtn);
  expect(screen.getByText('1')).toBeInTheDocument();
});
```

---

## Які методи пошуку елементів в React Testing Library й коли їх використовувати?

RTL пропонує три основні категорії методів: `getBy*`, `queryBy*`, `findBy*`. Вибір залежить від того, **коли** та **як часто** повинен існувати елемент.

| Метод | Коли помилка | Повертає | Коли використовувати |
|---|---|---|---|
| `getBy*` | Кидає помилку | Елемент | Елемент має存在 одразу |
| `queryBy*` | Повертає `null` | Елемент або `null` | Елемент НЕ повинен існувати |
| `findBy*` | Кидає помилку | Promise | Елемент з'явиться асинхронно |

```jsx
// getBy* — елемент існує одразу (синхронно)
it('показує надпис "Вітаємо"', () => {
  render(<Greeting name="Тарас" />);
  // Кине помилку, якщо елемент не знайдено
  expect(screen.getByText('Вітаємо, Тарас!')).toBeInTheDocument();
});

// queryBy* — перевіряємо відсутність елемента
it('спочатку не показує помилку', () => {
  render(<LoginForm />);
  // Повертає null, не кидає помилку
  expect(screen.queryByText(/помилка автентифікації/i)).not.toBeInTheDocument();
});

// findBy* — елемент з'явиться пізніше (асинхронно)
it('завантажує й показує користувачів', async () => {
  render(<UserList />);
  // Чекаємо на асинхронне завантаження
  const userItem = await screen.findByText('Іван Петренко');
  expect(userItem).toBeInTheDocument();
});
```

**Пріоритет пошуку (за доступністю):**
1. `getByRole()` — найкраще (користувачі переганяються очами на основі ролей)
2. `getByLabelText()` — для form inputs
3. `getByPlaceholderText()` — для форм без label
4. `getByText()` — для контенту
5. `getByTestId()` — останнє середство (не тестуємо доступність)

```jsx
// ДОБРИЙ ПОРЯДОК пошуку
it('показує й обробляє форму', async () => {
  render(<ContactForm />);
  
  // 1️⃣ Role — найбільш доступно
  const emailInput = screen.getByRole('textbox', { name: /імейл/i });
  
  // 2️⃣ Label — для форм
  const nameInput = screen.getByLabelText(/ім'я/i);
  
  // 3️⃣ Text — для видимого контенту
  const submitBtn = screen.getByRole('button', { name: /надіслати/i });
  
  // 4️⃣ TestId — тільки якщо немаєкраще варіанту
  const formStatus = screen.getByTestId('form-status');
});
```

---

## Чим userEvent відрізняється від fireEvent й чому userEvent краще?

`fireEvent` симулює браузер-события на низькому рівні (прямо тригерить event listeners), а `userEvent` **симулює реальні дії користувача** (вводить текст символ за символом, триголить change, click, blur).

```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { fireEvent } from '@testing-library/react';

// Компонент для тестування
function SearchInput({ onSearch }) {
  const [value, setValue] = React.useState('');
  
  return (
    <>
      <input 
        value={value}
        onChange={(e) => setValue(e.target.value)}
        onBlur={() => onSearch(value)}
      />
    </>
  );
}

// ❌ fireEvent — НЕ виконує blur, неправдоподібно
it('повинна викликати onSearch (ПОГАНО)', () => {
  const onSearch = jest.fn();
  render(<SearchInput onSearch={onSearch} />);
  
  const input = screen.getByRole('textbox');
  fireEvent.change(input, { target: { value: 'react' } });
  // onSearch не викликалася, тому що не було blur!
  expect(onSearch).not.toHaveBeenCalled();
});

// ✅ userEvent — імітує реальні дії (введення, blur)
it('повинна викликати onSearch при blur (ДОБРЕ)', async () => {
  const onSearch = jest.fn();
  render(<SearchInput onSearch={onSearch} />);
  
  const input = screen.getByRole('textbox');
  await userEvent.type(input, 'react'); // введення символ за символом
  await userEvent.tab(); // або userEvent.click(input); — blur
  
  // Тепер onSearch була викликана!
  expect(onSearch).toHaveBeenCalledWith('react');
});
```

**Коли користувати:**
- **userEvent** — майже ЗАВЖДИ (це поведінка користувача)
- **fireEvent** — рідко, для низькорівневого тестування зміни подій

---

## Як тестувати React hooks?

Для тестування хуків існує утиліта `renderHook` з `@testing-library/react`. Хуки не можна викликати поза React-компонентом, тому `renderHook` обгортає хук у тестовий компонент.

```jsx
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('повинна повертати 0 на старті', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  // ❌ ПОГАНО: забуває про act()
  it('збільшує счетчик при натисканні (НЕПРАВИЛЬНО)', () => {
    const { result } = renderHook(() => useCounter());
    result.current.increment(); // Warning: setState без act()!
    expect(result.current.count).toBe(1);
  });

  // ✅ ДОБРЕ: обгортаємо в act()
  it('збільшує счетчик при натисканні (ПРАВИЛЬНО)', () => {
    const { result } = renderHook(() => useCounter());
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  // Тестування хука з залежностями
  it('сумує числа при кожній зміні', () => {
    const { result, rerender } = renderHook(
      ({ num1, num2 }) => useSum(num1, num2),
      { initialProps: { num1: 2, num2: 3 } }
    );
    
    expect(result.current).toBe(5);
    
    // rerender змінює props
    rerender({ num1: 10, num2: 20 });
    expect(result.current).toBe(30);
  });
});
```

**useEffect у тестах:**
```jsx
import { renderHook, waitFor } from '@testing-library/react';
import { useFetchUser } from './useFetchUser';

it('завантажує користувача при mount', async () => {
  const { result } = renderHook(() => useFetchUser('123'));
  
  // Спочатку дані не завантажені
  expect(result.current.loading).toBe(true);
  
  // Чекаємо на асинхронне завантаження
  await waitFor(() => {
    expect(result.current.loading).toBe(false);
  });
  
  expect(result.current.user).toEqual({ id: '123', name: 'Іван' });
});
```

---

## Коли мокувати й як мокувати модулі в тестах (jest.mock, vi.mock)?

Мокування використовується для **ізольованого тестування** — замінюємо залежності на підробки. Мокуй:
- Зовнішні API та вебсервіси
- Складні утиліти та бібліотеки
- Модулі, які складно налаштувати в тесті

**НЕ мокуй:**
- Особливо малі утиліти без побічних ефектів
- Код, який ти тестуєш (це приховає регресії)

```javascript
// api.js
export async function fetchUser(id) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

// component.js
import { fetchUser } from './api';

export function UserProfile({ id }) {
  const [user, setUser] = React.useState(null);
  
  React.useEffect(() => {
    fetchUser(id).then(setUser);
  }, [id]);
  
  return user ? <div>{user.name}</div> : <div>Завантаження...</div>;
}

// ✅ JEST: мокування на рівні модуля
// UserProfile.test.jsx
import { render, screen } from '@testing-library/react';
import { fetchUser } from './api';
import UserProfile from './UserProfile';

jest.mock('./api'); // замінюємо весь модуль api

it('показує ім\'я користувача при завантаженні', async () => {
  fetchUser.mockResolvedValue({ id: '1', name: 'Марія' });
  
  render(<UserProfile id="1" />);
  
  const userName = await screen.findByText('Марія');
  expect(userName).toBeInTheDocument();
  expect(fetchUser).toHaveBeenCalledWith('1');
});
```

```javascript
// ✅ VITEST: такий же синтаксис (vi замість jest)
import { vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import { fetchUser } from './api';
import UserProfile from './UserProfile';

vi.mock('./api');

it('показує ім\'я користувача', async () => {
  vi.mocked(fetchUser).mockResolvedValue({ id: '1', name: 'Марія' });
  
  render(<UserProfile id="1" />);
  expect(await screen.findByText('Марія')).toBeInTheDocument();
});
```

**Мокування конкретної функції в модулі:**
```javascript
// ❌ НЕПРАВИЛЬНО: мокуємо і компонент, якийтестуємо
jest.mock('./calculations', () => ({
  sum: jest.fn(() => 10)
}));

// ✅ ПРАВИЛЬНО: мокуємо залежність, не код якийтестуємо
jest.mock('./userService');
jest.mock('axios'); // зовнішня бібліотека
```

---

## Як мокувати API запити з MSW (Mock Service Worker)?

MSW (Mock Service Worker) — це бібліотека для перехоплення мережевих запитів на рівні worker. Це кращий підхід, ніж мокування `fetch` чи `axios`, тому що:
- Код не знає про мокування
- Можна використати той самий MSW у браузерних тестах й E2E-тестах
- Легко налаштовувати поточні запити

```javascript
// mocks/handlers.js
import { http, HttpResponse } from 'msw';

export const handlers = [
  // GET /api/users/1
  http.get('/api/users/:id', ({ params }) => {
    const { id } = params;
    return HttpResponse.json(
      { id, name: 'Іван Петренко', email: 'ivan@example.com' },
      { status: 200 }
    );
  }),

  // POST /api/users
  http.post('/api/users', async ({ request }) => {
    const data = await request.json();
    return HttpResponse.json(
      { id: '999', ...data },
      { status: 201 }
    );
  }),

  // Помилка 404
  http.get('/api/nonexistent', () => {
    return HttpResponse.json(
      { error: 'Not found' },
      { status: 404 }
    );
  }),
];

// mocks/server.js (для Node.js тестів)
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// setup.js (глобальна налаштування для тестів)
import { server } from './mocks/server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// ✅ Тест з MSW
// UserProfile.test.jsx
import { render, screen } from '@testing-library/react';
import { server } from './mocks/server';
import { http, HttpResponse } from 'msw';
import UserProfile from './UserProfile';

it('завантажує й показує користувача', async () => {
  render(<UserProfile id="1" />);
  
  const userName = await screen.findByText('Іван Петренко');
  expect(userName).toBeInTheDocument();
});

it('обробляє помилку 404', async () => {
  // Перевизначаємо обробник тільки для цього тесту
  server.use(
    http.get('/api/users/:id', () => {
      return HttpResponse.json(
        { error: 'User not found' },
        { status: 404 }
      );
    })
  );
  
  render(<UserProfile id="999" />);
  
  const errorMsg = await screen.findByText(/користувача не знайдено/i);
  expect(errorMsg).toBeInTheDocument();
});
```

---

## Як тестувати асинхронну поведінку (waitFor, findBy)?

Асинхронні операції (запити, таймери, state updates) потребують чекання. RTL пропонує `waitFor` та `findBy*` для цього.

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import SearchResults from './SearchResults';

// Компонент з затримкою в пошуку
function SearchResults() {
  const [results, setResults] = React.useState([]);
  const [loading, setLoading] = React.useState(false);

  const handleSearch = async (query) => {
    setLoading(true);
    await new Promise(r => setTimeout(r, 300)); // затримка API
    setResults(['результат 1', 'результат 2']);
    setLoading(false);
  };

  return (
    <>
      <button onClick={() => handleSearch('test')}>Пошук</button>
      {loading && <div>Завантаження...</div>}
      <ul>
        {results.map((r, i) => <li key={i}>{r}</li>)}
      </ul>
    </>
  );
}

// ✅ findBy* — для елементів, які з'являться асинхронно
it('показує результати пошуку', async () => {
  render(<SearchResults />);
  
  // Натискаємо на кнопку й чекаємо на результати
  await userEvent.click(screen.getByRole('button'));
  
  const result = await screen.findByText('результат 1');
  expect(result).toBeInTheDocument();
});

// ✅ waitFor — для умов, які встановлюються асинхронно
it('приховує завантаження після отримання результатів', async () => {
  render(<SearchResults />);
  
  await userEvent.click(screen.getByRole('button'));
  
  // Чекаємо, коли завантаження зникне
  await waitFor(() => {
    expect(screen.queryByText('Завантаження...')).not.toBeInTheDocument();
  });
});

// ❌ НЕПРАВИЛЬНО: забуває await
it('тест буде нестабільним (ПОГАНО)', () => {
  render(<SearchResults />);
  screen.getByRole('button').click();
  // Не чекаємо, тест закінчиться ДО того, як результати завантажуються
  expect(screen.getByText('результат 1')).toBeInTheDocument(); // Помилка!
});
```

---

## Як тестувати Context й Redux?

Компоненти, які залежать від провайдерів (Context, Redux), потребують обгортання у тестах.

```jsx
// AuthContext.jsx
const AuthContext = React.createContext();

export function AuthProvider({ children }) {
  const [user, setUser] = React.useState(null);
  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  );
}

// UserCard.jsx — залежить від AuthContext
export function UserCard() {
  const { user } = React.useContext(AuthContext);
  return user ? <div>Привіт, {user.name}</div> : <div>Не авторизований</div>;
}

// ✅ Обгортаємо компонент у провайдер
// UserCard.test.jsx
import { render, screen } from '@testing-library/react';
import { AuthProvider } from './AuthContext';
import UserCard from './UserCard';

// Функція-помічник для налаштування тесту
function renderWithAuth(component, { user = null } = {}) {
  return render(
    <AuthProvider initialUser={user}>
      {component}
    </AuthProvider>
  );
}

it('показує ім\'я авторизованого користувача', () => {
  renderWithAuth(<UserCard />, { user: { id: '1', name: 'Марія' } });
  expect(screen.getByText(/привіт, марія/i)).toBeInTheDocument();
});

it('показує "не авторизований" без користувача', () => {
  renderWithAuth(<UserCard />);
  expect(screen.getByText(/не авторизований/i)).toBeInTheDocument();
});
```

**Redux з React-Redux:**
```jsx
// store.js
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './authSlice';

export const store = configureStore({
  reducer: { auth: authReducer }
});

// component.test.jsx
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './authSlice';
import UserProfile from './UserProfile';

function renderWithRedux(component, { initialState = {}, store: customStore } = {}) {
  const store = customStore || configureStore({
    reducer: { auth: authReducer },
    preloadedState: initialState
  });
  
  return render(
    <Provider store={store}>
      {component}
    </Provider>
  );
}

it('показує користувача з Redux store', () => {
  const initialState = {
    auth: { user: { id: '1', name: 'Іван' } }
  };
  
  renderWithRedux(<UserProfile />, { initialState });
  expect(screen.getByText(/іван/)).toBeInTheDocument();
});
```

---

## Коли дорошно використовувати snapshot тестування й коли це антипатерн?

**Snapshot** — це збережений вивід компонента. При змінах RTL порівнює новий вивід зі старим. Це корисно рідко.

✅ **Добрі випадки:**
- UI каталоги (крупні статичні компоненти без логіки)
- Код генератори

❌ **Антипатерни:**
- Для компонентів з логікою й динамічним контентом (тести часто "update snapshot" без аналізу)
- Замість конкретних assertions

```jsx
// Button.test.jsx
import { render } from '@testing-library/react';
import Button from './Button';

// ❌ ПОГАНО: snapshot для кнопки зі змінюваними пропсами
it('має рендеритися як snapshot', () => {
  const { container } = render(<Button variant="primary">Натисніть</Button>);
  expect(container).toMatchSnapshot();
  // Якщо змініш CSS, тест обрує "update snapshot" — це не добре
});

// ✅ ДОБРЕ: snapshot для статичного каталогу
it('рендерить усі варіанти кнопок', () => {
  const variants = ['primary', 'secondary', 'danger'];
  const { container } = render(
    <div>
      {variants.map(v => <Button key={v} variant={v}>{v}</Button>)}
    </div>
  );
  expect(container).toMatchSnapshot();
});

// ✅ ДОБРЕ: конкретні assertions замість snapshot
it('рендерить з правильними класами', () => {
  const { container } = render(<Button variant="primary">Натисніть</Button>);
  const btn = container.querySelector('button');
  expect(btn).toHaveClass('btn-primary');
  expect(btn).toHaveTextContent('Натисніть');
});
```

---

## Коротко про E2E тестування: Playwright vs Cypress

| Характеристика | Playwright | Cypress |
|---|---|---|
| Мова | JavaScript / Python / Java / C# | JavaScript |
| Браузери | Chrome, Firefox, Safari, Edge, WebKit | Chrome, Firefox, Edge |
| Налаштування | Простіше, немає залежностей | Складніше (особливо в CI) |
| Швидкість | Дуже швидко | Повільніше |
| DevTools | Хороші (Trace viewer) | Хороші (Time-travel) |
| API | Низькорівневий, гнучкий | Вищорівневий, більше магії |
| Мобільні | Підтримує | Не підтримує |
| Коли | Коли потрібна швидкість, мобіль | Коли потрібна простота |

```javascript
// Playwright
import { test, expect } from '@playwright/test';

test('користувач реєструється', async ({ page }) => {
  await page.goto('http://localhost:3000/signup');
  await page.fill('input[name="email"]', 'user@test.com');
  await page.fill('input[name="password"]', 'Pass123!');
  await page.click('button[type="submit"]');
  
  await expect(page).toHaveURL('**/dashboard');
});

// Cypress
describe('Sign up', () => {
  it('реєструє користувача', () => {
    cy.visit('http://localhost:3000/signup');
    cy.get('input[name="email"]').type('user@test.com');
    cy.get('input[name="password"]').type('Pass123!');
    cy.get('button[type="submit"]').click();
    
    cy.url().should('include', '/dashboard');
  });
});
```

---

## Як тестувати доступність (a11y) з axe-core й jest-axe?

Автоматизовані A11y тести виловлюють порушення доступності (контрастність, aria-labels, semantic HTML).

```jsx
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';
import Button from './Button';

// Розширюємо expect для axe матчерів
expect.extend(toHaveNoViolations);

it('не має порушень доступності', async () => {
  const { container } = render(
    <Button>
      Натисніть мене
    </Button>
  );
  
  // axe проходить контейнер і виявляє проблеми
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

it('компонент має правильну ARIA роль', async () => {
  const { container } = render(
    <div role="navigation">
      <a href="/">Домашня</a>
      <a href="/about">Про</a>
    </div>
  );
  
  const results = await axe(container);
  // Не повинно бути помилок з ARIA
  expect(results).toHaveNoViolations();
});

// ❌ ЗНАЙДЕ ПРОБЛЕМУ: недостатній контраст, відсутність label
it('виявляє проблеми доступності', async () => {
  const { container } = render(
    <input type="text" /> {/* ❌ немає label */}
  );
  
  const results = await axe(container);
  // Результат буде містити порушення з цієї помилки
  expect(results.violations.length).toBeGreaterThan(0);
});
```

---

## Як міряти код coverage й що таке "достатньо"?

Coverage показує, який відсоток коду виконується тестами.

```bash
# Запустити тести з report'ом coverage
npm test -- --coverage

# Результат у console:
# File         | % Statements | % Branches | % Funcs | % Lines |
# Button.jsx   |      100     |     95     |   100   |   100   |
# utils.js     |       85     |     80     |    90   |    85   |
# index.js     |       60     |     50     |    70   |    60   |
```

**Що таке "достатньо":**
- **Критичний код** (auth, платежі): 90%+
- **Core business logic**: 80%+
- **UI компоненти**: 70-80%
- **Utils, config**: 60-70%

❌ **ПОГАНО:** Гонитися за 100% coverage
```javascript
// У тебе 100% покриття, але тест не тестує поведінку
function add(a, b) {
  return a + b;
}

it('має 100% coverage', () => {
  add(1, 2); // Лінія виконана, поведінка не перевірена!
});
```

✅ **ДОБРЕ:** Цільове покриття критичного коду
```javascript
function calculatePrice(items, taxRate) {
  return items.reduce((sum, item) => sum + item.price, 0) * (1 + taxRate);
}

it('розраховує ціну з податком', () => {
  const total = calculatePrice(
    [{ price: 100 }, { price: 50 }],
    0.1
  );
  expect(total).toBe(165); // 150 * 1.1
});

it('обробляє пустий масив', () => {
  expect(calculatePrice([], 0.1)).toBe(0);
});
```

---

## Типові пастки при тестуванні React

1. **Забуває `await` при асинхронних операціях**
```javascript
// ❌ ПОГАНО
it('показує дані', () => {
  render(<UserList />);
  expect(screen.getByText('Іван')).toBeInTheDocument(); // Раніше, ніж дані завантажуються!
});

// ✅ ДОБРЕ
it('показує дані', async () => {
  render(<UserList />);
  expect(await screen.findByText('Іван')).toBeInTheDocument();
});
```

2. **Тестує деталі реалізації, а не поведінку**
```javascript
// ❌ ПОГАНО
expect(component.state.isOpen).toBe(true);
expect(component.props.onClick).toHaveBeenCalled();

// ✅ ДОБРЕ
expect(screen.getByText('Меню')).toBeVisible();
```

3. **Мокує код, який тестує**
```javascript
// ❌ ПОГАНО
jest.mock('./Button');
render(<Button />); // Тестуєш мок, а не компонент!

// ✅ ДОБРЕ
jest.mock('./api'); // Мокуй залежності, не компонент
render(<Button />);
```

4. **Не використовує `userEvent`, використовує `fireEvent`**
```javascript
// ❌ НЕПРАВДОПОДІБНО
fireEvent.click(button);
fireEvent.change(input, { target: { value: 'text' } });

// ✅ РЕАЛІСТИЧНО
await userEvent.click(button);
await userEvent.type(input, 'text');
```

5. **Забуває обгортати state updates в `act()`**
```javascript
// ❌ ПОПЕРЕДЖЕННЯ: setState не в act()
fireEvent.click(screen.getByRole('button'));

// ✅ ПРАВИЛЬНО
act(() => {
  fireEvent.click(screen.getByRole('button'));
});
// Або краще — використовуй userEvent
```

6. **Покладається на дані, що залежать від часу**
```javascript
// ❌ ПОГАНО: дата змінюється, тест нестабільний
expect(screen.getByText(/2026-04-17/)).toBeInTheDocument();

// ✅ ДОБРЕ: мокуй дату
jest.useFakeTimers();
jest.setSystemTime(new Date('2026-04-17'));
expect(screen.getByText(/2026-04-17/)).toBeInTheDocument();
jest.useRealTimers();
```

7. **Не чистить мокування між тестами**
```javascript
// ❌ ПОГАНО
jest.mock('axios');
it('test 1', () => { axios.get.mockResolvedValue(...) });
it('test 2', () => { /* axios всі ще змокований з test 1 */ });

// ✅ ДОБРЕ
afterEach(() => jest.clearAllMocks());
```
