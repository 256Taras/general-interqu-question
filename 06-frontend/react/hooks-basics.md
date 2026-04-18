# React Hooks — Основи

## Що таке React hooks і навіщо вони з'явилися?

React hooks — це функції, які дозволяють використовувати стан та інші React-можливості в функціональних компонентах. До версії 16.8 (лютий 2019) функціональні компоненти були без стану (stateless), і весь логік зберігався у класових компонентах. Hooks розв'язали цю проблему, дозволяючи функціональним компонентам мати свій власний стан, life-cycle логік та інші React-можливості без написання класів.

**Чому це важливо?** Класові компоненти — це більше коду, складніший синтаксис з `this`, зв'язування методів. Реальне життя показало, що логік часто розпилюється по різних методам (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`), і невеликі компоненти превращались у 200+ рядків коду. Hooks дозволяють групувати логік за функціональністю, а не за life-cycle фазами.

```jsx
// До hooks (класовий компонент)
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
  }

  componentDidMount() {
    document.title = `Clicked ${this.state.count} times`;
  }

  componentDidUpdate() {
    document.title = `Clicked ${this.state.count} times`;
  }

  render() {
    return (
      <button onClick={() => this.setState({ count: this.state.count + 1 })}>
        Click me ({this.state.count})
      </button>
    );
  }
}

// З hooks (функціональний компонент)
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `Clicked ${count} times`;
  }, [count]); // Запускається при зміні count

  return (
    <button onClick={() => setCount(count + 1)}>
      Click me ({count})
    </button>
  );
}
```

Функціональні компоненти з hooks — це майбутнє React. React Team активно рекомендує hooks і більше не розвивають класи.

---

## Правила hooks — ЧОМУ вони існують?

React hooks мають два суворі правила:

1. **Викликайте hooks тільки на верхньому рівні компонента** — не всередині умов (if/else), циклів або вкладених функцій
2. **Викликайте hooks тільки з React функцій** — з компонентів або власних hooks

```jsx
// ✅ ПРАВИЛЬНО
function MyComponent() {
  const [count, setCount] = useState(0);
  useEffect(() => {}, []);
  // ...
}

// ❌ НЕПРАВИЛЬНО — усередину умови
function BadComponent({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null); // ПОМИЛКА!
  }
}

// ❌ НЕПРАВИЛЬНО — всередину циклу
function AnotherBad() {
  for (let i = 0; i < 5; i++) {
    useState(i); // ПОМИЛКА!
  }
}
```

**Чому це правило існує?** React внутрішньо зберігає стан компонента в масиві (подумайте як на списку). Коли ви викликаєте `useState` першого разу, React поміщає значення у позицію 0. При другому виклику — у позицію 1. React встановлює відповідність між викликами hooks і їх індексами у внутрішньому масиві.

```
Call order:          Index:
useState(0)   →      [0] = state for count
useEffect(...)→      [1] = effect for side-effects
useState(null)→      [2] = state for user
```

Якщо ви викликаєте hooks всередину умови, порядок може змінитися, і React отримає неправильні значення стану. Наприклад:

```jsx
// Уявіть, що isLoggedIn змінюється
function Dangerous({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null); // Іноді індекс 0, іноді не вызивается
  }
  const [theme, setTheme] = useState("light"); // Іноді індекс 0, іноді індекс 1
  // React не знає, які значення до чого належать!
}
```

React надає ESLint plugin (`eslint-plugin-react-hooks`), який автоматично наловлює такі помилки.

---

## useState — базове використання

`useState` — це функція, яка додає локальний стан до функціонального компонента. Повертає масив з двома елементами: поточне значення стану та функція для його оновлення.

```jsx
import { useState } from 'react';

function Counter() {
  // Деструктурування масиву [поточниеЗначення, функцияОновленням]
  const [count, setCount] = useState(0);

  // Синтаксичний цукор для декількох станів
  const [name, setName] = useState('');
  const [isVisible, setIsVisible] = useState(false);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      
      <input 
        value={name} 
        onChange={(e) => setName(e.target.value)} 
      />
      
      <button onClick={() => setIsVisible(!isVisible)}>Toggle</button>
    </div>
  );
}
```

