# Vue 3 Composition API

## Options API vs Composition API — коли яке використовувати?

У Vue 3 можна писати компоненти двома способами: Options API (традиційний) і Composition API (новий, рекомендований для нових проектів).

**Options API** — старий стиль, дані в `data`, методи в `methods`, хуки в `mounted` тощо:

```javascript
export default {
  data() {
    return { count: 0 };
  },
  methods: {
    increment() { this.count++; }
  },
  mounted() {
    console.log("Готово");
  }
};
```

**Composition API** — функціональний підхід, близький до React Hooks. Вся логіка в `setup()`:

```javascript
import { ref } from 'vue';

export default {
  setup() {
    const count = ref(0);
    const increment = () => count.value++;
    
    onMounted(() => console.log("Готово"));
    
    return { count, increment };
  }
};
```

**Коли використовувати:**

- **Options API**: маленькі компоненти, простий state, команда незна́йома з Composition API, або якщо Options API розробиться як в проекті
- **Composition API**: складна логіка, reusable composables (аналог custom hooks), більші компоненти, новий код

**Переваги Composition API:**
1. Логіка пов'язана з однією фічею знаходиться в одному місці (не розпорошена по `data`, `methods`, `computed`)
2. Легко витягти логіку в custom composable для переиспользования
3. Більш гнучкий, ближче до функціонального стилю, як у React

**Різниця від React Hooks:**
- Vue не має ейбіль dependency array як `useEffect([dep])` — натомість є `watch` з явною залежністю
- У Vue не треба запам'ятовувати порядок Hooks, вони можуть бути в будь-якому порядку

---

## setup() — як працює, життєвий цикл

`setup()` — функція, що викликається **перед** виконанням рушія (до `created`). Це місце, де ти ініціалізуєш весь state й логіку Composition API.

```javascript
import { ref, onMounted, onUnmounted } from 'vue';

export default {
  setup() {
    const message = ref("Привіт");
    
    // setup() викликається ДО mounted, created і т.д.
    // Тому this тут недоступний — немає ще екземпляра компоненту
    console.log("setup() виконується ПЕРШИМ");
    
    onMounted(() => {
      console.log("onMounted (2-й)");
      console.log(message.value); // "Привіт" — реактивно працює
    });
    
    onUnmounted(() => {
      console.log("Компонент знищується");
    });
    
    // ОБОВ'ЯЗКОВО повернути объект з усім, що потрібно в template
    return { message };
  }
};
```

**Життєвий цикл Composition API:**

```
Порядок виконання:
1. setup() ← функція запускається ПЕРШОЮ
2. Потім Options API хуки (beforeCreate, created не викликаються з Composition API)
3. onBeforeMount
4. onMounted ← DOM готовий
5. onBeforeUpdate (перед оновленням через реактивність)
6. onUpdated ← DOM оновлений
7. onBeforeUnmount
8. onUnmounted ← компонент видалений
```

**Важливо:** у Composition API немає `beforeCreate` і `created` — замість них використовується синхронний код у `setup()`.

```javascript
setup() {
  // Це виконується на місці beforeCreate/created
  const store = useMyStore();
  
  onMounted(() => {
    // Це виконується на місці mounted
  });
}
```

---

## script setup syntax — спрощена версія, що дає, приклади

`<script setup>` — це синтаксичний цукор, який дозволяє писати Composition API мовою, близькою до простого кода. Змінні та функції автоматично експортуються в template, нема потреби у `return {}`.

**Без `<script setup>`:**

```vue
<script>
import { ref } from 'vue';
import MyButton from './MyButton.vue';

export default {
  components: { MyButton },
  setup() {
    const count = ref(0);
    const title = "Лічильник";
    const increment = () => count.value++;
    
    return { count, title, increment, MyButton };
  }
};
</script>

<template>
  <div>
    <h1>{{ title }}</h1>
    <p>Count: {{ count }}</p>
    <MyButton @click="increment" />
  </div>
</template>
```

**З `<script setup>` (сучасний, рекомендований способ):**

```vue
<script setup>
import { ref } from 'vue';
import MyButton from './MyButton.vue';

const count = ref(0);
const title = "Лічильник";
const increment = () => count.value++;
// Все автоматично доступно в template
</script>

<template>
  <div>
    <h1>{{ title }}</h1>
    <p>Count: {{ count }}</p>
    <MyButton @click="increment" />
  </div>
</template>
```

