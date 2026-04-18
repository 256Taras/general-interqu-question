# React — Питання для інтерв'ю

## Що таке Virtual DOM і як він працює?

**Virtual DOM** — це легковагий JavaScript об'єкт, який є копією реального DOM. React використовує його для оптимізації оновлень інтерфейсу.

**Як працює:**
1. При зміні стану React створює новий Virtual DOM
2. Порівнює новий Virtual DOM зі старим (процес називається **Reconciliation**)
3. Обчислює мінімальний набір змін (**diffing algorithm**)
4. Застосовує лише необхідні зміни до реального DOM (**batch update**)

**Чому це швидко:**
- Маніпуляції з JavaScript об'єктами набагато швидші, ніж з реальним DOM
- Batch updates — React групує кілька змін в одне оновлення
- Мінімальна кількість операцій з реальним DOM

**Важливий нюанс:** Virtual DOM сам по собі не робить React швидшим за ручні DOM маніпуляції. Він робить **декларативний** підхід достатньо швидким, щоб не думати про оптимізацію вручну.

---

## Lifecycle methods vs Hooks

**Class components (lifecycle methods):**
```jsx
class MyComponent extends React.Component {
  componentDidMount() { /* після монтування */ }
  componentDidUpdate(prevProps) { /* після оновлення */ }
  componentWillUnmount() { /* перед розмонтуванням */ }
  shouldComponentUpdate(nextProps) { /* оптимізація */ }
}
```

**Function components (Hooks):**
```jsx
function MyComponent() {
  useEffect(() => {
    // componentDidMount + componentDidUpdate
    return () => { /* componentWillUnmount */ };
  }, [deps]);
}
```

**Ключові відмінності:**
- Hooks дозволяють розділити логіку за **функціональністю**, а не за lifecycle фазами
- В одному компоненті можна мати кілька `useEffect` для різної логіки
- Hooks простіше перевикористовувати (custom hooks)
- Class components все ще підтримуються, але нові фічі React доступні лише для hooks

---

## useState, useEffect, useCallback, useMemo, useRef — пояснити кожен

### useState
Зберігає локальний стан компонента:
```jsx
const [count, setCount] = useState(0);
// count — поточне значення
// setCount — функція для оновлення
// 0 — початкове значення
```
**Важливо:** `setState` може приймати функцію для оновлення на основі попереднього стану:
```jsx
setCount(prev => prev + 1); // безпечно при кількох оновленнях
```

### useEffect
Виконує побічні ефекти (API виклики, підписки, таймери):
```jsx
useEffect(() => {
  const subscription = subscribe(id);
  return () => subscription.unsubscribe(); // cleanup
}, [id]); // перезапускається тільки при зміні id
```
- `[]` — виконається один раз (mount)
- `[dep]` — при зміні dep
- без масиву — після кожного рендеру (майже ніколи не потрібно)

### useCallback
Мемоізує **функцію** (повертає ту саму посилку, поки deps не змінились):
```jsx
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
```
**Коли потрібен:** коли передаєш callback у дочірній компонент, обгорнутий у `React.memo`.

### useMemo
Мемоізує **результат обчислення:**
```jsx
const sortedList = useMemo(() => {
  return items.sort((a, b) => a.name.localeCompare(b.name));
}, [items]);
```
**Коли потрібен:** для важких обчислень, які не треба повторювати при кожному рендері.

### useRef
Зберігає мутабельне значення, яке **не викликає ре-рендер:**
```jsx
const inputRef = useRef(null);
const intervalRef = useRef(null);

// Доступ до DOM елемента
inputRef.current.focus();

// Зберігання значення між рендерами без ре-рендеру
intervalRef.current = setInterval(tick, 1000);
```

---

## React.memo vs useMemo vs useCallback — різниця

| Інструмент | Що робить | Коли використовувати |
|---|---|---|
| `React.memo` | Мемоізує **компонент** — не ре-рендерить, якщо props не змінились | Дочірній компонент рендериться часто з тими самими props |
| `useMemo` | Мемоізує **значення** — не перераховує, якщо deps не змінились | Важкі обчислення всередині компонента |
| `useCallback` | Мемоізує **функцію** — зберігає посилку | Передача callback у `React.memo` компонент |

