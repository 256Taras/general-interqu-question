# Pinia та Vue Router

## Що таке Pinia й чому замість Vuex?

Pinia -- це сучасний state manager для Vue.js, який замінив Vuex як офіційне рішення. Основні відмінності:

1. **Простота API** -- Pinia дозволяє писати магазини з Option API або Composition API, без боргото 'mutations'. У Vuex обов'язково розділення на state, mutations, actions.
2. **Реактивність по замовчуванню** -- Vuex вимагав оборотів з getters та commits, Pinia базується на Vue 3 Composition API і реактивна за природою.
3. **DevTools інтеграція** -- краща: відслідковування дій, часові подорожі, миттєві знімки стану.
4. **TypeScript підтримка** -- Pinia має вивід типів автоматично без додаткових обертань.
5. **Модульність** -- кожна store -- це власний файл, без концепції 'modules' з вкладенням простору імен.

```javascript
// Vuex (старий підхід)
const store = createStore({
  state: () => ({ count: 0 }),
  mutations: { increment(state) { state.count++ } },
  actions: { increment({ commit }) { commit('increment') } }
});

// Pinia (новий)
const useCountStore = defineStore('count', {
  state: () => ({ count: 0 }),
  actions: {
    increment() { this.count++ } // без commit!
  }
});
```

**Когда использовать:** Для нових Vue 3 проектів — завжди Pinia. Для легаці на Vue 2 з Composition API polyfill — можна Pinia. Для старого Vue 2 з Vue 2 Composition API — Vuex.

---

## Store у Pinia: state, getters, actions (Options style)

Store у Pinia визначається через `defineStore()`. Найпоширеніший підхід -- **Options API style**:

```javascript
// stores/todoStore.js
import { defineStore } from 'pinia';

export const useTodoStore = defineStore('todos', {
  // Реактивний стан
  state: () => ({
    todos: [],
    filter: 'all', // all | active | completed
    loading: false,
    error: null
  }),

  // Обчислювальні властивості (похідні)
  getters: {
    // Прості getters (як обчислювальні властивості)
    activeTodos(state) {
      return state.todos.filter(t => !t.completed);
    },
    
    // Getters що повертають функцію (параметризовані)
    getTodoById: (state) => (id) => {
      return state.todos.find(todo => todo.id === id);
    },
    
    // Getter який викликає інший getter
    completedCount(state) {
      return state.todos.length - this.activeTodos.length; // цей this буде store!
    },
    
    // Доступ до інших store через inject
    withUserInfo(state) {
      const userStore = useUserStore(); // підключення інишої store
      return state.todos.map(todo => ({
        ...todo,
        author: userStore.users[todo.userId]
      }));
    }
  },

  // Методи (синхронні та асинхронні операції)
  actions: {
    // Синхронна дія
    addTodo(title) {
      this.todos.push({
        id: Date.now(),
        title,
        completed: false,
        createdAt: new Date()
      });
    },

    // Асинхронна дія (може мати сайд-ефекти)
    async fetchTodos() {
      this.loading = true;
      this.error = null;
      try {
        const response = await fetch('/api/todos');
        this.todos = await response.json();
      } catch (err) {
        this.error = err.message;
      } finally {
        this.loading = false;
      }
    },

    // Дія що викликає іншу дію
    async toggleTodo(id) {
      const todo = this.getTodoById(id); // викликаємо getter
      if (!todo) return;
      
      todo.completed = !todo.completed;
      // Виклик інший дії поточної store
      await this.updateTodoOnServer(id, todo);
    },

    async updateTodoOnServer(id, todo) {
      try {
        const response = await fetch(`/api/todos/${id}`, {
          method: 'PATCH',
          body: JSON.stringify(todo)
        });
        if (!response.ok) throw new Error('Update failed');
      } catch (err) {
        this.error = err.message;
        // Можемо откатить зміни
        const original = await this.fetchTodos();
      }
    },

    // Дія що змінює стан іншої store
    clearAndReset() {
      this.todos = [];
      this.filter = 'all';
      // Очищення в інший store
      const userStore = useUserStore();
      userStore.clearUserData();
    }
  }
});
```

---

## Store у Pinia: Setup style (Composition API)

Альтернативний підхід -- **Setup style**, який дозволяє писати store як функцію, подібну до `setup()` компонента:

