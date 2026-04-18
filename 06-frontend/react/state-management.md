# State Management у React

## Які типи state у React застосунку?

State у React можна розділити на кілька категорій залежно від його місцезнаходження та значення:

| Тип | Місцезнаходження | Приклад |
|---|---|---|
| **Local (Component) State** | Всередині компоненту | `useState()`, мова форми |
| **Lifted State** | Батьківський компонент | Стан між сестринськими компонентами |
| **Shared State** | Декілька компонентів | Стан користувача, меню |
| **Server State** | На сервері | Дані з API, бази даних |
| **URL State** | В URL (query params) | Фільтри, пагінація, поточна вкладка |
| **Derived State** | Обчислена з іншого state | Відфільтровані списки, розраховані значення |

```jsx
// 1. LOCAL STATE — тільки всередині компоненту
function SearchBox() {
  const [query, setQuery] = useState('');
  // Цей state потрібний тільки цьому компоненту
  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Пошук..."
    />
  );
}

// 2. LIFTED STATE — переміщено в батька
function Parent() {
  const [activeTab, setActiveTab] = useState('profile');
  // Стан піднято, щоб синхронізувати вкладки та контент
  return (
    <>
      <Tabs active={activeTab} onChange={setActiveTab} />
      <TabContent tab={activeTab} />
    </>
  );
}

// 3. DERIVED STATE — обчислена з іншого
function FilteredList({ items, searchQuery }) {
  // Це НЕ є state, це обчислена властивість
  const filtered = items.filter(
    item => item.name.includes(searchQuery)
  );
  return <ul>{filtered.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
}
```

---

## Lifting State Up — класичний pattern

Lifting State Up - це pattern, коли ви переміщаєте state з дочірніх компонентів до їхнього найближчого спільного батька. Це необхідно, щоб синхронізувати стан між кількома дочірніми компонентами.

```jsx
// ❌ ПОГАНО: State в обох компонентах (не синхронізовані)
function BadExample() {
  return (
    <>
      <InputA />
      <InputB />
    </>
  );
}

function InputA() {
  const [value, setValue] = useState('');
  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
}

function InputB() {
  const [value, setValue] = useState(''); // Окремий state!
  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
}

// ✅ ДОБРЕ: State в батька, props в дітей
function GoodExample() {
  const [sharedValue, setSharedValue] = useState('');
  
  return (
    <>
      <InputA value={sharedValue} onChange={setSharedValue} />
      <InputB value={sharedValue} onChange={setSharedValue} />
    </>
  );
}

function InputA({ value, onChange }) {
  return (
    <input
      value={value}
      onChange={(e) => onChange(e.target.value)}
      placeholder="Input A"
    />
  );
}
```

### Коли Lifting State Up межить:

1. **Глибока ієрархія (prop drilling)** — якщо батьків більше 3-4 рівнів, lepше використати Context або Zustand.
2. **Багато часто оновлюваного state** — батьківський компонент буде часто re-render.
3. **Незалежні дерева компонентів** — часто потрібна глобальна система управління.

```jsx
// Prop drilling — занадто багато проміжних пропсів
function App() {
  const [user, setUser] = useState(null);
  return <Level1 user={user} setUser={setUser} />;
}

function Level1({ user, setUser }) {
  return <Level2 user={user} setUser={setUser} />;
}

function Level2({ user, setUser }) {
  return <Level3 user={user} setUser={setUser} />;
}

function Level3({ user, setUser }) {
  return <div>Користувач: {user?.name}</div>;
}

// Краще: Context API або Zustand
```

---

## Context API — коли доречно, коли НЕ варто

Context API дозволяє передавати state без prop drilling. **Однак**: Context перемальовує весь subtree при кожній зміні, що проблемно для частих оновлень.

