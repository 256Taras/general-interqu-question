# Web Performance для Senior Engineer

## Що таке Critical Rendering Path?

Critical Rendering Path (CRP) -- це послідовність кроків, які браузер виконує для перетворення HTML, CSS та JavaScript на піксели на екрані. Оптимізація CRP критична для швидкого першого відображення контенту.

Браузер проходить наступні етапи:

```
1. HTML Parsing → 2. DOM Tree Creation
                ↓
3. CSS Parsing → 4. CSSOM Creation
                ↓
5. Render Tree (DOM + CSSOM)
                ↓
6. Layout (обчислення позицій та розмірів)
                ↓
7. Paint (малювання пікселів)
                ↓
8. Composite (з'єднання шарів)
```

Детальніша ASCII-діаграма:

```
┌────────────────────────────────────────────────────────┐
│                    HTML Document                        │
└─────────────────────┬──────────────────────────────────┘
                      │
        ┌─────────────┴──────────────┐
        ▼                            ▼
   ┌────────────┐            ┌──────────────┐
   │ DOM Parser │            │  CSS Parser  │
   └─────┬──────┘            └──────┬───────┘
         │                          │
         ▼                          ▼
    ┌────────┐              ┌──────────┐
    │  DOM   │              │   CSSOM  │
    │  Tree  │              │  Rules   │
    └────┬───┘              └────┬─────┘
         │                       │
         └───────────┬───────────┘
                     ▼
            ┌─────────────────┐
            │  Render Tree    │
            │ (visible nodes  │
            │  + styles)      │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │  Layout         │ ← обчисління позицій (reflow)
            │  (box model)    │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │  Paint          │ ← відображення (repaint)
            │  (rasterize)    │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │  Composite      │ ← з'єднання шарів (GPU)
            │  (GPU layers)   │
            └─────────────────┘
```

**Ключові моменти:**
- DOM та CSSOM конструюються паралельно
- JavaScript блокує парсинг HTML (якщо не вказаний `defer` або `async`)
- CSS блокує парсинг HTML та перевідображення (Render Tree не можна побудувати без CSSOM)
- Paint та Composite коштують дорого; потрібно мінімізувати

```html
<!-- ПОГАНО: блокує парсинг -->
<head>
  <script src="blocking.js"></script>
  <link rel="stylesheet" href="style.css">
</head>

<!-- ДОБРЕ: асинхронна загрузка -->
<head>
  <link rel="stylesheet" href="style.css">
  <script src="deferred.js" defer></script>
</head>
```

---

## Що такі Core Web Vitals та як їх вимірювати?

Core Web Vitals -- це три метрики, які Google вважає найкритичнішими для user experience. Вони впливають на SEO рейтинг.

### LCP (Largest Contentful Paint)

Час до найбільшого видимого елементу контенту.

- **Ціль:** < 2500 мс (Good), < 4000 мс (Needs Improvement)
- **Міряє:** Коли основний контент стає видимим користувачеві
- **Елементи:** `<img>`, `<video>`, `<div>` з фоновим зображенням, текстові блоки

```javascript
// Вимірювання LCP
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1];
  console.log("LCP:", lastEntry.renderTime || lastEntry.loadTime);
});

observer.observe({ entryTypes: ["largest-contentful-paint"] });
```

### INP (Interaction to Next Paint)

Час від користувацької дії (клік, натиск) до наступного відображення (замінив FID).

- **Ціль:** < 200 мс (Good), < 500 мс (Needs Improvement)
- **Міряє:** Responsiveness界面до користувацьких дій
- **Включає:** input delay + processing time + presentation delay

```javascript
// Вимірювання INP
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach((entry) => {
    console.log("INP:", entry.duration);
  });
});

observer.observe({ entryTypes: ["event"] });
```

### CLS (Cumulative Layout Shift)

Сума неочікуваних змін в макеті сторінки.

