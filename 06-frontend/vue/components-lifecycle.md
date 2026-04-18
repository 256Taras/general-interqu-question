# Vue 3 Components: Lifecycle & Advanced Patterns

## Які lifecycle hooks у Vue 3 та в якому порядку вони викликаються?

Vue 3 надає набір lifecycle hooks для взаємодії з різними етапами існування компонента. На відміну від Vue 2 (beforeCreate, created тощо), Vue 3 з Composition API пропонує функціональний підхід: `onBeforeMount`, `onMounted`, `onBeforeUpdate`, `onUpdated`, `onBeforeUnmount`, `onUnmounted`, `onErrorCaptured`.

```
┌─────────────────────────────────────────────────────────┐
│                                                           │
│   Компонент створюється (instance ready)                 │
│                                                           │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────────┐
        │ onBeforeMount()    │ ← DOM ще не монтований
        └────────┬───────────┘
                 │
                 ▼
        ┌────────────────────────────────┐
        │ onMounted()                    │ ← DOM монтований, видимий
        └────────┬───────────────────────┘
                 │
                 ▼ (дані змінюються)
        ┌────────────────────┐
        │ onBeforeUpdate()   │ ← DOM буде оновлено
        └────────┬───────────┘
                 │
                 ▼
        ┌────────────────────┐
        │ onUpdated()        │ ← DOM оновлено
        └────────┬───────────┘
                 │
                 ▼ (цикл може повторюватися)
        ┌────────────────────┐
        │ onBeforeUnmount()  │ ← DOM буде видалено
        └────────┬───────────┘
                 │
                 ▼
        ┌────────────────────┐
        │ onUnmounted()      │ ← DOM видалено, компонент видален
        └────────────────────┘
```

```vue
<script setup>
import { onBeforeMount, onMounted, onBeforeUpdate, onUpdated, onBeforeUnmount, onUnmounted, ref } from 'vue'

const count = ref(0)

onBeforeMount(() => {
  console.log("1. Перед монтуванням — дані готові, DOM ще ні")
})

onMounted(() => {
  console.log("2. Монтовано — DOM доступний, можна звертатися до ref")
  // Тут безпечно звертатися до DOM, інізіалізувати бібліотеки
})

onBeforeUpdate(() => {
  console.log("3. Перед оновленням — старі дані ще в DOM")
})

onUpdated(() => {
  console.log("4. Після оновлення — нові дані в DOM")
})

onBeforeUnmount(() => {
  console.log("5. Перед видаленням — компонент ще видимий")
})

onUnmounted(() => {
  console.log("6. Видалено — очищення ресурсів (таймери, слухачі)")
})
</script>
```

---

## Що робити у якому hook'і? Рекомендовані практики.

| Hook | Коли використовувати | Приклади |
|---|---|---|
| `onBeforeMount` | Рідко. Підготовка перед DOM | Ініціалізація state, завантаження даних |
| `onMounted` | Найчастіше. Робота з DOM, бібліотеками | API запити, DOM маніпуляція, отримання ref, ініціалізація графіків |
| `onBeforeUpdate` | Дуже рідко. Логування або захист | Зберегти старі дані перед оновленням |
| `onUpdated` | Рідко. Робота після оновлення DOM | Додаткові DOM операції, які залежать від нових даних |
| `onBeforeUnmount` | Заборона та попередження | "Ви впевнені?" діалоги перед закриттям |
| `onUnmounted` | Завжди для cleanup | Видалення таймерів, слухачів подій, скасування запитів |

```vue
<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'

const data = ref(null)
const loading = ref(true)
const timerId = ref(null)

// ❌ НЕПРАВИЛЬНО: асинхронна операція у setup без контролю
// API запит буде завжди, навіть якщо компонент видален

// ✅ ПРАВИЛЬНО: асинхронна операція в onMounted
onMounted(async () => {
  try {
    const response = await fetch('/api/data')
    data.value = await response.json()
  } catch (error) {
    console.error('Помилка завантаження:', error)
  } finally {
    loading.value = false
  }

  // Таймери мають бути видалені в onUnmounted
  timerId.value = setInterval(() => {
    console.log('Тик-так')
  }, 1000)
})

// ОБОВ'ЯЗКОВО очищуйте ресурси
onUnmounted(() => {
  if (timerId.value) {
    clearInterval(timerId.value)
  }
  // Скасувати активні запити, видалити слухачі
})

// Хак для cleaner-up: composable з автоматичним cleanup
function useTimer(interval = 1000) {
  const timerId = ref(null)
  
  onMounted(() => {
    timerId.value = setInterval(() => {
      console.log('Автоматичний cleanup')
    }, interval)
  })
  
  onUnmounted(() => {
    clearInterval(timerId.value)
  })
  
  return { timerId }
}
</script>
```