```jsx
// ✅ ДОБРЕ: Рідкі оновлення (тема, мова, користувач)
const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// ❌ ПОГАНО: Частодовані оновлення (фільтри, пошук)
const SearchContext = createContext();

function SearchProvider({ children }) {
  const [query, setQuery] = useState('');
  // Кожна зміна query → перемальовує весь subtree!
  
  return (
    <SearchContext.Provider value={{ query, setQuery }}>
      {children}
    </SearchContext.Provider>
  );
}

function SearchBox() {
  const { query, setQuery } = useContext(SearchContext);
  
  return (
    <input
      value={query}
      // Кожна літера викликає перемальовування всіх компонентів
      // у SearchContext.Provider
      onChange={(e) => setQuery(e.target.value)}
    />
  );
}

// ПАСКА: Весь subtree перемальовується на кожну букву!
// Крім батька input-у, якщо у нього нема useMemo() або memo()
```

### Оптимізація Context:

```jsx
// Розділи Context на малі, незалежні частини
const UserContext = createContext(); // Рідко мінюється
const FilterContext = createContext(); // Часто мінюється

function Store({ children }) {
  const [user, setUser] = useState(null);
  const [filters, setFilters] = useState({});
  
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <FilterContext.Provider value={{ filters, setFilters }}>
        {children}
      </FilterContext.Provider>
    </UserContext.Provider>
  );
}

// Або вибери Zustand для частих оновлень
```

---

## Redux (RTK) — архітектура та коли варто

Redux — це централізована сховища (store) з предиктабельним управлінням станом. Redux Toolkit (RTK) скорочує boilerplate.

```jsx
// Redux Toolkit — мінімалістичний Redux
import { createSlice, configureStore } from '@reduxjs/toolkit';

// 1. Slice — комбінація reducers, actions, selectors
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => {
      state.value += 1; // Immer дозволяє мутувати
    },
    decrement: (state) => {
      state.value -= 1;
    },
    incrementByAmount: (state, action) => {
      state.value += action.payload;
    }
  }
});

// 2. Store
const store = configureStore({
  reducer: {
    counter: counterSlice.reducer,
    // інші reducers...
  }
});

// 3. Використання в компоненті
import { useDispatch, useSelector } from 'react-redux';

function Counter() {
  const dispatch = useDispatch();
  // Selector визначає, який state потрібен
  const count = useSelector((state) => state.counter.value);
  
  return (
    <div>
      <p>Лічильник: {count}</p>
      <button onClick={() => dispatch(counterSlice.actions.increment())}>
        +1
      </button>
      <button onClick={() => dispatch(counterSlice.actions.incrementByAmount(5))}>
        +5
      </button>
    </div>
  );
}
```

### Архітектура Redux:

| Частина | Завдання | Приклад |
|---|---|---|
| **Action** | Описує подію | `{ type: 'counter/increment', payload: 5 }` |
| **Reducer** | Оновлює state | `(state, action) => { ... return newState }` |
| **Selector** | Витягує дані зі store | `(state) => state.counter.value` |
| **Middleware** | Перехоплює actions | Thunk, Saga, Logger |

### Коли Redis ВАРТО:

1. **Багато компонентів вживають один state** — користувач, налаштування, авторизація.
2. **Складна бізнес-логіка** — обчислення, валідація, побічні ефекти.
3. **Time-travel debugging, dev tools** — Redux DevTools чудові для дебагу.
4. **Команда звична до Redux** — consistency важлива.

### Коли Redux НЕ варто:

1. **Малий застосунок** — useState достатньо.
2. **Мало спільного state** — Context або Zustand простіше.
3. **Частодовані оновлення (форми, анімація)** — Redux має оверхед.

---

## Redux Middleware — Thunk та Saga

Middleware дозволяє обробляти асинхронні операції перед редюсерами.

```jsx
// THUNK — функція замість об'єкту action
import { createAsyncThunk } from '@reduxjs/toolkit';

const fetchUser = createAsyncThunk(
  'user/fetchUser',
  async (userId, { rejectWithValue }) => {
    try {
      const res = await fetch(`/api/users/${userId}`);
      if (!res.ok) throw new Error('Помилка завантаження');
      return res.json();
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, loading: false, error: null },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.loading = false;
        state.data = action.payload;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      });
  }
});

function UserProfile() {
  const dispatch = useDispatch();
  const { data, loading, error } = useSelector((state) => state.user);
  
  useEffect(() => {
    dispatch(fetchUser(123)); // Dispatching async thunk
  }, [dispatch]);
  
  if (loading) return <div>Завантаження...</div>;
  if (error) return <div>Помилка: {error}</div>;
  return <div>{data?.name}</div>;
}
```