```javascript
// stores/counterStore.js (Setup style)
import { defineStore } from 'pinia';
import { ref, computed, watch } from 'vue';

export const useCounterStore = defineStore('counter', () => {
  // Стан (ref або reactive)
  const count = ref(0);
  const history = ref([]);
  const name = ref('Counter');

  // Getters (computed)
  const doubled = computed(() => count.value * 2);
  const isEven = computed(() => count.value % 2 === 0);

  // Actions (обиклі функції)
  function increment() {
    count.value++;
    history.value.push(count.value);
  }

  function decrement() {
    count.value--;
    history.value.push(count.value);
  }

  function reset() {
    count.value = 0;
    history.value = [];
  }

  function setName(newName) {
    name.value = newName;
  }

  // можемо мати watch, але в store його рідко використовують
  // краще підписатись в компонентах

  // Повертаємо лише те, що мають бути public
  return {
    count,
    doubled,
    isEven,
    history,
    name,
    increment,
    decrement,
    reset,
    setName
  };
});
```

**Порівняння:**
- **Options** -- знайомо розробникам з Vue 2, чітка структура
- **Setup** -- більше гнучкості, легше переносити логику, ближче до функціональних компонентів

---

## Підписка на зміни через watch та $subscribe

Важливо слідкувати за змінами в store для побічних ефектів (логування, синхронізація з API, оновлення UI):

```javascript
// У компоненті (script setup)
<script setup>
import { watch } from 'vue';
import { useTodoStore } from '@/stores/todoStore';

const todoStore = useTodoStore();

// 1. watch через computed getter
watch(
  () => todoStore.activeTodos.length,
  (newCount, oldCount) => {
    console.log(`Активних TODO: ${newCount} (було ${oldCount})`);
    // API запит, аналітика, тощо
  }
);

// 2. watch на кілька реактивних змін
watch(
  () => [todoStore.filter, todoStore.todos.length],
  ([filter, len], [oldFilter, oldLen]) => {
    console.log(`Фільтр: ${filter}, Всього: ${len}`);
  }
);

// 3. Глибокий watch на весь стан (обережно, дорого!)
watch(
  () => todoStore.$state,
  (newState) => {
    console.log('Весь стан змінився:', newState);
  },
  { deep: true } // дорого для великих об'єктів
);
</script>
```

```javascript
// 4. $subscribe -- специфічний для Pinia спосіб слідкування
const todoStore = useTodoStore();

// Підписуємось на будь-які зміни
const unsubscribe = todoStore.$subscribe(
  (mutation, state) => {
    // mutation.type: 'direct' | 'patch object' | 'patch function'
    console.log(`Тип змін: ${mutation.type}`, mutation.payload);
    console.log('Новий стан:', state);
  },
  { detached: true } // detached: працює навіть після unmount компонента
);

// Скасування підписки
unsubscribe();
```

```javascript
// 5. $subscribe з фільтруванням (підписка на окремі поля)
todoStore.$subscribe(
  (mutation, state) => {
    if (mutation.type === 'patch object') {
      console.log('Оновилися поля:', Object.keys(mutation.payload));
    }
  },
  { 
    // Викликати лише під час змін у компоненті (не detached)
    detached: false
  }
);
```

---

## Plugins у Pinia -- persist, logging

Plugins дозволяють розширити функціонал всіх store в один момент:

```javascript
// plugins/persistPlugin.js
export function createPersistPlugin() {
  return (context) => {
    const { store } = context;

    // store.$subscribe викликається при кожній зміні
    store.$subscribe((mutation, state) => {
      // Збереження у localStorage при кожній зміні
      const storeName = store.$id; // 'todos', 'counter', тощо
      localStorage.setItem(
        `pinia_${storeName}`,
        JSON.stringify(state)
      );
    });

    // При ініціалізації store -- завантаження з localStorage
    const saved = localStorage.getItem(`pinia_${store.$id}`);
    if (saved) {
      store.$patch(JSON.parse(saved));
    }
  };
}
```

```javascript
// plugins/loggingPlugin.js
export function createLoggingPlugin() {
  return (context) => {
    const { store } = context;

    store.$subscribe((mutation, state) => {
      console.log(
        `[${store.$id}] ${mutation.type}`,
        mutation.payload,
        state
      );
    });

    // Логування виклику дій
    store.$onAction(({ name, store, args, after, onError }) => {
      console.log(`Action ${name} called with:`, args);
      
      after((result) => {
        console.log(`Action ${name} resolved with:`, result);
      });

      onError((error) => {
        console.error(`Action ${name} failed:`, error);
      });
    });
  };
}
```