---

## Reactive vs non-reactive props. Як правильно передавати дані у компонент?

У Vue 3 **props завжди реактивні** при передачі через батьків, але є нюанси з об'єктами та масивами. Основне правило: **не мутуйте props безпосередньо** — це порушує однонаправленість потоку даних.

```vue
<!-- Parent.vue -->
<script setup>
import { ref } from 'vue'
import Child from './Child.vue'

const user = ref({ name: 'Taras', age: 30 })
const items = ref([1, 2, 3])
</script>

<template>
  <!-- Передача реактивних значень -->
  <Child :user="user" :items="items" />
</template>
```

```vue
<!-- Child.vue -->
<script setup>
import { defineProps, computed } from 'vue'

const props = defineProps({
  user: { type: Object, required: true },
  items: { type: Array, default: () => [] }
})

// ✅ ПРАВИЛЬНО: Обчислювальне властивість для трансформацій
const userDisplay = computed(() => {
  return `${props.user.name} (${props.user.age})`
})

// ❌ НЕПРАВИЛЬНО: Пряма мутація props
const incrementAge = () => {
  // props.user.age++ // ❌ Порушує однонаправленість даних
  
  // ✅ ПРАВИЛЬНО: Emititi подія батькові
  // (див. defineEmits нижче)
}
</script>

<template>
  <div>{{ userDisplay }}</div>
  <!-- items.length також реактивний -->
  <p>Елементів: {{ items.length }}</p>
</template>
```

---

## defineProps / defineEmits у script setup з TypeScript. Як скористатися?