**Що дає `<script setup>`:**

1. Коротший и зрозуміліший код
2. Змінні автоматично експортуються (нема потреби у `return`)
3. Прямолінійна реактивність без огорнення в хуки
4. Props и emits декларуються просто через `defineProps` й `defineEmits`

```vue
<script setup>
// Props
const props = defineProps({
  msg: String,
  count: Number
});

// Emits
const emit = defineEmits(['update']);

const handleClick = () => {
  emit('update', props.count + 1);
};
</script>

<template>
  <button @click="handleClick">{{ msg }}: {{ count }}</button>
</template>
```

---

## ref() — що це (reactive primitive), .value, auto-unwrap у template

`ref()` — це функція для створення **реактивної примітивної величини** (число, рядок, булев). Вона обертає значення в об'єкт з властивістю `.value`, щоб Vue відслідковував зміни.

```javascript
import { ref } from 'vue';

const count = ref(0);
const name = ref("Іван");
const isActive = ref(false);

// Щоб прочитати або змінити значення, треба звернутися до .value
count.value++; // Тепер 1
console.log(name.value); // "Іван"
isActive.value = true;
```

**Чому `.value`?** Vue потребує це огорнення, щоб відслідковувати доступ до значення через Proxy. Без цього JavaScript не матиме способу дізнатися, що значення змінилось.

**Auto-unwrap у template:**

У шаблоні (`<template>`) Vue автоматично розгортує `.value`, тому ти можеш писати просто `{{ count }}`:

```vue
<script setup>
import { ref } from 'vue';

const count = ref(42);
const message = ref("Привіт");
</script>

<template>
  <!-- Auto-unwrap: Vue робить це автоматично -->
  <p>{{ count }}</p> <!-- Показує 42, а не {{ count.value }} -->
  <p>{{ message }}</p> <!-- Показує "Привіт" -->
  
  <!-- В JavaScript коді ПОТРІБЕН .value -->
  <button @click="() => count.value++">Інкремент</button>
</template>
```

**Порівняння з React:**

React:
```javascript
const [count, setCount] = useState(0);
// Читаємо: count
// Змінюємо: setCount(count + 1)
```

Vue:
```javascript
const count = ref(0);
// Читаємо: count.value
// Змінюємо: count.value++
// (у template: {{ count }} — auto-unwrap)
```

---

## reactive() — для об'єктів. Різниця з ref

`reactive()` — це функція для створення **реактивного об'єкта**. На відміну від `ref`, при роботі з об'єктами не потрібен `.value`:

```javascript
import { reactive } from 'vue';

const state = reactive({
  user: { name: 'Іван', age: 30 },
  todos: [],
  settings: { theme: 'dark' }
});

// Прямий доступ, БЕЗ .value
state.user.name = 'Петро';
state.todos.push({ id: 1, title: 'Задача' });
state.settings.theme = 'light';

console.log(state.user.name); // "Петро"
```

**`reactive()` у template:**

```vue
<script setup>
import { reactive } from 'vue';

const state = reactive({
  count: 0,
  message: 'Привіт'
});

const increment = () => state.count++;
</script>

<template>
  <div>
    <p>{{ state.count }}</p> <!-- No .value needed -->
    <p>{{ state.message }}</p>
    <button @click="increment">+1</button>
  </div>
</template>
```

**Різниця `ref()` vs `reactive()`:**

| | `ref()` | `reactive()` |
|---|---|---|
| Тип даних | Примітиви й об'єкти | Тільки об'єкти |
| Доступ до значення | `.value` (в JS коді) | Прямий, без `.value` |
| Деструктуризація | Втрачає реактивність (див. toRefs) | Втрачає реактивність |
| Переприсвоєння | `ref` можна переприсвоїти: `count = ref(5)` | Неможна переприсвоїти |
| Коли використовувати | Примітиви (числа, рядки), окремі властивості | Цілі об'єкти стану |

**Коли яке використовувати:**

```javascript
// Примітив → ref()
const count = ref(0);
const name = ref("Іван");

// Об'єкт → reactive()
const user = reactive({
  name: "Іван",
  age: 30,
  address: { city: 'Киї' }
});

// Змішано: ref() для окремих властивостей, reactive() для групування
const formData = reactive({
  email: ref("test@mail.com"), // ref можна вкладати в reactive
  password: ref("")
});
```

---

## computed — замість React useMemo, декларативний

