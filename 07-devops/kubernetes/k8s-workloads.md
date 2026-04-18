# Kubernetes: Workloads (Deployment, StatefulSet, DaemonSet, Job)

## Що таке Deployment і навіщо він поверх Pods?

Deployment -- це декларативний контролер для stateless застосунків. Він керує **ReplicaSet**, який, своєю чергою, керує Pods.

```
Deployment
   └── ReplicaSet (v1)
         ├── Pod
         ├── Pod
         └── Pod
```

**Що дає Deployment:**
- **Кількість replicas** -- завжди тримає задане число Pods.
- **Rolling updates** -- оновлення без downtime.
- **Rollback** -- повернення до попередньої версії.
- **Self-healing** -- якщо Pod зник, ReplicaSet створює новий.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # скільки +1 Pods допустимо понад replicas
      maxUnavailable: 0    # скільки можуть бути недоступні одночасно
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myregistry/api:v1.2.3
          ports:
            - containerPort: 3000
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 256Mi }
          readinessProbe:
            httpGet: { path: /ready, port: 3000 }
          livenessProbe:
            httpGet: { path: /health, port: 3000 }
```

---

## Як працює Rolling Update і як зробити rollback?

При зміні `spec.template` Deployment створює **новий ReplicaSet** і поступово:
1. Масштабує новий RS вгору.
2. Старий RS вниз.

**З `maxSurge=1, maxUnavailable=0`:**
```
Initial:   [v1][v1][v1]                    # 3 pods v1
Step 1:    [v1][v1][v1][v2]                # +1 v2 (maxSurge)
Step 2:    [v1][v1][v2]                    # -1 v1 (v2 готовий)
Step 3:    [v1][v1][v2][v2]
Step 4:    [v1][v2][v2]
...
Final:     [v2][v2][v2]
```

**Перевірити статус:**
```bash
kubectl rollout status deployment/api
```

**Історія та rollback:**
```bash
kubectl rollout history deployment/api
# REVISION  CHANGE-CAUSE
# 1         Initial deploy
# 2         Upgrade to v1.2.3
# 3         Upgrade to v1.2.4

kubectl rollout undo deployment/api                   # до попередньої
kubectl rollout undo deployment/api --to-revision=2   # до конкретної
```

Щоб CHANGE-CAUSE заповнювалось:
```bash
kubectl annotate deployment api kubernetes.io/change-cause="Upgrade to v1.2.4"
```

**Пауза і resume** (корисно для canary / тестів):
```bash
kubectl rollout pause deployment/api
# ... зміни до Deployment не застосовуються одразу
kubectl rollout resume deployment/api
```

---

## Коли використовувати StatefulSet замість Deployment?

StatefulSet потрібен для застосунків, яким критичні:
- **Стабільне ім'я** -- `mysql-0`, `mysql-1`, `mysql-2` (а не випадкові суфікси).
- **Стабільне мережеве ім'я** -- через headless Service: `mysql-0.mysql.default.svc`.
- **Стабільний storage** -- кожен Pod має свій PersistentVolumeClaim, який не зникає при рестарті.
- **Впорядкований розгортання/завершення** -- `mysql-0` створиться перший, `mysql-2` -- останній. Під час видалення -- у зворотному порядку.

**Приклад:** БД (PostgreSQL, MongoDB), брокери (Kafka, RabbitMQ), розподілені системи (Elasticsearch, etcd).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql        # headless Service
  replicas: 3
  selector:
    matchLabels: { app: mysql }
  template:
    metadata:
      labels: { app: mysql }
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 10Gi
```

Кожен Pod отримає:
- `mysql-0` з PVC `data-mysql-0` (10Gi).
- `mysql-1` з PVC `data-mysql-1` (10Gi).
- `mysql-2` з PVC `data-mysql-2` (10Gi).

**Якщо Pod `mysql-1` падає** -- створюється новий з тим самим іменем і тим самим PVC. Дані зберігаються.

