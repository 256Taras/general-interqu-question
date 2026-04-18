# Моніторинг і Observability

## Що таке Three Pillars of Observability і чим вони відрізняються?

Observability -- це здатність зрозуміти внутрішній стан системи на основі її зовнішніх сигналів. Три стовпи (pillars) -- це три типи сигналів, які ми збираємо: **метрики (metrics)**, **логи (logs)** та **трейси (traces)**. Кожен з них відповідає на різні питання.

```
┌─────────────────────────────────────────────────────────────┐
│                  THREE PILLARS OF OBSERVABILITY              │
│                                                               │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   │
│   │   METRICS     │   │    LOGS       │   │   TRACES      │   │
│   │               │   │               │   │               │   │
│   │ Числові       │   │ Текстові      │   │ Шлях запиту   │   │
│   │ агреговані    │   │ події з       │   │ через         │   │
│   │ дані          │   │ контекстом    │   │ сервіси       │   │
│   │               │   │               │   │               │   │
│   │ "Що           │   │ "Що саме      │   │ "Де саме      │   │
│   │ відбувається  │   │ відбулось     │   │ сповільнення? │   │
│   │ з системою?"  │   │ у момент X?"  │   │ Який span?"   │   │
│   │               │   │               │   │               │   │
│   │ Prometheus,   │   │ Loki, ELK,    │   │ Jaeger,       │   │
│   │ Datadog       │   │ CloudWatch    │   │ Tempo, Zipkin │   │
│   └──────────────┘   └──────────────┘   └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Metrics (метрики)** -- це числові агреговані значення в часі: request rate, CPU usage, queue length. Дешеві у зберіганні (time series), швидкі у запитах, добре підходять для алертів і дашбордів. Недолік -- втрачається контекст конкретного запиту.

**Logs (логи)** -- це дискретні події з максимальним контекстом (user_id, request_id, stack trace). Дорогі у зберіганні та пошуку. Незамінні для root cause analysis, коли вже знаєш приблизно коли і де шукати.

**Traces (трейси)** -- це послідовність операцій (spans) одного запиту, який проходить через декілька сервісів. Показує таймінг і залежності. Без них у мікросервісах майже неможливо зрозуміти, де саме "тормозить".

```
Запит користувача -> API Gateway -> Auth Service -> Order Service -> Payment Service
                        50ms          30ms            200ms            800ms  <- знайшли виновника