**Lazy initialization** — якщо початковий стан дорогий у обчисленні (наприклад, парсинг localStorage, складні калькуляції), передавайте функцію замість значення:

```jsx
// ❌ НЕПРАВИЛЬНО — запускається при кожному렌더
const [count, setCount] = useState(expensiveCalculation());

// ✅ ПРАВИЛЬНО — запускається тільки один раз при монтуванні
const [count, setCount] = useState(() => expensiveCalculation());

// Реальний приклад з localStorage
const [savedData, setSavedData] = useState(() => {
  const saved = localStorage.getItem('myData');
  return saved ? JSON.parse(saved) : { items: [] };
});
```

**Функціональний setState** — замість прямого присвоєння нового значення, передавайте функцію, яка отримує попереднє значення. Це гарантує, що ви завжди маєте актуальний стан:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    // ❌ Небезпечно — якщо onClick викликається швидко, можуть бути race conditions
    setCount(count + 1);
    setCount(count + 1); // Виконується з тим же count, не +2
  };

  const handleClickSafe = () => {
    // ✅ Правильно — завжди використовує найновіший стан
    setCount((prev) => prev + 1);
    setCount((prev) => prev + 1); // Правильно буде +2
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={handleClick}>Небезпечно +1</button>
      <button onClick={handleClickSafe}>Безпечно +2</button>
    </div>
  );
}
```

---

## useState гойчини: stale closure та batch updates

**Stale closure** — одна з найпотужніших гойчин. Коли функція обертає у себе змінні з зовнішньої області видимості, вона «запам'ятовує» їх значення на момент створення. У React це проблема, коли callback вже створений, але стан змінюється потім.

```jsx
function StaleClosureExample() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    // Цей callback був створений при рендері з count = 0
    setTimeout(() => {
      alert(`Count is ${count}`); // Завжди буде 0, навіть якщо count = 5
    }, 3000);
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={handleClick}>Check count after 3s</button>
    </div>
  );
}
```

**Рішення:** Використовуйте `useRef` або перенесіть логік всередину `useEffect`:

```jsx
function FixedExample() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count; // Завжди зберігаємо останнє значення
  }, [count]);

  const handleClick = () => {
    setTimeout(() => {
      alert(`Count is ${countRef.current}`); // Тепер правильно
    }, 3000);
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={handleClick}>Check count after 3s</button>
    </div>
  );
}
```

**Batch updates** — в React 18+, якщо ви викликаєте `setState` кілька разів підряд, React автоматично групує їх в один рендер:

```jsx
function BatchExample() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');

  const handleSubmit = () => {
    setCount(count + 1);   // Батчується
    setName('John');       // Батчується
    setCount(count + 2);   // Батчується
    // React виконає ДВА державних оновлення за один рендер, не три!
  };

  return (
    <div>
      <p>Count: {count}, Name: {name}</p>
      <button onClick={handleSubmit}>Update both</button>
    </div>
  );
}