| Deployment                         | StatefulSet                          |
|------------------------------------|---------------------------------------|
| Випадкові імена Pods               | `name-0, name-1, ...`                 |
| Pods взаємозамінні                 | Кожен має ідентичність                |
| Спільний PVC (або без)             | PVC per Pod                           |
| Паралельне створення               | Послідовне                            |
| Stateless                          | Stateful (БД, брокери)                |

---

## Для чого DaemonSet?

DaemonSet запускає **по одному Pod на кожному Node** (або на вибраних через nodeSelector). При додаванні Node -- автоматично з'являється новий Pod.

**Use cases:**
- **Log shippers** -- Fluent Bit, Filebeat, Vector.
- **Metrics agents** -- Node Exporter для Prometheus, Datadog Agent.
- **Network plugins** -- Cilium, Calico.
- **Storage daemons** -- Ceph, GlusterFS agents.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
spec:
  selector:
    matchLabels: { app: fluent-bit }
  template:
    metadata:
      labels: { app: fluent-bit }
    spec:
      tolerations:
        - operator: Exists           # запускатись на всіх вузлах, включно з control plane
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

---

## Що таке Job і CronJob?

**Job** -- для one-time задач, які мають завершитися успішно. Запускає один або кілька Pods до досягнення заданої кількості успішних завершень.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 4           # скільки разів пробувати при фейлі
  activeDeadlineSeconds: 600 # максимальний час виконання
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: myapp:v1.2.3
          command: ["npm", "run", "migrate"]
```

**Use cases:** міграції БД, batch обробка, експорт даних.

**Паралельний Job:**
```yaml
spec:
  parallelism: 5            # 5 pods паралельно
  completions: 100          # виконати 100 успішних завершень
```

**CronJob** -- Job за розкладом:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"                    # 02:00 щоденно (UTC)
  concurrencyPolicy: Forbid                # не запускати паралельно
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: report
              image: reports:v1.0
              command: ["./generate-report.sh"]
```

**Важливо:**
- `concurrencyPolicy: Forbid` -- якщо попередній Job ще біжить, новий скіпається.
- `concurrencyPolicy: Replace` -- новий Job вб'є старий.
- `startingDeadlineSeconds` -- якщо Job не запустився у цей час (наприклад, контроль-плейн лежав) -- скіпається.

---

## Як правильно налаштувати liveness, readiness, startup probes?

**Liveness probe** -- "чи застосунок живий?"
- Якщо fail → контейнер перезапускається.
- Використовуйте для виявлення зависань (deadlock, infinite loop).
- **НЕ ставте тут важкі перевірки** (БД, зовнішні API) -- це спричинить каскадний crash loop.

**Readiness probe** -- "чи готовий приймати трафік?"
- Якщо fail → Pod вилучається з endpoints Service (трафік не йде).
- Контейнер **не** перезапускається.
- Можна перевіряти залежності (БД, кеш).

**Startup probe** -- "чи застосунок вже стартував?"
- Працює до першого успіху, потім вимикається.
- Поки startup не success -- liveness/readiness не запускаються.
- Для повільних застосунків (Java app із JIT warmup, сервіси, що мігрують схему).

```yaml
containers:
  - name: app
    image: myapp:1.0

    startupProbe:
      httpGet: { path: /health, port: 3000 }
      failureThreshold: 30
      periodSeconds: 10       # дозволити 300 сек на старт

    livenessProbe:
      httpGet: { path: /health, port: 3000 }
      periodSeconds: 10
      failureThreshold: 3      # 3 фейли підряд = restart

    readinessProbe:
      httpGet: { path: /ready, port: 3000 }
      periodSeconds: 5
      failureThreshold: 3
```

**Типи probes:**
```yaml
httpGet:
  path: /health
  port: 3000

# Або
tcpSocket:
  port: 3000

# Або
exec:
  command: ["pg_isready", "-U", "postgres"]
```

**Приклад /health vs /ready у застосунку:**
```javascript
// Liveness: тільки подія, що процес живий
app.get('/health', (req, res) => res.send('ok'));

// Readiness: перевірка критичних залежностей
app.get('/ready', async (req, res) => {
  try {
    await db.ping();
    await redis.ping();
    res.send('ready');
  } catch (e) {
    res.status(503).send('not ready');
  }
});
```