`computed()` — це реактивна властивість, що автоматично перераховується при зміні залежностей. Це аналог React `useMemo`, але декларативніший.

```javascript
import { ref, computed } from 'vue';

const firstName = ref('Іван');
const lastName = ref('Петренко');

// computed автоматично отримує залежності з замикання
const fullName = computed(() => {
  console.log("Перераховую fullName"); // Виконується тільки коли firstName або lastName змінилися
  return `${firstName.value} ${lastName.value}`;
});

firstName.value = 'Петро'; // Тригер перерахунку
console.log(fullName.value); // "Петро Петренко", логи показав один раз
```

**Коли re-compute?** Тільки коли змінилася залежність (firstName або lastName).

**Писаний computed (setter):**

```javascript
const firstName = ref('Іван');
const lastName = ref('Петренко');

const fullName = computed({
  get() {
    return `${firstName.value} ${lastName.value}`;
  },
  set(newValue) {
    // Розбиваємо "Петро Петренко" на parts
    const [first, last] = newValue.split(' ');
    firstName.value = first;
    lastName.value = last;
  }
});

// Читання
console.log(fullName.value); // "Іван Петренко"

// Запис через setter
fullName.value = 'Петро Котенко';
console.log(firstName.value); // "Петро"
console.log(lastName.value); // "Котенко"
```

**Порівняння з React useMemo:**

React:
```javascript
const fullName = useMemo(() => {
  return `${firstName} ${lastName}`;
}, [firstName, lastName]); // Явна залежність
```

Vue:
```javascript
const fullName = computed(() => {
  return `${firstName.value} ${lastName.value}`;
  // Залежності отримуються автоматично із замикання
});
```

Vue тут простіший — не треба писати масив залежностей.

---

## watch vs watchEffect — коли яке, приклади

`watch()` й `watchEffect()` — це механізм спостереження за змінами реактивного стану і виконання side effects (як `useEffect` у React, але розділений на два способи).

**`watch()` — явна залежність, як React useEffect з dependencies:**

```javascript
import { ref, watch } from 'vue';

const count = ref(0);
const message = ref('');

// watch(залежність, callback)
watch(count, (newVal, oldVal) => {
  console.log(`count змінився з ${oldVal} на ${newVal}`);
  message.value = `Нове значення: ${newVal}`;
});

count.value = 5; // Логи: "count змінився з 0 на 5"
```

**`watch()` кілька залежностей:**

```javascript
const firstName = ref('Іван');
const lastName = ref('Петренко');

// Спостерігаємо за масивом залежностей
watch([firstName, lastName], ([first, last]) => {
  console.log(`Ім'я змінилось: ${first} ${last}`);
});

firstName.value = 'Петро'; // Тригер
```

**`watchEffect()` — автоматична залежність (краще для простих випадків):**

```javascript
import { ref, watchEffect } from 'vue';

const count = ref(0);
const multiplier = ref(2);

// watchEffect відстежує все, що викликається всередину callback
watchEffect(() => {
  // Автоматично залежить від count і multiplier
  console.log(`Result: ${count.value * multiplier.value}`);
});

count.value = 5; // Логи: "Result: 10"
multiplier.value = 3; // Логи: "Result: 15"
```

**Коли яке використовувати:**

```javascript
// watch() — коли потрібна явна залежність
const searchQuery = ref('');
watch(searchQuery, (query) => {
  fetchResults(query); // Явно спостерігаємо за searchQuery
});

// watchEffect() — коли залежність складна або багато змінних
const user = reactive({ name: 'Іван', age: 30 });
const query = ref('');

watchEffect(() => {
  // Автоматично залежить від user.name, user.age, query
  console.log(`${user.name}, ${user.age}, пошук: ${query.value}`);
});
```

**Важливо: `immediate` опція:**

```javascript
watch(count, (newVal) => {
  console.log(newVal);
}, { immediate: true }); // Виконається одразу, не чекаючи зміни
```

---

## watchEffect cleanup — як зробити (аналог useEffect cleanup)

У React:

```javascript
useEffect(() => {
  const subscription = subscribe();
  return () => subscription.unsubscribe(); // Cleanup
}, []);
```

У Vue, `watchEffect` повертає функцію cleanup:

```javascript
import { watchEffect } from 'vue';