- **Ціль:** < 0.1 (Good), < 0.25 (Needs Improvement)
- **Міряє:** Стабільність макету; запобігає акцидентальним кліцям
- **Причини:** завантаження зображень без розмірів, реклама, спливаючі вікна

```javascript
// Вимірювання CLS
let clsValue = 0;
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach((entry) => {
    if (!entry.hadRecentInput) {
      clsValue += entry.value;
      console.log("CLS:", clsValue);
    }
  });
});

observer.observe({ entryTypes: ["layout-shift"] });
```

```html
<!-- ДОБРЕ: зазначаємо width/height для уникнення CLS -->
<img src="image.jpg" width="1200" height="800" alt="фото">

<!-- або на модерних браузерах -->
<img src="image.jpg" loading="lazy" alt="фото" style="aspect-ratio: 16/9;">
```

---

## Lighthouse та PageSpeed Insights

Lighthouse -- це автоматизований інструмент для аудиту якості сайтів. PageSpeed Insights -- це веб-версія Lighthouse від Google.

### Що показує Lighthouse:

- **Performance (0-100):** LCP, FCP, CLS, INP, TBT
- **Accessibility:** контрастність, ARIA-атрибути, сенантична семантика
- **Best Practices:** HTTPS, CSP, deprecated APIs
- **SEO:** мета-теги, OpenGraph, мобільна адаптивність
- **PWA:** service workers, manifest.json

### Як користуватися:

1. Chrome DevTools → Lighthouse → Select device (Mobile/Desktop) → Analyze page load
2. Див. Opportunities та Diagnostics для пропозицій
3. Перевір Performance metrics у деталях

```javascript
// Программатичне користування (npm install lighthouse)
import lighthouse from "lighthouse";
import * as chromeLauncher from "chrome-launcher";

const chrome = await chromeLauncher.launch({ chromeFlags: ["--headless"] });
const options = {
  logLevel: "info",
  output: "json",
  port: chrome.port,
};

const runnerResult = await lighthouse("https://example.com", options);
console.log(runnerResult.lhr.scores); // { performance: 92, accessibility: 95, ... }

await chromeLauncher.kill(chrome.pid);
```

---

## Оптимізація зображень

Зображення часто становлять 50%+ розміру сторінки. Їхня оптимізація критична.

### WebP та AVIF форматів:

- **WebP:** 25-35% менше від JPEG, підтримка прозорості; IE не підтримує
- **AVIF:** на 20% менше від WebP; нова технологія, обмежена підтримка
- **JPEG:** універсальний, але більший розмір
- **PNG:** з'ясовування, тільки якщо потрібна прозорість

```html
<!-- Fallback для старих браузерів -->
<picture>
  <!-- Новіші браузери спробують спочатку AVIF -->
  <source srcset="image.avif" type="image/avif">
  <!-- Потім WebP -->
  <source srcset="image.webp" type="image/webp">
  <!-- Fallback -->
  <img src="image.jpg" alt="опис" loading="lazy">
</picture>
```

### srcset та sizes:

```html
<!-- srcset: браузер вибирає найоптимальніший зображення для екрана -->
<img
  src="image-small.jpg"
  srcset="
    image-small.jpg 480w,
    image-medium.jpg 800w,
    image-large.jpg 1200w
  "
  sizes="(max-width: 600px) 100vw, (max-width: 1200px) 50vw, 33vw"
  alt="адаптивне зображення"
  loading="lazy"
>
```

### lazy loading:

```html
<!-- Браузер завантажує зображення тільки коли воно близько до viewport -->
<img src="below-fold.jpg" loading="lazy" alt="отримає довідку ленивого завантаження">
```

**Інструменти:** ImageOptim (Mac), TinyPNG, Squoosh CLI

---

## Code Splitting та Dynamic Imports

Code splitting -- це розділення bundle на менші чанки, які завантажуються за потребою.

### Route-based splitting (найпоширеніший):

