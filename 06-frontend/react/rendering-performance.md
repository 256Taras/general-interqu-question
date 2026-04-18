# React: Rendering Performance

## Як React рендерить компоненти? Що таке Virtual DOM?

React не працює напряму з DOM браузера при кожній зміні. Замість цього він використовує **Virtual DOM** — легкий JavaScript-представлення дерева компонентів. Коли стан або props змінюються, React:

1. Створює новий Virtual DOM на основі сучасного стану
2. Порівнює його зі старим (процес **reconciliation**)
3. Обчислює мінімальні зміни (diff)
4. Застосовує лише необхідні оновлення до реального DOM

Це значно швидше, ніж оновлювати весь DOM, тому що:
- Маніпуляція DOM дорога (браузерні reflows/repaints)
- Порівняння JS об'єктів набагато дешевше
- Батчування оновлень оптимізує рендеринг

```jsx
// Приклад: React батчує оновлення
function Counter() {
  const [count, setCount] = useState(0);
  
  const handleClick = () => {
    // Обидва setState викликають ОДИН рендер, не два
    setCount(count + 1);
    setCount(count + 2);
  };
  
  return <button onClick={handleClick}>{count}</button>;
}
```

```
Virtual DOM процес:

1. Зміна стану
        ▼
2. Новий Virtual DOM дерево створюється
        ▼
3. Diff-алгоритм порівнює старе і нове дерево
        ▼
4. Батчування: збір мінімальних змін
        ▼
5. Один раз оновити реальний DOM
        ▼
6. Браузер: parse → layout → paint (один цикл)
```

---

## Що таке React Fiber? Як це відрізняється від старого reconciler?

**Fiber** — це переписане ядро React (з версії 16), яке замінило стару Stack Reconciler. Головна різниця: **переривання роботи**.

**Стара система (Stack Reconciler):**
- Почавши рендерити дерево, React не міг зупинитися
- Якщо дерево велике, браузер заморожується (60мс+ = видно затримку)
- Немає пріоритизації: весь код має однаковий пріоритет

**Fiber (сучасна система):**
- Робота розбита на малі одиниці (units of work)
- Може переривати середину рендерингу і повернутися пізніше
- Пріоритизація: оновлення UI має вищий пріоритет за background роботу
- Фундамент для **Concurrent Rendering** і Suspense

```jsx
// Приклад: без Fiber браузер би завис при рендерингу 10000 елементів
function BigList({ items }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          {/* Fiber дозволяє рендерити це поступово,
              не блокуючи інші взаємодії користувача */}
          <ExpensiveComponent data={item} />
        </li>
      ))}
    </ul>
  );
}
```

**Concurrent Rendering** — здатність перериватися при вищому пріоритеті:

```jsx
// startTransition ознайомлює React, що це non-urgent update
import { startTransition } from 'react';

function SearchUsers() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  const handleChange = (e) => {
    const value = e.target.value;
    // Це URGENT: оновити input миттєво
    setQuery(value);
    
    // Пошук — NON-URGENT, може чекати
    startTransition(() => {
      setResults(searchUsers(value));
    });
  };
  
  return (
    <div>
      <input value={query} onChange={handleChange} />
      {/* Input реагує відразу, пошук обробляється коли браузер вільний */}
      {results.map(user => <UserCard key={user.id} user={user} />)}
    </div>
  );
}
```

---

## Коли компонент re-render-иться? Які тригери?

React re-render компонент коли:

