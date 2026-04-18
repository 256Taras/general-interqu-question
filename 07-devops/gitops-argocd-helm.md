# GitOps, ArgoCD, Flux, Helm

## Що таке GitOps і які його чотири принципи?

GitOps -- це операційна модель для Kubernetes (та інфраструктури загалом), в якій **Git є єдиним джерелом правди (single source of truth)** для стану системи. Замість того щоб оператор виконував `kubectl apply` вручну чи через CI pipeline, в кластері живе **агент**, який постійно порівнює стан кластера з маніфестами в Git і приводить його до бажаного стану.

Термін ввела компанія Weaveworks у 2017 році. У 2021-му CNCF OpenGitOps Working Group сформулювала чотири принципи:

### 1. Declarative (декларативність)

Весь бажаний стан системи описаний декларативно, а не через послідовність імперативних команд. Не "створи Deployment з 3 репліками", а "стан системи = Deployment з 3 репліками".

```yaml
# Декларативно: "ось який стан я хочу"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  # ...
```

### 2. Versioned and Immutable (версіоновано та незмінно)

Бажаний стан зберігається в Git -- а отже, отримує повну історію, аудит, підписи комітів, code review через PR. Будь-яка зміна -- це коміт. Rollback -- це `git revert`.

### 3. Pulled Automatically (автоматичний pull)

Агенти в кластері **самі підтягують** зміни з Git. Не CI штовхає зміни в кластер (push), а кластер тягне їх (pull). Це ключовий зсув парадигми.

### 4. Continuously Reconciled (безперервна реконсиляція)

Агент постійно (кожні N секунд/хвилин) порівнює реальний стан кластера з бажаним і усуває drift. Якщо хтось вручну змінив щось через `kubectl edit` -- агент це помітить і відкотить (якщо ввімкнено selfHeal).

```
   ┌──────────────────────────────────────────────────────┐
   │                  GitOps Loop                          │
   │                                                      │
   │   ┌─────────┐    pull      ┌──────────────┐        │
   │   │   Git    │ ──────────▶ │ GitOps Agent  │        │
   │   │  (truth) │              │  (in-cluster) │        │
   │   └─────────┘              └──────┬───────┘        │
   │                                    │ diff & apply    │
   │                                    ▼                  │
   │                              ┌──────────┐           │
   │                              │ K8s API   │           │
   │                              └──────────┘           │
   │                                    │                  │
   │                                    │ observed state  │
   │                                    ▼                  │
   │                              ┌──────────┐           │
   │                              │ Cluster   │◀──drift   │
   │                              │  State    │  detect   │
   │                              └──────────┘           │
   └──────────────────────────────────────────────────────┘
```

**Чому це з'явилося?** До GitOps типовий pipeline виглядав так: CI збирає образ → CI виконує `kubectl apply` з credentials кластера → готово. Проблеми:
- CI мав повні права в production-кластері (security risk).
- Ніхто не відстежував drift -- хтось `kubectl edit` і стан розходиться.
- Rollback -- це запустити попередній job, а не `git revert`.
- Аудит розмазаний між CI-логами, kubectl history, etcd.

GitOps переносить **апплай** з CI всередину кластера, залишаючи CI відповідати тільки за build+push image.

---

## Push vs Pull модель деплою. У чому різниця і чому Pull кращий?

Це фундаментальна відмінність GitOps від класичного CI/CD.

### Push model (традиційний CI/CD)

CI pipeline виконує `kubectl apply` ззовні:

```
  ┌─────────┐  build   ┌─────────┐  push    ┌──────────────┐
  │ Developer│──git──▶ │   CI    │ ──image──▶│  Registry    │
  └─────────┘         └────┬────┘           └──────────────┘
                            │
                            │ kubectl apply
                            │ (CI has cluster creds!)
                            ▼
                      ┌──────────────┐
                      │  Kubernetes  │
                      │   cluster    │
                      └──────────────┘
```