`defineProps` та `defineEmits` — це макроси Vite, які працюють без імпорту у `<script setup>`. Вони мають 2 синтаксису: runtime (об'єкти) та типи (TypeScript).

```vue
<script setup lang="ts">
import { defineProps, defineEmits, computed } from 'vue'

// ✅ Runtime синтаксис (валідація на runtime)
const props = defineProps({
  title: String,
  count: { type: Number, default: 0, required: true },
  items: Array as PropType<string[]>,
  disabled: Boolean
})

// ✅ TypeScript синтаксис (type-safe, але без runtime валідації)
// Це РЕКОМЕНДОВАНО у production
interface Props {
  title?: string
  count?: number
  items?: string[]
  disabled?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
  items: () => [],
  disabled: false
})

// defineEmits — для відправки подій батькові
const emit = defineEmits<{
  // Ім'я подія: параметри
  update: [value: string]
  delete: [id: number]
  'custom-event': [payload: { name: string; age: number }]
}>()

// Викликати подію
const handleClick = () => {
  emit('update', 'нове значення')
  emit('delete', 123)
  emit('custom-event', { name: 'Taras', age: 30 })
}
</script>

<template>
  <div>
    <h1>{{ title }}</h1>
    <p>Лічильник: {{ count }}</p>
    <button @click="handleClick">Натисни</button>
  </div>
</template>
```

Runtime синтаксис з TypeScript:

```vue
<script setup lang="ts">
import { defineProps, PropType } from 'vue'

const props = defineProps({
  message: String,
  count: {
    type: Number,
    required: true,
    validator: (value: number) => value > 0 // Власна валідація
  },
  items: {
    type: Array as PropType<string[]>,
    default: () => []
  }
})
</script>
```

---

## v-model у кастомних компонентах. Як це працює?

`v-model` — це синтаксичний цукерок для синхронізації даних батьків і дитини. Під капотом це `prop + emit`. У Vue 3 можна мати **кілька v-models** у одному компоненті.

```vue
<!-- Parent.vue -->
<script setup>
import { ref } from 'vue'
import CustomInput from './CustomInput.vue'

const text = ref('Hello')
const checked = ref(true)
</script>

<template>
  <!-- v-model:modelValue는 :modelValue + @update:modelValue -->
  <CustomInput v-model="text" v-model:checked="checked" />
  <p>Text: {{ text }}</p>
  <p>Checked: {{ checked }}</p>
</template>
```

```vue
<!-- CustomInput.vue -->
<script setup lang="ts">
import { defineProps, defineEmits } from 'vue'

// v-model:modelValue вимагає prop "modelValue" та emit "update:modelValue"
defineProps<{
  modelValue: string
  checked?: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
  'update:checked': [value: boolean]
}>()

const handleInput = (event: Event) => {
  const target = event.target as HTMLInputElement
  // Батьківська змінна text оновиться автоматично
  emit('update:modelValue', target.value)
}

const toggleChecked = () => {
  emit('update:checked', !(checked))
}
</script>

<template>
  <div>
    <input :value="modelValue" @input="handleInput" />
    <label>
      <input type="checkbox" :checked="checked" @change="toggleChecked" />
      Позначити
    </label>
  </div>
</template>
```

```
v-model="text" 
    ↓ (розширюється)
:modelValue="text" @update:modelValue="text = $event"

v-model:checked="checked"
    ↓ (розширюється)
:checked="checked" @update:checked="checked = $event"
```

---

## Slots — default, named, scoped. Приклади використання.

Slots дозволяють компоненту приймати вміст від батька. Vue 3 має 3 типи: **default**, **named**, **scoped**.

```vue
<!-- Modal.vue -->
<script setup>
import { defineProps, defineSlots } from 'vue'

defineSlots<{
  default: (props: {}) => any // Default slot
  header: (props: { title: string }) => any
  footer: (props: { onCancel: () => void }) => any
}>()

const handleClose = () => {
  console.log('Закрити модаль')
}
</script>

<template>
  <div class="modal">
    <!-- Named slot: header -->
    <div class="modal-header">
      <slot name="header" :title="'Заголовок модалі'" />
    </div>

    <!-- Default slot -->
    <div class="modal-body">
      <slot />
    </div>

    <!-- Named scoped slot: footer -->
    <div class="modal-footer">
      <slot name="footer" :on-cancel="handleClose" />
    </div>
  </div>
</template>
```

```vue
<!-- Parent.vue — використання slots -->
<script setup>
import Modal from './Modal.vue'
</script>

<template>
  <Modal>
    <!-- Default slot content -->
    <p>Це вміст за замовчуванням всередині modal-body</p>

    <!-- Named slot: header (користувацька залози) -->
    <template #header="{ title }">
      <h2>{{ title }} — Користувацький заголовок</h2>
    </template>

    <!-- Named scoped slot: footer -->
    <template #footer="{ onCancel }">
      <button @click="onCancel">Скасувати</button>
      <button>OK</button>
    </template>
  </Modal>
</template>
```

Скорочена синтаксис для default slot:

```vue
<Modal>
  <p>Default slot без #default</p>
  <template #header="{ title }">
    <h2>{{ title }}</h2>
  </template>
</Modal>
```

---

## provide / inject — вертикальна комунікація крізь дерево компонентів.

`provide / inject` дозволяє передавати дані від батька до глибоких нащадків **без пропс drilling**. Це альтернатива React Context API.

```vue
<!-- App.vue — постачальник (provider) -->
<script setup>
import { provide, ref } from 'vue'
import Grandchild from './Grandchild.vue'

const theme = ref('dark')
const user = { name: 'Taras', role: 'admin' }

// Надати дані всім нащадкам
provide('theme', theme) // Реактивно: змін вернеться
provide('user', user)   // Статично
</script>

<template>
  <div :class="`theme-${theme}`">
    <Grandchild />
    <!-- theme та user будуть доступні в Grandchild без передачі props -->
  </div>
</template>
```

```vue
<!-- Grandchild.vue — споживач (injector) -->
<script setup>
import { inject, computed } from 'vue'

// Отримати надані дані
const theme = inject('theme', 'light') // Дефолтне значення 'light'
const user = inject('user', {})

// Реактивно використовувати
const isDarkMode = computed(() => theme.value === 'dark')
</script>

<template>
  <div>
    <p>Тема: {{ theme }}</p>
    <p>Користувач: {{ user.name }} ({{ user.role }})</p>
    <p v-if="isDarkMode">Темний режим включений!</p>
  </div>
</template>
```

```
App.vue (provide)
    │
    ├─ Child.vue
    │   │
    │   └─ Grandchild.vue (inject) ✓
    │       │
    │       └─ GreatGrandchild.vue (inject) ✓
    │
    └─ AnotherChild.vue
        └─ DeepNested.vue (inject) ✓
```

Виключення: Якщо батько передав `theme = ref(...)`, то нащадок отримає **реактивну посилання**.

---

## Teleport — передача компонента в інший DOM вузол. Коли це потрібно?

`<Teleport>` переміщує компонент у інший DOM вузол (зазвичай у `<body>` для модалей). Це уникає проблем із стилями та z-index.

```vue
<!-- Modal.vue -->
<script setup>
import { ref } from 'vue'

const isOpen = ref(false)
</script>

<template>
  <!-- ✅ ПРАВИЛЬНО: Модаль телепортується у body, не під батьком -->
  <Teleport to="body">
    <Transition>
      <div v-show="isOpen" class="modal-overlay">
        <div class="modal-content">
          <h2>Модаль</h2>
          <p>Цей вміст в body, не у батька</p>
          <button @click="isOpen = false">Закрити</button>
        </div>
      </div>
    </Transition>
  </Teleport>

  <button @click="isOpen = true">Відкрити модаль</button>
</template>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0; left: 0;
  width: 100%; height: 100%;
  background: rgba(0, 0, 0, 0.5);
  z-index: 1000; /* Тепер працює коректно, бо модаль у body */
}

.modal-content {
  position: absolute;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  background: white;
  padding: 2rem;
  border-radius: 8px;
}
</style>
```

Основне правило: **Модалі, тултіпи, дропдауни — завжди Teleport!**

---

## KeepAlive — кешування неактивних компонентів. Коли використовувати?

`<KeepAlive>` зберігає стан неактивних компонентів у пам'яті замість видалення. Корисно для дорогих компонентів (графіки, форми з даними).

```vue
<!-- TabsExample.vue -->
<script setup>
import { ref } from 'vue'
import FormTab from './FormTab.vue'
import ChartTab from './ChartTab.vue'

const activeTab = ref('form')
</script>

<template>
  <div class="tabs">
    <button
      v-for="tab in ['form', 'chart']"
      :key="tab"
      :class="{ active: activeTab === tab }"
      @click="activeTab = tab"
    >
      {{ tab }}
    </button>
  </div>

  <!-- ❌ БЕЗ KeepAlive: компонент видаляється при переключенні, стан втрачається -->
  <!-- <component :is="activeTab === 'form' ? FormTab : ChartTab" /> -->

  <!-- ✅ З KeepAlive: стан збережено, компонент не перемонтовується -->
  <KeepAlive :include="['FormTab', 'ChartTab']" :max="10">
    <component :is="activeTab === 'form' ? FormTab : ChartTab" />
  </KeepAlive>
</template>
```

```vue
<!-- FormTab.vue -->
<script setup>
import { ref, onActivated, onDeactivated } from 'vue'

const formData = ref({ name: '', email: '' })

onActivated(() => {
  console.log('Форма активована (або першого разу показана)')
})

onDeactivated(() => {
  console.log('Форма дезактивована, але не видалена')
})
</script>

<template>
  <form>
    <input v-model="formData.name" placeholder="Ім'я" />
    <input v-model="formData.email" placeholder="Email" />
  </form>
</template>
```

---

## Suspense — обробка асинхронних операцій у setup. Vue 3 експериментально.

`<Suspense>` дозволяє компоненту з `async setup()` чекати на результат до рендерингу. Показує fallback контент тоді час очікування.

```vue
<!-- AsyncDataComponent.vue -->
<script setup>
import { ref } from 'vue'

// Компонент очікує на цей Promise перед рендерингом
const data = await (async () => {
  const response = await fetch('/api/data')
  if (!response.ok) throw new Error('Помилка завантаження')
  return response.json()
})()

const error = ref(null)
</script>

<template>
  <div>
    <h2>Дані завантажені:</h2>
    <pre>{{ JSON.stringify(data, null, 2) }}</pre>
  </div>
</template>
```

```vue
<!-- App.vue — Suspense обгортка -->
<script setup>
import AsyncDataComponent from './AsyncDataComponent.vue'
</script>

<template>
  <!-- Suspense показує fallback поки AsyncDataComponent завантажується -->
  <Suspense>
    <template #default>
      <AsyncDataComponent />
    </template>

    <!-- Fallback контент під час очікування -->
    <template #fallback>
      <div class="loading-skeleton">
        <div class="skeleton-box"></div>
        <div class="skeleton-box"></div>
      </div>
    </template>
  </Suspense>
</template>

<style>
.loading-skeleton {
  display: flex;
  gap: 1rem;
}

.skeleton-box {
  width: 100px;
  height: 50px;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
</style>
```

---

## Dynamic components (<component :is>) та асинхронні компоненти (defineAsyncComponent).

```vue
<!-- DynamicTabs.vue -->
<script setup>
import { ref, defineAsyncComponent } from 'vue'
import FormTab from './FormTab.vue'

const activeTab = ref('form')

// ✅ Асинхронне завантаження (code splitting)
const ChartTab = defineAsyncComponent(
  () => import('./ChartTab.vue')
)

const SettingsTab = defineAsyncComponent({
  loader: () => import('./SettingsTab.vue'),
  loadingComponent: LoadingComponent, // fallback під час завантаження
  delay: 200,
  timeout: 10000, // Таймаут
  errorComponent: ErrorComponent // fallback при помилці
})

const components = { FormTab, ChartTab, SettingsTab }
</script>

<template>
  <div>
    <!-- Динамічне перемикання компонентів -->
    <component :is="components[activeTab]" />

    <button
      v-for="tab in Object.keys(components)"
      :key="tab"
      @click="activeTab = tab"
    >
      {{ tab }}
    </button>
  </div>
</template>
```

---

## Error handling: errorCaptured hook та app-level errorHandler.

```vue
<!-- App.vue — глобальна обробка помилок -->
<script setup>
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Глобальний обробник помилок
app.config.errorHandler = (err, instance, info) => {
  console.error(`Помилка в ${info}: ${err.message}`)
  // Відправити у Sentry, Bugsnag, тощо
  // trackError(err, { info })
}

app.mount('#app')
</script>

<!-- Component.vue — локальна обробка помилок -->
<script setup>
import { onErrorCaptured, ref } from 'vue'

const errorMessage = ref('')

// Ловить помилки у дочірніх компонентах
onErrorCaptured((err, instance, info) => {
  errorMessage.value = `Помилка у ${info}: ${err.message}`
  
  // Вернути false для обробки, або true для поширення
  return false // Не пускати помилку вище
})
</script>

<template>
  <div v-if="errorMessage" class="error">
    {{ errorMessage }}
  </div>
  <ComponentThatMightError />
</template>
```

---

## Типові пастки при роботі з lifecycle та компонентами.

### 1. Таймери в onMounted без cleanup

```vue
<!-- ❌ ПОМИЛКА: таймер ніколи не зупиняється -->
<script setup>
import { onMounted } from 'vue'

onMounted(() => {
  setInterval(() => {
    console.log('Тик') // Буде тікати ще після видалення компонента!
  }, 1000)
})
</script>

<!-- ✅ ПРАВИЛЬНО: видалити в onUnmounted -->
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const timerId = ref(null)

onMounted(() => {
  timerId.value = setInterval(() => console.log('Тик'), 1000)
})

onUnmounted(() => {
  clearInterval(timerId.value)
})
</script>
```

### 2. Мутація props безпосередньо

```vue
<!-- ❌ ПОМИЛКА: мутація prop -->
<script setup>
import { defineProps } from 'vue'

const props = defineProps({ count: Number })

// ❌ Не робіть так!
props.count++

// ✅ Правильно: emit подія
const emit = defineEmits(['update:count'])
const increment = () => emit('update:count', props.count + 1)
</script>
```

### 3. Нескінченні цикли в watch

```vue
<!-- ❌ ПОМИЛКА: watch викликає вихідну дану, яка оновлює watch -->
<script setup>
import { ref, watch } from 'vue'

const count = ref(0)

watch(count, (newVal) => {
  count.value++ // ❌ Нескінченний цикл!
})
</script>

<!-- ✅ ПРАВИЛЬНО: watch зміни, не причину -->
<script setup>
import { ref, watch } from 'vue'

const count = ref(0)
const doubled = ref(0)

watch(count, (newVal) => {
  doubled.value = newVal * 2 // ✅ Окремена змінна
})
</script>
```

### 4. Забування про dependency array в watch

```vue
<!-- ❌ ПОМИЛКА: watch першої версії без dependencies -->
<script setup>
import { ref, watch } from 'vue'

const count = ref(0)

watch(() => count.value, () => {
  // Ця функція називається ЩОРАЗУ, коли count змінюється
  // Якщо в ній API запит — це N запитів для N змін!
})
</script>

<!-- ✅ ПРАВИЛЬНО: для дорогих операцій використовуйте throttle/debounce -->
<script setup>
import { ref, watch } from 'vue'
import { debounce } from 'lodash-es'

const searchQuery = ref('')

const handleSearch = debounce(async (query) => {
  const results = await fetch(`/api/search?q=${query}`)
}, 300)

watch(searchQuery, handleSearch)
</script>
```

### 5. Забування про cleanup у composables

```vue
<!-- ❌ ПОМИЛКА: composable не очищує ресурси -->
<script setup>
import { useMouseTracking } from './composables'

// Ця функція створює слухача eventi, але не видаляє його!
const { x, y } = useMouseTracking()
</script>

<!-- ✅ ПРАВИЛЬНО: composable з автоматичним cleanup -->
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

function useMouseTracking() {
  const x = ref(0)
  const y = ref(0)

  const handleMouseMove = (event) => {
    x.value = event.clientX
    y.value = event.clientY
  }

  onMounted(() => {
    window.addEventListener('mousemove', handleMouseMove)
  })

  onUnmounted(() => {
    window.removeEventListener('mousemove', handleMouseMove)
  })

  return { x, y }
}
</script>
```