1. **Його власний стан змінився** (`useState`, `useReducer`)
2. **Props змінилися** (навіть якщо значення однакове, але це новий об'єкт)
3. **Батьківський компонент re-render-ився** (автоматично re-render-ять всіх дітей)
4. **Context значення змінилось** (якщо компонент це обслуговує)

```jsx
// Приклад 1: re-render від батька
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Click</button>
      {/* Child ЗАВЖДИ re-render-ится коли Parent re-render-ується,
          навіть якщо props однакові */}
      <Child name="Bob" />
    </div>
  );
}

function Child({ name }) {
  console.log('Child re-rendered'); // Вивід при кожному кліку на батька
  return <div>Hello, {name}</div>;
}
```

```jsx
// Приклад 2: inline props створюють нові об'єкти
function Parent() {
  const [count, setCount] = useState(0);
  
  // ПРОБЛЕМА: новий об'єкт при кожному рендері
  const config = { theme: 'dark' };
  
  return <Child config={config} />;
  // Child re-render-ується кожного разу, тому що config це новий об'єкт
}

function Child({ config }) {
  return <div>Theme: {config.theme}</div>;
}
```

---

## React.memo — як це працює? Коли використовувати?

`React.memo` обгортає компонент і перевіряє props перед рендерингом. Якщо props не змінилися, компонент не re-render-ується.

**За замовчуванням:** поверхневе порівняння props (shallow comparison). Для глибокого порівняння — другий аргумент.

```jsx
// Без оптимізації
function Button({ label, onClick }) {
  console.log('Button rendered');
  return <button onClick={onClick}>{label}</button>;
}

// З оптимізацією
const Button = React.memo(function Button({ label, onClick }) {
  console.log('Button rendered');
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  
  // ПРОБЛЕМА: новіша функція при кожному рендері
  const handleClick = () => console.log('clicked');
  
  return (
    <div>
      <Button label="Click me" onClick={handleClick} />
      {/* Кнопка re-render-ується кожний раз, тому що handleClick — новий */}
      <div>{count}</div>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

**Пастка: inline props**

```jsx
// ПОГАНО: нові об'єкти/функції при кожному рендері
<MemoizedChild 
  config={{ theme: 'dark' }}  // новий об'єкт кожний раз
  onClick={() => alert('hi')} // новіша функція кожний раз
/>

// ДОБРЕ: стабільні посилання
const config = { theme: 'dark' };
const handleClick = () => alert('hi');

<MemoizedChild 
  config={config}
  onClick={handleClick}
/>
```

**Спеціальне порівняння:**

```jsx
const User = React.memo(
  function User({ id, data }) {
    return <div>{data.name}</div>;
  },
  (prevProps, nextProps) => {
    // Повернути true, якщо props ОДНАКОВІ (НЕ РЕНДИТИ)
    // Повернути false, якщо props РІЗНІ (РЕНДИТИ)
    return prevProps.id === nextProps.id;
  }
);
```

---

## useMemo і useCallback для performance. Коли реально допомагає?

Ці хуки мемоізують значення/функції щоб уникнути перестворення при кожному рендері.

**useMemo:** мемоізує результат обчислення

```jsx
function ExpensiveComponent({ items }) {
  // ПРОБЛЕМА: дорога операція запускається при кожному рендері
  const sorted = items.sort((a, b) => a.value - b.value);
  
  return <div>{sorted.map(i => <span key={i.id}>{i.value}</span>)}</div>;
}

// РІШЕННЯ: мемоізуємо результат
function ExpensiveComponent({ items }) {
  const sorted = useMemo(() => {
    console.log('Sorting...'); // Вивід тільки коли items змінилися
    return items.sort((a, b) => a.value - b.value);
  }, [items]); // deps: пересипати тільки коли items змінилися
  
  return <div>{sorted.map(i => <span key={i.id}>{i.value}</span>)}</div>;
}
```

**useCallback:** мемоізує функцію

```jsx
const Parent = React.memo(function Parent({ children }) {
  const [count, setCount] = useState(0);
  
  // ПРОБЛЕМА: новіша функція при кожному рендері
  const handleClick = () => console.log('clicked');
  
  return (
    <div>
      {/* MemoChild re-render-ується тому що handleClick змінилася */}
      <MemoChild onClick={handleClick} />
    </div>
  );
});

// РІШЕННЯ: useCallback
function Parent() {
  const [count, setCount] = useState(0);
  
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []); // deps: функція ніколи не пересипається
  
  return <MemoChild onClick={handleClick} />;
}
```

**Коли це реально допомагає:**
- Компонент обгорнутий у `React.memo`
- Обчислення дійсно дорого (сортування 10000+ елементів)
- Функція передається дітям, обгорнутим у `React.memo`

**Коли НЕ варто використовувати (марне):**

```jsx
// МАРНЕ: useMemo для простого значення
const value = useMemo(() => 42, []); // Просто напишіть const value = 42