watchEffect((onCleanup) => {
  const subscription = subscribe();
  
  // Передай cleanup функцію в onCleanup
  onCleanup(() => {
    subscription.unsubscribe();
  });
});
```

**Практичний приклад — слухання подій:**

```javascript
import { watchEffect } from 'vue';

const inputRef = ref(null);

watchEffect((onCleanup) => {
  const el = inputRef.value;
  if (!el) return;
  
  const handleClick = () => console.log('Клік на input');
  
  el.addEventListener('click', handleClick);
  
  // Cleanup: видалити слухача при знищенні або перезапуску
  onCleanup(() => {
    el.removeEventListener('click', handleClick);
  });
});
```

**Порівняння:** Vue тут більш явний — нема сховання cleanup в return, вона передається явно.

---

## Custom composables (Vue-версія custom hooks) — useDebounce, useFetch приклади

**Composable** — це функція, що повертає реактивний стан і методи. Це аналог React custom hook.

**Приклад: useFetch (асинхронна загрузка):**

```javascript
// composables/useFetch.js
import { ref, reactive, watchEffect } from 'vue';

export function useFetch(url) {
  const data = ref(null);
  const error = ref(null);
  const loading = ref(false);

  watchEffect((onCleanup) => {
    loading.value = true;
    error.value = null;

    let cancelled = false;

    fetch(url)
      .then(res => res.json())
      .then(json => {
        if (!cancelled) {
          data.value = json;
        }
      })
      .catch(err => {
        if (!cancelled) {
          error.value = err;
        }
      })
      .finally(() => {
        if (!cancelled) loading.value = false;
      });

    // Cleanup: анулювати запит, якщо URL змінився
    onCleanup(() => {
      cancelled = true;
    });
  });

  return { data, error, loading };
}
```

**Використання:**

```vue
<script setup>
import { useFetch } from './composables/useFetch';

const { data, error, loading } = useFetch('/api/user');
</script>

<template>
  <div v-if="loading">Загружаємо...</div>
  <div v-else-if="error">Помилка: {{ error }}</div>
  <div v-else>{{ data }}</div>
</template>
```

**Приклад: useDebounce (затримка введення):**

```javascript
// composables/useDebounce.js
import { ref, watch } from 'vue';

export function useDebounce(value, delay = 500) {
  const debouncedValue = ref(value.value);

  const timer = ref(null);

  watch(value, (newVal) => {
    // Скасувати попередній таймер
    clearTimeout(timer.value);
    
    // Встановити новий
    timer.value = setTimeout(() => {
      debouncedValue.value = newVal;
    }, delay);
  });

  return debouncedValue;
}
```

**Використання:**

```vue
<script setup>
import { ref } from 'vue';
import { useDebounce } from './composables/useDebounce';

const searchQuery = ref('');
const debouncedQuery = useDebounce(searchQuery, 300);

// Спостерігаємо за debounced версією для API запита
import { watch } from 'vue';
watch(debouncedQuery, (query) => {
  if (query) {
    fetchResults(query);
  }
});
</script>

<template>
  <input v-model="searchQuery" placeholder="Пошук..." />
  <p>Пошукуємо: {{ debouncedQuery }}</p>
</template>
```

**Різниця від React hooks:**
- У Vue composables тво можна дивляти на реактивне значення напряму (через `ref.value`)
- Немає потреби у dependency array, як у React

---

## toRef / toRefs — коли треба, як уникнути втрати реактивності при деструктуризації

При деструктуризації реактивного об'єкта втрачається реактивність:

```javascript
import { reactive } from 'vue';

const user = reactive({
  name: 'Іван',
  age: 30
});

// ✗ НЕПРАВИЛЬНО: деструктуризація втрачає реактивність
const { name, age } = user;
name = 'Петро'; // Не тригер оновлення компоненту
console.log(user.name); // Все ще 'Іван'
```

**Рішення: `toRefs()`**

```javascript
import { reactive, toRefs } from 'vue';

const user = reactive({
  name: 'Іван',
  age: 30
});

// ✓ ПРАВИЛЬНО: toRefs конвертує властивості в ref'и
const { name, age } = toRefs(user);

name.value = 'Петро'; // Реактивно!
console.log(user.name); // 'Петро'
```

**Коли використовувати `toRef()` (на одну властивість):**

```javascript
import { reactive, toRef } from 'vue';

const state = reactive({
  count: 0,
  message: 'Привіт'
});

// Якщо тобі потрібна тільки одна властивість як ref
const count = toRef(state, 'count');

