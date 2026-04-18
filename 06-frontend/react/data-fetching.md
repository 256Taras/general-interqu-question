# Data Fetching у React

## Якою різниця між fetch-on-render та render-as-you-fetch?

**Fetch-on-render** (нерекомендований) — компонент рендерується першим, потім у `useEffect` запускається запит. Це створює **waterfall**: рендер → бачимо пустий стан → запит стартує → чекаємо → отримуємо дані.

**Render-as-you-fetch** (рекомендований) — запит стартує ДО рендеру компонента. Коли компонент монтується, дані вже наступають або близькі до готовності. Це мінімізує loading-стан.

```jsx
// ❌ FETCH-ON-RENDER: waterfall
function UserProfile({ userId }) {
  const [user, setUser] = React.useState(null);
  const [loading, setLoading] = React.useState(true);

  React.useEffect(() => {
    // 1. Компонент монтується
    // 2. useEffect запускає запит (пізно!)
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]);

  if (loading) return <div>Завантажуємо...</div>;
  return <div>{user.name}</div>;
}

// ✅ RENDER-AS-YOU-FETCH: паралельно
const userResource = fetchUser(userId); // Запит стартує ОДРАЗУ, ще до рендеру

function UserProfile() {
  // Компонент читає проміс, який вже завантажується у фоні
  const user = userResource.read(); // Suspense обробляє стан очікування
  return <div>{user.name}</div>;
}

// Або з React Query (найпростіше)
function UserProfile({ userId }) {
  // Запит стартує автоматично при монтуванні чи зміні userId
  const { data: user, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
  });

  if (isLoading) return <div>Завантажуємо...</div>;
  return <div>{user.name}</div>;
}
```

**Waterfall проблема**: якщо для сторінки потрібно 3 асинхронних запити в ланцюжку (користувач → посади → деталі посади), то час завантаження = sum(всіх часів). Паралельно буде швидше: max(всіх часів).

---

## Які пастки useEffect для fetch?

### 1. Race Conditions

Якщо користувач змінить параметр (наприклад, `userId`) до завершення попереднього запиту, старий запит може перезаписати нові дані.

```jsx
// ❌ RACE CONDITION
function UserProfile({ userId }) {
  const [user, setUser] = React.useState(null);

  React.useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => setUser(data)); // Якщо userId змінився, дані можуть бути неправильні
  }, [userId]);

  return <div>{user?.name}</div>;
}

// ✅ CLEANUP / ABORT
function UserProfile({ userId }) {
  const [user, setUser] = React.useState(null);

  React.useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(r => r.json())
      .then(data => setUser(data));

    return () => controller.abort(); // Скасуємо запит при unmount чи зміні userId
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

### 2. Забуття cleanup

Якщо підписуємося на WebSocket або таймер, потрібно їх відпустити.

```jsx
// ❌ MEMORY LEAK
function RealtimeData() {
  const [data, setData] = React.useState(null);

  React.useEffect(() => {
    const ws = new WebSocket('ws://...');
    ws.onmessage = (e) => setData(e.data); // WebSocket НІКОЛИ не закриється!
  }, []);

  return <div>{data}</div>;
}

// ✅ CLEANUP
function RealtimeData() {
  const [data, setData] = React.useState(null);

  React.useEffect(() => {
    const ws = new WebSocket('ws://...');
    ws.onmessage = (e) => setData(e.data);

    return () => ws.close(); // Закриваємо при unmount
  }, []);

  return <div>{data}</div>;
}
```

### 3. Подвійний запит у Strict Mode

У development режимі React умисно монтує/демонтує компоненти двічі для виявлення проблем. Якщо не обробити cleanup, запит запуститься двічі.

```jsx
// React 18 Strict Mode демонтує/монтує, тому useEffect запустится дві рази
// Це НОРМАЛЬНО, але потрібно мати cleanup
React.useEffect(() => {
  // Цей код виконається 2 рази у development
  fetch('...').then(...)

  return () => {
    // Cleanup виконається 1 раз після другого монтування
  };
}, []);
```

---

## Що таке React Query (TanStack Query) і що він дає?

React Query — це бібліотека управління станом для асинхронних даних. Замість ручного `useState` + `useEffect`, ви описуєте запит один раз, і React Query обробляє cache, refetch, deduplication та многе інше.

### Ключові концепції

- **queryKey** — унікальний ідентифікатор для кешування. `['users', userId]` = разні кеші для кожного userId
- **staleTime** — скільки часу дані вважаються "свіжими" (за замовчуванням 0). Якщо свіжі, не roboti запит
- **cacheTime** (gcTime у v5) — скільки часу тримати неактивні дані в пам'яті (за замовчуванням 5 хв)
- **deduplication** — якщо 2 компоненти одночасно запросять те саме, запит запуститься один раз
- **stale-while-revalidate** — служити дані з кешу, одночасно оновлювати у фоні

```jsx
import { useQuery } from '@tanstack/react-query';

