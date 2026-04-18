# Система реактивності Vue 3

## Як працює реактивність у Vue 3 — Proxy під капотом?

Vue 3 переписав реактивну систему на Proxy. На відміну від Vue 2, який використовував `Object.defineProperty()`, Vue 3 використовує ES6 Proxy для перехоплення операцій над об'єктами.

Основна ідея: коли ви звертаєтесь до властивості або змінюєте її, Proxy перехоплює цю операцію, відслідковує залежності (яким компонентам це потрібно) та запускає перерендер. Proxy дозволяє обробляти операції на рівні об'єкта, а не кожної властивості окремо.

```javascript
// Vue 2: Object.defineProperty (відслідковування кожної властивості окремо)
const obj = {};
Object.defineProperty(obj, 'count', {
  get() {
    console.log('Читання count'); // Виконується при доступі
    return 5;
  },
  set(newValue) {
    console.log('Запис count:', newValue); // Виконується при зміні
  }
});

obj.count; // Читання count
obj.count = 10; // Запис count: 10

// Недоліки:
// - Не замічає нові властивості (потрібен Vue.set())
// - Не спрацьовує при видаленні властивостей
// - Не працює з Array методами без спеціальної обробки
// - Кожна властивість — окремий Object.defineProperty
```

```javascript
// Vue 3: Proxy (перехоплення на рівні об'єкта)
const state = { count: 0, name: 'Alice' };

const handler = {
  get(target, key) {
    console.log(`Читання: ${key}`);
    track(target, key); // Відслідковування залежності
    return target[key];
  },
  set(target, key, value) {
    if (target[key] !== value) {
      target[key] = value;
      console.log(`Запис: ${key} = ${value}`);
      trigger(target, key); // Запуск оновлень
    }
    return true;
  }
};

const reactive = new Proxy(state, handler);

reactive.count; // Читання: count
reactive.count = 5; // Запис: count = 5
reactive.newProp = 'автоматично спрацює'; // Читання: newProp | Запис: newProp = ...

// Переваги:
// - Перехоплює ВСІ операції (навіть нові властивості)
// - Працює з Array методами (push, splice, filter)
// - Одна точка контролю
// - Простіше для розуміння
```

```vue
<!-- Практично у Vue 3 -->
<script setup>
import { ref, reactive } from 'vue';

// ref() — обгортка для примітивів
const count = ref(0);
console.log(count.value); // 0 — доступ через .value

// reactive() — прямий Proxy для об'єктів
const user = reactive({
  name: 'Alice',
  age: 30
});

// При зміні — автоматично спрацює реактивність
const increment = () => {
  count.value++; // Proxy перехоплює, запускає перерендер
};

const updateName = () => {
  user.name = 'Bob'; // Також перехоплюється Proxy
};
</script>

<template>
  <p>Count: {{ count }}</p>
  <p>User: {{ user.name }}, {{ user.age }}</p>
  <button @click="increment">Increment</button>
  <button @click="updateName">Update Name</button>
</template>
```

---

## Dependency tracking — як Vue знає, що треба перерендерити?

Vue 3 використовує механізм **track** та **trigger**. Коли компонент читає реактивну властивість, Vue записує цей зв'язок ("компонент залежить від цієї властивості"). Коли властивість змінюється, Vue запускає перерендер всіх залежних компонентів.