```javascript
// main.js
import { createPinia } from 'pinia';
import { createPersistPlugin } from '@/plugins/persistPlugin';
import { createLoggingPlugin } from '@/plugins/loggingPlugin';

const pinia = createPinia();

// Реєструємо plugins
pinia.use(createPersistPlugin());
pinia.use(createLoggingPlugin());

app.use(pinia);
```

---

## Використання store у компонентах

```vue
<script setup>
import { useTodoStore } from '@/stores/todoStore';

const todoStore = useTodoStore();

// ❌ ПАСТКА: Деструктуризація втрачає реактивність
// const { count, increment } = todoStore; // count буде НЕ реактивним!

// ✅ РІШЕННЯ 1: Доступ через store
const handleClick = () => {
  todoStore.increment(); // через store
  console.log(todoStore.count); // реактивна!
};

// ✅ РІШЕННЯ 2: storeToRefs для деструктуризації
import { storeToRefs } from 'pinia';
const { count, filter } = storeToRefs(todoStore);
const { increment, fetchTodos } = todoStore; // методи без storeToRefs!
// Тепер count та filter реактивні в шаблоні
</script>

<template>
  <div>
    <!-- Прямий доступ через store -->
    <p>Count: {{ todoStore.count }}</p>
    <button @click="todoStore.increment">+1</button>

    <!-- Або через deconstructured refs -->
    <p>Filter: {{ filter }}</p>

    <!-- Асинхронна дія -->
    <button @click="fetchTodos" :disabled="todoStore.loading">
      {{ todoStore.loading ? 'Loading...' : 'Fetch' }}
    </button>

    <!-- Обчислювальні властивості (getters) -->
    <p>Completed: {{ todoStore.completedCount }}</p>
  </div>
</template>
```

```javascript
// Деструктуризація разом зі стежанням
import { storeToRefs } from 'pinia';
import { watch } from 'vue';

const todoStore = useTodoStore();
const { todos, filter } = storeToRefs(todoStore);

// Тепер можемо використовувати watch на деструктуровані refs
watch(filter, (newFilter) => {
  console.log('Filter changed to:', newFilter);
});
```

---

## Testing stores

Тестування store у Pinia є простішим, ніж у Vuex:

```javascript
// __tests__/todoStore.spec.js
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { createPinia, setActivePinia } from 'pinia';
import { useTodoStore } from '@/stores/todoStore';

describe('TodoStore', () => {
  beforeEach(() => {
    // Кожний тест має свій экземпляр pinia
    setActivePinia(createPinia());
  });

  it('додає todo', () => {
    const store = useTodoStore();
    store.addTodo('Test todo');
    
    expect(store.todos).toHaveLength(1);
    expect(store.todos[0].title).toBe('Test todo');
  });

  it('активні todos повертаються через getter', () => {
    const store = useTodoStore();
    store.todos = [
      { id: 1, completed: false, title: 'Active' },
      { id: 2, completed: true, title: 'Done' }
    ];
    
    expect(store.activeTodos).toHaveLength(1);
    expect(store.activeTodos[0].id).toBe(1);
  });

  it('асинхронна дія fetchTodos', async () => {
    // Мокуємо fetch
    global.fetch = vi.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve([
          { id: 1, title: 'Todo 1' }
        ])
      })
    );

    const store = useTodoStore();
    expect(store.loading).toBe(false);
    
    await store.fetchTodos();
    
    expect(store.loading).toBe(false);
    expect(store.todos).toHaveLength(1);
    expect(global.fetch).toHaveBeenCalledWith('/api/todos');
  });

  it('обробляє помилки при fetch', async () => {
    global.fetch = vi.fn(() =>
      Promise.reject(new Error('Network error'))
    );

    const store = useTodoStore();
    await store.fetchTodos();
    
    expect(store.error).toBe('Network error');
    expect(store.loading).toBe(false);
  });

  it('$subscribe викликається при змінах', () => {
    const store = useTodoStore();
    const unsubscribe = vi.fn();
    
    store.$subscribe((mutation, state) => {
      expect(mutation.type).toBe('patch object');
      expect(state.todos.length).toBe(1);
    });

    store.addTodo('Test');
  });
});
```

---

## Vuex (legacy) -- розуміння старого коду

Vuex -- офіційний state manager для Vue до появи Pinia. Структура:

```javascript
// store.js (Vuex)
const store = new Vuex.Store({
  state: {
    count: 0,
    todos: []
  },

  // mutations -- синхронні зміни STATE
  mutations: {
    INCREMENT(state) {
      state.count++;
    },
    SET_TODOS(state, todos) {
      state.todos = todos;
    }
  },

  // actions -- асинхронні операції, комітять mutations
  actions: {
    async fetchTodos({ commit }) {
      const todos = await fetch('/api/todos').then(r => r.json());
      commit('SET_TODOS', todos); // мусимо викликати mutation!
    },
    
    incrementAsync({ commit }) {
      setTimeout(() => {
        commit('INCREMENT'); // обов'язково commit
      }, 1000);
    }
  },

  // getters -- обчислювальні властивості
  getters: {
    completedTodos: (state) => state.todos.filter(t => t.completed),
    todoCount: (state, getters) => getters.completedTodos.length
  },

  // modules -- глобальні namespace
  modules: {
    user: {
      namespaced: true,
      state: { name: 'John' },
      mutations: { SET_NAME(state, name) { state.name = name } },
      actions: {
        setName({ commit }, name) {
          commit('SET_NAME', name);
        }
      }
    }
  }
});

// У компоненті:
// this.$store.state.count  -- прямий доступ
// this.$store.commit('INCREMENT')  -- синхронна зміна
// this.$store.dispatch('incrementAsync')  -- асинхронна дія
// this.$store.getters.completedTodos  -- обчислювальна властивість
```

**Головна відмінність:** Vuex вимагає явного розділення mutations (синхронно) та actions (асинхронно), Pinia дозволяє змінювати стан напрямую в actions.

---

## Vue Router: setup, dynamic routes, nested routes

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router';
import Home from '@/pages/Home.vue';
import About from '@/pages/About.vue';

const routes = [
  {
    path: '/',
    component: Home,
    name: 'home', // назва для router.push({ name: 'home' })
    meta: { title: 'Home' }
  },

  // Динамічні маршрути (параметри)
  {
    path: '/user/:id',
    component: UserDetail,
    name: 'user-detail',
    // :id -- параметр, може змінюватись
    // Доступ: route.params.id
  },

  // Багаторівневі параметри
  {
    path: '/posts/:postId/comments/:commentId',
    component: CommentDetail,
    name: 'comment'
    // Доступ: route.params.postId, route.params.commentId
  },

  // Опційні параметри (0 або 1 раз)
  {
    path: '/files/:pathMatch(.*)?',
    name: 'NotFound',
    component: NotFound
    // :pathMatch(.*)?  -- catch-all для невідомих маршрутів
  },

  // Вкладені маршрути (nested routes)
  {
    path: '/dashboard',
    component: DashboardLayout,
    children: [
      {
        path: '', // /dashboard
        component: DashboardHome,
        name: 'dashboard-home'
      },
      {
        path: 'settings', // /dashboard/settings
        component: DashboardSettings,
        name: 'dashboard-settings'
      },
      {
        path: 'profile', // /dashboard/profile
        component: DashboardProfile,
        name: 'dashboard-profile'
      }
    ]
  },

  // Іменовані маршрути з компонентами
  {
    path: '/layout',
    component: LayoutWithSidebar,
    children: [
      {
        path: 'list',
        components: {
          default: ItemList,
          sidebar: ItemSidebar // названий view
        },
        name: 'item-list'
      }
    ]
  }
];

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  // Поведінка при прокручуванні
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      return savedPosition; // Повернути на місце при назад/вперед
    }
    return { left: 0, top: 0 }; // Інакше вверх сторінки
  }
});

export default router;
```

```javascript
// main.js
import { createApp } from 'vue';
import App from './App.vue';
import router from './router';

const app = createApp(App);
app.use(router);
app.mount('#app');
```

```vue
<!-- App.vue -->
<template>
  <nav>
    <!-- router-link -- навігаційні посилання -->
    <router-link to="/">Home</router-link>
    <router-link to="/about">About</router-link>
    <router-link :to="{ name: 'user-detail', params: { id: 123 } }">
      User 123
    </router-link>
  </nav>

  <!-- router-view -- місце для відображення компонента маршруту -->
  <router-view></router-view>