// Базовий паттерн
function UserProfile({ userId }) {
  const { 
    data: user, 
    isLoading, 
    error, 
    isFetching, // true коли відбувається запит (навіть якщо є кешовані дані)
  } = useQuery({
    queryKey: ['user', userId], // Унікальний ключ
    queryFn: async () => {
      const res = await fetch(`/api/users/${userId}`);
      if (!res.ok) throw new Error('Помилка завантаження');
      return res.json();
    },
    staleTime: 5 * 60 * 1000, // 5 хвилин дані вважаються свіжими
    gcTime: 10 * 60 * 1000, // 10 хвилин тримаємо в пам'яті
  });

  if (isLoading) return <div>Завантажуємо...</div>;
  if (error) return <div>Помилка: {error.message}</div>;

  return (
    <div>
      <div>{user.name}</div>
      {isFetching && <small>Оновлюємо у фоні...</small>}
    </div>
  );
}

// Deduplication: обидва компоненти запросять разом, запит запуститься 1 раз
<>
  <UserProfile userId={1} />
  <UserProfile userId={1} /> {/* Ж копіює дані з першого компонента */}
</>

// Stale-while-revalidate
function UserList() {
  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
    staleTime: 0, // ОДРАЗУ вважаємо застарілим
    // Результат: служимо СТАРІ дані (якщо є), одночасно запускаємо новий запит
  });

  return users.map(u => <div key={u.id}>{u.name}</div>);
}
```

---

## Що таке React Query mutations і як робити optimistic updates?

Mutation — це операція, яка змінює дані на сервері (POST, PUT, DELETE). React Query надає `useMutation` з автоматичною обробкою error, loading та cache invalidation.

```jsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function UpdateUserForm({ userId }) {
  const queryClient = useQueryClient();

  // Базовий mutation
  const { mutate, isPending, error } = useMutation({
    mutationFn: async (newData) => {
      const res = await fetch(`/api/users/${userId}`, {
        method: 'PUT',
        body: JSON.stringify(newData),
      });
      return res.json();
    },
    onSuccess: (updatedUser) => {
      // Після успіху, invalidate кеш щоб перезавантажити дані
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    },
    onError: (err) => {
      console.error('Помилка оновлення:', err);
    },
  });

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      mutate({ name: 'Нове ім\'я' });
    }}>
      <button disabled={isPending}>
        {isPending ? 'Збереження...' : 'Зберегти'}
      </button>
      {error && <div>{error.message}</div>}
    </form>
  );
}

// OPTIMISTIC UPDATE: покажемо нові дані ОДРАЗУ, навіть до сервера
function OptimisticUpdateForm({ userId, currentName }) {
  const queryClient = useQueryClient();

  const { mutate } = useMutation({
    mutationFn: async (newData) => {
      // Мережевий запит
      const res = await fetch(`/api/users/${userId}`, {
        method: 'PUT',
        body: JSON.stringify(newData),
      });
      return res.json();
    },
    onMutate: async (newData) => {
      // Скасуємо тимчасово усі fetch запити для цього ключа
      await queryClient.cancelQueries({ queryKey: ['user', userId] });

      // Зберігаємо СТАРІ дані для rollback у разі помилки
      const previousUser = queryClient.getQueryData(['user', userId]);

      // ОДРАЗУ оновлюємо UI (optimistic)
      queryClient.setQueryData(['user', userId], (old) => ({
        ...old,
        name: newData.name,
      }));

      return { previousUser }; // Повертаємо для onError
    },
    onError: (err, newData, context) => {
      // Якщо помилка, повертаємо СТАРІ дані
      if (context?.previousUser) {
        queryClient.setQueryData(['user', userId], context.previousUser);
      }
    },
    onSuccess: () => {
      // Якщо успіх, invalidate щоб отримати актуальні дані з сервера
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    },
  });

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      mutate({ name: 'Нове ім\'я' });
    }}>
      <button>Зберегти</button>
    </form>
  );
}
```

---

## Чим SWR відрізняється від React Query?

SWR (stale-while-revalidate) — легша альтернатива React Query, розроблена Vercel. Менше features, але достатня для більшості випадків.

| Характеристика | React Query | SWR |
|---|---|---|
| Розмір | ~13kb (gzipped) | ~3kb |
| Кеш | Складний, TTL, staleTime | Простий, браузер + пам'ять |
| Mutations | Вбудовані, зручні | Потрібно творити вручну |
| Deduplication | Так | Так |
| Middleware | Так | Не за замовчуванням |
| TypeScript | Відмінна підтримка | Хороша |
| Dev Tools | Включені | Окремий пакет |

```jsx
import useSWR from 'swr';

