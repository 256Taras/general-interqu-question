# Kubernetes: основи

## Що таке Kubernetes і яку проблему він вирішує?

Kubernetes (k8s) -- це платформа-оркестратор для контейнеризованих застосунків. Вона автоматизує розгортання, масштабування, self-healing та керування мережею для контейнерів.

**Проблеми, які вирішує:**
- **Планування (scheduling)** -- де саме (на якому вузлі) запустити контейнер?
- **Self-healing** -- що робити, коли контейнер або вузол впав?
- **Масштабування** -- як додати/зменшити кількість екземплярів залежно від навантаження?
- **Service discovery** -- як контейнеру знайти інший контейнер?
- **Balanced load** -- як розподілити запити між екземплярами?
- **Rolling updates** -- як оновити застосунок без downtime?
- **Конфігурація та секрети** -- як передати конфіг, не перебилдуючи образ?

Docker вирішує, **як запакувати** застосунок. Kubernetes вирішує, **як ним керувати** у production.

---

## Яка архітектура Kubernetes-кластера?

Кластер = **Control Plane** + **Worker Nodes**.

```
┌─────────────────────────── Control Plane ────────────────────────┐
│                                                                  │
│  ┌────────────┐  ┌─────────────┐  ┌──────────┐  ┌─────────────┐ │
│  │ API Server │  │   etcd      │  │Scheduler │  │  Controller │ │
│  │            │  │ (datastore) │  │          │  │   Manager   │ │
│  └────────────┘  └─────────────┘  └──────────┘  └─────────────┘ │
│       ▲                                                          │
└───────┼──────────────────────────────────────────────────────────┘
        │  (kubectl, controllers, nodes читають і пишуть через API)
        │
┌───────┴──────────────────────────────────────────────────────────┐
│                       Worker Nodes                                │
│                                                                   │
│  ┌────────────────────┐   ┌────────────────────┐                  │
│  │   Node 1           │   │   Node 2           │                  │
│  │  ┌───────┐         │   │  ┌───────┐         │                  │
│  │  │kubelet│         │   │  │kubelet│         │                  │
│  │  ├───────┤         │   │  ├───────┤         │                  │
│  │  │kube-  │         │   │  │kube-  │         │                  │
│  │  │proxy  │         │   │  │proxy  │         │                  │
│  │  ├───────┤         │   │  ├───────┤         │                  │
│  │  │Container Runtime│   │  │Container Runtime│                  │
│  │  │(containerd)     │   │  │(containerd)     │                  │
│  │  └───────┘         │   │  └───────┘         │                  │
│  │   Pods...          │   │   Pods...          │                  │
│  └────────────────────┘   └────────────────────┘                  │
└───────────────────────────────────────────────────────────────────┘
```

**Control Plane компоненти:**
- **kube-apiserver** -- REST API, єдиний entry point до кластера. Все інше спілкується через нього.
- **etcd** -- розподілене key-value сховище. Зберігає весь стан кластера (конфігурація, стан pods, secrets).
- **kube-scheduler** -- вибирає вузол для нового pod (дивиться на ресурси, affinity, taints/tolerations).
- **kube-controller-manager** -- запускає контролери (ReplicaSet, Deployment, Node controller тощо). Контролери приводять стан у відповідність до desired state.
- **cloud-controller-manager** -- інтеграція з хмарою (створення LoadBalancer, volumes).

**Worker Node компоненти:**
- **kubelet** -- агент, що спілкується з API Server і запускає pods на вузлі.
- **kube-proxy** -- керує мережевими правилами (iptables/IPVS) для Service.
- **Container runtime** -- containerd / CRI-O, запускає контейнери.

---

## Що таке Pod і чому це мінімальна одиниця?

Pod -- це найменша одиниця деплою в Kubernetes. Це **група з одного або кількох контейнерів**, які:
- запускаються на одному вузлі,
- діляться одним мережевим namespace (localhost, той самий IP),
- діляться storage volumes,
- стартують і завершуються разом.

**Чому не "контейнер"?** Іноді одному компоненту потрібен sidecar -- допоміжний контейнер (logging agent, service mesh proxy, init container). Pod об'єднує їх у логічну одиницю.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 3000
    - name: logging-sidecar
      image: fluentd:1.16
      # обидва бачать один /var/log через shared volume
      volumeMounts:
        - name: logs
          mountPath: /var/log
  volumes:
    - name: logs
      emptyDir: {}