**Типова помилка:** використовувати `useMemo`/`useCallback` всюди. Мемоізація має свою ціну (пам'ять, порівняння deps). Використовуй лише коли є реальна проблема з продуктивністю.

```jsx
// React.memo — обгортка компонента
const ExpensiveChild = React.memo(({ data, onClick }) => {
  return <div onClick={onClick}>{data.name}</div>;
});

// Батьківський компонент
function Parent({ items }) {
  const sorted = useMemo(() => sortItems(items), [items]);
  const handleClick = useCallback(() => selectItem(id), [id]);

  return <ExpensiveChild data={sorted} onClick={handleClick} />;
}
```

---

## Context API vs Redux vs Zustand — коли що?

### Context API
- Вбудований у React, не потребує бібліотек
- Добре для **рідко змінюваних** даних (тема, мова, auth)
- **Проблема:** кожна зміна context перерендерює ВСІ компоненти-споживачі
- Не підходить для частих оновлень (форми, анімації)

### Redux (+ Redux Toolkit)
- Передбачуваний state management з **єдиним store**
- Middleware (thunk, saga) для асинхронної логіки
- DevTools для дебагу (time travel)
- **Коли:** великий додаток, складні взаємодії між модулями, потрібна історія змін
- **Мінус:** багато boilerplate (хоча RTK значно зменшив)

### Zustand
- Мінімалістичний, без boilerplate
- Не потребує Provider
- Працює поза React (vanilla JS)
- **Коли:** середні додатки, потрібен простий глобальний стан
- Набагато простіший за Redux

**Рекомендація на інтерв'ю:** "Обираю інструмент під задачу. Для простих випадків — Context або Zustand, для складних enterprise додатків з командою — Redux Toolkit."

---

## Що таке Server Components?

**React Server Components (RSC)** — компоненти, які рендеряться **тільки на сервері** і не включаються у JavaScript bundle клієнта.

**Переваги:**
- Менший bundle size — серверні компоненти не відправляються клієнту
- Прямий доступ до БД, файлової системи, API без API routes
- Автоматичний code splitting

**Як працює:**
```jsx
// Server Component (за замовчуванням у Next.js App Router)
async function UserProfile({ id }) {
  const user = await db.query(`SELECT * FROM users WHERE id = $1`, [id]);
  return <div>{user.name}</div>;
}

// Client Component (потрібна директива)
"use client";
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Обмеження Server Components:**
- Не можуть використовувати hooks (useState, useEffect)
- Не мають доступу до browser API
- Не можуть обробляти user events (onClick)

**Правило:** Server Components за замовчуванням, "use client" тільки коли потрібна інтерактивність.

---

## React Suspense та Concurrent Mode

### Suspense
Дозволяє показувати fallback UI поки компонент завантажується:
```jsx
<Suspense fallback={<Spinner />}>
  <LazyComponent />
</Suspense>
```

**Використання:**
- Lazy loading компонентів (`React.lazy`)
- Data fetching (з бібліотеками типу React Query, SWR, або RSC)
- Вкладені Suspense boundaries для різних частин UI

### Concurrent Mode (Concurrent Features)
React може **переривати** рендеринг і повертатися до нього пізніше:

- **useTransition** — позначає оновлення як "не термінове":
```jsx
const [isPending, startTransition] = useTransition();
startTransition(() => {
  setSearchQuery(input); // це оновлення може бути перервано
});
```

- **useDeferredValue** — відкладає оновлення значення:
```jsx
const deferredQuery = useDeferredValue(query);
// deferredQuery оновиться коли React матиме час
```

**Навіщо:** щоб UI залишався відзивчивим навіть при важких оновленнях (фільтрація великих списків, навігація).

---

## Custom Hooks — як створювати і коли?

Custom Hook — це функція, яка починається з `use` і може використовувати інші hooks.

**Коли створювати:**
- Повторювана логіка в кількох компонентах
- Складна логіка, яку варто інкапсулювати
- Абстракція над побічними ефектами

**Приклади:**
```jsx
// Хук для API запитів
function useApi(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    fetch(url, { signal: controller.signal })
      .then(res => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}

// Хук для localStorage
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// Хук для debounce
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

**Правила custom hooks:**
1. Назва починається з `use`
2. Може викликати інші hooks
3. Кожен виклик хука має свій незалежний стан
4. Не ділить стан між компонентами (для цього є Context або state management)

---

## Як працює Reconciliation алгоритм?

**Reconciliation** — процес порівняння двох дерев Virtual DOM для визначення мінімальних змін.

**Два ключових евристики React:**
1. **Елементи різних типів** — повне перемонтування піддерева
2. **Елементи одного типу** — оновлення лише змінених атрибутів

**Алгоритм (спрощено):**
1. Порівнює кореневі елементи
2. Якщо тип різний (`<div>` → `<span>`) — видаляє старе дерево, створює нове
3. Якщо тип однаковий — порівнює props і оновлює лише змінені
4. Рекурсивно обходить дочірні елементи
5. Для списків використовує **keys** для ефективного порівняння

**Складність:** O(n) замість O(n³) для загального алгоритму порівняння дерев.

**React Fiber** (з React 16) — переписаний reconciler, який:
- Може переривати та відновлювати роботу
- Призначає пріоритети різним типам оновлень
- Підтримує concurrent features

---

## Keys в React — навіщо?

Keys допомагають React ідентифікувати елементи у списках і визначати, які елементи змінились, додались або видалились.

```jsx
// ✅ Правильно — стабільний унікальний ключ
{users.map(user => (
  <UserCard key={user.id} user={user} />
))}

// ❌ Погано — індекс як ключ (якщо список змінюється)
{users.map((user, index) => (
  <UserCard key={index} user={user} />
))}
```

**Чому індекс — поганий ключ:**
- При вставці елемента на початок — всі ключі зміщуються
- React вважає що ВСІ елементи змінились
- Стан компонентів може "перескочити" на інший елемент

**Коли індекс допустимий:**
- Статичний список, який ніколи не змінюється
- Немає стану в елементах списку

**Трюк з key для скидання стану:**
```jsx
// Зміна key примусово перемонтує компонент
<UserForm key={selectedUserId} user={selectedUser} />
```

---

## Controlled vs Uncontrolled components

### Controlled
React контролює значення через state:
```jsx
function ControlledInput() {
  const [value, setValue] = useState("");
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}
```
- React є "єдиним джерелом правди"
- Можна валідувати, трансформувати на кожну зміну
- Потрібен handler для кожного input

### Uncontrolled
DOM сам зберігає значення, доступ через ref:
```jsx
function UncontrolledInput() {
  const inputRef = useRef(null);
  const handleSubmit = () => console.log(inputRef.current.value);
  return <input ref={inputRef} defaultValue="" />;
}
```
- Менше коду
- Простіше інтегрувати з не-React бібліотеками
- Складніше валідувати в реальному часі

**Рекомендація:** controlled для більшості форм, uncontrolled для file inputs і простих випадків.

---

## Error Boundaries

Error Boundaries ловлять JavaScript помилки в дочірньому дереві компонентів і показують fallback UI.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <h1>Щось пішло не так</h1>;
    }
    return this.props.children;
  }
}