// SWR: простіше, але менше контролю
function UserProfile({ userId }) {
  const { data: user, error, isLoading } = useSWR(
    `/api/users/${userId}`,
    fetch // Передаємо fetcher функцію
  );

  if (isLoading) return <div>Завантажуємо...</div>;
  if (error) return <div>Помилка</div>;

  return <div>{user.name}</div>;
}

// За замовчуванням SWR автоматично revalidate, коли вікно отримує фокус
// Можна також встановити interval для periodic refetch
const { data } = useSWR(
  '/api/posts',
  fetcher,
  {
    refreshInterval: 5000, // Refetch кожні 5 сек
    focusThrottleInterval: 300000, // Revalidate коли window får фокус (кожні 5 хвилин макс)
  }
);

// SWR для mutations потрібно творити вручну
const { trigger: updateUser } = useSWRMutation(
  '/api/users/123',
  async (url, { arg }) => {
    const res = await fetch(url, {
      method: 'PUT',
      body: JSON.stringify(arg),
    });
    return res.json();
  }
);

// Викликаємо mutation
<button onClick={() => updateUser({ name: 'Нове ім\'я' })}>
  Оновити
</button>
```

**Коли вибрати:**
- **React Query**: складні додатки, багато запитів, потрібна оптимізація, advanced patterns
- **SWR**: simple додатки, Next.js, коли розмір критичний

---

## Як Suspense працює для data fetching?

Suspense дозволяє компоненту "призупинити" рендер і чекати на ресурс (дані, lazy-loaded компонент). Батьківський Suspense boundary показує fallback UI доки ресурс не готовий.

```jsx
// Suspense потребує ресурс, який має метод .read()
function createFetchResource(promise) {
  let status = 'pending';
  let data;
  let error;

  const suspender = promise
    .then(res => {
      status = 'success';
      data = res;
    })
    .catch(err => {
      status = 'error';
      error = err;
    });

  return {
    read() {
      if (status === 'pending') throw suspender; // Suspense ловить цей промис
      if (status === 'error') throw error;
      return data;
    }
  };
}

const userResource = createFetchResource(
  fetch('/api/users/1').then(r => r.json())
);

// Компонент читає дані, Suspense обробляє очікування
function UserProfile() {
  const user = userResource.read(); // Якщо дані не готові, throw promise
  return <div>{user.name}</div>;
}

// Suspense boundary перехоплює throw й показує fallback
function App() {
  return (
    <Suspense fallback={<div>Завантажуємо...</div>}>
      <UserProfile />
    </Suspense>
  );
}

// React 18+ підтримує Suspense для даних (поки експериментальна)
// React Query також має useQuerySuspense
import { useSuspenseQuery } from '@tanstack/react-query';

function UserProfileWithQuery() {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', 1],
    queryFn: () => fetch('/api/users/1').then(r => r.json()),
  });

  return <div>{user.name}</div>;
}