### Saga (складніша альтернатива):

```javascript
// Redux Saga — для складних flow
import { call, put, takeEvery } from 'redux-saga/effects';

function* fetchUserSaga(action) {
  try {
    const user = yield call(fetch, `/api/users/${action.payload}`);
    yield put({ type: 'USER_LOADED', payload: user });
  } catch (error) {
    yield put({ type: 'USER_ERROR', payload: error.message });
  }
}

function* rootSaga() {
  yield takeEvery('FETCH_USER_REQUEST', fetchUserSaga);
}
```

Saga складніша, але краща для:
- Паралельних операцій
- Рідких запитань до API
- Скасування операцій

---

## Zustand — мінімалістичний state manager

Zustand — це легкий (2.2 KB) альтернатива Redux без boilerplate.

```jsx
import { create } from 'zustand';

// Визначаємо store з одиницею коду
const useAuthStore = create((set) => ({
  user: null,
  isLoading: false,
  
  // Дії просто оновлюють state
  login: async (email, password) => {
    set({ isLoading: true });
    try {
      const res = await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify({ email, password })
      });
      const user = await res.json();
      set({ user, isLoading: false });
    } catch (error) {
      set({ isLoading: false });
    }
  },
  
  logout: () => set({ user: null })
}));

function LoginForm() {
  const user = useAuthStore((state) => state.user);
  const isLoading = useAuthStore((state) => state.isLoading);
  const login = useAuthStore((state) => state.login);
  
  return (
    <div>
      {user ? (
        <p>Привіт, {user.name}!</p>
      ) : (
        <button
          onClick={() => login('user@example.com', 'pass')}
          disabled={isLoading}
        >
          Увійти
        </button>
      )}
    </div>
  );
}

// Zustand НЕ викликає re-render, якщо вибірка не змінилася
// Автоматична оптимізація без memo()
```

---

## Jotai і Recoil — atomic state management

Atomic state — кожен атом це окремий стан. Тільки компоненти, що читають змінений атом, перемальовуються.

```jsx
// JOTAI — простіший Recoil
import { atom, useAtom } from 'jotai';

const countAtom = atom(0);
const textAtom = atom('');

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  
  return (
    <div>
      <p>Лічильник: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}

function TextInput() {
  // Цей компонент НЕ перемальовується, якщо count змінюється
  const [text, setText] = useAtom(textAtom);
  
  return (
    <input value={text} onChange={(e) => setText(e.target.value)} />
  );
}

// Атом-селектор (derived state)
const doubleCountAtom = atom(
  (get) => get(countAtom) * 2 // читання інших атомів
);

function DoubleCount() {
  const [doubleCount] = useAtom(doubleCountAtom);
  return <p>Подвійно: {doubleCount}</p>;
}
```

**Jotai vs Zustand:**

| Аспект | Jotai | Zustand |
|---|---|---|
| **Архітектура** | Атомарна (кожен атом окремо) | Монолітний store |
| **Rendering** | Тільки атомарні зміни | Весь store update |
| **API** | `useAtom()` | Selectors і дії |
| **Складність** | Більше для великих app | Простіша для малих |
| **Dev Tools** | Базові | Редюкс dev tools |

---

## React Query / TanStack Query — server state окремо від client state

