# Vue.js — Питання для інтерв'ю

## Reactive system — як працює reactivity у Vue 3?

Vue 3 використовує **Proxy** (замість Object.defineProperty у Vue 2) для відстеження змін.

**Як працює:**
1. Коли ти створюєш `reactive()` або `ref()`, Vue обгортає об'єкт у Proxy
2. При **читанні** властивості — Vue запам'ятовує, який ефект (компонент) залежить від цієї властивості (**track**)
3. При **записі** — Vue сповіщує всі залежні ефекти (**trigger**)

```
reactive({ count: 0 })
         │
         ▼
    Proxy handler
    ├── get(target, key) → track(target, key) → повертає значення
    └── set(target, key, value) → trigger(target, key) → оновлює залежності
```

**Переваги Proxy над defineProperty (Vue 2):**
- Відстежує додавання/видалення властивостей (не потрібен `Vue.set()`)
- Відстежує зміни в масивах (push, splice, індекси)
- Менше overhead — не потрібно рекурсивно обходити всі властивості при ініціалізації

---

## Options API vs Composition API

### Options API (класичний)
```vue
<script>
export default {
  data() {
    return { count: 0, name: "" };
  },
  computed: {
    doubled() { return this.count * 2; }
  },
  methods: {
    increment() { this.count++; }
  },
  mounted() {
    console.log("mounted");
  }
};
</script>
```

### Composition API (Vue 3)
```vue
<script setup>
import { ref, computed, onMounted } from "vue";

const count = ref(0);
const name = ref("");
const doubled = computed(() => count.value * 2);

function increment() {
  count.value++;
}

onMounted(() => {
  console.log("mounted");
});
</script>
```

**Переваги Composition API:**
- Логіка групується за **функціональністю**, а не за типом (data/methods/computed)
- Легше перевикористовувати логіку через **composables** (аналог custom hooks)
- Краща підтримка TypeScript
- Менший bundle size (tree-shakable)

**Коли Options API ще доречний:**
- Прості компоненти
- Команда, яка звикла до Vue 2
- Options API все ще повністю підтримується

---

## ref() vs reactive() — різниця