function App() {
  return (
    <Suspense fallback={<div>Завантажуємо...</div>}>
      <UserProfileWithQuery />
    </Suspense>
  );
}
```

**Переваги Suspense:**
- Один fallback для багатьох компонентів (не дублюємо loading-стан)
- Вищий рівень абстракції: логіка даних не мішається з UI
- Error Boundary природно доповнює Suspense

**Обмеження:**
- Поки для fetch потрібні бібліотеки (React Query, Next.js), не нативна
- Всі компоненти у Suspense чекають на найповільніший запит

---

## Як Server Components (React 19 / Next.js) змінюють data fetching?

Server Components дозволяють fetch дані прямо у компоненті БЕЗ `useEffect` чи `useState`. Компонент може бути `async`, дані завантажуються на сервері, тільки готовий HTML/JSON йде клієнту.

```jsx
// ✅ SERVER COMPONENT: дані завантажуються на сервері
// app/users/[id]/page.js (Next.js App Router)
async function UserPage({ params }) {
  // Запит виконується на сервері, у клієнті немає loading-стану!
  const user = await fetch(`https://api.example.com/users/${params.id}`)
    .then(r => r.json());

  // Можемо безпосередньо запросити БД, якщо на одному сервері
  // const user = await db.users.findById(params.id);

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// ❌ CLIENT COMPONENT: потрібен useEffect
'use client'; // Директива для клієнтського компонента

import { useEffect, useState } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]);

  if (loading) return <div>Завантажуємо...</div>;
  return <div>{user.name}</div>;
}

// КОМБІНОВАНИЙ ПАТТЕРН: Server + Client Component
// app/users/[id]/page.js
async function UserPage({ params }) {
  // Завантажуємо на сервері
  const user = await fetch(`/api/users/${params.id}`).then(r => r.json());

  return (
    <div>
      <UserHeader user={user} /> {/* Server Component */}
      <ClientSideComponent userId={params.id} /> {/* Client Component */}
    </div>
  );
}

// Переваги Server Components:
// 1. Дані завантажуються на сервері — у клієнта немає loading-стану
// 2. Немає N+1 проблеми (один запит, всі дані)
// 3. Акордеон токени, БД-паролі залишаються на сервері
// 4. Гарніша SEO (контент у HTML)
// 5. Менше JavaScript у клієнта (менше бандл)

export default UserPage;
```

---

## Next.js data patterns: getServerSideProps vs fetch у RSC

### Pages Router (старий) — getServerSideProps

```javascript
// pages/users/[id].js
export async function getServerSideProps({ params }) {
  // Виконується на сервері КОЖНОГО запиту
  const user = await fetch(`/api/users/${params.id}`).then(r => r.json());

  if (!user) {
    return { notFound: true }; // 404 сторінка
  }

  return {
    props: { user }, // Передаємо як props компоненту
    revalidate: 60, // ISR: перегенерувати сторінку через 60 сек
  };
}

function UserPage({ user }) {
  return <div>{user.name}</div>;
}
```

### App Router (нові) — fetch у Server Component

```jsx
// app/users/[id]/page.js
async function UserPage({ params }) {
  // Дані завантажуються на сервері
  const user = await fetch(
    `https://api.example.com/users/${params.id}`,
    {
      // Next.js автоматично кешує fetch результати
      cache: 'force-cache', // За замовчуванням кешуються як статичні
      // next: { revalidate: 60 }, // ISR альтернатива
    }
  ).then(r => r.json());

  return <div>{user.name}</div>;
}

export default UserPage;
```

### generateStaticParams для static generation

```jsx
// app/users/[id]/page.js
export async function generateStaticParams() {
  // На build-time генеруємо сторінки для цих користувачів
  const users = await fetch('/api/users').then(r => r.json());

  return users.map(user => ({
    id: user.id.toString(),
  }));
}

async function UserPage({ params }) {
  const user = await fetch(`/api/users/${params.id}`).then(r => r.json());
  return <div>{user.name}</div>;
}
```

---

## Як обробляти помилки: Error Boundaries vs try/catch?

**Error Boundary** — React компонент, який ловить помилки у дочірніх компонентах.

**try/catch** — обробка помилок усередину компонента для асинхронного коду.

```jsx
// ERROR BOUNDARY: ловить помилки у рендері дочірніх компонентів
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Логувати помилку у сервіс
    console.error('Помилка:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <div>Щось пішло не так: {this.state.error.message}</div>;
    }

    return this.props.children;
  }
}

// TRY/CATCH: обробка помилок у асинхронному коді
async function UserProfile({ userId }) {
  try {
    const user = await fetch(`/api/users/${userId}`).then(r => {
      if (!r.ok) throw new Error('Помилка завантаження');
      return r.json();
    });
    return <div>{user.name}</div>;
  } catch (error) {
    // Помилка у fetch/parsing не ловиться Error Boundary
    // Потрібно явно обробити
    return <div>Помилка: {error.message}</div>;
  }
}