// МАРНЕ: useCallback для функції без залежностей
const fn = useCallback(() => doSomething(), []); // Просто const fn = () => doSomething()

// МАРНЕ: якщо дитина не мемоізована
function Parent() {
  const expensiveValue = useMemo(() => compute(), [dep]);
  return <Child value={expensiveValue} />; // Child НЕ обгорнутий у memo
}
```

---

## Чому keys у списках важливі? Типові помилки?

**Keys** допомагають React визначити, які елементи змінилися, додалися або були видалені. Без keys React використовує індекс, що призводить до багів.

```jsx
// ПРОБЛЕМА: index як key
function UserList({ users }) {
  return (
    <ul>
      {users.map((user, index) => (
        // ПОГАНО: якщо користувача видалити, индекси переставляються
        <li key={index}>
          <input type="text" defaultValue={user.name} />
          {user.name}
        </li>
      ))}
    </ul>
  );
}

// Сценарій:
// 1. users = [{id:1, name:'Alice'}, {id:2, name:'Bob'}]
//    렌деры: <li key=0>Alice...</li>, <li key=1>Bob...</li>
// 2. Видаляємо Alice: users = [{id:2, name:'Bob'}]
//    React думає: "element 0 тепер Bob" → не переборудовувати
//    Результат: input все ще показує "Alice", але текст "Bob" (БАГ!)
```

```jsx
// ДОБРЕ: стабільний унікальний ключ
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => (
        // ДОБРЕ: user.id не змінюється при переупорядкуванні
        <li key={user.id}>
          <input type="text" defaultValue={user.name} />
          {user.name}
        </li>
      ))}
    </ul>
  );
}
```

**Чому keys важливі:**

```
Без key (index):
  users[0] = Alice    → key=0
  users[1] = Bob      → key=1

  Видаляємо Alice:
  users[0] = Bob      → key=0 (React переиспользует DOM для key=0)
                         state/input остаются от Alice!

З key (id):
  id=1: Alice         → key=id-1
  id=2: Bob           → key=id-2

  Видаляємо Alice:
  id=2: Bob           → key=id-2 (React знає, що це інший елемент)
                         state оновлюється правильно