```javascript
// Спрощена модель реактивної системи Vue

// 1. Глобальний контекст: який компонент/ефект зараз активний
let activeEffect = null;

// 2. Граф залежностей: властивість → список залежних ефектів
const targetMap = new WeakMap(); // obj → Map(key → Set(effects))

// 3. Функція для запису залежності
function track(target, key) {
  if (!activeEffect) return; // Ніхто не читає цю властивість
  
  let depsMap = targetMap.get(target);
  if (!depsMap) {
    depsMap = new Map();
    targetMap.set(target, depsMap);
  }
  
  let dep = depsMap.get(key);
  if (!dep) {
    dep = new Set();
    depsMap.set(key, dep);
  }
  
  // Записуємо залежність: цей ефект залежить від цієї властивості
  dep.add(activeEffect);
}

// 4. Функція для запуску оновлень
function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  
  const dep = depsMap.get(key);
  if (dep) {
    // Запускаємо всі ефекти, які залежать від цієї властивості
    dep.forEach(effect => effect());
  }
}

// 5. Створення реактивного об'єкта з track/trigger
function createReactive(target) {
  return new Proxy(target, {
    get(obj, key) {
      track(obj, key); // Записуємо залежність
      return obj[key];
    },
    set(obj, key, value) {
      if (obj[key] !== value) {
        obj[key] = value;
        trigger(obj, key); // Запускаємо залежні ефекти
      }
      return true;
    }
  });
}

// 6. Функція для запуску ефекту з трекінгом
function watchEffect(fn) {
  activeEffect = fn; // Встановлюємо активний ефект
  fn(); // Виконуємо функцію, вона прочитає реактивні властивості
  activeEffect = null; // Очищаємо після виконання
}

// Приклад
const state = createReactive({ count: 0, name: 'Alice' });

watchEffect(() => {
  console.log(`Count: ${state.count}`); // Читає state.count → фіксується залежність
});

watchEffect(() => {
  console.log(`Name: ${state.name}`); // Читає state.name → фіксується залежність
});

// Коли змінюємо count — перший ефект запускається автоматично
state.count = 5; // Виводить: Count: 5

// Коли змінюємо name — другий ефект запускається
state.name = 'Bob'; // Виводить: Name: Bob

// При читанні властивості, яка не впливає на активний ефект — залежність не записується
const unused = state.count; // Читаємо, але activeEffect = null, так що залежність не записується
```

```javascript
// У Vue 3 реально:
import { reactive, watchEffect } from 'vue';

const state = reactive({
  count: 0,
  doubleCount: 0
});

// watchEffect — це функція, яка запускає наш callback
// та відслідковує всі читання реактивних властивостей
watchEffect(() => {
  // Внутрішньо Vue встановлює activeEffect на цей callback
  console.log(`Count is: ${state.count}`); // Залежність записується
  state.doubleCount = state.count * 2;
});

state.count = 5; // watchEffect запускається автоматично, виводить: Count is: 5
```

---

## Reactive graph — як Vue внутрішньо зберігає граф залежностей?

```
┌─────────────────────────────────────────────────────────────┐
│                    targetMap (WeakMap)                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  state object ───► depsMap (Map)                             │
│                    ├─ 'count'  ──► Set(effect1, effect2)    │
│                    ├─ 'name'   ──► Set(effect2, effect3)    │
│                    └─ 'age'    ──► Set(effect1)             │
│                                                               │
│  user object  ───► depsMap (Map)                             │
│                    ├─ 'email'  ──► Set(effect4)             │
│                    └─ 'role'   ──► Set(effect4, effect5)    │
│                                                               │
└─────────────────────────────────────────────────────────────┘

         ▼ При зміні state.count ▼

  trigger(state, 'count')
           │
           ├─► effect1() ── перерендер компонента A
           │
           └─► effect2() ── перерендер компонента B
```

```javascript
// Приклад: як Vue знає, який компонент перерендерити

const state = reactive({ count: 0, message: 'Hello' });

// Компонент A залежить від count
watchEffect(() => {
  console.log('Component A:', state.count);
  // Записується: effect_A залежить від state.count
});

// Компонент B залежить від обох
watchEffect(() => {
  console.log('Component B:', state.count, state.message);
  // Записується: effect_B залежить від state.count та state.message
});

// Компонент C не залежить від реактивності
const x = 5;
watchEffect(() => {
  console.log('Component C:', x);
  // Записується: effect_C залежить від... нічого (x не реактивний)
});

// При зміні count:
state.count = 10;
// Внутрішньо:
// - trigger(state, 'count') знаходить: effect_A, effect_B
// - Запускає обидва:
//   Component A: 10
//   Component B: 10 Hello

// При зміні message:
state.message = 'World';
// - trigger(state, 'message') знаходить: effect_B
// - Запускає тільки effect_B:
//   Component B: 10 World
```

---

## Чому React порівнює через референси, а Vue відслідковує точково?

React: **Порівняння за рефенціями** (shallow equality)
Vue: **Точкове відслідковування** (fine-grained reactivity)

