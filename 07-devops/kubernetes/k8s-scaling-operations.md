# Kubernetes: Масштабування, RBAC, Обслуговування

## Як масштабуються застосунки в Kubernetes?

Є **три рівні масштабування**:

1. **Manual scaling** -- вручну змінюєте кількість replicas.
2. **Horizontal Pod Autoscaler (HPA)** -- автоматично змінює кількість replicas.
3. **Vertical Pod Autoscaler (VPA)** -- автоматично підлаштовує requests/limits.
4. **Cluster Autoscaler** -- додає/видаляє Nodes у кластері.

```
                Load increases
                      │
                      ▼
       ┌────────────────────────────┐
       │  HPA: більше replicas      │
       └────────────────────────────┘
                      │
                      ▼
       ┌────────────────────────────┐
       │  Чи є ресурси на Nodes?    │
       │  Ні → Cluster Autoscaler   │
       │       додає новий Node     │
       └────────────────────────────┘
```

---

## HPA: горизонтальне автомасштабування

HPA моніторить метрики (CPU, memory, кастомні) і змінює `replicas` у Deployment.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70          # середня утилізація CPU по Pods
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0       # масштабувати вгору одразу
      policies:
        - type: Percent
          value: 100                      # максимум подвоїти
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300     # зачекати 5 хв перед зменшенням
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

**Як HPA рахує replicas:**
```
desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)
```

Наприклад: 3 replicas, середній CPU 90%, target 70%:
```
3 × 90 / 70 = 3.86 → 4 replicas
```

**Вимоги:**
- `metrics-server` встановлений у кластері.
- Всі Pods мають `resources.requests` (інакше HPA не знає референсу для utilization).

**Кастомні метрики** (requests/sec, queue length) -- через Prometheus Adapter або KEDA:
```yaml
metrics:
  - type: External
    external:
      metric:
        name: sqs_messages_visible
        selector:
          matchLabels:
            queue: orders
      target:
        type: AverageValue
        averageValue: "100"
```

---

## KEDA: event-driven autoscaling

KEDA (Kubernetes Event-Driven Autoscaling) -- розширення для масштабування на основі зовнішніх подій: довжина черги (SQS, RabbitMQ, Kafka), метрики Prometheus, cron-розклад, розмір S3 bucket тощо.

**Плюс KEDA:** scale to zero -- коли немає подій, Pods = 0. Для HPA мінімум 1 Pod.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: consumer
spec:
  scaleTargetRef:
    name: consumer
  minReplicaCount: 0           # scale to zero!
  maxReplicaCount: 20
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123/orders
        queueLength: "10"
        awsRegion: us-east-1
```

Підходить для worker-подібних застосунків з періодичним навантаженням.

---

## VPA: вертикальне автомасштабування

VPA аналізує реальне споживання Pods і рекомендує (або автоматично застосовує) requests/limits.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Auto"          # Off | Initial | Recreate | Auto
```

**Режими:**
- `Off` -- тільки рекомендації (дивитись через `kubectl describe vpa`).
- `Initial` -- ставить requests при створенні Pod, не чіпає існуючі.
- `Recreate` -- рестартує Pod, щоб застосувати нові requests.

**Важливо:** VPA і HPA по CPU/memory одночасно на той самий Deployment **конфліктують**. Якщо HPA по CPU -- VPA має бути `Off` для CPU (або використовувати кастомні метрики для HPA).

---

## Cluster Autoscaler і Karpenter

**Cluster Autoscaler** -- додає Nodes, коли Pods не можуть бути заплановані через брак ресурсів.

Логіка:
1. Pod у `Pending` стані, причина -- немає ресурсів.
2. Cluster Autoscaler симулює: "якщо додам Node типу X, чи Pod запуститься?"
3. Якщо так -- збільшує ASG / Node Pool.
4. Коли Nodes довго недовантажені -- зменшує кластер.