// Примітка: async операції НЕ батчуються автоматично
const handleAsync = async () => {
  await fetchData();
  setCount(count + 1);    // Окремий рендер
  setName('Jane');        // Окремий рендер
  // В React 18 можна обернути у flushSync для явного батчування
};
```

---

## useEffect — mental model та dependency array

`useEffect` дозволяє виконувати побічні ефекти (side effects) у функціональному компоненті: отримання даних, передплати на события, зміну DOM. Запускається ПІСЛЯ того, як React намалював компонент на екрані.

**Mental model:** Уявіть, що ваш компонент — це синхронізація даних. `useEffect` говорить: "Коли ці залежності змінилися, виконай цю функцію, щоб синхронізувати з зовнішнім світом."

```jsx
function DataFetcher() {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  // Запускається один раз при монтуванні (порожній dependency array)
  useEffect(() => {
    fetch('/api/data')
      .then((res) => res.json())
      .then((json) => {
        setData(json);
        setLoading(false);
      })
      .catch((err) => {
        setError(err.message);
        setLoading(false);
      });
  }, []); // Порожній масив = запускається один раз

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  return <div>{data}</div>;
}
```

**Dependency array (залежності):**

- **Порожній масив `[]`** — запускається один раз при монтуванні
- **Без масива** — запускається після кожного рендеру (НЕ робіть так, буде infinite loop!)
- **Масив зі значеннями `[count, name]`** — запускається, коли `count` або `name` змінилися

```jsx
function DependencyExample() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');

  // Запускається при кожному рендері
  useEffect(() => {
    console.log('Рендер без залежностей');
  }); // ❌ Уникайте цього

  // Запускається при зміні count
  useEffect(() => {
    console.log(`Count змінився на ${count}`);
  }, [count]);

  // Запускається один раз при монтуванні
  useEffect(() => {
    console.log('Компонент змонтувався');
  }, []);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <input value={name} onChange={(e) => setName(e.target.value)} />
    </div>
  );
}
```

---

## useEffect cleanup — коли і чому потрібен

Функція cleanup (очистки) виконується ПЕРЕД повторним запуском ефекту або перед видаленням компонента. Потрібна для скасування передплат, timers, або звільнення ресурсів.

```jsx
function SubscriptionExample() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    // Передплатимося на онлайн-статус
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    // Функція cleanup повертається з ефекту
    return () => {
      // Відписуємося, коли компонент демонтується або ефект перезапускається
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []); // Підписуємося один раз при монтуванні

  return <div>Status: {isOnline ? 'Online' : 'Offline'}</div>;
}
```

**Interval з cleanup:**

```jsx
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setSeconds((prev) => prev + 1);
    }, 1000);

    // Cleanup: очищаємо interval, щоб не накопичувалися
    return () => clearInterval(interval);
  }, []);

  return <div>Timer: {seconds}s</div>;
}
```

**Важливо:** Якщо забудете cleanup, можуть накопичуватися передплати, timers, і створювати memory leaks.

---

## Коли НЕ треба useEffect

Типова помилка — використовувати `useEffect` для логіки, яка повинна виконуватися синхронно під час рендеру.

```jsx
// ❌ НЕПРАВИЛЬНО — обчислення давай у useEffect
function BadComputation() {
  const [count, setCount] = useState(0);
  const [doubled, setDoubled] = useState(0);

  useEffect(() => {
    setDoubled(count * 2); // Лишня оновлення, додатковий рендер
  }, [count]);

  return <div>{count} × 2 = {doubled}</div>;
}

// ✅ ПРАВИЛЬНО — обчислюй під час рендеру
function GoodComputation() {
  const [count, setCount] = useState(0);
  const doubled = count * 2; // Синхронне обчислення

  return <div>{count} × 2 = {doubled}</div>;
}
```

**Event handlers** — обробники подій не потребують `useEffect`:

```jsx
// ❌ НЕПРАВИЛЬНО
function BadHandler() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const handleClick = () => setCount(count + 1);
    return () => {}; // Навіщо?
  }, [count]);

  return <button>Count: {count}</button>;
}

// ✅ ПРАВИЛЬНО
function GoodHandler() {
  const [count, setCount] = useState(0);

  const handleClick = () => setCount(count + 1); // Просто функція

  return <button onClick={handleClick}>Count: {count}</button>;
}
```

---

## useContext — навіщо це потрібне?

`useContext` дозволяє читати значення з контексту без обгортання компоненту в `<Consumer>`. Це корисно для передачі даних глибоко у дерево компонентів без передачі пропсів на кожному рівні ("prop drilling").

```jsx
import { createContext, useContext } from 'react';

// Створюємо контекст теми
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={theme}>
      <Header />
      <MainContent />
      <Footer />
      {/* Всі вложені компоненти можуть читати theme без пропсів */}
    </ThemeContext.Provider>
  );
}