Server state (дані з API) потребують спеціальної обробки: кешування, синхронізація, оновлення у фоні.

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Query — отримання даних
function Users() {
  const {
    data: users,
    isLoading,
    error,
    refetch // Функція для ручного перезавантаження
  } = useQuery({
    queryKey: ['users'], // Унікальний ключ
    queryFn: async () => {
      const res = await fetch('/api/users');
      return res.json();
    },
    staleTime: 5 * 60 * 1000, // Кеш 5 хвилин
    gcTime: 10 * 60 * 1000 // Видалити через 10 хвилин
  });
  
  if (isLoading) return <div>Завантаження...</div>;
  if (error) return <div>Помилка: {error.message}</div>;
  
  return (
    <div>
      {users?.map(user => <div key={user.id}>{user.name}</div>)}
      <button onClick={() => refetch()}>Оновити</button>
    </div>
  );
}

// Mutation — зміна даних
function CreateUser() {
  const queryClient = useQueryClient();
  
  const { mutate, isPending } = useMutation({
    mutationFn: async (newUser) => {
      const res = await fetch('/api/users', {
        method: 'POST',
        body: JSON.stringify(newUser)
      });
      return res.json();
    },
    onSuccess: (newUser) => {
      // Оновити кеш автоматично
      queryClient.setQueryData(['users'], (oldUsers) => [
        ...oldUsers,
        newUser
      ]);
    }
  });
  
  return (
    <button
      onClick={() => mutate({ name: 'John', email: 'john@example.com' })}
      disabled={isPending}
    >
      {isPending ? 'Збереження...' : 'Створити'}
    </button>
  );
}
```

---

## Коли вибирати що?

| Сценарій | Рішення | Причина |
|---|---|---|
| Форма, вкладки, меню у одного компонента | `useState()` | Простота |
| State між 2-3 сестринськими компонентами | Lifting State Up | Мінімальна складність |
| Користувач, тема, мова (рідко мінюється) | Context API | Немає overhead |
| Частодовані оновлення (пошук, фільтри) | Zustand / Jotai | Оптимізація rendering |
| Складна архітектура, багато логіки | Redux (RTK) | Scalability, dev tools |
| Server data, API, кешування | React Query | Синхронізація сервера |

```jsx
// Комбінований підхід на реальному проекті
const useAuthStore = create((set) => ({
  // Zustand для локального стану
  user: null,
  setUser: (user) => set({ user })
}));

function App() {
  // Context для теми
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <QueryClientProvider client={queryClient}>
        <Routes>
          <Route path="/users" element={<UsersPage />} />
        </Routes>
      </QueryClientProvider>
    </ThemeContext.Provider>
  );
}

function UsersPage() {
  // React Query для сервер-state
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(r => r.json())
  });
  
  // Zustand для фільтрів (локальний)
  const filters = useFiltersStore((s) => s.filters);
  
  const filtered = users?.filter(u => matchesFilters(u, filters));
  return <UserList users={filtered} />;
}
```

---

## State Normalization — чому і як

Normalized state усуває дублювання та спрощує оновлення.

```javascript
// ❌ ПОГАНО: Дублювання (коли користувач мінюється, вик. скрізь)
const badState = {
  users: [
    { id: 1, name: 'Alice', posts: [
      { id: 101, title: 'Post 1', userId: 1 }
    ]},
    { id: 2, name: 'Bob', posts: [] }
  ]
};

// ✅ ДОБРЕ: Нормалізована (окремі entities)
const goodState = {
  entities: {
    users: {
      '1': { id: 1, name: 'Alice' },
      '2': { id: 2, name: 'Bob' }
    },
    posts: {
      '101': { id: 101, title: 'Post 1', userId: 1 },
      '102': { id: 102, title: 'Post 2', userId: 1 }
    }
  },
  result: {
    userIds: ['1', '2'],
    postIds: ['101', '102']
  }
};

// Оновлення користувача легко
goodState.entities.users['1'].name = 'Alice Updated';
// Усі посилання на Alice оновлюються автоматично

// Selector для відновлення
const selectUserWithPosts = (state, userId) => {
  const user = state.entities.users[userId];
  const userPosts = state.entities.posts;
  const posts = Object.values(userPosts).filter(
    p => p.userId === userId
  );
  return { ...user, posts };
};
```

### Redux Toolkit normalizationSlice:

```jsx
import { createEntityAdapter } from '@reduxjs/toolkit';

const usersAdapter = createEntityAdapter();