```javascript
// React Router
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));

function App() {
  return (
    <Routes>
      <Route
        path="/dashboard"
        element={
          <Suspense fallback={<div>Завантаження...</div>}>
            <Dashboard />
          </Suspense>
        }
      />
      <Route path="/settings" element={<Settings />} />
    </Routes>
  );
}
```

### Component-level splitting:

```javascript
// Завантаження компонента тільки при його рендері
const Modal = lazy(() => import("./Modal"));

function Form() {
  const [showModal, setShowModal] = useState(false);

  return (
    <>
      <button onClick={() => setShowModal(true)}>Відкрити</button>
      {showModal && (
        <Suspense fallback={null}>
          <Modal onClose={() => setShowModal(false)} />
        </Suspense>
      )}
    </>
  );
}
```

### Dynamic import for heavy libraries:

```javascript
// Завантажуємо бібліотеку тільки коли потрібна
async function handleChartClick() {
  const { Chart } = await import("chart.js");
  const ctx = document.getElementById("myChart").getContext("2d");
  new Chart(ctx, { /* конфіг */ });
}
```

---

## Resource Hints

Resource hints дозволяють браузеру завантажувати ресурси раніше.

```html
<!-- dns-prefetch: розв'язати DNS пізніше -->
<link rel="dns-prefetch" href="https://cdn.example.com">

<!-- preconnect: відкрити з'єднання (DNS + TCP + TLS) раніше -->
<link rel="preconnect" href="https://cdn.example.com">

<!-- preload: завантажити ресурс негайно, використовувати пізніше -->
<link rel="preload" href="font.woff2" as="font" crossorigin>
<link rel="preload" href="critical.css" as="style">

<!-- prefetch: завантажити ресурс з низьким пріоритетом -->
<link rel="prefetch" href="next-page.js">

<!-- modulepreload: завантажити ES модуль у фоні -->
<link rel="modulepreload" href="heavy-module.js">
```

### Коли що використовувати:

| Hint | Коли | Приклад |
|---|---|---|
| `dns-prefetch` | Багато додаткових доменів | соціальні мережі API, аналітика |
| `preconnect` | Критичні ресурси від CDN | шрифти, стилі з CDN |
| `preload` | Ресурси потрібні на поточній сторінці | критичні шрифти, CSS |
| `prefetch` | Ресурси для наступної сторінки | JS для следующой маршруту |
| `modulepreload` | Великі ES модулі | bundles, library chunks |

```html
<!-- Практичний приклад -->
<head>
  <!-- Критичні ресурси -->
  <link rel="preload" href="fonts/Inter.woff2" as="font" crossorigin>
  <link rel="preload" href="critical.css" as="style">

  <!-- Зовнішні API -->
  <link rel="preconnect" href="https://api.example.com">
  <link rel="dns-prefetch" href="https://analytics.google.com">

  <!-- Для наступної сторінки -->
  <link rel="prefetch" href="about.js">
</head>
```

---

## HTTP/2, HTTP/3, Brotli Compression

### HTTP/2 vs HTTP/1.1:

**HTTP/2 переваги:**
- Multiplexing: багато запитів над одним з'єднанням (немає проблеми Head-of-Line blocking)
- Server push: сервер посилає ресурси без запиту клієнта
- Header compression (HPACK)
- Binary framing (ефективніше, ніж текст)

**Результат:** краще на медленних мережах, перевантаженні сервері

### HTTP/3 (QUIC):

- На базі UDP замість TCP
- Криптографія за замовчуванням
- Швидша установка з'єднання
- Кращі перепрацювання пакетів
- **Підтримка:** ~94% браузерів (2026)

```nginx
# Nginx: включення HTTP/2 та HTTP/3
server {
  listen 443 ssl http2;
  listen 443 quic reuseport; # HTTP/3

  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_certificate cert.pem;
  ssl_certificate_key key.pem;

  add_header Alt-Svc 'h3=":443"; ma=86400';

  location / {
    proxy_pass http://backend;
  }
}
```

### Brotli Compression:

Brotli стискає краще за gzip на 15-20%, особливо для текстового контенту.

```nginx
# Nginx: Brotli
load_module modules/ngx_http_brotli_filter_module.so;
load_module modules/ngx_http_brotli_static_module.so;

http {
  brotli on;
  brotli_comp_level 6; # 1-11, за замовчуванням 6
  brotli_types text/plain text/css application/json application/javascript text/xml application/xml;
}
```

```javascript
// Перевірка Brotli підтримки на клієнтові
fetch("/api/data").then((res) => {
  const encoding = res.headers.get("content-encoding");
  console.log("Encoding:", encoding); // "br" (Brotli), "gzip", або не вказаний
});
```

---

## CDN та Caching Headers

### Cache-Control header:

```
Cache-Control: max-age=3600, public, must-revalidate
```

- `max-age`: скільки секунд кешувати
- `public`: браузер + CDN можуть кешувати
- `private`: тільки браузер
- `must-revalidate`: після isteka перевірити з сервером
- `no-cache`: не кешувати; завжди перевіряти з сервером
- `no-store`: ніколи не кешувати

```javascript
// Node.js / Express
app.get("/api/user", (req, res) => {
  res.set("Cache-Control", "private, max-age=600"); // 10 хвилин, тільки браузер
  res.json({ name: "John" });
});

app.get("/assets/logo.png", (req, res) => {
  res.set("Cache-Control", "public, max-age=31536000, immutable"); // 1 рік, статичні
  res.sendFile("logo.png");
});

app.get("/api/posts", (req, res) => {
  res.set("Cache-Control", "no-cache"); // Завжди перевіряти з сервером
  res.json(posts);
});
```

### ETag та Last-Modified:

```javascript
// Браузер не завантажує ресурс, якщо ETag не змінився
app.get("/data.json", (req, res) => {
  const data = JSON.stringify({ /* ... */ });
  const hash = crypto.createHash("md5").update(data).digest("hex");

  res.set("ETag", hash);
  res.set("Cache-Control", "no-cache"); // Перевіряти з сервером

  if (req.get("If-None-Match") === hash) {
    return res.sendStatus(304); // Not Modified
  }

  res.json({ /* ... */ });
});
```

### CDN вибір:

- **Cloudflare:** 200+ POP, дешевий, DDoS protection
- **Fastly:** для крупных медіа, потоковості
- **AWS CloudFront:** інтеграція з AWS
- **Akamai:** глобальні інфраструктури, дорого

---

## Render-blocking Resources

### CSS блокує парсинг:

```html
<!-- БЛОКУЄ: парсинг HTML зупиняється -->
<head>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <!-- Браузер не почне парсити це, поки не завантажить та обробить CSS -->
</body>
```

**Вирішення:**
```html
<!-- 1. Inline критичний CSS -->
<head>
  <style>
    /* Критичні стилі для LCP елементів (~14KB) */
    body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; }
    .hero { /* ... */ }
  </style>
  <link rel="stylesheet" href="style.css" media="print" onload="this.media='all'">
</head>

<!-- 2. Асинхронна загрузка CSS через media queries -->
<link rel="stylesheet" href="dark-theme.css" media="(prefers-color-scheme: dark)">

<!-- 3. Завантажити пізніше -->
<link rel="stylesheet" href="non-critical.css">
```

### JavaScript блокує парсинг:

```html
<!-- БЛОКУЄ: парсинг HTML зупиняється -->
<head>
  <script src="app.js"></script>
</head>

<!-- ДОБРЕ: асинхронна загрузка, виконується коли готово -->
<script src="app.js" async></script>

<!-- КРАЩЕ: виконується після парсингу HTML, перед DOMContentLoaded -->
<script src="app.js" defer></script>
```