// REACT QUERY: вбудована обробка помилок
function UserProfile({ userId }) {
  const { data: user, error, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: async () => {
      const res = await fetch(`/api/users/${userId}`);
      if (!res.ok) throw new Error('Помилка завантаження');
      return res.json();
    },
  });

  if (isLoading) return <div>Завантажуємо...</div>;
  if (error) return <div>Помилка: {error.message}</div>;

  return <div>{user.name}</div>;
}

// КОМБІНОВАНО: Error Boundary + React Query + try/catch
<ErrorBoundary>
  <Suspense fallback={<div>Завантажуємо...</div>}>
    <UserProfile userId={1} />
  </Suspense>
</ErrorBoundary>
```

---

## Як обробляти loading states: skeleton, spinner, optimistic UI?

```jsx
// 1. SKELETON: показуємо макет без даних
function UserProfileSkeleton() {
  return (
    <div className="skeleton-card">
      <div className="skeleton-avatar"></div>
      <div className="skeleton-text"></div>
      <div className="skeleton-text short"></div>
    </div>
  );
}

function UserProfile({ userId }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: fetchUser,
  });

  if (isLoading) return <UserProfileSkeleton />;
  return <div>{user.name}</div>;
}

// 2. SPINNER: простіший loading UI
function UserProfile({ userId }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: fetchUser,
  });

  if (isLoading) return <Spinner />;
  return <div>{user.name}</div>;
}

// 3. OPTIMISTIC UI: показуємо нові дані одразу
function UpdateNameForm({ userId, currentName }) {
  const queryClient = useQueryClient();
  const [inputValue, setInputValue] = React.useState(currentName);

  const { mutate } = useMutation({
    mutationFn: (newName) =>
      fetch(`/api/users/${userId}`, {
        method: 'PUT',
        body: JSON.stringify({ name: newName }),
      }).then(r => r.json()),

    onMutate: (newName) => {
      setInputValue(newName); // Одразу оновлюємо input
      queryClient.setQueryData(['user', userId], (old) => ({
        ...old,
        name: newName,
      }));
    },

    onError: (err, newName, context) => {
      // Якщо помилка, поверта попереднє значення
      setInputValue(currentName);
    },
  });

  return (
    <input
      value={inputValue}
      onChange={(e) => setInputValue(e.target.value)}
      onBlur={() => mutate(inputValue)}
    />
  );
}

// 4. STALE-WHILE-REVALIDATE: служимо СТАРІ дані, оновлюємо у фоні
function UserProfile({ userId }) {
  const { data: user, isFetching } = useQuery({
    queryKey: ['user', userId],
    queryFn: fetchUser,
    staleTime: 0, // ОДРАЗУ застарілі
  });

  return (
    <div>
      <div>{user?.name}</div>
      {isFetching && <small>Оновлюємо...</small>}
    </div>
  );
}
```

---

## Як реалізувати infinite scrolling з React Query?

```jsx
import { useInfiniteQuery } from '@tanstack/react-query';
import { useInView } from 'react-intersection-observer';

function InfinitePostList() {
  const { ref, inView } = useInView(); // Контролюємо, коли елемент у viewport

  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: async ({ pageParam = 1 }) => {
      // pageParam = номер сторінки, передаємо з initialPageParam
      const res = await fetch(`/api/posts?page=${pageParam}`);
      return res.json();
    },
    initialPageParam: 1, // З якої сторінки почати
    getNextPageParam: (lastPage, allPages) => {
      // Визначаємо, чи є наступна сторінка
      if (lastPage.posts.length === 0) return undefined;
      return allPages.length + 1; // Номер наступної сторінки
    },
  });

  // Коли останній елемент у viewport, завантажуємо наступну сторінку
  React.useEffect(() => {
    if (inView && hasNextPage) {
      fetchNextPage();
    }
  }, [inView, hasNextPage, fetchNextPage]);

  return (
    <div>
      {data?.pages.map((page) =>
        page.posts.map((post) => (
          <div key={post.id}>{post.title}</div>
        ))
      )}

      {/* Спеціальний ref-елемент у кінці списку */}
      <div ref={ref}>
        {isFetchingNextPage ? 'Завантажуємо...' : null}
      </div>

      {!hasNextPage && <div>Конец списку</div>}
    </div>
  );
}
```

---

## Real-time data: WebSocket, SSE, polling — що вибрати?

| Технологія | Спосіб | Latency | Браузер | Сервер | Коли використовувати |
|---|---|---|---|---|---|
| **Polling** | Клієнт запитує кожні N сек | 0-N сек | Простий | Простий | Низька частота оновлень |
| **Long Polling** | Клієнт чекає відповідь на сервері | 0-сек | Простий | Складніший | Рідко, legacy |
| **Server-Sent Events (SSE)** | Сервер відправляє в одну сторону | ~100мс | Вбудований, з fallback | Простий | Push-повідомлення, live updates |
| **WebSocket** | Двостороння сокет з'єднання | ~10мс | Вбудований | Складніший | Chat, collaborative editing, high-freq |

```jsx
// 1. POLLING: просто, але неефективно
function RealtimePosts() {
  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    refetchInterval: 5000, // Refetch кожні 5 сек
  });

  return posts?.map(p => <div key={p.id}>{p.title}</div>);
}

