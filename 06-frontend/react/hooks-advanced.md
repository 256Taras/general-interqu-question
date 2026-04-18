# Advanced React Hooks

## Що таке useMemo і коли він реально потрібен?

`useMemo` мемоізує результат обчислення та повертає збережений результат, доки залежності (dependency array) не змінились. Ідея звучить просто, але на практиці це частий вибір за умовчанням, який часто шкодить.

Базова синтаксис:
```javascript
const memoizedValue = useMemo(() => {
  // дорога обробка (計算)
  return computeExpensiveValue(a, b);
}, [a, b]);
```

### Коли useMemo дійсно допомагає:

1. **Дорогі обчислення** — якщо функція виконує важкі операції (сортування великих масивів, крипто-операції)

```jsx
// Реальний case: фільтрація та сортування великого списку
const SortedUserList = ({ users, sortBy, filter }) => {
  const sortedUsers = useMemo(() => {
    console.log("Переобчислюю список"); // detta відбувається ТІЛЬКИ при змінах deps
    return users
      .filter(u => u.name.toLowerCase().includes(filter.toLowerCase()))
      .sort((a, b) => {
        if (sortBy === 'name') return a.name.localeCompare(b.name);
        if (sortBy === 'age') return a.age - b.age;
        return 0;
      });
  }, [users, sortBy, filter]);

  return (
    <ul>
      {sortedUsers.map(user => (
        <li key={user.id}>{user.name} ({user.age})</li>
      ))}
    </ul>
  );
};
```

2. **Стабілізація об'єктів для дітей** — коли дочірня компонента очікує об'єкт і залежить від нього в своєму effect

```jsx
const Parent = ({ userId }) => {
  // БЕЗ useMemo: новий об'єкт кожен render → дітей перерендерять
  // З useMemo: об'єкт стабільний → дітей не перерендерять без потреби
  const userData = useMemo(() => ({
    id: userId,
    timestamp: new Date().getTime(),
  }), [userId]);

  return <Child userData={userData} />;
};
```

3. **Передача як залежність до effect** — щоб стабілізувати залежність в useEffect

```jsx
const DataFetcher = ({ filters }) => {
  const memoizedFilters = useMemo(() => filters, [filters]);

  useEffect(() => {
    // fetchData виконується ТІЛЬКИ якщо filters дійсно змінились
    // а не кожен render з новим об'єктом
    fetchData(memoizedFilters);
  }, [memoizedFilters]);
};
```

### Коли useMemo шкодить (зменшує продуктивність):

1. **Простих значень та функцій** — витрати на мемоізацію вищі за вигоду
```jsx
// ПОГАНО: useMemo на простій операції — гірше за без нього
const expensiveValue = useMemo(() => a + b, [a, b]);

// ДОБРЕ: просто обчислюйте
const simpleValue = a + b;
```

2. **Глибокі порівняння залежностей** — якщо вы передаєте складний об'єкт в dependency array, React все одно робить порівняння
```jsx
const Parent = ({ config }) => {
  // ПОГАНО: config — новий об'єкт кожен render
  // React порівнює config кожен раз, це не допомагає
  const memoized = useMemo(() => expensive(config), [config]);
};
```

Коли сумніваєтесь — вимірюйте з DevTools Profiler. Часто useMemo гірше, ніж без нього.

---

## Що таке useCallback і чому його часто неправильно використовують?

`useCallback` мемоізує саму функцію. Повертає нову функцію ТІЛЬКИ якщо залежності змінились, інакше — ту ж саму (за посиланням).

```javascript
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### Коли useCallback реально потрібен:

1. **Передача функції як prop до memoized дитини**

```jsx
const Parent = () => {
  const [count, setCount] = useState(0);

  // БЕЗ useCallback: нова функція кожен render → Child перерендерять
  // З useCallback: та ж функція → Child НЕ перерендерять без потреби
  const handleClick = useCallback(() => {
    console.log("Клік:", count);
  }, [count]);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <MemoizedChild onEvent={handleClick} />
    </>
  );
};