```html
<!-- Практичний приклад -->
<html>
  <head>
    <!-- Inline критичний CSS -->
    <style>
      /* Critical CSS від Lighthouse: ~10KB */
      html { margin: 0; padding: 0; }
      body { font-family: sans-serif; }
      .hero { /* ... */ }
    </style>

    <!-- Некритичний CSS асинхронно -->
    <link rel="preload" href="non-critical.css" as="style" onload="this.rel='stylesheet'">
  </head>
  <body>
    <div id="app"></div>

    <!-- Критичний JavaScript відкладається -->
    <script src="vendor.js" defer></script>
    <script src="app.js" defer></script>
  </body>
</html>
```

---

## Web Workers

Web Workers дозволяють виконувати JavaScript в фоні без блокування главного потоку (main thread).

```javascript
// main.js
const worker = new Worker("heavy-calc.js");

// Надіслати дані в worker
worker.postMessage({ numbers: Array.from({ length: 1e7 }, () => Math.random()) });

// Отримати результат
worker.onmessage = (event) => {
  console.log("Результат:", event.data);
};
```

```javascript
// heavy-calc.js (worker thread)
self.onmessage = (event) => {
  const { numbers } = event.data;

  // Важкі обчислення не блокують UI
  const sum = numbers.reduce((a, b) => a + b, 0);
  const avg = sum / numbers.length;

  self.postMessage({ sum, avg });
};
```

### Коли використовувати:

- Обробка великих масивів (sort, filter, map)
- Шифрування / хешування
- Image processing (Canvas manipulation)
- WebGL обчислення
- Синтаксичний аналіз JSON (для великих файлів)

```javascript
// Практичний приклад: обробка великого файлу
const fileInput = document.querySelector("input[type=file]");

fileInput.addEventListener("change", async (e) => {
  const file = e.target.files[0];
  const data = await file.arrayBuffer();

  // Передаємо в worker замість блокування UI
  const worker = new Worker("csv-parser.js");
  worker.postMessage({ buffer: data });

  worker.onmessage = (event) => {
    renderTable(event.data); // 60fps, UI не зависает
  };
});
```

---

## Virtual Scrolling для великих списків

Virtual scrolling (windowing) рендерить тільки видимі елементи, економлячи пам'ять та CPU.

```javascript
// Без virtual scrolling: 10,000 DOM節點= замерзання
const BigList = ({ items }) => (
  <div>
    {items.map((item) => (
      <div key={item.id} style={{ height: 50 }}>
        {item.name}
      </div>
    ))}
  </div>
);

// З react-window: тільки ~30 видимих елементів
import { FixedSizeList } from "react-window";

const BigList = ({ items }) => (
  <FixedSizeList
    height={600}
    itemCount={items.length}
    itemSize={50}
    width="100%"
  >
    {({ index, style }) => (
      <div style={style} key={items[index].id}>
        {items[index].name}
      </div>
    )}
  </FixedSizeList>
);
```

```javascript
// Альтернатива: Intersection Observer для lazy-loading
const InfiniteScroll = ({ items, loadMore }) => {
  const sentinel = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) loadMore();
    });

    if (sentinel.current) observer.observe(sentinel.current);
    return () => observer.disconnect();
  }, [loadMore]);

  return (
    <>
      {items.map((item) => (
        <div key={item.id}>{item.name}</div>
      ))}
      <div ref={sentinel} />
    </>
  );
};
```

---

## Profiling: Chrome DevTools

### Performance Panel:

1. Вкрити DevTools → Performance → запис → user interaction → зупинити запис
2. Див. Flame Chart: які функції кошвеят час?
3. Див. Bottom-Up: яка вкупа коштує найбільше?

```javascript
// Performance API для інструментування
performance.mark("data-fetch-start");

fetch("/api/data")
  .then((res) => res.json())
  .then((data) => {
    performance.mark("data-fetch-end");
    performance.measure(
      "data-fetch",
      "data-fetch-start",
      "data-fetch-end"
    );

    // Зберегти для análizes
    const measure = performance.getEntriesByName("data-fetch")[0];
    console.log(`Fetch час: ${measure.duration}ms`);
  });
```