const usersSlice = createSlice({
  name: 'users',
  initialState: usersAdapter.getInitialState(),
  reducers: {
    userAdded: usersAdapter.addOne,
    userUpdated: usersAdapter.updateOne,
    userRemoved: usersAdapter.removeOne
  }
});

function UserList() {
  // Selectors автоматично дають ids, byId, all
  const users = useSelector((state) => usersAdapter.selectAll(state.users));
  return users.map(user => <div key={user.id}>{user.name}</div>);
}
```

---

## Форма-state: react-hook-form vs formik

Форми часто є найскладнішою частиною state management.

```jsx
// REACT-HOOK-FORM — мінімалістичний, швидкий
import { useForm } from 'react-hook-form';

function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    defaultValues: { email: '', password: '' }
  });
  
  const onSubmit = (data) => console.log(data);
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register('email', {
          required: 'Потрібна пошта',
          pattern: { value: /^.+@.+$/, message: 'Невалідна пошта' }
        })}
        placeholder="Email"
      />
      {errors.email && <span>{errors.email.message}</span>}
      
      <input
        {...register('password', { required: 'Потрібен пароль', minLength: 6 })}
        type="password"
        placeholder="Пароль"
      />
      {errors.password && <span>{errors.password.message}</span>}
      
      <button type="submit">Увійти</button>
    </form>
  );
}

// FORMIK — більше контролю, підтримка вкладених форм
import { Formik, Form, Field, ErrorMessage } from 'formik';
import * as Yup from 'yup';

const validationSchema = Yup.object({
  email: Yup.string().email().required('Потрібна пошта'),
  password: Yup.string().min(6).required('Потрібен пароль')
});

function LoginFormFormik() {
  return (
    <Formik
      initialValues={{ email: '', password: '' }}
      validationSchema={validationSchema}
      onSubmit={(values) => console.log(values)}
    >
      <Form>
        <Field name="email" placeholder="Email" />
        <ErrorMessage name="email" component="div" />
        
        <Field name="password" type="password" placeholder="Пароль" />
        <ErrorMessage name="password" component="div" />
        
        <button type="submit">Увійти</button>
      </Form>
    </Formik>
  );
}
```

| Аспект | react-hook-form | formik |
|---|---|---|
| **Розмір** | 8.5 KB | 13 KB |
| **Performance** | Краща (менше re-renders) | Задовільна |
| **API** | register(), unregister() | useField(), connect() |
| **Вкладені поля** | Гарні (arrays) | Краще (FieldArray) |
| **Контроль** | Менше | Більше |

---

## URL як source of truth для фільтрів і пагінації

Зберігання фільтрів в URL дозволяє: bookmark, share, back/forward.

```jsx
import { useSearchParams } from 'react-router-dom';

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  
  // Читання з URL
  const page = parseInt(searchParams.get('page')) || 1;
  const category = searchParams.get('category') || 'all';
  const sort = searchParams.get('sort') || 'name';
  
  // Запит з параметрами URL
  const { data: products } = useQuery({
    queryKey: ['products', page, category, sort],
    queryFn: () => fetch(
      `/api/products?page=${page}&category=${category}&sort=${sort}`
    ).then(r => r.json())
  });
  
  const handleCategoryChange = (newCategory) => {
    // Оновити URL
    setSearchParams({
      page: '1', // Reset на першу сторінку
      category: newCategory,
      sort
    });
  };
  
  const handlePageChange = (newPage) => {
    setSearchParams({ page: newPage, category, sort });
  };
  
  return (
    <div>
      <select value={category} onChange={(e) => handleCategoryChange(e.target.value)}>
        <option value="all">Усі</option>
        <option value="electronics">Електроніка</option>
        <option value="books">Книги</option>
      </select>
      
      <div>
        {products?.map(p => <div key={p.id}>{p.name}</div>)}
      </div>
      
      <button onClick={() => handlePageChange(page - 1)} disabled={page === 1}>
        Назад
      </button>
      <span>Сторінка {page}</span>
      <button onClick={() => handlePageChange(page + 1)}>
        Далі
      </button>
    </div>
  );
}
```

---

## Типові пастки

### 1. Infinite Re-renders з Context

```jsx
// ❌ ПОГАНО
function Provider({ children }) {
  const [value, setValue] = useState(0);
  const contextValue = { value, setValue }; // Новий об'єкт на кожен render!
  
  return (
    <MyContext.Provider value={contextValue}> {/* Увесь subtree re-render */}
      {children}
    </MyContext.Provider>
  );
}