### ref()
```js
const count = ref(0);
console.log(count.value); // 0 — потрібен .value
count.value++;

const user = ref({ name: "John" });
user.value.name = "Jane"; // .value для доступу до об'єкта
```
- Працює з **будь-яким типом** (примітиви, об'єкти, масиви)
- Потребує `.value` у JavaScript (в template — автоматично розгортається)
- Зберігає **реактивне посилання** — можна перезаписати повністю

### reactive()
```js
const state = reactive({ count: 0, name: "John" });
console.log(state.count); // 0 — без .value
state.count++;
```
- Працює тільки з **об'єктами** (не примітиви)
- Не потребує `.value`
- **Не можна** перезаписати повністю (`state = newState` — втрата реактивності)
- **Не можна** деструктуризувати без `toRefs()`

**Рекомендація:** використовуй `ref()` за замовчуванням — він універсальніший і передбачуваніший.

---

## Computed vs Watch vs WatchEffect

### computed
Обчислюване значення, яке **кешується** і перераховується тільки при зміні залежностей:
```js
const fullName = computed(() => `${firstName.value} ${lastName.value}`);
```
- Кешований (на відміну від методів)
- Тільки для **синхронних** обчислень
- Має бути **без побічних ефектів** (pure)

### watch
Спостерігає за конкретними джерелами і виконує callback при зміні:
```js
watch(userId, async (newId, oldId) => {
  const user = await fetchUser(newId);
  userData.value = user;
}, { immediate: true });

// Кілька джерел
watch([firstName, lastName], ([newFirst, newLast]) => {
  console.log(`${newFirst} ${newLast}`);
});
```
- Доступ до **старого і нового** значення
- Підтримує async операції
- `immediate: true` — виконати одразу
- `deep: true` — глибоке спостереження об'єктів

### watchEffect
Автоматично відстежує **всі** реактивні залежності всередині callback:
```js
watchEffect(() => {
  console.log(`User: ${user.value.name}, Count: ${count.value}`);
  // Автоматично спостерігає за user та count
});
```
- Не потрібно вказувати джерела явно
- Виконується **одразу** (на відміну від watch)
- Немає доступу до старого значення

---

## Vue Lifecycle Hooks

```
setup()                      ← Composition API entry point
  │
  ▼
onBeforeMount()              ← перед монтуванням у DOM
  │
  ▼
onMounted()                  ← компонент у DOM (доступ до $el)
  │
  ▼
onBeforeUpdate()             ← перед оновленням DOM
  │
  ▼
onUpdated()                  ← DOM оновлено
  │
  ▼
onBeforeUnmount()            ← перед видаленням
  │
  ▼
onUnmounted()                ← компонент видалено (cleanup)
```

**Додаткові:**
- `onActivated()` / `onDeactivated()` — для `<KeepAlive>`
- `onErrorCaptured()` — аналог Error Boundary

**Коли що використовувати:**
- `onMounted` — API запити, доступ до DOM, ініціалізація бібліотек
- `onUnmounted` — cleanup підписок, таймерів
- `onUpdated` — рідко, лише коли потрібно працювати з DOM після оновлення

---

## Provide / Inject

Аналог React Context — передача даних через дерево компонентів без prop drilling:

```vue
<!-- Батьківський компонент -->
<script setup>
import { provide, ref } from "vue";

const theme = ref("dark");
const toggleTheme = () => {
  theme.value = theme.value === "dark" ? "light" : "dark";
};

provide("theme", { theme, toggleTheme });
</script>

<!-- Дочірній компонент (будь-яка глибина) -->
<script setup>
import { inject } from "vue";

const { theme, toggleTheme } = inject("theme");
</script>
```

**Best practices:**
- Використовуй Symbol як ключ для уникнення конфліктів
- Надавай readonly значення якщо дочірні не повинні змінювати
- Для TypeScript створюй типізовані injection keys

```ts
// keys.ts
import type { InjectionKey, Ref } from "vue";
export const ThemeKey: InjectionKey<Ref<string>> = Symbol("theme");
```

---

## Teleport та Suspense

### Teleport
Рендерить вміст в інше місце DOM (поза поточним компонентом):
```vue
<template>
  <div>
    <h1>Мій компонент</h1>
    <Teleport to="body">
      <div class="modal">Це модальне вікно рендериться у body</div>
    </Teleport>
  </div>
</template>
```
**Використання:** модальні вікна, тултіпи, нотифікації — все, що має бути поза основним layout.

### Suspense (експериментальний)
```vue
<template>
  <Suspense>
    <template #default>
      <AsyncComponent />
    </template>
    <template #fallback>
      <LoadingSpinner />
    </template>
  </Suspense>
</template>
```
Працює з async setup() або async компонентами.

---

## Vue Router — Navigation Guards

```js
// Глобальний guard
router.beforeEach((to, from) => {
  if (to.meta.requiresAuth && !isAuthenticated()) {
    return { name: "Login" };
  }
});

// Per-route guard
const routes = [
  {
    path: "/admin",
    component: Admin,
    beforeEnter: (to, from) => {
      if (!isAdmin()) return { name: "Home" };
    }
  }
];

// In-component guard (Composition API)
import { onBeforeRouteLeave } from "vue-router";

onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges.value) {
    return confirm("Є незбережені зміни. Покинути сторінку?");
  }
});
```

**Порядок виконання guards:**
1. `beforeRouteLeave` (компонент що покидається)
2. `beforeEach` (глобальний)
3. `beforeEnter` (route)
4. `beforeRouteEnter` (компонент що завантажується)
5. `afterEach` (глобальний)

---

## Pinia vs Vuex

### Pinia (рекомендований для Vue 3)
```js
import { defineStore } from "pinia";

export const useUserStore = defineStore("user", () => {
  const user = ref(null);
  const isLoggedIn = computed(() => !!user.value);

  async function login(credentials) {
    user.value = await api.login(credentials);
  }

  return { user, isLoggedIn, login };
});

// Використання в компоненті
const userStore = useUserStore();
userStore.login({ email, password });
```

### Vuex (legacy)
```js
const store = createStore({
  state: { user: null },
  mutations: { SET_USER(state, user) { state.user = user; } },
  actions: {
    async login({ commit }, credentials) {
      const user = await api.login(credentials);
      commit("SET_USER", user);
    }
  },
  getters: { isLoggedIn: state => !!state.user }
});
```

**Чому Pinia краще:**
- Немає mutations — менше boilerplate
- Повна підтримка TypeScript
- Composition API стиль
- DevTools інтеграція
- Менший bundle size
- Модульна архітектура за замовчуванням

---

## v-model під капотом

`v-model` — це синтаксичний цукор для двостороннього зв'язування:

```vue
<!-- Це: -->
<input v-model="search" />

<!-- Еквівалентно: -->
<input :value="search" @input="search = $event.target.value" />
```

**На кастомному компоненті (Vue 3):**
```vue
<!-- Батьківський -->
<MyInput v-model="name" />

<!-- Еквівалентно: -->
<MyInput :modelValue="name" @update:modelValue="name = $event" />

<!-- Дочірній -->
<script setup>
const props = defineProps(["modelValue"]);
const emit = defineEmits(["update:modelValue"]);
</script>

<template>
  <input :value="modelValue" @input="emit('update:modelValue', $event.target.value)" />
</template>
```

**Кілька v-model (Vue 3):**
```vue
<UserForm v-model:firstName="first" v-model:lastName="last" />
```

---

## Slots (default, named, scoped)

### Default slot
```vue
<!-- Батьківський -->
<Card>
  <p>Цей контент піде у slot</p>
</Card>

<!-- Card.vue -->
<template>
  <div class="card">
    <slot>Fallback контент</slot>
  </div>
</template>
```

### Named slots
```vue
<!-- Батьківський -->
<Layout>
  <template #header>
    <h1>Заголовок</h1>
  </template>
  <template #default>
    <p>Основний контент</p>
  </template>
  <template #footer>
    <p>Футер</p>
  </template>
</Layout>
```

### Scoped slots
Дочірній компонент передає дані у slot:
```vue
<!-- List.vue -->
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <slot :item="item" :index="index">{{ item.name }}</slot>
    </li>
  </ul>
</template>

<!-- Використання -->
<List :items="users">
  <template #default="{ item }">
    <strong>{{ item.name }}</strong> — {{ item.email }}
  </template>
</List>
```

**Scoped slots** — потужний патерн для створення flexible, reusable компонентів (таблиці, списки, форми).

---

## TypeScript інтеграція з Vue 3

```vue
<script setup lang="ts">
import { ref, computed } from "vue";

interface User {
  id: number;
  name: string;
  email: string;
}

const user = ref<User | null>(null);
const userName = computed(() => user.value?.name ?? "Guest");

// Props з TypeScript
const props = defineProps<{
  title: string;
  count?: number;
  items: User[];
}>();

// Props з default values
const props = withDefaults(defineProps<{
  title: string;
  count?: number;
}>(), {
  count: 0,
});

// Emits
const emit = defineEmits<{
  (e: "update", id: number): void;
  (e: "delete", id: number): void;
}>();

// Expose
defineExpose({
  reset: () => { user.value = null; }
});
</script>
```

**Best practices:**
- Завжди використовуй `lang="ts"` у `<script setup>`
- Типізуй props через generics `defineProps<T>()`
- Типізуй emits через generics `defineEmits<T>()`
- Використовуй `Ref<T>` для типізації ref

---

## Script setup синтаксис

`<script setup>` — компіляторна оптимізація для Composition API:

```vue
<script setup>
// Все що оголошено тут — автоматично доступне у template
import { ref } from "vue";
import MyComponent from "./MyComponent.vue";

const count = ref(0);
function increment() { count.value++; }
</script>

<template>
  <MyComponent />
  <button @click="increment">{{ count }}</button>
</template>
```

**Переваги:**
- Менше boilerplate (не потрібен `export default`, `setup()`, `return`)
- Кращий TypeScript inference
- Кращий runtime performance (компілюється в setup function)
- Імпортовані компоненти автоматично реєструються

**defineProps, defineEmits, defineExpose, defineOptions** — компіляторні макроси, не потребують import.

---

## Як оптимізувати Vue додаток?

1. **v-once** — рендерити елемент лише один раз (статичний контент)
2. **v-memo** — мемоізація частини template
3. **Computed properties** замість методів для кешованих обчислень
4. **KeepAlive** — кешувати компоненти при навігації
5. **Lazy loading** routes і компонентів
6. **Virtual scrolling** для великих списків (vue-virtual-scroller)
7. **shallowRef / shallowReactive** — уникнути глибокої реактивності для великих об'єктів
8. **Уникати** непотрібних watchers — надавати перевагу computed
9. **Tree shaking** — імпортувати лише потрібне з бібліотек
10. **SSR/SSG** з Nuxt.js для кращого first load

---

## Custom Directives

```js
// Глобальна директива
app.directive("focus", {
  mounted(el) {
    el.focus();
  }
});

// Використання
// <input v-focus />

// Директива з параметрами
app.directive("tooltip", {
  mounted(el, binding) {
    el.title = binding.value;
  },
  updated(el, binding) {
    el.title = binding.value;
  }
});

// <span v-tooltip="'Підказка'">Наведи</span>
```

**Lifecycle hooks директив:** created, beforeMount, mounted, beforeUpdate, updated, beforeUnmount, unmounted.

**Коли використовувати:** низькорівневий доступ до DOM, який не варто виносити у компонент (focus, click outside, intersection observer).

---

## Nuxt.js — SSR фреймворк для Vue

**Основні концепції:**
- **File-based routing** — файли у `pages/` автоматично стають маршрутами
- **Auto-imports** — компоненти, composables, утиліти
- **SEO** — вбудована підтримка meta tags
- **Data fetching** — `useFetch`, `useAsyncData`

**Rendering modes:**
- **SSR** — server-side rendering (за замовчуванням)
- **SSG** — `nuxt generate` для статичних сайтів
- **SPA** — `ssr: false`
- **Hybrid** — різні режими для різних маршрутів

```vue
<!-- pages/users/[id].vue -->
<script setup>
const route = useRoute();
const { data: user } = await useFetch(`/api/users/${route.params.id}`);
</script>

<template>
  <div>{{ user.name }}</div>
</template>
```