// 2. SERVER-SENT EVENTS (SSE): push від сервера
function RealtimePosts() {
  const [posts, setPosts] = React.useState([]);

  React.useEffect(() => {
    const eventSource = new EventSource('/api/posts/stream');

    eventSource.onmessage = (e) => {
      const newPost = JSON.parse(e.data);
      setPosts(prev => [newPost, ...prev]); // Нова пост у топ
    };

    eventSource.onerror = () => eventSource.close();

    return () => eventSource.close();
  }, []);

  return posts.map(p => <div key={p.id}>{p.title}</div>);
}

// 3. WEBSOCKET: двостороння комунікація
function RealtimeChat() {
  const [messages, setMessages] = React.useState([]);

  React.useEffect(() => {
    const ws = new WebSocket('wss://api.example.com/chat');

    ws.onopen = () => {
      ws.send(JSON.stringify({ type: 'SUBSCRIBE', channel: 'main' }));
    };

    ws.onmessage = (e) => {
      const message = JSON.parse(e.data);
      setMessages(prev => [...prev, message]);
    };

    ws.onerror = (err) => console.error('WS помилка:', err);

    return () => {
      if (ws.readyState === WebSocket.OPEN) ws.close();
    };
  }, []);

  const sendMessage = (text) => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'MESSAGE', text }));
    }
  };

  return (
    <div>
      {messages.map((m, i) => <div key={i}>{m.text}</div>)}
      <input
        onKeyPress={(e) => {
          if (e.key === 'Enter') sendMessage(e.target.value);
        }}
      />
    </div>
  );
}

// Рекомендація:
// - Polling: прогноз погоди, курсовий план, рідко оновлювані дані
// - SSE: натискання, live-feed, eventi лог
// - WebSocket: chat, collaborative docs, interactive games
```

---

## Типові пастки

1. **Waterfall запитів** — запити запускаються послідовно замість паралельно. Рішення: `render-as-you-fetch`, React Query, або Suspense.

2. **Race conditions** — старий запит перезаписує нові дані. Рішення: `AbortController`, cleanup у useEffect, React Query.

3. **Memory leaks** — забуті cleanup функції в useEffect. Рішення: завжди закривати WebSocket, таймери, слухачі подій.

4. **Множинні fetches однакових даних** — кожен компонент робить свій запит. Рішення: React Query deduplication, SWR, або глобальний стейт.

5. **N+1 проблема** — цикл запитів де кожна пост потребує окремий запит для автора. Рішення: batch API, Server Components, або JOIN у SQL.

6. **Забуті скасування** — fetch у useEffect без cleanup до AbortController. Рішення: `const controller = new AbortController()` та `return () => controller.abort()`.

7. **Старі дані у UI** — користувач бачить застарілі інформацію. Рішення: `staleTime`, manual invalidation, чи polling.

8. **Loading spinner на всю сторінку** — замість skeleton для окремих частин. Рішення: Suspense з меншими boundaries, skeleton UI за частинами.

9. **Не обробити помилки fetch** — спробувати парсити null response. Рішення: проверяти `response.ok`, throw Error, обробляти в Error Boundary чи try/catch.

10. **Infinite loop у useEffect** — забули додати dependency array. Рішення: `useEffect(..., [])` з усіма залежностями.

11. **Optimize UI під час fetch** — заморозити UI під час запиту. Рішення: optimistic updates, стейт синхронізації, чи isFetching flag.

12. **Не кешувати дані** — кожна навігація перезавантажує. Рішення: React Query, SWR, або Next.js fetch caching.