// ✅ ДОБРЕ
function Provider({ children }) {
  const [value, setValue] = useState(0);
  const contextValue = useMemo(
    () => ({ value, setValue }),
    [value] // Тільки коли value змінюється
  );
  
  return (
    <MyContext.Provider value={contextValue}>
      {children}
    </MyContext.Provider>
  );
}
```

### 2. Забуття re-render оптимізації в Redux

```jsx
// ❌ ПОГАНО: re-render на будь-яку зміну store
const User = () => {
  const state = useSelector((state) => state); // Увесь store!
  return <div>{state.user.name}</div>;
};

// ✅ ДОБРЕ: Тільки той стан, який потрібен
const User = () => {
  const userName = useSelector((state) => state.user.name);
  return <div>{userName}</div>;
};

// ✅ ДО РЕЧІ: Reselect для складних селекторів
import { createSelector } from '@reduxjs/toolkit';

const selectUserName = createSelector(
  (state) => state.user,
  (user) => user.name
);
```

### 3. Не закриття subscriptions (Zustand)

```jsx
// ❌ ПОГАНО: Store subscription витічає пам'ять
useEffect(() => {
  const unsubscribe = useAuthStore.subscribe(
    (state) => state.user,
    (user) => console.log(user)
  );
  // Забули unsubscribe!
}, []);

// ✅ ДОБРЕ: Очистити subscription
useEffect(() => {
  const unsubscribe = useAuthStore.subscribe(
    (state) => state.user,
    (user) => console.log(user)
  );
  return () => unsubscribe(); // Cleanup
}, []);
```

### 4. Race condition у React Query

```jsx
// ❌ ПОГАНО: Два паралельні запити, другий перезаписує перший
useEffect(() => {
  fetchUserData(userId); // Запит 1
}, [userId]);

useEffect(() {
  fetchUserData(userId); // Запит 2 (перезаписує 1!)
}, [userId]);

// ✅ ДОБРЕ: React Query обробляє це автоматично
const { data } = useQuery({
  queryKey: ['user', userId], // userId в ключі
  queryFn: () => fetchUserData(userId)
});
```

### 5. Зберігання функцій у Zustand

```jsx
// ❌ ПОГАНО: Функція оновлюється на кожен render
const useStore = create((set) => ({
  onDelete: (id) => console.log(id) // Нова функція!
}));

// ✅ ДОБРЕ: useCallback або вбудовується в store
const useStore = create((set) => ({
  deleteItem: (id) => {
    // Логіка видалення
  }
}));
```

---

## Порівняльна таблиця бібліотек

| Критерій | useState | Context | Redux (RTK) | Zustand | Jotai | React Query |
|---|---|---|---|---|---|---|
| **Розмір** | 0 | 0 | 3 KB | 2.2 KB | 4 KB | 12 KB |
| **Крива навчання** | Легка | Легка | Важка | Легка | Середня | Середня |
| **Дев-інструменти** | Ні | Ні | Чудові | Базові | Базові | Чудові |
| **Async операції** | useEffect | useEffect | Thunk/Saga | Вбудоване | Вбудоване | Основна фіча |
| **Server state** | Ні | Ні | Можна | Можна | Можна | Так |
| **TypeScript** | Відмінно | Хорошо | Відмінно | Відмінно | Відмінно | Відмінно |
| **Коли варто** | Локальний | Рідкі звичайні | Велика архітектура | Малі/середні | Атомарна | Server-first |
| **Масштабованість** | Низька | Низька | Висока | Висока | Висока | Висока |