**Налаштовується на рівні хмари** (AWS: EKS + ASG, GKE: Node Pool, AKS: Node Pool).

**Karpenter** (AWS) -- альтернатива Cluster Autoscaler для EKS. Підбирає інстанси точно під Pods (right-sizing), швидше, підтримує Spot instances.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: [c, m, r]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [spot, on-demand]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
```

---

## Як працює RBAC (Role-Based Access Control)?

RBAC у k8s -- хто (субʼєкт) може робити що (дія) з чим (ресурс).

**4 об'єкти:**
- **Role** / **ClusterRole** -- набір дозволів.
- **RoleBinding** / **ClusterRoleBinding** -- прив'язка Role до субʼєктів (User, Group, ServiceAccount).

**Role vs ClusterRole:**
- `Role` -- namespaced (дозволи в одному namespace).
- `ClusterRole` -- cluster-scoped (для Nodes, PVs, cluster-wide ресурсів; або для переюзу в кількох namespaces через RoleBinding).

**Приклад: read-only доступ до Pods у namespace `dev`:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-reader
  namespace: dev
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Verbs:**
- `get`, `list`, `watch` -- читання.
- `create`, `update`, `patch`, `delete` -- запис.
- `*` -- усе (уникайте).

**Перевірка дозволів:**
```bash
kubectl auth can-i list pods --namespace=dev --as=alice@example.com
# yes / no
```

---

## ServiceAccount: як Pods автентифікуються?

Кожен Pod запускається від імені **ServiceAccount** (дефолт -- `default` у namespace). Через це Pod може звертатись до API Server.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  namespace: dev

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-sa-reader
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: api-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: v1
kind: Pod
metadata:
  name: api
spec:
  serviceAccountName: api-sa
  containers:
    - name: api
      image: myapp:1.0
```

У Pod автоматично монтується token у `/var/run/secrets/kubernetes.io/serviceaccount/token`.

**Важливо:**
- Дефолтний ServiceAccount має мінімальні права, але краще створювати окремий для кожного застосунку.
- Для хмарних credentials (IAM) -- використовуйте **IRSA** (IAM Roles for Service Accounts) на EKS або **Workload Identity** на GKE/AKS, замість статичних ключів.

---

## Як дебажити проблемний Pod?

**Крок 1: стан Pod**
```bash
kubectl get pod myapp-abc123
# NAME           READY   STATUS             RESTARTS   AGE
# myapp-abc123   0/1     CrashLoopBackOff   5          10m
```

**Крок 2: опис і events**
```bash
kubectl describe pod myapp-abc123
# Events:
#   Warning  FailedScheduling  no nodes available
#   Normal   Scheduled         Successfully assigned
#   Warning  BackOff           Back-off restarting failed container
```

**Крок 3: логи**
```bash
kubectl logs myapp-abc123                 # поточний контейнер
kubectl logs myapp-abc123 --previous      # попередній рестарт (важливо для CrashLoop)
kubectl logs myapp-abc123 -c init-migrate # конкретний init container
```

**Крок 4: exec всередину**
```bash
kubectl exec -it myapp-abc123 -- sh
```

**Крок 5: якщо Pod впав і рестартується -- ephemeral debug container** (k8s 1.25+):
```bash
kubectl debug -it myapp-abc123 --image=busybox --target=app
```

**Типові причини поломок:**

| Статус               | Що перевірити                                            |
|----------------------|----------------------------------------------------------|
| Pending              | Ресурси на нодах, PVC, nodeSelector, taints              |
| ImagePullBackOff     | Тег образу, imagePullSecrets, мережа до registry          |
| CrashLoopBackOff     | Логи попереднього рестарту, exit code, liveness probe    |
| OOMKilled            | Memory limit замалий, реальне споживання                 |
| Error                | Exit code != 0, застосунок сам впав                      |
| CreateContainerError | volumeMounts, secretKeyRef не знайдено                   |
| Completed            | Для Job -- нормально. Для Deployment -- баг у застосунку |