// Використання
<ErrorBoundary>
  <UserProfile />
</ErrorBoundary>
```

**Що ловлять:** помилки в рендері, lifecycle методах, конструкторах
**Що НЕ ловлять:** event handlers, async код, SSR, помилки в самому Error Boundary

**Примітка:** Error Boundaries доступні лише як class components. Для hooks можна використати бібліотеку `react-error-boundary`.

---

## Code splitting та lazy loading

```jsx
// Lazy loading компонента
const HeavyChart = React.lazy(() => import("./HeavyChart"));

function Dashboard() {
  return (
    <Suspense fallback={<Skeleton />}>
      <HeavyChart />
    </Suspense>
  );
}
```

**Стратегії code splitting:**
1. **Route-based** — розділення по сторінках (найпоширеніше)
2. **Component-based** — великі компоненти (модальні вікна, графіки)
3. **Vendor splitting** — окремий chunk для бібліотек

**Dynamic imports:**
```jsx
// Завантаження бібліотеки за потребою
const handleExport = async () => {
  const { exportToPDF } = await import("./export-utils");
  exportToPDF(data);
};
```

---

## Як оптимізувати продуктивність React додатку?

1. **React.memo** для компонентів, які часто ре-рендеряться з тими самими props
2. **useMemo / useCallback** для важких обчислень і стабільних callback-ів
3. **Code splitting** з React.lazy та Suspense
4. **Virtualization** великих списків (react-window, react-virtuoso)
5. **Debounce** для пошуку та input-ів
6. **Уникати inline objects/functions** в JSX (створюють нові посилки щоразу)
7. **Keys** — використовувати стабільні унікальні ключі
8. **Мінімізувати state** — зберігати мінімум необхідних даних
9. **useTransition** для не-термінових оновлень
10. **Image optimization** — lazy loading, правильні формати (WebP, AVIF)
11. **Bundle analysis** — webpack-bundle-analyzer для пошуку великих залежностей

**Правило:** не оптимізуй передчасно. Спочатку виміряй (React DevTools Profiler), потім оптимізуй.

---

## useEffect cleanup — коли і навіщо?

Cleanup функція виконується:
1. **Перед розмонтуванням** компонента
2. **Перед кожним наступним виконанням** ефекту (коли deps змінились)

```jsx
useEffect(() => {
  // Setup
  const subscription = eventBus.subscribe("update", handler);
  const timer = setInterval(tick, 1000);
  const controller = new AbortController();

  fetch(url, { signal: controller.signal });

  // Cleanup — запобігає memory leaks
  return () => {
    subscription.unsubscribe();
    clearInterval(timer);
    controller.abort();
  };
}, [url]);
```

**Коли потрібен cleanup:**
- Підписки (WebSocket, EventEmitter, DOM events)
- Таймери (setInterval, setTimeout)
- Fetch запити (AbortController)
- Сторонні бібліотеки, які потребують cleanup

**Без cleanup:**
- Запис у console
- Одноразові обчислення
- Оновлення document.title

---

## Prop drilling та як його уникнути

**Prop drilling** — передача props через кілька рівнів компонентів, які самі ці props не використовують.

```jsx
// ❌ Prop drilling
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <UserAvatar user={user} />  {/* тільки тут потрібен user */}
    </Sidebar>
  </Layout>