Проблеми:
- CI має secrets/kubeconfig production-кластера. Компрометація CI = компрометація prod.
- Якщо кластер за VPN / private network -- CI має туди достукатись (firewall headache).
- Drift detection відсутній -- CI не знає, що хтось вручну змінив стан.
- Стан кластера ≠ останній successful job. Якщо хтось перезапустив старий pipeline -- rollback "випадково".

### Pull model (GitOps)

Агент всередині кластера тягне зміни:

```
  ┌─────────┐  git push  ┌─────────┐
  │ Developer│──────────▶│   Git   │
  └─────────┘           └────┬────┘
                              │
                              │ (poll / webhook)
                              │  pull
                              ▼
                      ┌──────────────────┐
                      │   Kubernetes      │
                      │  ┌────────────┐   │
                      │  │ GitOps agent│  │
                      │  │ (ArgoCD /   │  │
                      │  │  Flux)      │   │
                      │  └─────┬──────┘   │
                      │        │ apply     │
                      │        ▼           │
                      │  ┌────────────┐   │
                      │  │ Workloads  │   │
                      │  └────────────┘   │
                      └──────────────────┘
```

Переваги pull:
- **Безпека**: кластеру не потрібен вхідний доступ ззовні. CI лише пушить image + маніфест у Git. Credentials production живуть лише в кластері.
- **Airgapped / private clusters** працюють легко -- їм треба лише вихідний HTTPS до Git.
- **Continuous reconciliation** -- агент бачить drift і править його.
- **Audit trail** -- усе, що в кластері, є в Git.
- **Rollback = `git revert`**, а не "запусти старий pipeline".

У типовому setup CI все одно існує, але його роль звужується: build image, run tests, **оновити image tag у values.yaml**, закомітити в Git. Далі бере на себе ArgoCD/Flux.

---

## Архітектура ArgoCD: компоненти і як це працює

ArgoCD -- найпопулярніший GitOps-агент для Kubernetes (CNCF graduated). Це Kubernetes-native застосунок (CRD + controllers), який деплоїться всередині кластера.

Основні компоненти:

```
   ┌─────────────────────────────────────────────────────────┐
   │                     ArgoCD                               │
   │                                                         │
   │  ┌────────────┐      ┌──────────────────┐              │
   │  │  API Server │◀────▶│  Application      │              │
   │  │  (UI/CLI/   │      │   Controller      │              │
   │  │   gRPC)     │      │  (reconcile loop) │              │
   │  └──────┬─────┘      └──────┬───────────┘              │
   │         │                    │                           │
   │         │                    │ manifests                 │
   │         ▼                    ▼                           │
   │  ┌─────────────┐     ┌──────────────┐                   │
   │  │    Redis    │     │  Repo Server  │                   │
   │  │   (cache)   │◀───▶│ (git clone,    │                   │
   │  └─────────────┘     │  helm template,│                   │
   │                       │  kustomize)    │                   │
   │                       └──────┬───────┘                   │
   │                              │ clone                     │
   │                              ▼                           │
   │                       ┌──────────────┐                   │
   │                       │   Git repo    │                   │
   │                       └──────────────┘                   │
   │                                                         │
   │  ┌─────────────────────────────┐                        │
   │  │    Dex (SSO, OIDC/LDAP)     │  опціонально           │
   │  └─────────────────────────────┘                        │
   │  ┌─────────────────────────────┐                        │
   │  │ ApplicationSet Controller   │  опціонально           │
   │  └─────────────────────────────┘                        │
   │  ┌─────────────────────────────┐                        │
   │  │ Notifications Controller    │  опціонально           │
   │  └─────────────────────────────┘                        │
   └─────────────────────────────────────────────────────────┘
```

### Що робить кожен компонент

**API Server** -- експонує gRPC/REST API, обслуговує UI та CLI. Тут RBAC, SSO, authentication. Це єдина точка входу для користувача.

**Repo Server** -- stateless-сервіс, який клонує Git-репозиторії та генерує маніфести. Вміє: plain YAML, Helm (робить `helm template`), Kustomize, Jsonnet, custom plugins. Результат -- кешується в Redis.