const MemoizedChild = React.memo(({ onEvent }) => {
  console.log("Child рендерить");
  return <button onClick={onEvent}>Call Event</button>;
});
```

2. **Функція як залежність в useEffect чи useLayoutEffect**

```jsx
const ApiClient = ({ apiKey }) => {
  // Функція стабільна → fetchUserData робиться ТІЛЬКИ при зміні apiKey
  const fetchUserData = useCallback(async (userId) => {
    const response = await fetch(`/api/users/${userId}`, {
      headers: { 'Authorization': apiKey }
    });
    return response.json();
  }, [apiKey]);

  useEffect(() => {
    fetchUserData(123);
  }, [fetchUserData]);
};
```

### Типова помилка: useCallback без мемоізованої дитини

```jsx
// ПОГАНО: useCallback без користі
const Parent = () => {
  const handleClick = useCallback(() => {
    console.log("Click");
  }, []);

  // Child не мемоізована → все одно перерендерить
  return <Child onClick={handleClick} />;
};

const Child = ({ onClick }) => {
  return <button onClick={onClick}>Click</button>;
};
```

---

## useMemo vs useCallback — в чому різниця?

| Аспект | useMemo | useCallback |
|---|---|---|
| Що мемоізує | Результат функції (value) | Саму функцію |
| Синтаксис | `useMemo(() => value, deps)` | `useCallback(() => {...}, deps)` |
| Повертає | Обчислене значення | Функцію |
| Використання | Дорога обробка, стабілізація об'єктів | Функція як prop або в effect deps |

Практично:
```jsx
// useMemo — при render вже є результат
const value = useMemo(() => expensiveComputation(x), [x]);

// useCallback — гарантія, що функція не змінюється
const fn = useCallback(() => doSomething(x), [x]);

// Часто разом:
const data = useMemo(() => processData(items), [items]);
const handleSort = useCallback(() => {
  setItems(sortArray(data)); // використовуємо data
}, [data]);
```

---

## Чому useReducer краще за useState для складного стану?

`useReducer` корисний коли стан залежить від попереднього стану або має складну логіку оновлення. Натомість `useState` краще для простих значень.

```jsx
// ПОГАНО: useState при складному стані
const useCounter = () => {
  const [count, setCount] = useState(0);
  const [history, setHistory] = useState([0]);
  const [canUndo, setCanUndo] = useState(false);

  const increment = () => {
    const newCount = count + 1;
    setCount(newCount);
    setHistory([...history, newCount]);
    setCanUndo(true); // множення рендерів!
  };

  // складно синхронізувати стан
};

// ДОБРЕ: useReducer централізує логіку
const initialState = { count: 0, history: [0], canUndo: false };

function counterReducer(state, action) {
  switch (action.type) {
    case 'INCREMENT': {
      const newCount = state.count + 1;
      return {
        count: newCount,
        history: [...state.history, newCount],
        canUndo: true,
      };
    }
    case 'UNDO': {
      const prev = state.history[state.history.length - 2];
      return {
        count: prev,
        history: state.history.slice(0, -1),
        canUndo: state.history.length > 1,
      };
    }
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

const Counter = () => {
  const [state, dispatch] = useReducer(counterReducer, initialState);

  return (
    <>
      <div>Count: {state.count}</div>
      <div>History: {state.history.join(' → ')}</div>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'UNDO' })} disabled={!state.canUndo}>↶</button>
      <button onClick={() => dispatch({ type: 'RESET' })}>Reset</button>
    </>
  );
};
```

### State machine приклад

```jsx
// Машина станів для форми: idle → loading → success/error
const formReducer = (state, action) => {
  switch (action.type) {
    case 'SUBMIT':
      return { ...state, status: 'loading', error: null };
    case 'SUCCESS':
      return { ...state, status: 'success', data: action.payload };
    case 'ERROR':
      return { ...state, status: 'error', error: action.payload };
    case 'RESET':
      return { status: 'idle', data: null, error: null };
    default:
      return state;
  }
};