</App>
```

**Рішення:**

1. **Context API:**
```jsx
const UserContext = createContext(null);

function App() {
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

function UserAvatar() {
  const user = useContext(UserContext);
  return <img src={user.avatar} />;
}
```

2. **Component composition:**
```jsx
function App() {
  return (
    <Layout sidebar={<UserAvatar user={user} />} />
  );
}
```

3. **State management** (Zustand, Redux) для глобального стану

---

## Higher-Order Components vs Render Props vs Hooks

### HOC (Higher-Order Component)
Функція, яка приймає компонент і повертає новий:
```jsx
function withAuth(Component) {
  return function AuthenticatedComponent(props) {
    const user = useAuth();
    if (!user) return <Redirect to="/login" />;
    return <Component {...props} user={user} />;
  };
}
const ProtectedPage = withAuth(Dashboard);
```

### Render Props
Компонент, який приймає функцію для рендеру:
```jsx
<Mouse render={({ x, y }) => <Cursor x={x} y={y} />} />
```

### Custom Hooks (сучасний підхід)
```jsx
function useAuth() {
  const [user, setUser] = useState(null);
  // ... логіка
  return { user, login, logout };
}
```

**Висновок:** Custom Hooks замінили HOC та Render Props у більшості випадків. Вони простіші, немає "wrapper hell", краще працюють з TypeScript.

---

## React 18/19 нові фічі

### React 18
- **Automatic batching** — автоматичне групування оновлень стану (навіть у setTimeout, fetch)
- **Concurrent Features** — useTransition, useDeferredValue
- **Suspense для SSR** — streaming server-side rendering
- **createRoot** — новий API для рендерингу (замість ReactDOM.render)
- **useId** — генерація унікальних ID для accessibility

### React 19
- **React Compiler** — автоматична мемоізація (не потрібен useMemo/useCallback)
- **Actions** — спрощена обробка форм та async операцій
- **useFormStatus** — стан форми без prop drilling
- **useOptimistic** — оптимістичні оновлення UI
- **use()** — новий хук для читання ресурсів (промісів, контексту)
- **Server Actions** — виклик серверних функцій напряму з компонентів
- **Document metadata** — `<title>`, `<meta>` прямо в компонентах
- **ref як prop** — більше не потрібен forwardRef

**На інтерв'ю:** покажіть що ви слідкуєте за розвитком React і розумієте напрямок — від ручної оптимізації до автоматичної (compiler), від client-first до server-first (RSC).

---

## Next.js — SSR vs SSG vs ISR

### SSR (Server-Side Rendering)
Сторінка генерується на сервері **при кожному запиті:**
```jsx
// Next.js App Router — за замовчуванням
export default async function Page() {
  const data = await fetch(url, { cache: "no-store" });
  return <div>{data}</div>;
}
```
**Коли:** дані часто змінюються, потрібен SEO, персоналізований контент.

### SSG (Static Site Generation)
Сторінка генерується **під час збірки:**
```jsx
export default async function Page() {
  const data = await fetch(url); // кешується за замовчуванням
  return <div>{data}</div>;
}
```
**Коли:** контент рідко змінюється (блог, документація, маркетинг).

### ISR (Incremental Static Regeneration)
Статична сторінка, яка **оновлюється через заданий інтервал:**
```jsx
export default async function Page() {
  const data = await fetch(url, { next: { revalidate: 60 } }); // кожні 60 сек
  return <div>{data}</div>;
}
```
**Коли:** потрібен баланс між продуктивністю SSG та свіжістю SSR.

---

## React Testing Library — підхід до тестування

**Філософія:** тестуй так, як користувач взаємодіє з додатком.

```jsx
import { render, screen, fireEvent } from "@testing-library/react";

test("should increment counter on click", () => {
  render(<Counter />);

  const button = screen.getByRole("button", { name: /increment/i });
  fireEvent.click(button);

  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});
```

**Принципи:**
1. Шукай елементи за **ролями, текстом, лейблами** — не за CSS класами чи test-id
2. Тестуй **поведінку**, не імплементацію
3. Не тестуй внутрішній стан компонента
4. Використовуй `userEvent` замість `fireEvent` для реалістичніших подій
5. `waitFor` та `findBy` для асинхронних операцій

**Пріоритет queries:**
1. `getByRole` — найкращий (accessibility)
2. `getByLabelText` — для форм
3. `getByPlaceholderText` — для inputs
4. `getByText` — для текстового контенту
5. `getByTestId` — останній варіант