```

**Важливо:** Pods -- **ephemeral**. Якщо Pod падає, k8s створює новий з новим IP. Ви не запускаєте Pods напряму в production -- ви використовуєте Deployment/StatefulSet, які керують Pods.

---

## Що таке desired state і reconciliation loop?

Kubernetes працює за декларативним принципом. Ви описуєте **бажаний стан** (desired state), а k8s постійно приводить **поточний стан** (actual state) до нього.

```
     You: kubectl apply -f deployment.yaml
              │
              ▼
     ┌──────────────────┐
     │  etcd: desired   │  "3 replicas of myapp:1.0"
     │  state           │
     └──────────────────┘
              │
              ▼
     ┌──────────────────┐
     │  Controller loop │
     │  ┌─────────────┐ │
     │  │ 1. observe  │ │  → 2 pods running (one died)
     │  │ 2. diff     │ │  → need 1 more
     │  │ 3. act      │ │  → create new pod
     │  └─────────────┘ │
     └──────────────────┘
              │
              ▼ (повторюється щосекунди)
```

Це ключ до self-healing:
- Контейнер впав? Kubelet рестартує його.
- Pod впав? ReplicaSet створить новий.
- Node впав? Controller переносить Pods на інші вузли.
- Хтось видалив Pod вручну? ReplicaSet створить заміну.

Тому не треба "натискати restart" -- достатньо задати правильний desired state.

---

## Як kubectl спілкується з кластером?

```
kubectl → kubeconfig → API Server (HTTPS) → etcd
                           │
                           └──→ controllers → kubelet → runtime → pods
```

Kubeconfig (`~/.kube/config`) містить:
- **cluster** -- адреса API Server і CA certificate.
- **user** -- credentials (client cert, token, OIDC).
- **context** -- комбінація cluster + user + namespace.

```bash
# Переглянути поточний контекст
kubectl config current-context

# Переключитись між кластерами
kubectl config use-context prod-us-east-1

# Побачити всі контексти
kubectl config get-contexts
```

**Важливо:** `kubectl` ходить до API Server за HTTPS, автентифікується за вашими credentials. Жодних SSH-з'єднань з Nodes у звичайному workflow.

---

## Основні команди kubectl

```bash
# ---- Огляд ----
kubectl get pods                          # pods у поточному namespace
kubectl get pods -A                       # у всіх namespaces
kubectl get pods -o wide                  # + node, IP
kubectl get pods --watch                  # стрімити зміни
kubectl get pods -l app=myapp             # фільтр за label

kubectl describe pod myapp-abc123         # детальна інформація + events

# ---- Логи та debug ----
kubectl logs myapp-abc123                 # логи контейнера
kubectl logs -f myapp-abc123              # стрімити
kubectl logs myapp-abc123 --previous      # з попереднього рестарту
kubectl logs -l app=myapp --tail=100      # з усіх pods з label

kubectl exec -it myapp-abc123 -- sh       # shell у контейнер
kubectl port-forward pod/myapp-abc123 8080:80   # локальний доступ

# ---- Застосування ресурсів ----
kubectl apply -f deployment.yaml          # застосувати/оновити
kubectl apply -k ./overlays/prod          # kustomize
kubectl delete -f deployment.yaml

# ---- Редагування на льоту ----
kubectl edit deployment myapp             # відкриє YAML в $EDITOR
kubectl scale deployment myapp --replicas=5
kubectl rollout restart deployment myapp  # перезапустити всі pods

# ---- Rollout ----
kubectl rollout status deployment myapp
kubectl rollout history deployment myapp
kubectl rollout undo deployment myapp

# ---- Діагностика кластера ----
kubectl get nodes
kubectl top node                          # CPU/memory (потрібен metrics-server)
kubectl top pod
kubectl get events --sort-by=.lastTimestamp
```

---

## Що таке namespace і як ним користуватись?

Namespace -- це віртуальний розподіл ресурсів у кластері. Аналог "папки" для об'єктів.

**Навіщо:**
- Ізоляція середовищ (`dev`, `staging`, `prod`) в одному кластері.
- Розділення команд / тенантів.
- ResourceQuota / LimitRange per namespace.
- RBAC -- дозволи на namespace.

```bash
kubectl create namespace staging
kubectl apply -f deployment.yaml -n staging
kubectl get pods -n staging

# Змінити default namespace у контексті
kubectl config set-context --current --namespace=staging
```

**Важливо:** не всі ресурси namespaced. `Node`, `PersistentVolume`, `ClusterRole`, `StorageClass` -- cluster-scoped, не прив'язані до namespace.

Міжnamespace-ний DNS: `service-name.namespace.svc.cluster.local`.

---

## Labels, Selectors, Annotations -- різниця

**Labels** -- key-value пари для **групування і вибору** об'єктів:
```yaml
metadata:
  labels:
    app: payments
    env: production
    version: v1.2.3