</template>
```

---

## Navigation guards: beforeEach, beforeEnter, in-component

```javascript
// Глобальні guards
router.beforeEach((to, from, next) => {
  // Виконується ДО переходу до будь-якого маршруту
  console.log(`Переходимо з ${from.path} на ${to.path}`);
  
  // Перевірка автентифікації
  const isAuthenticated = !!localStorage.getItem('token');
  if (to.meta.requiresAuth && !isAuthenticated) {
    next({ name: 'login' }); // Перенаправити на логін
    return;
  }

  // Встановлення заголовка сторінки
  document.title = to.meta.title || 'App';

  next(); // Дозволити перехід
  // Або next(false)  -- скасувати перехід
  // Або next('/other') -- перенаправити на інший маршрут
});

router.beforeResolve((to, from, next) => {
  // Виконується ПІСЛЯ beforeEach, але ДО преходу
  // Корисно для завантаження даних перед переходом
  next();
});

router.afterEach((to, from) => {
  // Виконується ПІСЛЯ переходу (без можливості скасування)
  console.log('Перехід завершено');
  // Логування, аналітика, тощо
});

// Guard для специфічного маршруту
const routes = [
  {
    path: '/admin',
    component: AdminPanel,
    beforeEnter: (to, from, next) => {
      // Guard лише для цього маршруту
      const isAdmin = !!localStorage.getItem('admin_token');
      if (!isAdmin) {
        next('/'); // Перенаправити
      } else {
        next();
      }
    }
  }
];

// In-component guards (у Vue компоненті)
// script setup не підтримує, используємо обычний <script>
export default {
  beforeRouteEnter(to, from, next) {
    // Виконується ДО входу в компонент
    // У цьому місці немає доступу до this
    console.log('Перед входом у компонент');
    next(); // ВАЖЛИВО викликати next()!
  },

  beforeRouteUpdate(to, from, next) {
    // Виконується, коли компонент уже монтований, але маршрут міняється
    // Наприклад, /user/1 -> /user/2 (один компонент, різні параметри)
    console.log(`Оновлюємо параметри з ${from.params.id} на ${to.params.id}`);
    next();
  },

  beforeRouteLeave(to, from, next) {
    // Виконується ДО виходу з компонента
    if (this.formDirty) {
      const answer = window.confirm('Ви впевнені? Зміни будуть втрачені.');
      if (answer) {
        next();
      } else {
        next(false); // Скасувати перехід
      }
    } else {
      next();
    }
  }
};
```

```vue
<!-- За допомогою Composition API -->
<script setup>
import { onBeforeRouteUpdate, onBeforeRouteLeave } from 'vue-router';

onBeforeRouteUpdate((to, from) => {
  console.log(`Параметр змінився: ${from.params.id} -> ${to.params.id}`);
});

onBeforeRouteLeave((to, from) => {
  if (formDirty.value) {
    return confirm('Скасувати зміни?');
  }
});
</script>
```

---

## Route params vs query -- коли що

```javascript
// PARAMS (:id) -- частина маршруту
// /user/123
// Використання: ідентифікатори, обов'язкові дані
{
  path: '/user/:id',
  component: UserDetail
}

// У компоненті:
const route = useRoute();
const userId = route.params.id; // '123'

// Навігація:
router.push(`/user/123`);
// Або
router.push({ name: 'user', params: { id: 123 } });

// ===================================

// QUERY (?key=value) -- рядок запиту
// /search?q=vue&page=2
// Використання: фільтри, сортування, пагінація
{
  path: '/search',
  component: SearchResults
}

// У компоненті:
const route = useRoute();
const query = route.query.q; // 'vue'
const page = route.query.page; // '2' (завжди рядок!)

// Навігація:
router.push(`/search?q=vue&page=2`);
// Або (краще):
router.push({
  name: 'search',
  query: { q: 'vue', page: '2' }
});

// ===================================

// ПРИКЛАД: Комбінація params + query
// /products/electronics?sort=price&order=asc
{
  path: '/products/:category',
  component: ProductList
}

const route = useRoute();
const category = route.params.category; // 'electronics'
const sort = route.query.sort; // 'price'
const order = route.query.order; // 'asc'
```

---

## Programmatic navigation: router.push, router.replace

```javascript
import { useRouter } from 'vue-router';

const router = useRouter();

// router.push -- додає запис у history (можна назад)
router.push('/');
router.push('/user/123');
router.push({ name: 'user-detail', params: { id: 123 } });
router.push({
  path: '/search',
  query: { q: 'vue', page: 2 }
});