---

## Як робити безпечне оновлення кластера?

**Кроки:**
1. **Backup etcd** -- або через Velero, або snapshotом.
2. **Drain вузлів по одному:**
   ```bash
   kubectl cordon node-1           # не планувати нові Pods
   kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
   # → Pods переносяться на інші вузли
   ```
3. **Upgrade control plane** -- один master за раз (або через managed service: EKS, GKE, AKS).
4. **Upgrade worker nodes** -- rolling, по одному.
5. **Uncordon:**
   ```bash
   kubectl uncordon node-1
   ```

**PodDisruptionBudget захищає** -- drain не зупинятиме Pod, якщо це порушить PDB.

**Правило Kubernetes version skew:**
- Контрольна площина і kubelet -- максимум 1 minor версію різниці.
- Не пропускайте minor версії: 1.27 → 1.28 → 1.29, не 1.27 → 1.29.

---

## Моніторинг та observability стек

Типовий стек:

**Metrics:**
- **Prometheus** -- збирає метрики з Pods, Nodes, Services.
- **Grafana** -- дашборди.
- **kube-state-metrics** -- експортер стану k8s обʼєктів.
- **node-exporter** -- метрики вузлів (CPU, memory, disk).
- **Metrics Server** -- для HPA і `kubectl top`.

**Logs:**
- **Fluent Bit / Fluentd / Vector** -- збирають логи з контейнерів.
- **Loki / Elasticsearch / CloudWatch / Datadog** -- зберігання та пошук.

**Traces:**
- **OpenTelemetry Collector** -- збирає traces.
- **Jaeger / Tempo / Datadog APM** -- перегляд.

**Чому важливо дивитися метрики:**
- CPU throttling (`container_cpu_cfs_throttled_seconds_total`) -- ліміт замалий.
- OOMKilled -- memory limit замалий.
- Pod restarts -- щось не так з застосунком / probes.
- Pending pods -- ресурси закінчилися.

---

## Чотири "золоті сигнали" моніторингу

За Google SRE book:
1. **Latency** -- час відповіді.
2. **Traffic** -- requests/second.
3. **Errors** -- error rate.
4. **Saturation** -- наскільки повно використані ресурси (CPU, memory, queue).

Для кожного сервісу у кластері -- ставте алерти на ці 4. Не всі алерти одразу, але поступово.

---

## Типові помилки з operations

**1. Немає requests/limits:**
- Pod у BestEffort QoS, вбивається першим.
- HPA не працює по CPU utilization.
- Кластер непередбачуваний.

**2. Немає PDB:**
- Drain на Node може випадково знищити всі replicas сервісу.

**3. Немає anti-affinity:**
- Всі replicas на одному Node -- Node fail = повний downtime.

**4. Тільки 1 replica у production:**
- Будь-який disruption = downtime. Мінімум 2, краще 3.

**5. Немає Resource Quota / LimitRange у namespace:**
- Один team може з'їсти всі ресурси кластера.

**6. Cluster Autoscaler без `overprovisioning`:**
- Нові Pods чекають, поки новий Node запуститься (1-3 хв).
- Рішення -- pause Pods з низьким priority, які вивільнюють місце при потребі.

**7. Прямий доступ до Pods через Pod IP:**
- Pod IPs нестабільні. Завжди через Service.

**8. Секрети у Git без шифрування:**
- Навіть sealed-secrets краще за plaintext. Оптимум -- ESO + Vault/AWS SM.

**9. Ignored liveness probe:**
- Без неї завислий процес продовжує "жити", поки k8s не рестартує Pod з інших причин.

**10. kubectl apply без review:**
- Використовуйте `kubectl diff -f file.yaml` перед apply.
- Або GitOps (ArgoCD, Flux) -- deploy тільки через PR.