```

**Selector** -- як Service / Deployment знаходить свої Pods:
```yaml
# Deployment
selector:
  matchLabels:
    app: payments

# Pod template
template:
  metadata:
    labels:
      app: payments      # ← збігається з selector
```

**Annotations** -- метадані **без семантики** для k8s. Для людей або зовнішніх інструментів (monitoring, git SHA, changelog):
```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Upgrade Node to 20"
    prometheus.io/scrape: "true"
    git.commit: "a1b2c3d"
```

| Атрибут      | Labels                    | Annotations                 |
|--------------|---------------------------|-----------------------------|
| Призначення  | Ідентифікація, selection  | Метадані для людей/tools    |
| Запит        | Можна фільтрувати          | Не фільтруються             |
| Обмеження    | 63 символи, обмежений формат | Будь-який рядок, до 256KB |

---

## Як k8s приймає рішення, куди планувати Pod?

**Scheduler** обирає вузол у 2 фази:

1. **Filtering** -- відкидає вузли, де Pod не може запуститися:
   - Недостатньо CPU / memory (resources.requests).
   - Не збігаються `nodeSelector` / `nodeAffinity`.
   - Node має taint, pod не має matching toleration.
   - Port conflict, PVC не доступний, тощо.

2. **Scoring** -- ранжує решту вузлів:
   - Найменш завантажений.
   - Spread across zones (для HA).
   - Image locality (якщо образ вже є на вузлі).

**Requests vs Limits:**
```yaml
resources:
  requests:
    cpu: 100m          # 0.1 vCPU -- для scheduling
    memory: 128Mi      # для scheduling (гарантовано)
  limits:
    cpu: 500m          # hard limit (throttling)
    memory: 256Mi      # hard limit (OOMKill)
```

- **Requests** -- скільки гарантовано виділено. Scheduler резервує на вузлі.
- **Limits** -- верхня межа. Перевищення CPU → throttling. Перевищення memory → OOMKilled.

**Правило:** завжди ставте requests і limits. Без них -- Pod в "BestEffort" QoS класі, його вб'ють першим при тисняві.

---

## Як виглядає життєвий цикл Pod?

```
Pending → ContainerCreating → Running ──┬── Succeeded (Job)
   │                             │       └── Failed
   │                             │
   │                         (restart on failure)
   │
   └── якщо не зміг запланувати → Failed
```

Фази:
- **Pending** -- об'єкт створено, pod ще не запустився (тягнеться образ, бракує ресурсів, Init контейнери йдуть).
- **Running** -- хоча б один контейнер стартував.
- **Succeeded** -- всі контейнери завершились з exit code 0 (для Jobs).
- **Failed** -- хоча б один контейнер завершився з помилкою і не рестартує.
- **Unknown** -- Node не відповідає.

**Restart policy:**
- `Always` -- рестартувати завжди (дефолт для Deployment).
- `OnFailure` -- тільки при non-zero exit (Job).
- `Never` -- не рестартувати.

`CrashLoopBackOff` -- не фаза, а статус контейнера: постійно рестартує. k8s збільшує паузу між спробами (10s → 20s → 40s → ... до 5 хв).

---

## Типові пастки для початківців

**1. `latest` тег:**
```yaml
image: myapp:latest    # ❌
```
Новий Pod може витягнути іншу версію з того самого тегу. Використовуйте конкретні версії.

**2. Немає `imagePullPolicy`:**
За замовчуванням `IfNotPresent` (тягне тільки якщо немає локально). Для `latest` дефолт `Always`. Для безпеки -- використовуйте `Always` + конкретний тег.

**3. Відсутні requests/limits:**
Pod у BestEffort QoS, буде вбитий першим. Також scheduler не знає, скільки ресурсів резервувати.

**4. Немає readiness/liveness probes:**
- Без **readiness** -- трафік йде до Pod, який ще не готовий.
- Без **liveness** -- завислий Pod продовжує бути "Running".

**5. Деплой pod напряму (без Deployment):**
Якщо Node падає -- pod не створюється знову. Завжди використовуйте контролер (Deployment / StatefulSet / DaemonSet).

**6. Забуті `--namespace`:**
Подивились `kubectl get pods` в `default`, не побачили своїх -- бо вони в іншому namespace. `-A` для перевірки всіх.