// Глибоко вложений компонент
function DeepComponent() {
  const theme = useContext(ThemeContext); // Безпосередньо отримуємо з контексту

  return (
    <div style={{ 
      backgroundColor: theme === 'dark' ? '#000' : '#fff',
      color: theme === 'dark' ? '#fff' : '#000'
    }}>
      Поточна тема: {theme}
    </div>
  );
}
```

**Приклад з auth:**

```jsx
const AuthContext = createContext(null);

function App() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Перевіряємо, чи користувач залогінений
    checkAuth().then((userData) => {
      setUser(userData);
      setLoading(false);
    });
  }, []);

  return (
    <AuthContext.Provider value={{ user, loading }}>
      <Router />
    </AuthContext.Provider>
  );
}

function ProtectedPage() {
  const { user, loading } = useContext(AuthContext);

  if (loading) return <div>Loading...</div>;
  if (!user) return <div>Redirect to login</div>;

  return <div>Welcome, {user.name}!</div>;
}
```

---

## Infinite loop у useEffect — типові причини

Infinite loop відбувається, коли `useEffect` оновлює залежність, яка спричиняє ефект запуститися знову.

```jsx
// ❌ INFINITE LOOP
function BadLoop() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setCount(count + 1); // Оновлює count
  }, [count]); // count у залежностях
  // count змінюється → запускається ефект → count змінюється → запускається ефект...
}
```

**Інші типові причини:**

```jsx
// ❌ Object як залежність — завжди новий об'єкт
function BadObject() {
  const [data, setData] = useState(null);
  const user = { id: 1, name: 'John' }; // Новий об'єкт при кожному рендері

  useEffect(() => {
    fetchUserData(user);
  }, [user]); // user завжди змінюється → infinite loop
}

// ✅ Винесіть об'єкт поза компонент або мемоізуйте
const DEFAULT_USER = { id: 1, name: 'John' };

function GoodObject() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchUserData(DEFAULT_USER);
  }, []);
}

// ❌ Функція як залежність — також завжди нова
function BadFunction() {
  const onComplete = () => console.log('done'); // Нова функція при кожному рендері

  useEffect(() => {
    fetch('/api').then(onComplete);
  }, [onComplete]); // onComplete завжди змінюється → infinite loop
}

// ✅ Винесіть функцію або скористайтесь useCallback
function GoodFunction() {
  const onComplete = useCallback(() => console.log('done'), []);

  useEffect(() => {
    fetch('/api').then(onComplete);
  }, [onComplete]);
}
```

**Як уникнути:**
1. Не оновлюйте залежності всередину ефекту
2. Для об'єктів і функцій — використовуйте `useMemo` та `useCallback`
3. Якщо потрібна залежність — подумайте, чи необхідна їй бути

---

## Dependency array pitfalls — об'єкти, масиви, функції

Це найважніша гойчина у роботі з `useEffect`. Залежності порівнюються за `===`, а не за вмістом.

```jsx
// ❌ Object як залежність
function BadObject() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('Effect ran');
  }, [{ count }]); // Новий об'єкт при кожному рендері
  // Effect запускається при кожному рендері, хоча count не змінався!
}

// ✅ Правильно — розпакуйте потрібні значення
function GoodObject() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('Effect ran');
  }, [count]); // Залежимо від примітивного значення
}

// ❌ Масив як залежність
function BadArray() {
  const [items, setItems] = useState(['a', 'b']);
  const filtered = items.filter((x) => x !== 'a'); // Новий масив

  useEffect(() => {
    console.log(filtered);
  }, [filtered]); // Infinite loop, filtered завжди новий
}

// ✅ Правильно — використовуйте useMemo
function GoodArray() {
  const [items, setItems] = useState(['a', 'b']);
  const filtered = useMemo(
    () => items.filter((x) => x !== 'a'),
    [items] // Мемоізуємо масив, щоб не був новим при кожному рендері
  );

  useEffect(() => {
    console.log(filtered);
  }, [filtered]);
}