```javascript
// React — явна залежність
function Component() {
  const [state, setState] = useState({ count: 0, name: 'Alice' });

  useEffect(() => {
    console.log('Effect запустився');
  }, [state]); // Залежність від ЦІЛ об'єкта state

  // Усередину updateCount
  const updateCount = () => {
    // Навіть якщо змінюємо ТІЛЬКИ count...
    setState({ count: state.count + 1, name: 'Alice' });
    // ...effect все одно запускається, бо references змінилась
    // Ефект НЕ знає, що name НЕ змінилась
  };

  return <button onClick={updateCount}>Count</button>;
}

// Розв'язання: розділити на окремі useState або використати useCallback/useMemo
const [count, setCount] = useState(0);
const [name, setName] = useState('Alice');

useEffect(() => {
  console.log('Count змінився');
}, [count]); // Залежність ТІЛЬКИ від count

useEffect(() => {
  console.log('Name змінився');
}, [name]); // Залежність ТІЛЬКИ від name
```

```javascript
// Vue — автоматичне точкове відслідковування
import { reactive, watch } from 'vue';

const state = reactive({ count: 0, name: 'Alice' });

// Вотчер A залежить ТІЛЬКИ від count
watch(
  () => state.count, // Функція-селектор
  (newCount) => {
    console.log('Count змінився:', newCount);
  }
);

// Вотчер B залежить ТІЛЬКИ від name
watch(
  () => state.name,
  (newName) => {
    console.log('Name змінився:', newName);
  }
);

// Або навіть простіше у шаблоні:
// <p>{{ state.count }}</p> — автоматично прив'язується до count
// <p>{{ state.name }}</p>  — автоматично прив'язується до name

const updateCount = () => {
  state.count++;
  // ТІЛЬКИ watch для count запускається, name-вотчер НЕ запускається
};

const updateName = () => {
  state.name = 'Bob';
  // ТІЛЬКИ watch для name запускається, count-вотчер НЕ запускається
};
```

**Плюси/мінуси:**