```

```jsx
// Типова помилка: key, залежна від індексу під час сортування
function SortedUsers({ users }) {
  const [sortBy, setSortBy] = useState('name');
  
  const sorted = useMemo(() => {
    return [...users].sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return a.id - b.id;
    });
  }, [users, sortBy]);
  
  return (
    <ul>
      {sorted.map((user, index) => (
        // ПОГАНО: key залежить від сортування
        <li key={`${sortBy}-${index}`}>{user.name}</li>
        // ДОБРЕ:
        // <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

---

## Code Splitting: React.lazy + Suspense. Коли це потрібне?

**Code Splitting** розбиває JavaScript бандл на менші чанки, які завантажуються за потребою. Це скорочує початкове завантаження.

```jsx
// Традиційно: весь код завантажується одразу
import HeavyModal from './HeavyModal'; // 500KB!

// З code splitting: завантажується тільки при відкритті
const HeavyModal = React.lazy(() => import('./HeavyModal'));

function App() {
  const [open, setOpen] = useState(false);
  
  return (
    <div>
      <button onClick={() => setOpen(true)}>Open Modal</button>
      {open && (
        <Suspense fallback={<div>Loading...</div>}>
          <HeavyModal onClose={() => setOpen(false)} />
        </Suspense>
      )}
    </div>
  );
}
```

**Коли використовувати:**
- Велика вкладка/сторінка з тяжким компонентом
- Модалі, які відкриваються рідко
- Адміністративні розділи для звичайних користувачів
- Складні редактори (rich text, 3D viewer)

**Типова помилка:**

```jsx
// ПОГАНО: лінива завантаження всього що можна
const A = React.lazy(() => import('./A'));
const B = React.lazy(() => import('./B'));
const C = React.lazy(() => import('./C'));
const D = React.lazy(() => import('./D'));

// Користувач чекає на 4 завантаження разом — погіршення UX

// ДОБРЕ: split-це по маршрутах або логічним розділам
const AdminPanel = React.lazy(() => import('./pages/AdminPanel'));
const UserProfile = React.lazy(() => import('./pages/UserProfile'));
```

---

## Що таке Suspense? Як його використовувати?

**Suspense** дозволяє компонентам "приспати" під час завантаження даних або коду. Батьківський компонент показує fallback UI до завершення.

```jsx
// С lazy components
const LazyComponent = React.lazy(() => import('./Heavy'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <LazyComponent />
    </Suspense>
  );
}
```

```jsx
// Експериментально: з data fetching (React 19+)
function User({ userId }) {
  // Це еще не стабільне API, але демонструє концепцію
  const user = use(fetchUser(userId)); // компонент "спить"
  return <div>{user.name}</div>;
}

function App({ userId }) {
  return (
    <Suspense fallback={<Spinner />}>
      <User userId={userId} />
    </Suspense>
  );
}
```

**Вложена Suspense — для більш тонкого контролю:**

```jsx
function App() {
  return (
    <div>
      <Header /> {/* Завантажується одразу */}
      
      {/* Ліва панель чекає */}
      <Suspense fallback={<SidebarLoader />}>
        <Sidebar />
      </Suspense>
      
      {/* Контент чекає окремо */}
      <Suspense fallback={<ContentLoader />}>
        <MainContent />
      </Suspense>
    </div>
  );
}
```

---

## Як профілювати React? React DevTools Profiler

**React DevTools Profiler** записує всі рендери, їх тривалість та причину.

```
1. Відкрити DevTools → Components tab
2. Натиснути кнопку Profiler
3. Виконати дію на сторінці (кліки, введення)
4. Зупинити запис
```

**Що дивитися:**
- **Ranked chart:** компоненти, відсортовані за часом рендерингу
- **Flamegraph:** візуалізація часу (ширше = довше)
- **Commit flame graph:** за окремими оновленнями

**Типові напрацювання:**

```
ПРОБЛЕМА: Component X takes 500ms

1. Дивимось, чому X re-render-ується (див "Reason")
2. Чи батька? Обгорнути X у React.memo
3. Чи props передалися? Мемоізувати props з useMemo/useCallback
4. Чи вивід від context? Розділити context на менші
5. Чи render logic дорогий? Перенести в useMemo
```

---

## Chrome Performance Tab — знаходимо bottlenecks

React DevTools показує логіку React. Chrome Performance tab показує браузерні оновлення (layout, paint).

```
1. Відкрити DevTools → Performance tab
2. Натиснути Record
3. Виконати дію
4. Натиснути Stop
```

**Дивимось на:**
- **Main thread graph:** червоне = заблокований основний потік
- **FCP (First Contentful Paint):** коли користувач вперше бачить контент
- **LCP (Largest Contentful Paint):** коли завантажується найбільший елемент
- **Layout shifts:** CLS (Cumulative Layout Shift)

```jsx
// Приклад: infinite loop в useEffect
function BadComponent() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    // ПРОБЛЕМА: не має deps, тому запускається при кожному рендері
    setCount(count + 1);
  });
  
  return <div>{count}</div>;
  // Результат: браузер залипає, Chrome Perf показує червоне графік
}

// РІШЕННЯ: додати deps
useEffect(() => {
  setCount(count + 1);
}, []); // Запускається один раз
```

---

## Типові причини повільного UI. Як їх знаходити?

1. **Re-render всього дерева від однієї зміни state:**
   - Рішення: поділити state на менші контексти або перемістити state нижче

2. **Inline props/callbacks:**
   - Рішення: мемоізація або useCallback

3. **Дорогі обчислення в render:**
   - Рішення: useMemo або перенести в worker

4. **Багато вложених контекстів:**
   - Рішення: атомарні контексти (Zustand, Jotai)

5. **Великі списки без віртуалізації:**
   - Рішення: react-window або react-virtual

```jsx
// Приклад: віртуалізація великих списків
import { FixedSizeList } from 'react-window';

function HugeList({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={35}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
// Рендерить тільки видимі 20 елементів, не 10000
```

---

## Server Components vs Client Components (React 19 / Next.js)

**Server Components (RSC):** рендеряться на сервері, відправляють JSON користувачу

**Переваги:**
- Доступ до БД, API ключів без відправки клієнту
- Менше JS у браузер
- SEO-friendly

**Обмеження:**
- Немає hooks (`useState`, `useEffect`)
- Немає браузерних API (`window`, `localStorage`)

```jsx
// app/page.jsx — Server Component за замовчуванням
export default async function Page() {
  // Pode rodar async code
  const users = await db.users.findAll();
  
  return (
    <div>
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}

// components/UserCard.jsx — обов'язково Client Component
'use client'; // Ця директива позначає клієнтський компонент

import { useState } from 'react';

export function UserCard({ user }) {
  const [liked, setLiked] = useState(false);
  return (
    <div>
      {user.name}
      <button onClick={() => setLiked(!liked)}>
        {liked ? '❤️' : '🤍'}
      </button>
    </div>
  );
}
```

**Коли використовувати що:**
- Server: для отримання даних, захисту ключів
- Client: для інтерактивності (форми, кліки, стейт)

---

## Concurrent Rendering і startTransition

**Concurrent Rendering** дозволяє React перериватися при важливих оновленнях.

```jsx
import { useState, startTransition } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  const handleSearch = (e) => {
    const value = e.target.value;
    
    // URGENT: миттєво оновити input
    setQuery(value);
    
    // NON-URGENT: пошук може чекати
    startTransition(() => {
      const found = expensiveSearch(value);
      setResults(found);
    });
  };
  
  return (
    <div>
      <input value={query} onChange={handleSearch} />
      {/* Пошук показує старий результат, поки завантажується новий */}
      {results.map(r => <Result key={r.id} result={r} />)}
    </div>
  );
}
```

**Коли це допомагає:**
- Дорога фільтрація під час введення
- Велике дерево компонентів при одному оновленні
- Поточні взаємодії мають пріоритет перед фоновими

---

## Типові пастки в React Performance

```javascript
// ПАСТКА 1: Object.is порівняння з хуками
const config = { theme: 'dark' }; // новий об'єкт при кожному рендері
useEffect(() => {
  // Цей ефект запускається КОЖНИЙ раз, хоча config має однаковий вміст
  applyTheme(config);
}, [config]); // ПОГАНО

// РІШЕННЯ: мемоізуємо об'єкт
const config = useMemo(() => ({ theme: 'dark' }), []);
useEffect(() => {
  applyTheme(config);
}, [config]); // Тепер лише один раз

// ПАСТКА 2: useCallback з занадто багато залежностей
const expensiveFunc = useCallback(() => {
  return data.map(computeExpensive);
}, [data, otherDep1, otherDep2, otherDep3]);
// Якщо кожна залежність часто змінюється, useCallback марний

// ПАСТКА 3: контекст спричиняє re-render всіх consumers
const ThemeContext = createContext();

function Provider({ children }) {
  const [theme, setTheme] = useState('light');
  const [otherState, setOtherState] = useState('...');
  
  return (
    // Коли otherState змінюється, ВСІ компоненти що обслуговують контекст re-render-ються
    <ThemeContext.Provider value={{ theme, otherState }}>
      {children}
    </ThemeContext.Provider>
  );
}

// РІШЕННЯ: розділити контексти
const ThemeContext = createContext();
const OtherContext = createContext();

// ПАСТКА 4: забування cleanup в useEffect
useEffect(() => {
  const interval = setInterval(() => {
    // утікає memory leak коли компонент unmount-ується
    console.log('tick');
  }, 1000);
  
  // ЗАБУВ: return () => clearInterval(interval);
}, []);

// ПАСТКА 5: мемоізація функції, але забув мемоізувати результат
const items = useMemo(() => transform(data), [data]);

// Але якщо передаємо function prop:
const handleClick = useCallback(() => {
  alert(items); // items змінюється при зміні data, но handleClick не змінюється
}, []); // ПОГАНО: handleClick застарілий

// ДОБРЕ:
const handleClick = useCallback(() => {
  alert(items);
}, [items]); // Залежність від items
```