const FormComponent = () => {
  const [state, dispatch] = useReducer(formReducer, { 
    status: 'idle', 
    data: null, 
    error: null 
  });

  const handleSubmit = async (e) => {
    e.preventDefault();
    dispatch({ type: 'SUBMIT' });
    try {
      const data = await fetchData();
      dispatch({ type: 'SUCCESS', payload: data });
    } catch (err) {
      dispatch({ type: 'ERROR', payload: err.message });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {state.status === 'idle' && <button type="submit">Submit</button>}
      {state.status === 'loading' && <p>Loading...</p>}
      {state.status === 'success' && <p>✓ Success!</p>}
      {state.status === 'error' && <p>✗ {state.error}</p>}
    </form>
  );
};
```

---

## useRef — три основні use cases

`useRef` повертає об'єкт з властивістю `.current`, яка зберігається між рендерами без спричинення нового рендеру.

### 1. Прямий доступ до DOM

```jsx
const TextInput = () => {
  const inputRef = useRef(null);

  const focusInput = () => {
    inputRef.current.focus(); // прямий доступ до input element
  };

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus Input</button>
    </>
  );
};
```

### 2. Мutable value, який не спричиняє render

```jsx
const Timer = () => {
  const intervalRef = useRef(null);

  const startTimer = () => {
    // Зберігаємо interval ID, щоб мати доступ у clearInterval
    // Це НЕ спричиняє render, на відміну від setState
    intervalRef.current = setInterval(() => {
      console.log("Тик");
    }, 1000);
  };

  const stopTimer = () => {
    clearInterval(intervalRef.current);
  };

  return (
    <>
      <button onClick={startTimer}>Start</button>
      <button onClick={stopTimer}>Stop</button>
    </>
  );
};
```

### 3. Зберігання попереднього значення

```jsx
const PrevValue = ({ value }) => {
  const prevValueRef = useRef();

  useEffect(() => {
    // Після рендеру зберігаємо поточне значення як попереднє
    prevValueRef.current = value;
  }, [value]);

  return (
    <div>
      Current: {value}
      <br />
      Previous: {prevValueRef.current}
    </div>
  );
};
```

---

## forwardRef та useImperativeHandle — коли потрібні?

`forwardRef` дозволяє отримати ref функціональної компоненти. `useImperativeHandle` дозволяє кастомізувати, що батько отримує через ref.

```jsx
// Дочірня компонента: експортує функцію через ref
const CustomInput = forwardRef(({ placeholder }, ref) => {
  const inputRef = useRef(null);

  // Визначаємо публічний API компоненти
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => {
      inputRef.current.value = '';
      inputRef.current.focus();
    },
    getValue: () => inputRef.current.value,
  }), []);

  return <input ref={inputRef} placeholder={placeholder} />;
});

// Батьківська компонента: використовує методи дочірньої
const Parent = () => {
  const customInputRef = useRef(null);

  const handleClearClick = () => {
    customInputRef.current.clear();
  };

  return (
    <>
      <CustomInput ref={customInputRef} placeholder="Type something..." />
      <button onClick={handleClearClick}>Clear</button>
      <button onClick={() => console.log(customInputRef.current.getValue())}>
        Get Value
      </button>
    </>
  );
};
```

Коли використовувати: тільки коли батьківська компонента потребує особливого контролю над дочірньою (фокус, скролл, крео-їх). Уникайте за замовчанням — краще передавати props.

---

## Custom hooks — правила й приклади

Custom hook — це функція, що починається з `use` і використовує інші hooks. Розділяє логіку між компонентами.

### Правило імені: `use*`

```jsx
// ДОБРЕ: імені починається з `use`
const useLocalStorage = (key, initialValue) => { };
const useFetch = (url) => { };
const useDebounce = (value, delay) => { };

// ПОГАНО: не починається з `use` — це звичайна функція, не hook
const getLocalStorage = (key) => { }; // не hook!
```

### Приклад 1: useDebounce

```jsx
const useDebounce = (value, delay = 500) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    // Таймер встановлюється при кожній зміні value
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    // Очистка: скасовуємо попередній таймер
    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
};

// Використання: затримка пошуку до 500мс після припинення введення
const SearchUsers = () => {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 500);

  useEffect(() => {
    if (debouncedSearchTerm.length > 0) {
      // Робимо запит ТІЛЬКИ після того, як користувач припинив печатати
      fetchSearchResults(debouncedSearchTerm);
    }
  }, [debouncedSearchTerm]);

  return (
    <input 
      value={searchTerm}
      onChange={(e) => setSearchTerm(e.target.value)}
      placeholder="Search..."
    />
  );
};
```

### Приклад 2: useLocalStorage

```jsx
const useLocalStorage = (key, initialValue) => {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error('Error reading localStorage:', error);
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      // Синхронізація з localStorage
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error('Error writing to localStorage:', error);
    }
  };

  return [storedValue, setValue];
};