| Характеристика | React | Vue |
|---|---|---|
| Явність залежностей | Так (dependency array) | Ні (автоматичне) |
| Гранулярність оновлень | Грубіша (порівнює об'єкти) | Тонша (точкове) |
| Продуктивність | Треба оптимізувати (useMemo, useCallback) | Автоматично оптимізується |
| Навчання | Потрібно розуміти shallow equality | Прозоро, інтуїтивно |

---

## Deep reactivity vs shallow — коли що важливо?

```javascript
import { reactive, shallowReactive, ref, shallowRef } from 'vue';

// Deep reactivity (за замовчуванням)
const deepState = reactive({
  user: {
    profile: {
      name: 'Alice',
      age: 30
    }
  }
});

// Глибокі зміни спрацьовують:
deepState.user.profile.name = 'Bob'; // ✓ Реактивно
deepState.user.profile.age = 31;     // ✓ Реактивно

// Shallow reactivity — тільки перший рівень
const shallowState = shallowReactive({
  user: {
    profile: {
      name: 'Alice',
      age: 30
    }
  }
});

// Неглибокі зміни спрацьовують:
shallowState.user = { profile: { name: 'Bob', age: 31 } }; // ✓ Реактивно

// Глибокі зміни НЕ спрацьовують:
shallowState.user.profile.name = 'Bob'; // ✗ НЕ реактивно!
shallowState.user.profile.age = 31;     // ✗ НЕ реактивно!

// Коли використовувати shallow:
// 1. Велика вложена структура даних — для продуктивності
// 2. Третьокупне API, де структура рідко змінюється

const largeData = shallowReactive({
  config: { /* великий об'єкт з 1000+ властивостями */ }
});

// Змінюємо весь об'єкт за раз:
largeData.config = newConfig; // ✓ Дешево за продуктивністю
```

```javascript
// shallowRef — для примітивів, аналог shallowReactive
const count = shallowRef({ value: 0 });

// Зміна self property спрацьовує:
count.value = { value: 1 }; // ✓ Реактивно

// Але глибокі зміни НЕ спрацьовують:
count.value.value = 2; // ✗ НЕ реактивно!

// Для примітивів це малозастосовно, але для великих об'єктів корисно:
const cachedData = shallowRef({ /* 10MB JSON */ });

// Заміна всього об'єкта:
cachedData.value = newData; // ✓ Реактивно, дешево
```

---

## Array mutation — чому push/splice триггерять оновлення?

```javascript
// Vue 2: спеціальна обробка масивів
// - vm.$set(array, index, value) для по-індексної заміни
// - Спеціальні оберки для push, splice, filter тощо
// - Прямі мутації некоректно спрацьовували

// Vue 3: Proxy перехоплює ВСІХ операції
import { reactive, watch } from 'vue';

const items = reactive([1, 2, 3]);

watch(
  () => items.length, // Залежність від length
  (newLength) => {
    console.log('Масив змінився, довжина:', newLength);
  }
);

// Всі методи автоматично спрацьовують:

items.push(4);
// ✓ Перехоплюється через set trap (items[3] = 4, length = 4)
// Виводить: Масив змінився, довжина: 4

items.splice(1, 1); // Видалити елемент на індексі 1
// ✓ Перехоплюється set трап (перінумерація індексів, length = 3)
// Виводить: Масив змінився, довжина: 3

items[0] = 10;
// ✓ Перехоплюється set трап (items[0] = 10)
// Виводить: ... залежить від того, чи length змінилась

// Чому це працює? Потому що Proxy перехоплює SET операції на масиві:
// push → [3] set to 4 + length set to 4
// splice → indices reassigned + length set to new value
// [0] = 10 → [0] set to 10
```

```javascript
// Детально: як Proxy перехоплює методи масиву

const handler = {
  set(target, key, value) {
    console.log(`set: target[${key}] = ${value}`);
    
    // Для масивів: коли вихідні length, спрацьовує trigger
    const oldLength = target.length;
    target[key] = value;
    
    if (key === 'length' && target.length !== oldLength) {
      trigger(target, 'length');
    }
    
    return true;
  }
};

const arr = new Proxy([1, 2, 3], handler);

arr.push(4);
// Внутрішньо:
// [3] = 4 → set: target[3] = 4
// length = 4 → set: target[length] = 4
// Обидва викликають trigger

arr.splice(0, 1);
// Внутрішньо перераховуються індекси
// [0] = 2, [1] = 3, length = 2
// Всі спричиняють trigger
```

```vue
<!-- Практичне застосування -->
<script setup>
import { reactive, watch } from 'vue';

const todos = reactive([
  { id: 1, text: 'Learn Vue', done: false }
]);

watch(
  () => todos.length,
  (newLength) => {
    console.log(`Кількість todos: ${newLength}`);
  }
);

const addTodo = () => {
  todos.push({ id: 2, text: 'Build app', done: false });
  // ✓ Автоматично запускає watch
};

const removeTodo = (index) => {
  todos.splice(index, 1);
  // ✓ Автоматично запускає watch
};

const toggleTodo = (index) => {
  todos[index].done = !todos[index].done;
  // ✓ Працює, бо Proxy перехоплює set
};
</script>

<template>
  <ul>
    <li v-for="(todo, idx) in todos" :key="todo.id">
      {{ todo.text }}
      <button @click="removeTodo(idx)">Remove</button>
    </li>
  </ul>
  <button @click="addTodo">Add Todo</button>
  <p>Всього: {{ todos.length }}</p>
</template>
```

---

## Set / Map / WeakMap у reactive — теж підтримуються?

```javascript
import { reactive, watch } from 'vue';

// ✓ Set як реактивна колекція
const tags = reactive(new Set(['vue', 'reactive']));

watch(
  () => tags.size,
  (newSize) => {
    console.log('Tags size:', newSize);
  }
);

tags.add('javascript'); // Спрацює trigger
// Виводить: Tags size: 3

tags.delete('vue'); // Спрацює trigger
// Виводить: Tags size: 2

// ✓ Map як реактивна колекція
const userMap = reactive(new Map([
  ['alice', { name: 'Alice', age: 30 }],
  ['bob', { name: 'Bob', age: 25 }]
]));

watch(
  () => userMap.size,
  (newSize) => {
    console.log('Map size:', newSize);
  }
);

userMap.set('charlie', { name: 'Charlie', age: 28 });
// Спрацює trigger
// Виводить: Map size: 3

userMap.delete('alice');
// Спрацює trigger
// Виводить: Map size: 2

// ✗ WeakMap НЕ можна зробити реактивним
// (WeakMap не має .size та не iterable)
// Але можна зберігати всередину reactive об'єкта:
const cache = reactive({
  _weakMap: new WeakMap() // Порядок, але не сама реактивна
});

// Коли потрібні weak references — просто використовуйте WeakMap в обичній формі
const listeners = new WeakMap(); // Без reactive()
```

```javascript
// Практичний приклад: Set для унікальних значень

import { reactive, computed } from 'vue';

const state = reactive({
  tags: new Set()
});

const uniqueTags = computed(() => Array.from(state.tags)); // Для рендерингу

const addTag = (tag) => {
  if (!state.tags.has(tag)) {
    state.tags.add(tag);
  }
};

const removeTag = (tag) => {
  state.tags.delete(tag);
};

// У шаблоні:
// <div v-for="tag in uniqueTags">{{ tag }}</div>
```

---

## Nested reactive — як Vue перетворює весь об'єкт у reactive рекурсивно?

```javascript
import { reactive, isReactive } from 'vue';

const state = reactive({
  user: {
    profile: {
      name: 'Alice',
      settings: {
        theme: 'dark',
        notifications: true
      }
    },
    posts: [
      { id: 1, title: 'Vue 3', comments: [{ text: 'Great!' }] }
    ]
  }
});

// Vue рекурсивно перетворює ВЕСЬ граф об'єктів в Proxy:

console.log(isReactive(state)); // true
console.log(isReactive(state.user)); // true
console.log(isReactive(state.user.profile)); // true
console.log(isReactive(state.user.profile.settings)); // true
console.log(isReactive(state.user.posts)); // true
console.log(isReactive(state.user.posts[0])); // true
console.log(isReactive(state.user.posts[0].comments)); // true

// Всі читання автоматично записуються у граф залежностей:
const name = state.user.profile.name;
// track(state, 'user')
// track(state.user, 'profile')
// track(state.user.profile, 'name')
// Три залежності зареєстровані!

// Заміна на глибоког рівні:
state.user.profile.settings.theme = 'light';
// trigger(state.user.profile.settings, 'theme')
// Перерендер спрацює
```

```javascript
// Як Vue 3 реалізує рекурсивну реактивність

function createReactive(target, seen = new WeakSet()) {
  // Уникаємо циклічних посилань
  if (seen.has(target)) {
    return target;
  }
  seen.add(target);

  // Якщо це вже Proxy — повертаємо як є
  if (isProxy(target)) {
    return target;
  }

  const handler = {
    get(obj, key) {
      const value = obj[key];
      
      // Якщо значення — об'єкт або масив, рекурсивно робимо його реактивним
      if (value !== null && typeof value === 'object') {
        return createReactive(value, seen); // Рекурсія!
      }
      
      track(obj, key);
      return value;
    },
    
    set(obj, key, value) {
      if (obj[key] !== value) {
        obj[key] = value;
        trigger(obj, key);
      }
      return true;
    }
  };

  return new Proxy(target, handler);
}

const state = createReactive({
  user: {
    profile: {
      name: 'Alice'
    }
  }
});

// При першому доступі до state.user.profile.name:
// 1. Читаємо state → get(state, 'user')
// 2. Створюємо Proxy для state.user → createReactive(state.user)
// 3. Читаємо state.user → get(state.user, 'profile')
// 4. Створюємо Proxy для state.user.profile → createReactive(state.user.profile)
// 5. Читаємо state.user.profile → get(state.user.profile, 'name')
// 6. Повертаємо 'Alice'
```

```vue
<!-- Практичне застосування вложеної реактивності -->
<script setup>
import { reactive, watch } from 'vue';

const data = reactive({
  company: {
    departments: [
      {
        name: 'Engineering',
        teams: [
          { name: 'Frontend', members: ['Alice', 'Bob'] }
        ]
      }
    ]
  }
});

// Вотчер на глибокій вложеній властивості
watch(
  () => data.company.departments[0].teams[0].members.length,
  (newLength) => {
    console.log('Frontend members:', newLength);
  }
);

// Будь-яка зміна на будь-якому рівні спрацює:
data.company.departments[0].teams[0].members.push('Charlie');
// ✓ Спрацює, виводить: Frontend members: 3
</script>

<template>
  <div v-for="dept in data.company.departments">
    <h3>{{ dept.name }}</h3>
    <div v-for="team in dept.teams">
      <h4>{{ team.name }}</h4>
      <p v-for="member in team.members">{{ member }}</p>
    </div>
  </div>
</template>
```

---

## readonly — щоб запобігти модифікації

```javascript
import { reactive, readonly, isReadonly } from 'vue';

const state = reactive({
  count: 0,
  message: 'Hello'
});

// readonly() створює Proxy, який дозволяє читати, але забороняє писати
const readonlyState = readonly(state);

console.log(readonlyState.count); // 0 — ✓ Читання працює
console.log(isReadonly(readonlyState)); // true

// Спроби модифікації:
readonlyState.count = 5; // ✗ Error в strict mode або мовчазна ігнорація

try {
  readonlyState.count = 5;
  // Error: Set operation on key "count" failed: target is readonly.
} catch (e) {
  console.error(e.message);
}

// Читання все ще спрацьовує та записує залежність:
const watcher = () => {
  console.log('Count:', readonlyState.count);
};
```

```javascript
// Коли використовувати readonly:

import { reactive, readonly, ref } from 'vue';

// 1. Батьківський компонент передає дані дочірньому
// Parent.vue
const parentState = reactive({
  users: [{ name: 'Alice' }, { name: 'Bob' }]
});

const childProps = readonly(parentState); // Передаємо дочірньому

// Child.vue
// props: {
//   data: readonly(parentState)
// }
// child не може змінювати дані батька

// 2. API результати, що не мають змінюватись
const apiResponse = readonly(ref({
  data: { id: 1, title: 'Post' }
}));

// 3. Конфігурація, яка не має змінюватись під час виконання
const appConfig = readonly({
  apiUrl: 'https://api.example.com',
  maxRetries: 3
});

// 4. Глибокий readonly — запобігає модифікаціям на всіх рівнях
const deepReadonly = readonly({
  user: {
    profile: {
      name: 'Alice'
    }
  }
});

deepReadonly.user.profile.name = 'Bob'; // ✗ Error
```

---

## markRaw / toRaw — коли треба "вивести" об'єкт з реактивної системи

```javascript
import { reactive, markRaw, toRaw, isReactive } from 'vue';

// markRaw() — припинити об'єкт реактивним під час створення

// Приклад: велика DOM-дерево або WebGL контекст
const canvasContext = markRaw(document.getElementById('canvas').getContext('2d'));

const state = reactive({
  canvas: canvasContext, // ✓ Він зберігається в reactive, але сам НЕ реактивний
  count: 0
});

console.log(isReactive(state.canvas)); // false — markRaw запобіг реактивності

// toRaw() — витягти оригінальний об'єкт з реактивного Proxy

const user = reactive({
  name: 'Alice',
  age: 30
});

const originalUser = toRaw(user);

// originalUser === user._object (внутрішня посилання на оригінал)
// originalUser.name = 'Bob'; // Змінить оригінал, але НЕ спрацює реактивність

console.log(isReactive(originalUser)); // false
console.log(originalUser.name); // 'Alice'
```

```javascript
// Коли використовувати markRaw та toRaw:

// 1. Великі об'єкти, які рідко змінюються
const largeDataStructure = markRaw({
  items: new Array(10000).fill(0).map((_, i) => ({ id: i, data: 'x' }))
});

const state = reactive({
  data: largeDataStructure // Економиться пам'ять на Proxy для великого об'єкта
});

// 2. Вирішення циклічних посилань
const circular = {};
circular.self = circular;
const reactiveCircular = reactive(circular); // Може спричинити проблеми
const markedCircular = reactive({ data: markRaw(circular) }); // Безпечніше

// 3. Отримання оригіналу для порівняння або модифікації без реактивності
const state = reactive({ values: [1, 2, 3] });

const original = toRaw(state);
original.values.push(4); // Модифікуємо БЕЗ спрацьовування реактивності

// 4. Плагін-інтеграція, де не потрібна реактивність
const thirdPartyConfig = {
  apiKey: 'secret',
  settings: { /* велика конфігурація */ }
};

const appState = reactive({
  _thirdPartyConfig: markRaw(thirdPartyConfig) // Не реактивний, але зберігається
});
```

---

## Performance: shallowRef для великих об'єктів, computed кешування

```javascript
import { ref, shallowRef, computed, reactive } from 'vue';

// ❌ Неефективно: deep ref для великого об'єкта
const largeData = ref({
  items: new Array(100000).fill({ nested: { data: 'x' } })
});

// Кожна зміна на будь-якому рівні спирає всю реактивну систему

// ✓ Ефективно: shallowRef для великого об'єкта
const largeDataShallow = shallowRef({
  items: new Array(100000).fill({ nested: { data: 'x' } })
});

// Змінюємо весь об'єкт за раз (дешево):
const updateData = () => {
  largeDataShallow.value = {
    items: newData // Нова посилання = перерендер
  };
};

// Глибокі зміни НЕ спрацьовують, але це нам не потрібно
largeDataShallow.value.items[0].nested.data = 'y'; // ✗ Не спрацює

// Computed з кешуванням — обчислюється тільки при зміні залежностей

const state = reactive({
  firstName: 'John',
  lastName: 'Doe',
  fullNameComputedCount: 0
});

// ✓ Кешується — обчислюється тільки коли firstName або lastName змінюються
const fullName = computed(() => {
  console.log('Computing fullName');
  state.fullNameComputedCount++;
  return `${state.firstName} ${state.lastName}`;
});

console.log(fullName.value); // Computing fullName | "John Doe" | count = 1
console.log(fullName.value); // "John Doe" (кешовано, count = 1)
console.log(fullName.value); // "John Doe" (кешовано, count = 1)

state.firstName = 'Jane'; // Змінилась залежність
console.log(fullName.value); // Computing fullName | "Jane Doe" | count = 2
```

```javascript
// Приклад оптимізації продуктивності:

import { reactive, shallowRef, computed, watch } from 'vue';

// Сценарій: велика таблиця з фільтруванням

const state = reactive({
  filters: { status: 'active', priority: 'high' },
  sortBy: 'date'
});

// ❌ Без оптимізації: перепрацює фільтрацію й сортування постійно
const filteredAndSorted = computed(() => {
  console.log('Filtering and sorting...');
  return allData
    .filter(item => item.status === state.filters.status)
    .filter(item => item.priority === state.filters.priority)
    .sort((a, b) => a[state.sortBy] > b[state.sortBy] ? 1 : -1);
}, { depCount: 0 });

// ✓ З оптимізацією: окремі computed для фільтрації та сортування

const filterKey = computed(
  () => `${state.filters.status}-${state.filters.priority}`
);

const filtered = computed(() => {
  console.log('Filtering...');
  return allData.filter(item =>
    item.status === state.filters.status &&
    item.priority === state.filters.priority
  );
});

const sorted = computed(() => {
  console.log('Sorting...');
  return [...filtered.value].sort((a, b) =>
    a[state.sortBy] > b[state.sortBy] ? 1 : -1
  );
});

// Тепер:
// - Фільтрація перепрацює тільки коли змінюються filters
// - Сортування перепрацює тільки коли змінюються sortBy
// - Якщо змінюється інше — нічого не перепрацьовується
```

---

## Як debug-ити реактивність: Vue DevTools, onTrack, onTrigger

```javascript
import { reactive, watch, onTrack, onTrigger } from 'vue';

const state = reactive({
  count: 0,
  message: 'Hello'
});

// watch з debug-обробниками
watch(
  () => state.count,
  (newValue) => {
    console.log('Count changed:', newValue);
  },
  {
    // onTrack — запускається коли вотчер читає залежність
    onTrack(event) {
      console.log('Track event:', event);
      // {
      //   target: state,
      //   type: 'get',
      //   key: 'count'
      // }
    },
    
    // onTrigger — запускається коли залежність змінюється
    onTrigger(event) {
      console.log('Trigger event:', event);
      // {
      //   target: state,
      //   type: 'set',
      //   key: 'count',
      //   newValue: 1,
      //   oldValue: 0
      // }
    }
  }
);

state.count = 1; // Спрацює onTrigger
// Консоль:
// Track event: { target: state, type: 'get', key: 'count' }
// Trigger event: { target: state, type: 'set', key: 'count', newValue: 1, oldValue: 0 }
// Count changed: 1
```

```javascript
// Розширеної debug-функції для Vue реактивності

function debugReactivity(target, name = 'target') {
  return new Proxy(target, {
    get(obj, key) {
      console.log(`[${name}] GET ${String(key)}`);
      return obj[key];
    },
    set(obj, key, value) {
      console.log(`[${name}] SET ${String(key)} = ${value}`);
      obj[key] = value;
      return true;
    }
  });
}

const state = debugReactivity(reactive({
  count: 0,
  user: { name: 'Alice' }
}), 'appState');

state.count; // [appState] GET count
state.count = 5; // [appState] SET count = 5
state.user.name = 'Bob'; // [appState] GET user | [appState] SET name = Bob
```

```vue
<!-- Vue DevTools для debug реактивності -->
<script setup>
import { reactive, computed, watch } from 'vue';

const state = reactive({
  count: 0,
  doubled: null
});

watch(
  () => state.count,
  (newValue) => {
    state.doubled = newValue * 2;
  }
);

// В Vue DevTools:
// 1. Перевіть на вкладку "Reactivity"
// 2. Побачите граф залежностей state → watch
// 3. При кліку на елемент побачите:
//    - Всі залежності цього значення
//    - Всі ефекти, які від нього залежать
// 4. Дебаг через inspect в консолі

const inspect = () => {
  console.log(state); // Розгорніть у Vue DevTools для детального огляду
};
</script>

<template>
  <p>Count: {{ state.count }}</p>
  <p>Doubled: {{ state.doubled }}</p>
  <button @click="state.count++">Increment</button>
  <button @click="inspect">Inspect in Console</button>
</template>
```

---

## Типові пастки

### 1. Зміна властивості динамічно доданою на об'єкт

```javascript
const state = reactive({});

// Vue 2 потребував Vue.set():
// Vue.set(state, 'newProp', value);

// Vue 3: Proxy перехоплює динамічні властивості
state.newProp = 'Hello'; // ✓ Автоматично реактивно

// Але якщо це Об'єкт з помилкою при доступі, це може бути проблемою:
const buggyObject = Object.create(null);
buggyObject.prop = 'value';

const state = reactive({ buggyObject });
state.buggyObject.newProp = 'x'; // ✓ Все одно працює
```

### 2. Заміна всього об'єкту замість зміни властивостей

```javascript
const state = reactive({ user: { name: 'Alice' } });

// ❌ Не мутує реактивно (втрачаємо посилання на oldUser)
state.user = { name: 'Bob' };

// ✓ Краще: мутувати усередину
state.user.name = 'Bob';

// Якщо ПОТРІБНА заміна всього об'єкту:
Object.assign(state.user, { name: 'Bob', age: 30 }); // Мутує на місці
```

### 3. Забувають про .value для ref

```javascript
import { ref } from 'vue';

const count = ref(0);

// ❌ Частова помилка
const increment = () => {
  count++; // Спрацює? НІ! count — об'єкт, не число
};

// ✓ Правильно
const increment = () => {
  count.value++; // Мутуємо .value
};

// У шаблоні .value НЕ потрібний:
// <p>{{ count }}</p> <!-- Vue автоматично розпаковує -->
```

### 4. Масиви і глубока реактивність

```javascript
const items = reactive([]);

// ❌ Прямо по індексу за межами array.length — не спрацює:
items[100] = 'value'; // Спрацює, але неефективно

// ✓ Використовуйте методи або push:
items.push('value'); // Спрацює оптимально

// ❌ Заміна елемента по індексу іноді не спрацює:
items[0] = newItem; // Спрацює у Vue 3, але у Vue 2 потребував vm.$set
```

### 5. Функції не повинні бути реактивними

```javascript
const state = reactive({
  count: 0,
  // ❌ Не зберігайте функції у reactive
  increment: function() {
    this.count++;
  }
});

// ✓ Функції відокремлюйте:
const increment = () => {
  state.count++;
};

// ✓ Або у setup компоненту:
// const increment = () => count.value++;
```

### 6. WeakMap / WeakSet не можна зробити реактивними

```javascript
// ❌ Це не працює:
const cache = reactive(new WeakMap());

// ✓ WeakMap залишіть поза reactive:
const cache = new WeakMap();

const state = reactive({
  data: {}
});

// Та використовуйте як допоміжну структуру
cache.set(state.data, 'computed-value');
```

### 7. Object.freeze() запобігає реактивності

```javascript
const state = reactive({
  config: Object.freeze({ apiUrl: 'https://api.example.com' })
});

state.config.apiUrl = 'new-url'; // ✗ НЕ спрацює, frozen

// ✓ Якщо конфігурація рідко змінюється, використовуйте readonly:
const state = reactive({
  config: readonly({ apiUrl: 'https://api.example.com' })
});
```

### 8. Циклічні залежності у computed

```javascript
// ❌ Циклічна залежність
const a = computed(() => b.value + 1); // залежить від b
const b = computed(() => a.value + 1); // залежить від a

// Результат: нескінченний цикл

// ✓ Розділіть логіку чи використовуйте watch:
const base = ref(0);
const a = computed(() => base.value + 1);
const b = computed(() => base.value + 2);
```