### Coverage Tab:

1. DevTools → More tools → Coverage
2. Запиши user flow
3. Див. скільки CSS/JS не використовується

**Цінна інформація:** якщо 60% CSS не використовується, є можливість для code splitting

---

## Bundle Analysis

### webpack-bundle-analyzer:

```bash
npm install --save-dev webpack-bundle-analyzer
```

```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require("webpack-bundle-analyzer").BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: "static",
      openAnalyzer: false,
    }),
  ],
};
```

### vite-bundle-visualizer:

```bash
npm install --save-dev vite-bundle-visualizer
```

```javascript
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
});
```

### Стратегія зменшення bundle:

1. **Вилучити великі бібліотеки:** замініть Lodash на native, moment.js на date-fns
2. **Code split:** динамічні імпорти для маршрутів
3. **Tree shake:** вилучіть неживий код (мертвий код)
4. **Lazy load:** fonts, images, heavy components
5. **Кешування:** використовуйте HTTP caching для стабільних ресурсів

```javascript
// Приклад: замінити Lodash
// ПЕРЕД (lodash = 70KB gzipped)
import { debounce } from "lodash";

// ПІСЛЯ (нативний = 0KB)
function debounce(func, wait) {
  let timeout;
  return (...args) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => func.apply(this, args), wait);
  };
}
```

---

## Типові пастки та Performance Checklist

### Типові пастки:

1. **Memory leaks:** eventos, timers, closures не видалятися
2. **N+1 queries:** завантажувати дані в циклі замість batch
3. **Layout thrashing:** читати/писати в DOM чергово (батчити операції)
4. **Reflow avalanche:** зміни стилів, які змінюють layout
5. **Giant bundles:** невідомо про розмір залежностей
6. **Sync XHR:** блокує UI поки чекає відповідь
7. **Полі внутрішні таблиці:** без pagination дорогу для БД

### Performance Checklist:

```markdown
## Performance Checklist

### Images
- [ ] Зображення оптимізовані (WebP/AVIF)
- [ ] srcset для адаптивних дизайнів
- [ ] loading="lazy" для below-fold
- [ ] width/height для уникнення CLS
- [ ] <picture> для фоллбеку

### CSS/JS
- [ ] Inline критичний CSS (~14KB)
- [ ] Code splitting для маршрутів
- [ ] defer/async для скриптів
- [ ] Видаліти неживий код (tree shaking)
- [ ] Асинхронна загрузка не-критичного CSS

### Resource Hints
- [ ] preload для критичних ресурсів
- [ ] prefetch для наступної сторінки
- [ ] preconnect для CDN/API

### Compression
- [ ] GZIP/Brotli увімкнена на сервері
- [ ] HTTP/2 або HTTP/3

### Caching
- [ ] Cache-Control headers встановлени
- [ ] ETag для динамічного контенту
- [ ] CDN використовується

### Metrics
- [ ] LCP < 2500ms
- [ ] INP < 200ms
- [ ] CLS < 0.1
- [ ] FCP < 1800ms
- [ ] Bundle size < 200KB (main)

### Tools
- [ ] Lighthouse score 90+
- [ ] Bundle analyzer перевірений
- [ ] Coverage tab перевірений (< 50% неживого коду)
- [ ] DevTools Performance профільовано
```

---

## Тесто-приклад: оптимізація сторінки

**Початок:** LCP 4200ms, INP 450ms, CLS 0.15, bundle 850KB

**Шаги:**

1. **Inline критичний CSS** → LCP 3100ms (-27%)
2. **Code split маршрути** → bundle 320KB (-62%)
3. **Lazy load зображення** → CLS 0.08 (-47%)
4. **Web Worker для обчислень** → INP 180ms (-60%)
5. **Brotli compression** → мережа -20%

**Результат:** LCP 2100ms ✓, INP 180ms ✓, CLS 0.08 ✓, bundle 140KB ✓