count.value++; // Реактивно оновлює state.count
console.log(state.count); // 1
```

**Типовий use case у composables:**

```javascript
// composables/useUser.js
import { reactive, toRefs } from 'vue';

export function useUser(userId) {
  const user = reactive({
    id: userId,
    name: '',
    email: '',
    loading: false
  });

  // Завантажити дані...

  // Повернути як refs, щоб у компоненті можна було деструктурувати
  return toRefs(user);
}
```

**Використання:**

```vue
<script setup>
import { useUser } from './composables/useUser';

const { name, email, loading } = useUser(123); // Реактивно!
</script>

<template>
  <div v-if="loading">Загружаємо...</div>
  <div v-else>
    <p>{{ name }}</p>
    <p>{{ email }}</p>
  </div>
</template>
```

---

## shallowRef / shallowReactive — коли використовувати (велика структура)

`shallowRef()` і `shallowReactive()` — це оптимізаційні версії для великих структур даних, де треба відстежувати тільки верхній рівень.

**Проблема:** якщо в `ref()` величезний об'єкт (мільйони вузлів дерева, великі масиви), Vue затратить много ресурсів на Proxy обгортання.

```javascript
import { ref } from 'vue';

// ✗ ПОВІЛЬНО: Vue обгортає весь об'єкт у Proxy
const largeObject = ref({
  data: [[...10000 елементів...], [...10000 елементів...]]
});

// Доступ до вкладеного масиву змінює ref повністю
largeObject.value.data[0].push(1); // Не тригер reaktivity
largeObject.value = { ...largeObject.value }; // Потрібна переприсвоєння
```

**Рішення: `shallowRef()`**

```javascript
import { shallowRef } from 'vue';

const largeObject = shallowRef({
  data: [[...10000 елементів...], [...10000 елементів...]]
});

// Змінювати тільки `.value` для оновлення
largeObject.value.data[0].push(1); // Не помічає зміну
largeObject.value = { ...largeObject.value }; // Тригер оновлення

// Або використовувати triggerRef для явного оновлення
import { triggerRef } from 'vue';
largeObject.value.data[0].push(1);
triggerRef(largeObject); // Явно скажемо Vue обновити
```

**`shallowReactive()` — аналог для об'єктів:**

```javascript
import { shallowReactive } from 'vue';

const state = shallowReactive({
  nested: {
    deeply: {
      value: 'Привіт' // Тільки верхній рівень реактивний
    }
  }
});

state.nested.deeply.value = 'Привіт!'; // НЕ тригер оновлення
state.nested = { deeply: { value: 'Новий' } }; // Тригер оновлення
```

**Коли використовувати:**
- Величезні структури даних (мільйони вузлів)
- Зовнішні бібліотеки, що мають свої об'єкти (D3.js, велике дерево в памяті)
- Коли треба лише окремі зміни на верхньому рівні

---

## Порівняння з React hooks — коли Vue робить те саме простіше, коли важче

| Фича | React | Vue | Простіше в Vue? |
|---|---|---|---|
| State примітив | `const [x, setX] = useState(0)` | `const x = ref(0)` | Складніше (`.value`) |
| State об'єкт | `const [obj, setObj] = useState({...})` | `const obj = reactive({...})` | Простіше |
| Computed value | `const val = useMemo(() => x + y, [x, y])` | `const val = computed(() => x.value + y.value)` | Простіше (автозалежності) |
| Side effect | `useEffect(() => {...}, [deps])` | `watchEffect(() => {...})` | Простіше (автозалежності) |
| Явні залежності | Потрібно писати `[x, y]` | `watch(x, ...)` або явно передай | Одно-одно |
| Cleanup | `return () => {...}` | `onCleanup(() => {...})` | Одно-одно |
| Custom hook | `function useX() { ... return [...] }` | `function useX() { ... return {...} }` | Простіше (об'єкт, не масив) |
| Деструктуризація | Втрачає реактивність | Втрачає реактивність, але `toRefs()` простіше | Одно-одно |
| Closure проблеми | Часті (забув `deps`) | Рідкі (залежності явні) | Простіше в Vue |

**Де Vue простіше:**
1. **computed() vs useMemo** — не треба писати dependencies, вони отримуються з замикання
2. **reactive() для об'єктів** — прямий доступ без setter функцій, як у React
3. **watchEffect()** — автоматична залежність, менше багів забутих deps
4. **composables vs custom hooks** — повертаємо об'єкт, а не масив, тому не хвилюємось про порядок

**Де Vue складніше:**
1. **ref() з `.value`** — додатковий boilerplate при роботі з примітивами
2. **Auto-unwrap тільки в template** — в JavaScript коді потрібен `.value`, це спантеличує
3. **Заповнення return {}** — раніше потрібно було явно повертати з `setup()` (тепер з `<script setup>` це не потрібно)

**Практичний приклад: лічильник з fetch-ом**

React:
```javascript
const [count, setCount] = useState(0);
const [data, setData] = useState(null);
const [loading, setLoading] = useState(false);