// Використання: автоматичне збереження теми
const ThemeSwitcher = () => {
  const [theme, setTheme] = useLocalStorage('theme', 'light');

  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Current: {theme}
    </button>
  );
};
```

### Приклад 3: useFetch

```jsx
const useFetch = (url, options = {}) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let isMounted = true; // Забезпечення того, що компонента ще змонтована

    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url, options);
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const result = await response.json();
        if (isMounted) setData(result);
      } catch (err) {
        if (isMounted) setError(err.message);
      } finally {
        if (isMounted) setLoading(false);
      }
    };

    fetchData();

    // Очистка: запобіганння memory leak при unmount
    return () => {
      isMounted = false;
    };
  }, [url, options]);

  return { data, loading, error };
};

// Використання: автоматичне завантаження даних
const UserProfile = ({ userId }) => {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <div>{user.name}</div>;
};
```

---

## useLayoutEffect vs useEffect — коли яка?

| Характеристика | useEffect | useLayoutEffect |
|---|---|---|
| Виконання | ПІСЛЯ рендеру й ображення DOM | ДО ображення DOM (синхронно) |
| Використання | 99% випадків | Вимірювання, scroll, DOM мутації |
| Блокування | Не блокує рисування | Блокує рисування (якщо повільна) |

```jsx
// ТЕСТУВАННЯ: разниця у timing

const LayoutEffectExample = () => {
  const divRef = useRef(null);

  useLayoutEffect(() => {
    // Виконується ДО того, як користувач побачить зміни
    console.log("useLayoutEffect: width =", divRef.current.offsetWidth);
    divRef.current.style.width = '200px';
  }, []);

  useEffect(() => {
    // Виконується ПІСЛЯ рендеру (користувач може бачити мигання)
    console.log("useEffect: width =", divRef.current.offsetWidth);
  }, []);

  return <div ref={divRef} style={{ width: '100px' }}>Content</div>;
};
```

### Реальна задача: вимірювання розміру елемента й встановлення позиції

```jsx
const Tooltip = ({ trigger, children }) => {
  const triggerRef = useRef(null);
  const [position, setPosition] = useState(null);

  useLayoutEffect(() => {
    // ВАЖЛИВО: useLayoutEffect, бо нам потрібна позиція ПЕРЕД ображенням
    if (triggerRef.current) {
      const rect = triggerRef.current.getBoundingClientRect();
      setPosition({
        top: rect.bottom + 10,
        left: rect.left,
      });
    }
  }, [trigger]);

  if (!position) return null;

  return (
    <>
      <button ref={triggerRef}>{trigger}</button>
      <div style={{ position: 'fixed', ...position }}>
        {children}
      </div>
    </>
  );
};
```

---

## useTransition та useDeferredValue (React 18)

Ці hooks дозволяють позначити оновлення як не-строго-обов'язкові (non-blocking). React може "відстрочити" їхнє виконання у користувача пишет.

### useTransition

```jsx
const SearchWithTransition = () => {
  const [searchTerm, setSearchTerm] = useState('');
  const [isPending, startTransition] = useTransition();

  // Великий список: 10000 елементів
  const filteredResults = searchTerm
    ? results.filter(r => r.name.includes(searchTerm))
    : results;

  const handleChange = (e) => {
    const value = e.target.value;
    
    // BEЗ startTransition: input "зависає" під час фільтрації
    // З startTransition: input响應, фільтрація відбувається в фоні
    startTransition(() => {
      setSearchTerm(value);
    });
  };

  return (
    <>
      <input 
        value={searchTerm}
        onChange={handleChange}
        placeholder="Search..."
      />
      {isPending && <p>Filtering...</p>}
      <ul>
        {filteredResults.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </>
  );
};
```

### useDeferredValue

```jsx
const ParentWithDeferredValue = () => {
  const [userInput, setUserInput] = useState('');
  
  // deferredInput оновлюється ПІСЛЯ userInput, але не блокує введення
  const deferredInput = useDeferredValue(userInput);

  return (
    <>
      <input 
        value={userInput}
        onChange={(e) => setUserInput(e.target.value)}
      />
      {/* ExpensiveComponent отримує deferredInput, тому не блокує введення */}
      <ExpensiveComponent searchTerm={deferredInput} />
    </>
  );
};

const ExpensiveComponent = ({ searchTerm }) => {
  const results = useMemo(() => {
    // Повільна обробка
    return bigDataset.filter(item => 
      item.text.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }, [searchTerm]);

  return <div>{results.length} результатів</div>;
};
```

---

## useId — для SSR і accessibility

`useId` генерує унікальний, стабільний ID (клієнт і сервер мають один ID). Важливо для SSR та accessibility.

```jsx
const FormField = ({ label }) => {
  // useId гарантує унікальність у SSR (на відміну від Math.random())
  const id = useId();

  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} type="text" />
    </>
  );
};