// router.replace -- НЕ додає у history (не можна назад)
// Корисно для переспрямування або заміни поточного маршруту
router.replace('/login');
router.replace({
  name: 'home'
});

// router.back() / router.forward() / router.go(n)
router.back(); // Як кнопка "назад"
router.forward(); // Як кнопка "вперед"
router.go(-1); // На 1 крок назад
router.go(2); // На 2 кроки вперед

// Перевірка можливості навігації
if (router.options.history.state.position !== -1) {
  router.back(); // Є історія, можемо назад
}
```

---

## History mode vs Hash mode

```javascript
// Hash mode (за замовчуванням у Vite)
// URL: http://localhost:5173/#/user/123
// Плюси: працює без серверної конфігурації
// Мінуси: # виглядає не чисто, проблеми з SEO

import { createRouter, createWebHashHistory } from 'vue-router';

const router = createRouter({
  history: createWebHashHistory(import.meta.env.BASE_URL),
  routes
});

// ===================================

// HTML5 History mode (рекомендовано)
// URL: http://localhost:5173/user/123
// Плюси: чисті URL, краще для SEO
// Мінуси: вимагає серверної конфігурації (fallback на index.html)

import { createRouter, createWebHistory } from 'vue-router';

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes
});

// Серверна конфігурація (необхідна для history mode):

// Vite server (vite.config.js)
export default {
  server: {
    middlewareMode: true,
    // Всі невідомі маршрути -> index.html
  }
};

// Express server
app.use(express.static('dist'));
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'dist', 'index.html'));
});

// Nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

---

## Scroll behavior та route transitions

```javascript
// Управління прокручуванням при змені маршруту
const router = createRouter({
  history: createWebHistory(),
  routes,
  scrollBehavior(to, from, savedPosition) {
    // savedPosition -- при клік "назад/вперед"
    if (savedPosition) {
      return savedPosition;
    }
    
    // Якщо маршрут має якір (#section)
    if (to.hash) {
      return {
        el: to.hash,
        behavior: 'smooth'
      };
    }

    // Інакше -- вверх сторінки
    return { left: 0, top: 0 };
  }
});
```

```vue
<!-- Анімація переходу між маршрутами -->
<template>
  <Transition name="fade" mode="out-in">
    <router-view :key="route.fullPath"></router-view>
  </Transition>
</template>

<script setup>
import { useRoute } from 'vue-router';
const route = useRoute();
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

---

## Типові пастки

**1. Деструктуризація store без storeToRefs**
```javascript
const { count, increment } = useTodoStore(); // count НЕ реактивний!
// Рішення: const { count } = storeToRefs(useTodoStore());
```

**2. Забування next() у guards**
```javascript
router.beforeEach((to, from, next) => {
  console.log('Проверка...');
  // ❌ Забули next() -- навігація зависне!
});
```

**3. Race condition при асинхронній навігації**
```javascript
// Запит 1 інціює, потім Запит 2 -- Запит 2 завершується першим!
async fetchData() {
  const data1 = await api.get(this.id);
  // this.id змінився, але ми продовжуємо...
}
// Рішення: Перевірити, чи поточний id ще релевантний
```

**4. Прямий доступ до state у getters без this**
```javascript
// ❌ Невірно
getters: {
  doubled: (state) => state.count * 2; // Забув return!
}
// ✅ Правильно
getters: {
  doubled: (state) => state.count * 2,
  tripled(state) { return state.count * 3; }
}
```

**5. Додавання нових властивостей до state без оголошення**
```javascript
// У компоненті:
store.newField = 'value'; // НЕ реактивний в Pinia!
// Рішення: store.$patch({ newField: 'value' });
```

**6. Параметри query завжди рядки**
```javascript
// URL: /search?page=2
const page = parseInt(route.query.page); // НЕ забути конвертувати!
```

**7. Nested routes без <router-view/> у parent**
```vue
<!-- ❌ Дочірні маршрути не матимуть місця для виведення -->
<template>
  <div>Parent</div>
</template>

<!-- ✅ Додати router-view у parent -->
<template>
  <div>
    <div>Parent</div>
    <router-view></router-view>
  </div>
</template>
```

**8. beforeRouteUpdate не викликається при зміні query**
```javascript
// URL: /user/1?tab=profile -> /user/1?tab=settings
// beforeRouteUpdate НЕ викликається! Параметри не змінилися.
// Рішення: watch на route.query у компоненті
watch(() => route.query, (newQuery) => {
  handleQueryChange(newQuery);
});
```