useEffect(() => {
  setLoading(true);
  fetch(`/api/count/${count}`)
    .then(r => r.json())
    .then(json => {
      setData(json);
      setLoading(false);
    });
}, [count]); // Потребить написати залежність
```

Vue:
```javascript
const count = ref(0);
const data = ref(null);
const loading = ref(false);

watchEffect(async () => {
  loading.value = true;
  const res = await fetch(`/api/count/${count.value}`);
  data.value = await res.json();
  loading.value = false;
  // Залежність автоматична: count.value
});
```

---

## Типові пастки

### Пастка 1: Забутий `.value` в JavaScript коді

```javascript
const count = ref(0);

// ✗ НЕПРАВИЛЬНО
count++; // count тепер об'єкт Ref, а не число
if (count === 0) { } // Ніколи не буде true

// ✓ ПРАВИЛЬНО
count.value++;
if (count.value === 0) { }
```

**Як уникнути:** у шаблоні (template) використовуй без `.value`, у скрипті — завжди з `.value`.

### Пастка 2: Деструктуризація втрачає реактивність

```javascript
const user = reactive({ name: 'Іван', age: 30 });

// ✗ НЕПРАВИЛЬНО
const { name, age } = user;
name = 'Петро'; // Не оновлює UI

// ✓ ПРАВИЛЬНО
const { name, age } = toRefs(user);
name.value = 'Петро'; // Оновлює UI
```

### Пастка 3: reactive() не можна переприсвоїти

```javascript
let user = reactive({ name: 'Іван' });

// ✗ НЕПРАВИЛЬНО
user = reactive({ name: 'Петро' }); // Втрачає реактивність

// ✓ ПРАВИЛЬНО
const user = reactive({ name: 'Іван' });
user.name = 'Петро'; // Або використовувати Object.assign
Object.assign(user, { name: 'Петро' });
```

### Пастка 4: Забув return у setup() (без `<script setup>`)

```javascript
export default {
  setup() {
    const count = ref(0);
    const increment = () => count.value++;
    
    // ✗ НЕПРАВИЛЬНО: забув return
    // const count і increment не будуть доступні в template
    
    // ✓ ПРАВИЛЬНО
    return { count, increment };
  }
};
```

(З `<script setup>` цієї пастки немає — все автоматично експортується.)

### Пастка 5: watch() без явної залежності на ref

```javascript
const count = ref(0);

// ✗ НЕПРАВИЛЬНО: watch не спостерігає count
watch(() => count, (newVal) => {
  console.log(newVal); // Функція, а не значення
});

// ✓ ПРАВИЛЬНО
watch(count, (newVal) => {
  console.log(newVal); // Число
});

// Або для складного вираження
watch(() => count.value * 2, (doubled) => {
  console.log(doubled);
});
```

### Пастка 6: Забув очистити listeners у watchEffect

```javascript
// ✗ НЕПРАВИЛЬНО: memory leak
watchEffect(() => {
  el.addEventListener('click', handleClick);
  // Немає cleanup, новий listener кожен раз
});

// ✓ ПРАВИЛЬНО
watchEffect((onCleanup) => {
  el.addEventListener('click', handleClick);
  onCleanup(() => {
    el.removeEventListener('click', handleClick);
  });
});
```

---

## Резюме

Vue 3 Composition API — це функціональний підхід, близький до React Hooks, але часто простіший:
- **script setup** — найсучасніший спосіб писати Vue компоненти
- **ref()** для примітивів, **reactive()** для об'єктів
- **computed** автоматично отримує залежності
- **watch** й **watchEffect** для side effects
- **Composables** — переиспользувана логіка, як custom hooks
- **toRefs** — врятує реактивність при деструктуризації
- Типові пастки: забутий `.value`, деструктуризація, забула return, memory leaks