// На сервері й клієнті одержимо один ID → гідрація успішна

// Приклад: listbox accessibility
const ComboBox = () => {
  const id = useId();

  return (
    <div>
      <input 
        id={`${id}-input`}
        aria-controls={`${id}-listbox`}
        aria-expanded={isOpen}
      />
      <ul id={`${id}-listbox`} role="listbox">
        {options.map((opt, idx) => (
          <li key={idx} id={`${id}-option-${idx}`}>
            {opt}
          </li>
        ))}
      </ul>
    </div>
  );
};
```

---

## useSyncExternalStore — коли дані поза React

`useSyncExternalStore` підписує компоненту на зовнішній store (Redux, Zustand, MobX). Гарантує консистентність між сервером і клієнтом.

```jsx
// Приклад: інтеграція з глобальним store
const useStore = (selector) => {
  return useSyncExternalStore(
    // subscribe: функція для підписки на zміни
    (listener) => {
      const unsubscribe = store.subscribe(listener);
      return unsubscribe;
    },
    // getSnapshot: отримання поточного значення
    () => selector(store.getState()),
    // getServerSnapshot: значення для SSR
    () => selector(store.getInitialState())
  );
};

// Використання
const UserWidget = () => {
  const userName = useStore(state => state.user.name);
  const isLoggedIn = useStore(state => state.isLoggedIn);

  return (
    <div>
      {isLoggedIn ? `Hello, ${userName}` : 'Login'}
    </div>
  );
};
```

---

## Типові пастки з advanced hooks

### 1.闭合в useCallback/useEffect

```jsx
// ПОМИЛКА: `count` не в залежностях — старе значення в callback
const Counter = () => {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log(count); // завжди логує 0!
  }, []); // ПОГАНО: забули [count]

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <button onClick={handleClick}>Log Count</button>
    </>
  );
};
```

### 2. Infinite loops через неправильні залежності

```jsx
// ПОМИЛКА: об'єкт в залежностях — новий кожен render
const Fetcher = ({ config }) => {
  useEffect(() => {
    fetch(config); // новий запит кожен render!
  }, [config]); // config = новий об'єкт

  // ВИПРАВКА: мемоізуйте config або розберіть його на примітивні значення
};
```

### 3. Forgotten cleanup у custom hooks

```jsx
// ПОМИЛКА: утечка пам'яти
const useBadListener = (eventName) => {
  useEffect(() => {
    const handler = () => console.log(eventName);
    window.addEventListener(eventName, handler); //没有 cleanup!
    // ПОМИЛКА: listener ніколи не видаляється
  }, [eventName]);
};

// ВИПРАВКА: додайте cleanup function
const useGoodListener = (eventName) => {
  useEffect(() => {
    const handler = () => console.log(eventName);
    window.addEventListener(eventName, handler);
    
    return () => {
      window.removeEventListener(eventName, handler); // cleanup!
    };
  }, [eventName]);
};
```

### 4. Забувати о ESLint rules

```jsx
// ПОМИЛКА: ESLint plugin:react-hooks/exhaustive-deps розповідь вам
const Bad = () => {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setCount(c => c + 1); // функціональне оновлення = OK без count в deps
    }, 1000);
    return () => clearInterval(timer);
  }, []); // ДОБРЕ: порожній array, бо немає зовнішніх залежностей
};
```