---

## Init containers і sidecars

**Init container** -- запускається **перед** основним контейнером. Наступний контейнер не стартує, поки попередній не завершиться успішно.

**Use cases:**
- Очікування на залежність (чекати на БД).
- Завантаження конфігу / сертифікатів.
- Міграція БД перед стартом основного контейнера.

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']

    - name: migrate
      image: myapp:1.0
      command: ['npm', 'run', 'migrate']

  containers:
    - name: app
      image: myapp:1.0
      command: ['node', 'server.js']
```

**Sidecar** -- контейнер, що запускається паралельно з основним, надаючи йому допоміжну функцію:
- Service mesh proxy (Envoy для Istio/Linkerd).
- Log shipper (якщо не через DaemonSet).
- Config reloader, secret rotator.

У Kubernetes 1.29+ з'явилися "native sidecars" -- Init containers з `restartPolicy: Always`. Вони виконуються як sidecars і стартують перед основними.

---

## Graceful shutdown у Kubernetes

Коли Pod видаляється (rolling update, scale down), відбувається:

```
1. API Server позначає Pod як "Terminating"
2. Pod виключається з Service endpoints (трафік зупиняється)
3. preStop hook виконується (якщо є)
4. SIGTERM надсилається до всіх контейнерів
5. Очікування terminationGracePeriodSeconds (дефолт 30s)
6. Якщо контейнер не вийшов → SIGKILL
```

**Проблема:** пункти 2 і 4 виконуються паралельно! Якщо процес вийде швидше, ніж kube-proxy оновить iptables -- in-flight запити провисуть.

**Рішення:** `preStop` hook:
```yaml
containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "10"]       # дати kube-proxy час оновити endpoints
    # Потім SIGTERM → застосунок закриє з'єднання
```

**У застосунку:**
```javascript
process.on('SIGTERM', async () => {
  // Припинити приймати нові запити
  server.close(() => console.log('HTTP closed'));

  // Дочекатися in-flight
  await Promise.all(inFlightRequests);

  // Закрити ресурси
  await db.end();
  process.exit(0);
});
```

`terminationGracePeriodSeconds` -- встановіть з запасом (наприклад, 60 сек), якщо запити можуть бути довгими.

---

## Resource requests і QoS класи

Kubernetes присвоює Pods один з трьох QoS класів:

**Guaranteed** -- `requests == limits` для всіх ресурсів. Не вбиваються при тисняві першими.
```yaml
resources:
  requests: { cpu: 500m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 256Mi }
```

**Burstable** -- requests < limits.
```yaml
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 500m, memory: 256Mi }
```

**BestEffort** -- немає requests/limits взагалі. Вбиваються першими при OOM.

**Коли Node під тисняву, k8s вбиває Pods у порядку:** BestEffort → Burstable → Guaranteed.

**Для критичних сервісів (БД, core API)** -- ставте Guaranteed. Для stateless сервісів з бурстами -- Burstable. BestEffort уникайте в production.

---

## Поради з роботи з workloads

**1. Завжди мінімум 2 replicas** для stateless сервісів:
```yaml
replicas: 2
```
Інакше під час rolling update або Node failure -- downtime.

**2. PodDisruptionBudget** -- обмежує, скільки Pods можна одночасно зупинити:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2       # завжди мінімум 2 доступні
  selector:
    matchLabels: { app: api }
```
Захищає від одночасного зупинення при cordon/drain вузлів.

**3. Anti-affinity** -- розосередити Pods по вузлах/зонах:
```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values: [api]
            topologyKey: kubernetes.io/hostname
```

Інакше всі 3 replicas можуть опинитися на одному Node, і падіння Node = повний downtime.

**4. Не деплойте Pods напряму:** завжди через контролер.

**5. Використовуйте Labels послідовно:** `app`, `component`, `version`, `part-of`, `managed-by` (recommended Kubernetes labels).