**Application Controller** -- серце ArgoCD. Це Kubernetes controller, який:
1. Дивиться на всі `Application` CRD в кластері.
2. Для кожного: через Repo Server отримує desired state (маніфести з Git).
3. Порівнює з live state (через K8s API).
4. Якщо drift -- оновлює статус `OutOfSync`. Якщо ввімкнено auto-sync -- апплає.

**Redis** -- кеш для репо-менеджера та контролера, щоб не клонувати Git щоразу.

**ApplicationSet Controller** -- генератор `Application` ресурсів (про це нижче).

### Як це працює end-to-end

```
1. Developer:     git commit + push у "gitops-repo"
2. Argo Repo Server: clone repo (або webhook тригер → sync now)
3. Controller:     compute desired (from Git) vs live (from K8s)
4. Якщо diff:      status = OutOfSync
5. Auto-sync ON?   kubectl apply маніфестів
6. Reconcile:      loop (default every 3min)
7. UI:             health + sync status для кожного app
```

---

## ArgoCD Application, AppProject, sync policies

### Application -- основна одиниця

`Application` -- це CRD, який описує "що" деплоїти та "куди".

```yaml
# Application: деплой одного сервісу
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-production
  namespace: argocd  # Application зазвичай живуть у namespace argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # видалить ресурси при видаленні app
spec:
  project: production  # посилання на AppProject
  source:
    repoURL: https://github.com/my-org/gitops.git
    targetRevision: main  # гілка / тег / commit SHA
    path: apps/api/overlays/production  # шлях у репо
    # Для Helm:
    # helm:
    #   valueFiles:
    #     - values-production.yaml
    #   parameters:
    #     - name: image.tag
    #       value: v1.2.3
  destination:
    server: https://kubernetes.default.svc  # in-cluster
    namespace: api-prod
  syncPolicy:
    automated:
      prune: true       # видаляти ресурси, яких вже немає в Git
      selfHeal: true    # відкочувати ручні зміни через kubectl
      allowEmpty: false # забороняти sync, якщо Git повернув 0 ресурсів (safety)
    syncOptions:
      - CreateNamespace=true  # створити namespace, якщо нема
      - PrunePropagationPolicy=foreground
      - ServerSideApply=true  # замість client-side apply
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Sync policies детально

- **automated** -- без нього sync робиться вручну (через UI/CLI). З ним -- автоматично при появі drift.
- **prune: true** -- якщо ви видалили маніфест з Git, ArgoCD видалить відповідний ресурс з кластера. Без цього -- "orphan resources" лишаються.
- **selfHeal: true** -- якщо оператор зробив `kubectl edit deployment api -n api-prod` і змінив replicas, ArgoCD поверне як у Git. Без selfHeal -- drift буде лише підсвічено в UI.
- **allowEmpty: false** -- захист від ситуації "хтось помилково видалив усі маніфести з Git", що призвело б до видалення всіх ресурсів з кластера.

### AppProject -- логічне групування + guardrails

```yaml
# AppProject: обмежує, куди і звідки можна деплоїти
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production workloads
  # З яких репозиторіїв можна брати маніфести
  sourceRepos:
    - 'https://github.com/my-org/gitops.git'
    - 'https://charts.bitnami.com/bitnami'
  # В які кластери/namespace можна деплоїти
  destinations:
    - server: https://kubernetes.default.svc
      namespace: 'api-prod'
    - server: https://kubernetes.default.svc
      namespace: 'web-prod'
  # Які cluster-scoped ресурси дозволені
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: 'rbac.authorization.k8s.io'
      kind: ClusterRole
  # Заборонити певні ресурси (наприклад, не дозволяти створювати CRD)
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
  # RBAC: хто може sync-ити apps у цьому проекті
  roles:
    - name: developer
      policies:
        - p, proj:production:developer, applications, sync, production/*, allow
      groups:
        - my-org:developers
```

AppProject -- це як "tenant boundary". Dev-команда не зможе випадково задеплоїти у production-namespace, якщо їхній Project це не дозволяє.

---

## ApplicationSet -- генерація Applications масово

Писати 50 Application YAML для 50 мікросервісів -- боляче. `ApplicationSet` -- це **шаблон + генератор**, який створює Application ресурси автоматично.

### Generators

**List Generator** -- статичний список:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: services
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - name: api
            namespace: api-prod
          - name: worker
            namespace: worker-prod
          - name: frontend
            namespace: web-prod
  template:
    metadata:
      name: '{{name}}-production'
    spec:
      project: production
      source:
        repoURL: https://github.com/my-org/gitops.git
        targetRevision: main
        path: 'apps/{{name}}/overlays/production'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**Git Generator** -- сканує директорії у Git і створює Application для кожної:

```yaml
spec:
  generators:
    - git:
        repoURL: https://github.com/my-org/gitops.git
        revision: main
        directories:
          # Кожна директорія apps/*/overlays/production стане окремим Application
          - path: apps/*/overlays/production
  template:
    metadata:
      # {{path.basename}} = 'production', потрібно взяти 2-й рівень
      name: '{{index .path.segments 1}}-production'
    spec:
      source:
        repoURL: https://github.com/my-org/gitops.git
        path: '{{path}}'
      # ...
```

**Cluster Generator** -- для multi-cluster. Перебирає всі кластери, зареєстровані в ArgoCD:

```yaml
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: production  # деплоїти тільки в prod-кластери
  template:
    metadata:
      name: 'monitoring-{{name}}'  # {{name}} = ім'я кластера
    spec:
      destination:
        server: '{{server}}'
        namespace: monitoring
      source:
        repoURL: https://github.com/my-org/gitops.git
        path: infra/monitoring
```

**Matrix Generator** -- комбінація двох генераторів (cartesian product). Наприклад: `clusters × git directories` -> деплоїти кожен сервіс у кожен кластер.

ApplicationSet незамінний для monorepo з десятками сервісів або multi-cluster federation.

---

## Flux CD: GitRepository, Kustomization, HelmRelease. Порівняння з ArgoCD

Flux -- другий великий GitOps-агент (також CNCF graduated). Архітектурно дуже модульний: набір окремих контролерів.

### Основні CRD

**GitRepository** -- джерело:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m  # як часто пулити Git
  url: https://github.com/my-org/gitops.git
  ref:
    branch: main
  secretRef:
    name: git-credentials  # для приватних repo
```

**Kustomization** -- що апплаїти:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: api-production
  namespace: flux-system
spec:
  interval: 5m       # інтервал reconcile
  path: ./apps/api/overlays/production
  prune: true        # видаляти видалені ресурси
  sourceRef:
    kind: GitRepository
    name: my-app
  targetNamespace: api-prod
  timeout: 2m
  # health checks
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: api
      namespace: api-prod
```

**HelmRelease** -- для Helm-чартів:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: redis
  namespace: cache
spec:
  interval: 5m
  chart:
    spec:
      chart: redis
      version: '18.x.x'
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    auth:
      enabled: true
    replica:
      replicaCount: 3
  # автоматичний rollback при помилці
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
```

### ArgoCD vs Flux

| Аспект | ArgoCD | Flux |
|---|---|---|
| UI | Потужний web UI "з коробки" | UI немає (є сторонні: Weave GitOps, Capacitor) |
| Модель | Моноліт із кількох компонентів | Набір окремих контролерів (source, kustomize, helm, notification, image) |
| Multi-cluster | Hub-and-spoke: один ArgoCD керує кількома | Кожен кластер має свій Flux |
| Tenant model | AppProjects | Namespaces + RBAC |
| Image updates | Argo Image Updater (окремий проект) | Image Automation Controller (вбудований) |
| Манифести | Helm, Kustomize, Jsonnet, plugins | Helm, Kustomize |
| Конфіг | Через CRD Application / UI | Через CRD (GitOps для самого Flux) |

**Коли що**: ArgoCD -- якщо потрібен UI для dev-команд, multi-cluster з однієї точки, або ви deploy-heavy (багато сервісів). Flux -- якщо хочеться модульніше, native image automation, і ви комфортні без UI.

---

## Helm: chart structure, templates, values, helpers, dependencies

Helm -- пакетний менеджер для Kubernetes. Замість голого YAML -- шаблонізовані маніфести з параметрами.

### Структура чарту

```
my-app/
├── Chart.yaml           # метадані чарту (name, version, appVersion)
├── values.yaml          # значення за замовчуванням
├── values.schema.json   # JSON schema для валідації values (опціонально)
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl     # переіспользовувані template-функції
│   └── NOTES.txt        # те, що покажеться після install
├── charts/              # sub-charts (dependencies)
└── crds/                # CustomResourceDefinitions (апплаяться першими)
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: Мій застосунок
type: application        # або library
version: 1.2.3           # версія чарту (SemVer)
appVersion: "2.0.0"      # версія самого застосунку
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled  # вмикати/вимикати через values
```

### Template з values

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}   # виклик helper'а
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- if .Values.env }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          {{- end }}
```

### _helpers.tpl

```gotemplate
{{/*
Повне ім'я ресурсу: releaseName-chartName (або override)
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Standard labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: ghcr.io/my-org/my-app
  tag: ""  # порожній -> використається .Chart.AppVersion
  pullPolicy: IfNotPresent

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

env:
  LOG_LEVEL: info
  DATABASE_URL: "postgres://..."

postgresql:
  enabled: true  # sub-chart буде апплаїтись
  auth:
    username: myapp
    database: myapp
```

### Основні команди

```bash
helm install my-app ./my-app                           # новий релізс
helm install my-app ./my-app -f values-prod.yaml       # з override values
helm install my-app ./my-app --set image.tag=v1.2.3    # inline override

helm upgrade my-app ./my-app -f values-prod.yaml       # upgrade
helm upgrade --install my-app ./my-app                 # upsert

helm rollback my-app 2                                 # відкат до ревізії 2
helm history my-app                                    # історія ревізій
helm list -A                                           # всі релізи в кластері
helm template my-app ./my-app -f values.yaml           # рендер без apply
helm lint ./my-app                                     # валідація
helm dep update ./my-app                               # підтягнути dependencies
```

Під капотом: Helm 3 зберігає стан релізу як Secret у namespace релізу (не в Tiller, який був у Helm 2). Історія ревізій -- це окремі Secrets типу `helm.sh/release.v1`.

---

## Helm vs Kustomize vs Raw YAML

Три способи керувати маніфестами. У кожного свої сценарії.

### Raw YAML

```
apps/
├── api/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
```

**Плюси**: прозоро, жодної магії, швидкий старт.
**Мінуси**: немає параметризації. Кожне оточення = копіпаста. 5 середовищ × 10 сервісів = 50 майже однакових файлів.
**Коли**: дуже маленький проект, одне середовище, або CRD manifests з фіксованими значеннями.

### Helm

**Плюси**: повна параметризація через values, versioned packages (chart repos), dependencies, готові charts для всього (Bitnami, Prometheus Operator, etc.).
**Мінуси**: Go templates -- найгірший у світі шаблонізатор (помилки в `{{- -}}` і пробілах). Важко робити невеликі правки "зверху" без форку чарту.
**Коли**: власне ПЗ зі складною конфігурацією, або коли потрібно дистрибутувати чарт кільком командам. Майже всі third-party operators публікуються як Helm charts.

### Kustomize

Декларативні **overlays** без шаблонізатора. Base маніфести + patches.

```
apps/api/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: api-prod
bases:
  - ../../base
patchesStrategicMerge:
  - replica-patch.yaml
images:
  - name: api
    newTag: v1.2.3
configMapGenerator:
  - name: api-config
    literals:
      - LOG_LEVEL=warn
```

**Плюси**: жодних шаблонів, чистий YAML. Вбудовано в `kubectl` (`kubectl apply -k`). Ідеально для environments.
**Мінуси**: складні patches не інтуїтивні. Немає dependencies / packaging / versioning на рівні бандлу.
**Коли**: у вас є власні маніфести, і ви хочете кілька оточень з мінімальними відмінностями.

### Helm + Kustomize (гібрид)

Можна комбінувати: Helm для third-party (Prometheus Operator), Kustomize для своїх сервісів. Або навіть -- Kustomize "поверх" рендереного Helm output (Flux/Argo вміють).

```yaml
# kustomization.yaml з helm chart inflation
helmCharts:
  - name: redis
    repo: https://charts.bitnami.com/bitnami
    version: 18.1.0
    releaseName: my-redis
    valuesFile: redis-values.yaml
patchesStrategicMerge:
  - redis-patch.yaml  # дрібна правка поверх
```

Загалом: **використовуйте Kustomize для оточень, Helm для пакетування/third-party**.

---

## Secret management в GitOps

Центральна проблема: Git -- публічний/шерений, secrets -- ні. Як зберігати secrets в Git, не витікаючи їх?

### Погані рішення

- Закомітити plain secrets у Git. **Ніколи.** Git history вічна, навіть якщо потім видалити.
- Зберігати secrets поза GitOps (ручний `kubectl apply`). Порушує принцип "Git = truth".

### Sealed Secrets (Bitnami)

Контролер у кластері генерує публічний ключ. Ви шифруєте Secret цим ключем через `kubeseal` CLI -- результат (`SealedSecret`) безпечно коміти в Git. У кластері контролер розшифровує за приватним ключем.

```bash
# Локально: створюємо Secret і запечатуємо
kubectl create secret generic db-password \
  --from-literal=password=super-secret \
  --dry-run=client -o yaml \
  | kubeseal --format=yaml > sealed-db-password.yaml
# sealed-db-password.yaml безпечно коміти в Git
```

```yaml
# SealedSecret -- шифрований, але в Git
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
  namespace: api-prod
spec:
  encryptedData:
    password: AgBy8hCy7nA...long-encrypted-blob...==
```

**Мінуси**: ключ шифрування живе в кластері. Якщо кластер знищено -- розшифрувати неможливо. Треба бекапити master key.

### SOPS з age/GPG/KMS

SOPS (Mozilla) шифрує **значення полів** у YAML, залишаючи ключі читабельними. Інтегрується з age (modern, рекомендовано), GPG, AWS KMS, GCP KMS, Azure Key Vault, Vault.

```yaml
# secret.enc.yaml -- закомічено в Git
apiVersion: v1
kind: Secret
metadata:
  name: db-password
stringData:
  password: ENC[AES256_GCM,data:xxx,iv:yyy,tag:zzz,type:str]
sops:
  kms:
    - arn: arn:aws:kms:us-east-1:123:key/abc
```

Flux має вбудовану підтримку SOPS (`decryption.provider: sops`). ArgoCD -- через плагін (`argocd-vault-plugin`) або KSOPS Kustomize plugin.

### External Secrets Operator (ESO)

Secret-маніфести в Git **не містять** самих секретів -- тільки **посилання** на external store (Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, 1Password).

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: api-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials  # ім'я створюваного K8s Secret
  data:
    - secretKey: password
      remoteRef:
        key: prod/api/db
        property: password
```

**Плюси**: secrets ніколи не в Git, навіть зашифровано. Центр управління -- зовнішній store (аудит, ротація, версіонування). Інтеграція з IAM.
**Мінуси**: зайва залежність (external system); треба налаштувати IRSA / Workload Identity для доступу.

**Рекомендація**: для production з AWS/GCP -- ESO + secret manager. Для self-hosted / простих сетапів -- SOPS+age.

---

## Progressive delivery: Argo Rollouts, Flagger

Класичний Kubernetes Deployment робить RollingUpdate -- але без перевірки метрик. Progressive delivery додає canary/blue-green з автоматичним rollback за Prometheus-метриками.

### Argo Rollouts

Custom resource `Rollout` замість `Deployment`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
spec:
  replicas: 10
  strategy:
    canary:
      # Поступове підвищення трафіку
      steps:
        - setWeight: 10      # 10% трафіку на new
        - pause: { duration: 5m }
        - setWeight: 30
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate  # AnalysisTemplate
        - setWeight: 60
        - pause: { duration: 10m }
        - setWeight: 100
      # traffic management через Istio/SMI/Nginx
      trafficRouting:
        istio:
          virtualService:
            name: api
  selector:
    matchLabels:
      app: api
  template:
    # ... як у Deployment
```

```yaml
# AnalysisTemplate: перевірка метрик
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{status=~"2..", service="api"}[1m]))
            /
            sum(rate(http_requests_total{service="api"}[1m]))
```

Якщо success rate падає нижче 95% -- Rollouts сам робить rollback. Canary проміжок показує проблеми до того, як трафік 100% перейде на новий релізс.

### Flagger

Аналог Argo Rollouts, інтегрується з Flux. Працює з Istio/Linkerd/Nginx/Contour/Gloo/App Mesh.

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: api
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  progressDeadlineSeconds: 600
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5          # 5 провалів поспіль -> rollback
    maxWeight: 50
    stepWeight: 10        # +10% трафіку кожну ітерацію
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 1m
    webhooks:
      - name: load-test
        url: http://load-tester.test/
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://api-canary/"
```

**Blue-green** теж підтримується обома -- замість поступового weight shift, два оточення (blue=old, green=new), switch однієї кнопкою (service selector change).

### GitOps + progressive delivery

Git містить описи Rollout/Canary. ArgoCD/Flux апплає їх. Далі Rollouts/Flagger самі керують процесом розгортання, читаючи Prometheus. Коли робимо image bump у Git -- запускається canary pipeline в кластері.

---

## Типові патерни GitOps

### App-of-Apps

Одне Application, яке посилається на Git-шлях із маніфестами **інших** Applications. Так починається bootstrap: ставимо один root-app, він апплаїть усі решта.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/my-org/gitops.git
    path: bootstrap  # ця тека містить YAML багатьох Application
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```
bootstrap/
├── monitoring-app.yaml      # Application для Prometheus
├── ingress-app.yaml         # Application для Nginx Ingress
├── cert-manager-app.yaml    # Application для cert-manager
└── services-appset.yaml     # ApplicationSet для сервісів
```

Зараз ApplicationSet здебільшого замінив класичний app-of-apps: він і генерує, і керує.

### Monorepo vs Polyrepo для маніфестів

**Monorepo** (один Git repo для всіх маніфестів):
- `+` просто бачити "що де живе", крос-сервісні зміни в одному PR, легко стандартизувати.
- `-` дозволи на ввесь repo, складніше розводити CODEOWNERS, великий repo = довгі clones.

**Polyrepo** (кожен сервіс має власний маніфест-repo, або маніфести поруч з кодом):
- `+` гранульовані permissions, кожна команда -- господар свого.
- `-` складніше координувати спільні зміни, більше репозиторіїв реєструвати в ArgoCD.

Популярний гібрид: **monorepo для маніфестів + polyrepo для коду**. Застосунки живуть у своїх code-repos, а маніфести централізовано в gitops-repo. CI пушить image + bump version у gitops-repo.

### Environments: overlays vs branches

**Overlays (Kustomize)** -- рекомендовано:

```
apps/api/
├── base/
└── overlays/
    ├── dev/
    ├── staging/
    └── production/
```
Один branch `main`. PR у `main` -> sync у відповідне оточення. Chronology зберігається, diff між оточеннями видно в одному view.

**Branches** -- антипатерн:

```
main (= prod)
├── staging
└── dev
```
Кожне оточення = окрема гілка. Проблеми: drift між гілками (cherry-pick hell), складний promotion workflow, неможливо бачити всі оточення одразу. **Уникайте.**

### ApplicationSet для multi-tenant

Кожна команда/tenant має власний namespace + AppProject. ApplicationSet перебирає список tenants і генерує Application для кожного:

```yaml
generators:
  - list:
      elements:
        - tenant: team-a
        - tenant: team-b
template:
  metadata:
    name: '{{tenant}}-app'
  spec:
    project: '{{tenant}}'
    destination:
      namespace: '{{tenant}}'
```

---

## Типові помилки і як їх уникати

### 1. Drift через ручні зміни

Хтось "швидко" робить `kubectl edit deployment api -n api-prod`, міняє replicas з 3 на 10. ArgoCD показує `OutOfSync`, але якщо `selfHeal: false` -- нічого не робить. Реальність розходиться з Git.

**Рішення**: увімкнути `selfHeal: true`. Крім того -- обмежити права `kubectl` у prod (RBAC: dev-команди мають тільки read). Тоді єдиний шлях змінити стан -- через Git + PR.

### 2. Secrets leak в Git

Хтось випадково закомітив `kubectl get secret ... -o yaml > secret.yaml` без шифрування. Навіть якщо видалити в наступному коміті -- Git history лишається.

**Рішення**:
- `pre-commit` hooks з `gitleaks` / `truffleHog` / `detect-secrets`.
- GitHub push protection + secret scanning.
- Завжди SOPS/Sealed Secrets/ESO (див. розділ про secrets).
- Якщо сталося -- **негайно ротувати секрет**, не лише видаляти коміт.

### 3. Broken sync -- застосунок у нескінченному OutOfSync

Приклади причин:
- Маніфест містить поле, яке Kubernetes додає серверно (`defaults`). ArgoCD бачить diff щохвилини.
- `managed-fields` конфлікт між різними controllers.
- Webhooks/admission controllers модифікують ресурс після apply.

**Рішення**:
- `ignoreDifferences` у Application:
  ```yaml
  spec:
    ignoreDifferences:
      - group: apps
        kind: Deployment
        jsonPointers:
          - /spec/replicas  # бо HPA керує цим
  ```
- ServerSideApply: `syncOptions: [ServerSideApply=true]`.
- Синхронізувати, які `managedFields` використовуються.

### 4. Циклічні залежності

Application A чекає на Application B (CRD). B чекає на namespace, який створює A. Deadlock.

**Рішення**: **Sync Waves** -- пріоритети апплаю.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # cluster resources (CRDs, namespaces)
---
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # operators
---
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # CRD instances
```

Менше число -- раніше апплається. CRD -> Operator -> Custom Resource.

### 5. "Тихе" накопичення orphan-ресурсів

Без `prune: true` видалений з Git маніфест лишається в кластері назавжди. Через рік -- ресурси-привиди, які ніхто не знає.

**Рішення**: `prune: true` + `PrunePropagationPolicy=foreground` + періодичний audit через `kubectl get all --show-labels` і порівняння з Git.

### 6. Pipeline пише в той самий гілку, яку апплає ArgoCD

CI пушить bump image tag у `main`. ArgoCD апплає. Але якщо CI написав broken маніфест -- ArgoCD зламає прод миттєво. Також -- loop: ArgoCD викликає webhook, CI щось змінює, і так по колу.

**Рішення**:
- PR-based flow: CI відкриває PR, людина/автотести ревʼюять, merge -> sync.
- Або: CI пише у `dev` branch, після тестів -- PR у `main` (prod).
- Staging environment з автотестами перед production sync.

### 7. Helm chart version != app version

Люди плутають `version` (версія чарту) і `appVersion` (версія застосунку). Апгрейд чарту без апгрейду застосунку і навпаки.

**Рішення**: завжди бампати **обидва**. Можна автоматизувати через `helm-docs` + `chart-releaser`.

### 8. ArgoCD сам-по-собі-не-GitOps'ний

ArgoCD встановили через `helm install argocd`, а конфіг через UI. Потім -- ніхто не знає, як відновити. Це порушення принципу "Git = truth" для самого ArgoCD.

**Рішення**: **Self-managed ArgoCD** -- ArgoCD керує власним чартом через Application, який посилається на Git. Cluster bootstrap: `helm install argocd` -> створити root-Application -> ArgoCD тепер дивиться на Git, включно з власним конфігом.