```

Коли який використовувати:
- Алерт "error rate > 1%" -- metrics
- "Чому саме в 14:32 впав цей конкретний запит?" -- logs
- "Чому p99 latency саме цього endpoint виросла?" -- traces

---

## Яка різниця між Monitoring і Observability?

**Monitoring** -- це збір заздалегідь визначених метрик і алертинг на відомих проблемах. Ти знаєш наперед, що моніторити: CPU, memory, error rate, latency. Працює добре, коли система передбачувана і ти знаєш її failure modes.

**Observability** -- ширша концепція. Це здатність **питати систему нові питання**, які ти раніше не передбачав, не додаючи нову інструментацію. Потрібна, коли failure modes непередбачувані -- тобто у складних розподілених системах.

```
Monitoring:  "Known unknowns"  -- знаю, що можу не знати CPU -> моніторю CPU
Observability: "Unknown unknowns" -- навіть не знаю, на що подивитись
```

Класичний приклад unknown unknown: API раптово почав повільно відповідати для користувачів з певного регіону, з певним типом підписки, між 14:00 і 15:00. Метрики показують тільки "p95 зріс" -- не ясно чому. З observability (high-cardinality labels або traces з багатим контекстом) ти можеш:

```promql
# "Згрупуй latency по region, subscription_tier, endpoint -- де саме проблема?"
histogram_quantile(0.95,
  sum by (region, subscription_tier, endpoint, le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Але тут виникає проблема **high cardinality**. Якщо у метриці багато labels з великою кількістю унікальних значень (user_id, request_id), кількість комбінацій (time series) вибухає. Prometheus цього не любить -- він зберігає кожну комбінацію labels як окрему time series у пам'яті.

| Аспект | Monitoring | Observability |
|---|---|---|
| Питання | Наперед відомі | Довільні, ad-hoc |
| Дані | Агреговані метрики | Metrics + logs + traces + events |
| Інструменти | Prometheus + Grafana | + OpenTelemetry, Honeycomb, Tempo |
| Cardinality | Низька (controlled) | Висока (потребує спец. сховищ) |
| Cost | Низький | Вищий |

---

## Як працює Prometheus? Яка його архітектура?

Prometheus -- це time-series база даних з **pull-based** моделлю збору метрик. Замість того щоб сервіси пушили метрики в Prometheus, сам Prometheus періодично опитує (scrape) HTTP-endpoint `/metrics` на кожному таргеті.

```
                    ┌──────────────────────────────┐
                    │     PROMETHEUS SERVER        │
                    │                               │
                    │  ┌─────────┐   ┌──────────┐ │
                    │  │ Scraper  │──▶│   TSDB    │ │
                    │  └────┬────┘   └────┬─────┘ │
                    │       │             │        │
                    │       ▼             ▼        │
                    │  ┌─────────┐   ┌──────────┐ │
                    │  │ Service  │   │ PromQL    │ │
                    │  │ Discovery│   │ Engine    │ │
                    │  └─────────┘   └────┬─────┘ │
                    └──────────────────────┼───────┘
                           │ pull          │
              ┌────────────┼────────────┐  │
              ▼            ▼            ▼  ▼
         ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐
         │ app-1   │  │ app-2   │  │ node_   │  │ Grafana  │
         │/metrics │  │/metrics │  │exporter │  │Alertmgr  │
         └─────────┘  └─────────┘  └─────────┘  └──────────┘
```

Основні компоненти:
- **Prometheus Server** -- скрапить, зберігає, обчислює PromQL.
- **Exporters** -- "перекладачі" для систем, які самі не віддають метрики у форматі Prometheus (`node_exporter` для ОС, `mysqld_exporter`, `redis_exporter`).
- **Pushgateway** -- виняток з pull-моделі. Потрібен для short-lived jobs (cron, batch), які не встигають бути "скрапнутими".
- **Alertmanager** -- окремий сервіс для обробки, дедуплікації, routing алертів.
- **Service Discovery** -- автоматичне знаходження таргетів у Kubernetes, Consul, EC2, DNS.

**Чому pull, а не push?** Pull-модель має ряд плюсів:
- Легко виявити, що таргет "впав" (scrape fails).
- Легше тестувати локально (просто curl `/metrics`).
- Централізований контроль над частотою збору.
- Менше ризиків DoS від сервісів у Prometheus.

Мінус -- треба щоб Prometheus міг мережево досягнути всіх таргетів.

Приклад конфігурації `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s       # як часто скрапити
  evaluation_interval: 15s   # як часто обчислювати alerting rules

# Підключаємо правила алертів
rule_files:
  - "alerts/*.rules.yml"

# Надсилаємо алерти в Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  # Статичний таргет: Node.js додаток
  - job_name: "node-api"
    static_configs:
      - targets: ["api-1:3000", "api-2:3000"]
    metrics_path: /metrics

  # Service discovery у Kubernetes
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Скрапимо лише поди з анотацією prometheus.io/scrape=true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
```

---

## Які є типи метрик у Prometheus? Що таке cardinality explosion?

Prometheus має 4 типи метрик:

**1. Counter** -- монотонно зростаюче число. Скидається тільки при рестарті. Приклади: `http_requests_total`, `errors_total`. Ніколи не використовуй counter для значень, які можуть зменшуватись.

**2. Gauge** -- значення, яке може зростати і зменшуватись. Температура, черга, кількість активних з'єднань. `memory_bytes`, `queue_length`.

**3. Histogram** -- розподіл значень по заздалегідь визначеним "bucket-ам". Створює декілька time series: `_bucket`, `_count`, `_sum`. Дозволяє обчислити квантилі на стороні сервера через `histogram_quantile()`.

**4. Summary** -- теж розподіл, але квантилі обчислюються на стороні клієнта. Точніші, але не агрегуються між інстансами (не можна взяти середній p95 по 10 подах).

```
Histogram = buckets (агрегований, сервер-side quantile)
  http_request_duration_seconds_bucket{le="0.1"} 124
  http_request_duration_seconds_bucket{le="0.5"} 567
  http_request_duration_seconds_bucket{le="1.0"} 890
  http_request_duration_seconds_bucket{le="+Inf"} 950
  http_request_duration_seconds_count 950
  http_request_duration_seconds_sum 234.5

Summary = quantiles на клієнті (не агрегується між інстансами!)
  http_request_duration_seconds{quantile="0.5"} 0.23
  http_request_duration_seconds{quantile="0.95"} 0.89
  http_request_duration_seconds{quantile="0.99"} 1.45
```

Правило на співбесіді: **якщо запитують "histogram чи summary" -- 99% випадків histogram**, бо він агрегується між подами.

**Cardinality explosion** -- головна пастка Prometheus. Кожна унікальна комбінація `metric_name{label1=value1, label2=value2}` -- це окрема time series у пам'яті. Якщо ти додаси `user_id` як label і маєш 1M користувачів -- у тебе 1M time series на одну метрику. Prometheus "впаде" по пам'яті.

```javascript
// ПОГАНО: user_id як label -> cardinality explosion
requestsCounter.inc({ user_id: req.user.id, endpoint: req.path });

// ДОБРЕ: тільки low-cardinality labels
requestsCounter.inc({ method: req.method, endpoint: req.route.path, status: res.statusCode });
```

Безпечний діапазон cardinality для одного label: одиниці--сотні значень. Route pattern (`/users/:id`) -- OK, конкретний user_id -- ні. Для розрізлення по user_id використовуй logs або traces.

---

## PromQL: як писати базові запити?

PromQL -- мова запитів Prometheus. Працює з двома типами даних: **instant vector** (значення в момент часу) і **range vector** (значення за проміжок `[5m]`).

Функція `rate()` -- найважливіша. Повертає середню швидкість зростання counter'а за секунду:

```promql
# Кількість запитів на секунду за останні 5 хв
rate(http_requests_total[5m])

# Error rate (%) -- скільки запитів з 5xx від усіх
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# Топ 5 найповільніших endpoints (p95)
topk(5,
  histogram_quantile(0.95,
    sum by (endpoint, le) (rate(http_request_duration_seconds_bucket[5m]))
  )
)

# Сума CPU по pod, згруповано по service
sum by (service) (rate(container_cpu_usage_seconds_total[5m]))

# Скільки інстансів не відповідають
count(up{job="node-api"} == 0)
```

Важливі правила:
- `rate()` завжди на counter, ніколи на gauge.
- `rate(metric[5m])` -- діапазон має бути мінімум у 4 рази більший за scrape_interval (якщо scrape 15s -> мінімум 1m).
- `sum by (labels)` -- правильний порядок: спершу `rate()`, потім `sum by()`. Не навпаки.
- `histogram_quantile(0.95, sum by (le) (...))` -- `le` **обов'язково** має бути у групуванні.

**Класичні запити (золотий набір):**

```promql
# 1. Request rate (RED - Rate)
sum by (service) (rate(http_requests_total[5m]))

# 2. Error rate (RED - Errors)
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
  /
sum by (service) (rate(http_requests_total[5m]))

# 3. Latency p95 (RED - Duration)
histogram_quantile(0.95,
  sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))
)

# 4. Saturation: CPU usage (USE - Utilization)
avg by (instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))

# 5. Апроксимація apdex score
(
  sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="1.2"}[5m])) / 2
) / sum(rate(http_request_duration_seconds_count[5m]))
```

---

## Як налаштувати алерти в Prometheus і Alertmanager?

Prometheus оцінює alerting rules (PromQL вирази, які повертають true/false) і пушить активні алерти у Alertmanager. Alertmanager відповідає за **routing, grouping, dedup, silencing, inhibition, notification**.

```
Prometheus                    Alertmanager
────────                      ────────────
evaluates rules   ─────push──▶  groups alerts
every 15s                       deduplicates
                                applies silences
                                routes by labels
                                sends to Slack/PD/Email
```

Приклад `alerts.rules.yml`:

```yaml
groups:
  - name: http-api
    interval: 30s
    rules:
      # Симптом-алерт: користувачі бачать 5xx
      - alert: HighErrorRate
        expr: |
          sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
            /
          sum by (service) (rate(http_requests_total[5m]))
          > 0.01
        for: 5m   # алерт спрацює, тільки якщо умова тримається 5 хвилин
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "Error rate > 1% на {{ $labels.service }}"
          description: "Сервіс {{ $labels.service }} повертає 5xx у {{ $value | humanizePercentage }} запитів"
          runbook: "https://wiki/runbooks/high-error-rate"

      # Симптом: латентність
      - alert: HighLatencyP95
        expr: |
          histogram_quantile(0.95,
            sum by (service, le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 1
        for: 10m
        labels:
          severity: warning

  - name: infrastructure
    rules:
      # Причина: інстанс мертвий
      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
```

Конфіг `alertmanager.yml` з routing та inhibition:

```yaml
route:
  receiver: "default"
  group_by: ["alertname", "service"]
  group_wait: 30s        # чекаємо 30s, щоб зібрати схожі алерти разом
  group_interval: 5m     # нове повідомлення по тій же групі не частіше 5m
  repeat_interval: 4h    # не нагадувати про той самий алерт частіше 4h
  routes:
    - match:
        severity: critical
      receiver: "pagerduty"
    - match:
        team: backend
      receiver: "slack-backend"

receivers:
  - name: "default"
    slack_configs:
      - channel: "#alerts"
  - name: "pagerduty"
    pagerduty_configs:
      - service_key: "..."
  - name: "slack-backend"
    slack_configs:
      - channel: "#backend-alerts"

# Inhibition: якщо впав цілий instance -- не надсилати алерти про low-level проблеми цього інстансу
inhibit_rules:
  - source_match:
      alertname: InstanceDown
    target_match_re:
      alertname: "(HighLatency|HighErrorRate)"
    equal: ["instance"]
```

Корисні концепції:
- **`for:`** -- "flap protection". Алерт не спрацює, якщо умова короткочасна.
- **Grouping** -- 100 подів з тим же алертом -> одне повідомлення.
- **Silencing** -- тимчасово заглушити алерт (наприклад, на час maintenance).
- **Inhibition** -- "якщо горить корінь, не повідомляй про листя".

---

## Як будувати дашборди в Grafana?

Grafana -- це візуалізатор поверх datasources (Prometheus, Loki, Tempo, PostgreSQL, CloudWatch). Сам не зберігає дані.

Ключові концепції:
- **Datasource** -- підключення до джерела метрик/логів.
- **Panel** -- один графік/таблиця/stat. Має запит і візуалізацію.
- **Dashboard** -- набір панелей, збережений як JSON.
- **Variables** -- плейсхолдери типу `$service`, `$region`, що дозволяють перемикатись між фільтрами.
- **Annotations** -- маркери подій на графіках (deploy, incident).

Фрагмент dashboard JSON з templated variables:

```json
{
  "title": "API Service Overview",
  "templating": {
    "list": [
      {
        "name": "service",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(http_requests_total, service)",
        "refresh": 1
      },
      {
        "name": "percentile",
        "type": "custom",
        "options": [
          {"text": "p50", "value": "0.5"},
          {"text": "p95", "value": "0.95"},
          {"text": "p99", "value": "0.99"}
        ]
      }
    ]
  },
  "panels": [
    {
      "title": "Request Rate by Service",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum by (service) (rate(http_requests_total{service=~\"$service\"}[5m]))",
          "legendFormat": "{{ service }}"
        }
      ]
    },
    {
      "title": "Latency ($percentile)",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile($percentile, sum by (service, le) (rate(http_request_duration_seconds_bucket{service=~\"$service\"}[5m])))"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "s",
          "thresholds": {
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 0.5},
              {"color": "red", "value": 1.0}
            ]
          }
        }
      }
    }
  ]
}
```

Принципи якісних дашбордів:
- **Top-down**: зверху -- загальний стан сервісу (RED), знизу -- деталі по інстансах.
- **Одна ціль -- одна панель**. Не треба змішувати метрики непов'язаних сервісів.
- **Одиниці виміру** (seconds, bytes, %) завжди вказані.
- **Thresholds** візуально виділені (green/yellow/red).
- Не зловживати Grafana-алертами -- краще робити алерти у Prometheus через rule files (вони в Git, review-able).

---

## Як організувати логування? Чому не просто console.log?

`console.log("user logged in")` -- неструктурований лог. Пошук "хто залогінився між 14:00 і 15:00 з Європи" вимагатиме regex-магії по текстовим рядкам. Це не масштабується.

**Structured logging** -- логи як JSON з полями. Агрегатор (Loki, Elasticsearch, CloudWatch Insights) індексує поля і дозволяє робити запити типу SQL.

```javascript
// Погано: unstructured
console.log(`User ${userId} logged in from ${ip} at ${new Date()}`);

// Добре: structured
logger.info({
  event: "user_login",
  user_id: userId,
  ip,
  region: geoLookup(ip).region,
  request_id: req.id,
}, "user logged in");
```

Виглядатиме у лог-агрегаторі так:

```json
{
  "level": "info",
  "time": "2026-04-17T14:32:11.234Z",
  "event": "user_login",
  "user_id": "u_123",
  "ip": "1.2.3.4",
  "region": "eu-west",
  "request_id": "req_abc",
  "service": "auth-api",
  "msg": "user logged in"
}
```

Запит у Loki: `{service="auth-api"} | json | event="user_login" | region="eu-west"`.

**Log levels** -- стандартна ієрархія:
- `TRACE` / `DEBUG` -- у dev, вимикаються у prod.
- `INFO` -- нормальні бізнес-події (запит, вхід, платіж).
- `WARN` -- ненормально, але система справляється (retry, fallback).
- `ERROR` -- зламано щось, що вимагає уваги.
- `FATAL` -- додаток не може продовжити роботу.

**Популярні стеки агрегації:**

| Стек | Особливості |
|---|---|
| ELK (Elasticsearch + Logstash + Kibana) | Потужний пошук, високий cost, треба тюнити |
| EFK (Fluentd замість Logstash) | Легший, популярний у Kubernetes |
| Loki + Grafana | "Prometheus для логів", індексує тільки labels, дешевий |
| CloudWatch Logs + Insights | Managed AWS, зручно якщо вже на AWS |
| Datadog Logs | SaaS, дорогий, але feature-rich |

**Золоті правила логування:**
- Завжди включай `request_id` / `trace_id` -- це ключ для correlation з traces.
- Не логуй PII (паролі, токени, ПІБ у повідомленнях банку). Санітайзуй.
- Не логуй у hot path (цикл обробки кожного елемента). Лог -- це I/O.
- Sampling для high-volume подій (наприклад, 1% успішних запитів, 100% помилок).

---

## Що таке distributed tracing і OpenTelemetry?

У мікросервісах один запит проходить через 5--20 сервісів. Логи кожного сервісу розкидані. **Trace** -- це "дерево" викликів одного запиту:

```
Trace: "POST /api/orders"  trace_id=abc123
│
├─ Span: API Gateway (5ms)                    span_id=s1
│   └─ Span: Auth Service (30ms)              span_id=s2, parent=s1
│       └─ Span: DB query users (15ms)        span_id=s3, parent=s2
│
├─ Span: Order Service (200ms)                span_id=s4, parent=s1
│   ├─ Span: DB insert order (50ms)           span_id=s5, parent=s4
│   └─ Span: Payment Service (120ms)          span_id=s6, parent=s4
│       └─ Span: Stripe HTTP call (100ms) <-- БАГАТО ЧАСУ ТУТ
│
└─ Span: Notification Service (async, 20ms)   span_id=s7, parent=s1
```

**Span** -- одиниця роботи з ім'ям, start/end timestamp, attributes, events. Має **trace_id** (загальний для всього запиту) і **span_id** (унікальний для цього span'у), а також **parent_span_id**.

**Trace context propagation** -- механізм передачі trace_id/span_id через сервіси. Стандарт W3C Trace Context використовує HTTP-заголовки:

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
            │  │                                 │                  │
            │  └ trace-id (16 bytes)             └ parent-span-id   └ flags
            └ version                                                 (sampled)
```

Коли сервіс A викликає сервіс B, він **обов'язково** має передати `traceparent`, інакше trace "розривається".

**OpenTelemetry (OTel)** -- стандарт і SDK для інструментації. Vendor-neutral: тим самим кодом збирай дані і шли в Jaeger, Tempo, Datadog, Honeycomb.

```
[App + OTel SDK] -> [OTel Collector] -> Jaeger / Tempo / Datadog
                                    \-> Prometheus (metrics)
                                    \-> Loki (logs)
```

Приклад інструментації Node.js:

```javascript
// tracing.js -- запускати першим у app
const { NodeSDK } = require("@opentelemetry/sdk-node");
const { OTLPTraceExporter } = require("@opentelemetry/exporter-trace-otlp-http");
const { getNodeAutoInstrumentations } = require("@opentelemetry/auto-instrumentations-node");
const { Resource } = require("@opentelemetry/resources");

const sdk = new NodeSDK({
  resource: new Resource({
    "service.name": "order-api",
    "service.version": "1.2.3",
    "deployment.environment": process.env.NODE_ENV,
  }),
  traceExporter: new OTLPTraceExporter({
    url: "http://otel-collector:4318/v1/traces",
  }),
  // Автоматично інструментує express, http, pg, redis, mongoose тощо
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

Створення ручного span'у для бізнес-операції:

```javascript
const { trace } = require("@opentelemetry/api");

async function processOrder(order) {
  const tracer = trace.getTracer("order-service");

  return tracer.startActiveSpan("processOrder", async (span) => {
    span.setAttribute("order.id", order.id);
    span.setAttribute("order.amount", order.amount);
    try {
      await validate(order);
      await charge(order);
      await notify(order);
      span.setStatus({ code: 1 }); // OK
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: 2, message: err.message }); // ERROR
      throw err;
    } finally {
      span.end();
    }
  });
}
```

**Sampling strategies** -- збирати 100% traces у продакшені дорого. Варіанти:
- **Head-based sampling** -- рішення при старті trace (наприклад, 1% запитів).
- **Tail-based sampling** -- рішення після завершення trace, на стороні collector'а. Дозволяє зберегти всі "цікаві" traces (з помилками, повільні) і 1% нормальних.

---

## Що таке RED, USE і Four Golden Signals?

Три фреймворки "що моніторити", залежно від типу системи.

**RED (Rate, Errors, Duration)** -- для **сервісів, що обробляють запити** (HTTP API, RPC, черги). Від Tom Wilkie (Grafana Labs).

```
Rate      = requests per second
Errors    = failed requests per second (або %)
Duration  = distribution of request latencies (p50, p95, p99)
```

**USE (Utilization, Saturation, Errors)** -- для **ресурсів** (CPU, disk, network, memory). Від Brendan Gregg.

```
Utilization = % часу, коли ресурс зайнятий
Saturation  = скільки роботи в черзі на ресурс (run queue length)
Errors      = кількість помилок ресурсу (disk errors, packet drops)
```

**Four Golden Signals** -- від Google SRE Book. Найбільш універсальний.

```
Latency     -- час відповіді (окремо для success та error!)
Traffic     -- скільки навантаження (RPS, transactions/s)
Errors      -- скільки помилок (explicit 5xx, implicit wrong response)
Saturation  -- наскільки система "повна" (queue, CPU > 80%)
```

Порівняння:

| Фреймворк | Для чого | Хто автор |
|---|---|---|
| RED | Request-handling services | Tom Wilkie |
| USE | Resources (hardware-ish) | Brendan Gregg |
| Four Golden | Будь-яка user-facing система | Google SRE |

На практиці: RED -- для API, USE -- для інфри (node_exporter), Four Golden -- для SLO і високорівневих дашбордів.

---

## Що таке SLO, SLI, error budget і burn rate?

Концепції з Google SRE, які дають об'єктивну мову для розмови "чи добре працює сервіс".

**SLI (Service Level Indicator)** -- метрика якості сервісу. Зазвичай це співвідношення:

```
SLI = good_events / total_events

Приклади:
- availability  = successful_requests / total_requests
- latency       = requests_faster_than_300ms / total_requests
- correctness   = correct_responses / total_responses
```

**SLO (Service Level Objective)** -- ціль для SLI за період. Формулюється як:

```
99.9% запитів за 30 днів мають повертатись швидше за 300ms
│      │                │
│      │                └ window
│      └ період вимірювання
└ target
```

**Error budget** -- "дозволена доля невдач". Для SLO 99.9% за 30 днів:

```
error_budget = 100% - 99.9% = 0.1%
             = 0.1% * 30 днів * 24 год * 60 хв
             = 43.2 хвилини downtime на місяць
```

Error budget -- це інструмент балансу між reliability і швидкістю feature delivery. Є бюджет -> релізи дозволені. Бюджет вичерпаний -> freeze релізів, фокус на reliability.

**Burn rate** -- як швидко "прогорає" error budget. Якщо за 1 годину втратили 2% бюджету за 30 днів -- це швидкість 14.4x (norm = 1x -> 30 днів). Burn rate alerts попереджають раніше, ніж традиційні поргові алерти.

**Multi-window multi-burn-rate alerts** (з SRE Workbook):

```yaml
# Швидкий burn: за 1 годину спалили 14.4x -> алерт page (critical)
- alert: ErrorBudgetBurnFast
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h])) / sum(rate(http_requests_total[1h]))
    ) > (14.4 * 0.001)
    AND
    (
      sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
    ) > (14.4 * 0.001)
  labels:
    severity: page

# Повільний burn: за 6 годин спалили 6x -> ticket (warning)
- alert: ErrorBudgetBurnSlow
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[6h])) / sum(rate(http_requests_total[6h]))
    ) > (6 * 0.001)
    AND
    (
      sum(rate(http_requests_total{status=~"5.."}[30m])) / sum(rate(http_requests_total[30m]))
    ) > (6 * 0.001)
  labels:
    severity: ticket
```

Другий предикат (коротше вікно) -- це "reset condition": коли проблема припинилась, алерт згасає швидко.

**Як формулювати SLO для HTTP сервісу:**

1. Визнач user journey (що саме важливо для користувача).
2. Вибери SLI: availability + latency -- мінімальний набір.
3. Вибери target: не 100% (цього неможливо), і не довільне число. Починай з поточного стану + невеликий запас.
4. Зафіксуй window: 28/30 днів -- стандарт.

```
Сервіс: checkout-api
SLO 1 (availability): 99.9% запитів до /api/checkout за 28 днів мають повернути 2xx/3xx/4xx (не 5xx)
SLO 2 (latency):      99% запитів до /api/checkout за 28 днів мають завершитись < 500ms
Error budget:         0.1% = 40m 19s downtime на 28 днів
```

---

## Як інструментувати Node.js додаток?

Дві ключові бібліотеки: **prom-client** (для метрик) і **@opentelemetry/*** (для traces, а також метрик через OTel).

**prom-client -- базова інструментація метрик:**

```javascript
const express = require("express");
const client = require("prom-client");

// Реєстр, в який попадають усі метрики
const register = new client.Registry();

// Default metrics: CPU, memory, event loop lag, GC
client.collectDefaultMetrics({ register });

// Counter: загальна кількість запитів
const httpRequestsTotal = new client.Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status"],
  registers: [register],
});

// Histogram: розподіл тривалості запитів
const httpRequestDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration in seconds",
  labelNames: ["method", "route", "status"],
  // Buckets підібрані під типовий веб-сервіс: 10ms..10s
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2.5, 5, 10],
  registers: [register],
});

// Gauge: поточна кількість активних з'єднань
const activeConnections = new client.Gauge({
  name: "http_active_connections",
  help: "Currently active HTTP connections",
  registers: [register],
});

const app = express();

// Middleware для збору метрик
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  activeConnections.inc();

  res.on("finish", () => {
    // ВАЖЛИВО: route pattern, не req.path (уникаємо cardinality explosion)
    const route = req.route?.path || "unknown";
    const labels = { method: req.method, route, status: res.statusCode };

    httpRequestsTotal.inc(labels);
    end(labels);
    activeConnections.dec();
  });

  next();
});

// Endpoint для Prometheus
app.get("/metrics", async (req, res) => {
  res.set("Content-Type", register.contentType);
  res.end(await register.metrics());
});

app.listen(3000);
```

**Custom business metric:**

```javascript
const ordersCreated = new client.Counter({
  name: "orders_created_total",
  help: "Total orders created",
  labelNames: ["payment_method", "tier"],
  registers: [register],
});

async function createOrder(order) {
  // ... бізнес-логіка
  ordersCreated.inc({
    payment_method: order.paymentMethod,  // card/paypal -- low cardinality OK
    tier: order.customerTier,             // free/pro/enterprise -- OK
    // НЕ додавай user_id, order_id -- cardinality explosion!
  });
}
```

**OpenTelemetry metrics (альтернатива або додаток):**

```javascript
const { metrics } = require("@opentelemetry/api");

const meter = metrics.getMeter("order-service");

const orderCounter = meter.createCounter("orders.created", {
  description: "Orders created",
});

orderCounter.add(1, { payment_method: "card", tier: "pro" });
```

OTel добре поєднує metrics + traces + logs в єдину модель, і експортує всюди через collector.

---

## Які типові помилки в моніторингу?

**1. Cardinality explosion**

```javascript
// ПОГАНО
requestCounter.inc({ user_id: req.user.id });  // 1M users -> 1M time series
requestCounter.inc({ path: req.url });         // /users/123 -> кожен id унікальний
requestCounter.inc({ trace_id: req.traceId }); // кожен запит унікальний -> хаос

// ДОБРЕ
requestCounter.inc({ user_tier: req.user.tier }); // free/pro/enterprise
requestCounter.inc({ route: req.route.path });    // /users/:id
```

Для high-cardinality контексту -- logs або traces, не metrics.

**2. Алерти на причини замість симптомів**

```
ПОГАНО (причина):     "CPU > 80%"        -- а якщо сервіс справляється?
ДОБРЕ (симптом):      "p95 latency > 1s" -- користувач реально страждає
```

Алерти на симптоми (user-facing) -- ті, що будять вночі. Алерти на причини -- корисні для dashboards і warning-level.

**3. Alert fatigue**

Занадто багато алертів -> люди ігнорують навіть справжні. Ознаки:
- Більше 10 алертів на день на команду.
- Більшість -- auto-resolved без дії.
- У чаті #alerts люди мутять канал.

Ліки: жорсткий review кожного алерту раз на квартал ("чи був він дією-actionable?"), inhibition rules, group_by у Alertmanager, SLO-based burn rate alerts замість threshold-алертів.

**4. Метрики без одиниць і з незрозумілими назвами**

```
ПОГАНО: app_time, requests_count, memory
ДОБРЕ:  http_request_duration_seconds, http_requests_total, process_resident_memory_bytes
```

Prometheus naming convention: `<namespace>_<name>_<unit>`, суфікси `_total` для counters, одиниці у базовій СІ (seconds, bytes).

**5. Дашборди-смітники**

30 графіків на одну стіну, половина не оновлювалась рік. Правило: якщо на панель ніхто не дивився місяць -- видаляй.

**6. Лог-left-pad**

Логуємо все підряд -> terabyte/day -> $$$$. Не логуй:
- Health-check запити (інакше sampling).
- Великі payload'и цілком.
- Те, що вже є у metrics.

**7. Відсутність runbooks**

Алерт без runbook -- злочин проти чергового. Annotation `runbook_url:` має вести на доку "що перевірити, як дебажити".

**8. Моніторинг моніторингу**

Prometheus впав -- і ти не бачиш, що він впав (бо і дашборди, і алерти проходять через нього). Рішення: Prometheus federation, dead-man-switch алерт (`Watchdog`, який завжди активний; якщо перестав приходити -- значить alerting pipeline зламаний).

```yaml
- alert: Watchdog
  expr: vector(1)
  labels:
    severity: none
  annotations:
    description: "Завжди активний. Якщо відсутній -- зламаний alerting pipeline."
```

---