// ❌ Функція як залежність
function BadFunction() {
  const fetchData = () => fetch('/api').then(r => r.json()); // Нова функція

  useEffect(() => {
    fetchData();
  }, [fetchData]); // Infinite loop
}

// ✅ Правильно — використовуйте useCallback
function GoodFunction() {
  const fetchData = useCallback(
    () => fetch('/api').then(r => r.json()),
    [] // Залежності для функції, не для ефекту
  );

  useEffect(() => {
    fetchData();
  }, [fetchData]);
}
```

---

## Strict Mode — чому ефекти запускаються двічі в dev

У React 18+ при включеному Strict Mode (зазвичай в `<React.StrictMode>` у `main.jsx` або `index.js`) ефекти запускаються двічі під час розробки. **Це не баг — це фіча!** React робить це, щоб виявити side effects, які не мають cleanup функції.

```jsx
// Приклад: Strict Mode запускає це двічі
function Example() {
  useEffect(() => {
    console.log('Ефект запустився'); // Буде залоговане двічі в dev!
    return () => console.log('Cleanup');
  }, []);
}

// Результат у dev консолі:
// "Ефект запустився"
// "Cleanup"
// "Ефект запустився" (дублювання для перевірки)

// У production це буде один раз
```

**Чому це робиться?** Щоб переконатися, що ваш код правильно очищується. Якщо ви ініціалізуєте interval без cleanup, Strict Mode виявить це.

```jsx
// ❌ Проблема виявляється у Strict Mode
function Timer() {
  useEffect(() => {
    const id = setInterval(() => console.log('tick'), 1000);
    // Забули очистити!
  }, []);
}

// Результат: два intervals запущені, що спричинює 'tick' кожні 500мс!

// ✅ Правильно з cleanup
function TimerFixed() {
  useEffect(() => {
    const id = setInterval(() => console.log('tick'), 1000);
    return () => clearInterval(id); // Cleanup функція
  }, []);
}

// Результат: один interval, чистий як сльоза
```

---

## Типові пастки

### 1. Запам'ятовування повернень усередину useEffect
```jsx
// ❌ Стан не оновляється
function Bad() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    return () => setCount(0); // setCount у cleanup — неправильне використання
  }, []);

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

### 2. Забути залежності в useEffect
```jsx
// ❌ ESLint попередить про це
function Bad({ userId }) {
  useEffect(() => {
    fetchUser(userId); // userId використовується, але не у залежностях!
  }, []); // Буде завжди робити запит з однаковим userId
}

// ✅ Правильно
function Good({ userId }) {
  useEffect(() => {
    fetchUser(userId);
  }, [userId]); // userId у залежностях
}
```

### 3. Оновлювання батька з дитини через useEffect
```jsx
// ❌ Може спричинити infinite loops
function Child({ onDataChange }) {
  const [data, setData] = useState('');

  useEffect(() => {
    onDataChange(data); // Батько змінює пропс → дитина перендерюється → ефект запускається
  }, [data, onDataChange]); // onDataChange завжди новий!
}

// ✅ Батько повинен мемоізувати callback
function Parent() {
  const handleDataChange = useCallback((data) => {
    console.log(data);
  }, []);

  return <Child onDataChange={handleDataChange} />;
}
```

### 4. Race conditions при fetch
```jsx
// ❌ Якщо userId змінюється швидко
function User({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]); // Стара fetch може перезаписати нову!
}

// ✅ Скасуйте попередній запит
function UserFixed({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let isMounted = true; // Прапорець

    fetchUser(userId).then((data) => {
      if (isMounted) setUser(data); // Встановлюй тільки, якщо компонент ще змонтований
    });

    return () => {
      isMounted = false; // Cleanup: встановіть false
    };
  }, [userId]);
}
```

---

**Висновок:** Hooks — це потужна і зручна система для управління станом у React. Ключ до їх правильного використання — розуміти, як React прив'язує стан до компонентів (через порядок викликів), зберігати залежності чистими, і завжди очищувати побічні ефекти.
