# Kubernetes: Storage, ConfigMap, Secrets

## Які типи volumes бувають у Kubernetes?

Volumes у k8s -- спосіб дати контейнеру доступ до зовнішнього сховища або спільних файлів. Пов'язані з життєвим циклом Pod (деякі) або існують довше (PersistentVolumes).

**Ephemeral (зникають з Pod):**
- **emptyDir** -- порожня директорія при старті Pod, спільна між контейнерами в Pod. Видаляється, коли Pod видаляється.
- **configMap / secret** -- монтування ConfigMap / Secret як файлів.
- **downwardAPI** -- доступ до метаданих Pod (ім'я, labels, IP).

**Persistent:**
- **persistentVolumeClaim** -- посилання на PersistentVolume.

**Host-bound:**
- **hostPath** -- монтування директорії з Node. Небезпечно, уникайте (крім DaemonSets).

**Legacy in-tree drivers** (awsElasticBlockStore, gcePersistentDisk) -- deprecated, замінюються CSI drivers.

---

## Що таке PersistentVolume і PersistentVolumeClaim?

**PersistentVolume (PV)** -- реальний шматок сховища (диск AWS EBS, GCE PD, NFS share тощо). Створюється або вручну адміном (static provisioning), або автоматично (dynamic provisioning).

**PersistentVolumeClaim (PVC)** -- заявка Pod на сховище. Pod не знає про PV напряму -- він створює PVC, а k8s зв'язує PVC з відповідним PV.

**StorageClass** -- шаблон для динамічного створення PV. Визначає провайдер, тип диска, параметри.

```
Pod → PVC → StorageClass → (provisioner створює) → PV → реальний диск
```

**Приклад з динамічним provisioning:**
```yaml
# StorageClass (зазвичай є в хмарному кластері за замовчуванням)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer   # створити диск, коли Pod планується
reclaimPolicy: Delete                     # видалити диск, коли PVC видалено

---
# PVC -- заявка на сховище
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3
---
# Pod, який використовує PVC
apiVersion: v1
kind: Pod
metadata:
  name: db
spec:
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data
```

---

## Access modes: ReadWriteOnce, ReadOnlyMany, ReadWriteMany

| Mode            | Abbrev | Значення                                   |
|-----------------|--------|--------------------------------------------|
| ReadWriteOnce   | RWO    | Може бути змонтовано read/write одним Node |
| ReadOnlyMany    | ROX    | Read-only на багатьох Nodes                |
| ReadWriteMany   | RWX    | Read/write на багатьох Nodes               |
| ReadWriteOncePod| RWOP   | RW одним Pod (k8s 1.27+)                   |

**Важливо:** тільки частина драйверів підтримує RWX:
- AWS EFS, Azure Files, GCP Filestore -- підтримують.
- AWS EBS, GCE PD, звичайні блочні -- **тільки RWO**.

Тому для shared storage (наприклад, кілька Pods пишуть у спільну папку) потрібен NFS-подібний storage.

**Практичне значення RWO:**
- StatefulSet з RWO -- кожен Pod має власний диск.
- Deployment з 3 replicas + RWO PVC -- НЕ спрацює, диск можна змонтувати лише на один Node.

---

## Reclaim policy: що стається з PV, коли PVC видалено?

**Delete** -- PV і реальний диск видаляються. Дані пропадають. (Дефолт для більшості хмарних StorageClass.)

**Retain** -- PV переходить у `Released` стан. Диск зберігається, можна витягти дані вручну.

**Recycle** -- deprecated, не використовується.

**Рекомендація для критичних даних:**
```yaml
reclaimPolicy: Retain
```
Або регулярний backup (Velero, пряме snapshots диску).

---

## ConfigMap: як передати конфігурацію в Pod?

ConfigMap -- об'єкт для збереження конфігураційних даних (key-value, файли) окремо від образів.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
  FEATURE_X_ENABLED: "true"
  config.json: |
    {
      "timeout": 30,
      "retries": 3
    }
```

**Використання як env variables:**
```yaml
spec:
  containers:
    - name: app
      envFrom:
        - configMapRef:
            name: app-config
      # Або окремо:
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

**Монтування як файлів:**
```yaml
spec:
  containers:
    - name: app
      volumeMounts:
        - name: config
          mountPath: /etc/app
  volumes:
    - name: config
      configMap:
        name: app-config
```
Всередині контейнера з'являться файли `/etc/app/LOG_LEVEL`, `/etc/app/config.json`.

**Важливо:**
- Оновлення ConfigMap **не** тригерить рестарт Pod. Якщо змонтовано як файли -- оновиться через ~1 хв. Якщо env -- не оновиться до рестарту.
- Для автоматичного рестарту -- використовуйте tools як `reloader` або `stakater/reloader`.
- Розмір ConfigMap -- до 1 MiB.

---

## Secret: як зберігати чутливі дані?

Secret -- схожий на ConfigMap, але для чутливих даних (паролі, токени, сертифікати).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: cG9zdGdyZXM=        # base64: "postgres"
  password: c3VwZXJzZWNyZXQ=    # base64: "supersecret"
```

**За замовчуванням Secret -- це НЕ шифрування.** Дані просто закодовані в base64. У etcd вони зберігаються в тому ж вигляді.

**Що треба для реального захисту:**
1. **Encryption at rest** для etcd -- налаштовується в API Server:
   ```yaml
   # EncryptionConfiguration
   resources:
     - resources: [secrets]
       providers:
         - aescbc:
             keys:
               - name: key1
                 secret: <base64 key>
   ```
2. **RBAC** -- обмеження, хто може читати secrets.
3. **External secret managers** -- AWS Secrets Manager, HashiCorp Vault (через External Secrets Operator, Secrets Store CSI Driver).

**Використання:**
```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

---

## External Secrets Operator: чому не просто Secret?

Проблеми з нативними Secrets:
- Base64 ≠ шифрування.
- Секрети в Git (навіть у зашифрованому вигляді через sealed-secrets) -- незручно ротувати.
- Немає audit trail, хто і коли звертався до секрету.

**External Secrets Operator** (ESO) синхронізує k8s Secrets з зовнішніх secret managers:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials        # створить k8s Secret з цим іменем
  data:
    - secretKey: password
      remoteRef:
        key: prod/db/password
```

Тепер справжнє сховище -- AWS Secrets Manager. K8s Secret -- це кеш, який автоматично оновлюється.

Альтернатива -- **Secrets Store CSI Driver**, який монтує секрети як файли напряму з Vault/AWS/Azure, минаючи etcd.

---

## Як оновлювати конфіг без downtime?

**Сценарій:** оновили `ConfigMap`, хочете щоб Pods прочитали нові значення.

**Варіант 1: перезапуск Deployment**
```bash
kubectl rollout restart deployment/api
```

**Варіант 2: checksum annotation** -- змінюється при зміні ConfigMap, тригерить rolling update:
```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include "app/configmap.yaml" . | sha256sum }}
```
(Helm патерн.)

**Варіант 3: Reloader** -- controller, який стежить за ConfigMap/Secret і рестартує Deployment:
```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

**Варіант 4: hot reload у застосунку** -- застосунок сам перечитує файл конфігу при зміні (inotify). Найсмачніше, але потребує підтримки в коді.

---

## Порівняння: ConfigMap vs Secret vs Environment

| Підхід                 | Плюси                           | Мінуси                          |
|------------------------|----------------------------------|---------------------------------|
| `env` hardcoded у Pod  | Простий                          | Для кожного середовища окремо  |
| ConfigMap (envFrom)    | Окремо від Pod, легко оновити    | Не рестартує автоматично        |
| ConfigMap (volumeMount)| Автоматичне оновлення файлів     | Треба hot reload у коді         |
| Secret                 | Для чутливих даних               | Base64, не шифрування           |
| ESO / Vault / SSM      | Справжнє шифрування + ротація    | Додаткова інфраструктура        |

**Правило:** все конфігураційне -- через ConfigMap/Secret. Нічого hardcoded, нічого в образі.

---

## Как працюють Kustomize та Helm для керування конфігурацією?

### Kustomize
Нативний у kubectl (`kubectl apply -k`). Працює з overlays -- базова версія + патчі для кожного середовища.

```
manifests/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replicas-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base
patches:
  - path: replicas-patch.yaml
configMapGenerator:
  - name: app-config
    envs: [prod.env]
images:
  - name: myapp
    newTag: v1.2.3
```

```bash
kubectl apply -k overlays/prod
```

### Helm
Template engine для YAML + package manager. Chart -- це архів з шаблонами і значеннями.

```
myapp/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

```bash
helm install myapp ./myapp -f values-prod.yaml
helm upgrade myapp ./myapp -f values-prod.yaml
helm rollback myapp 1
```

**Kustomize vs Helm:**

| Аспект           | Kustomize                      | Helm                             |
|------------------|--------------------------------|----------------------------------|
| Template         | Немає, чистий YAML + патчі     | Go templates                     |
| Package mgmt     | Ні                             | Так (chart repos)                |
| Складність       | Нижча                          | Вища                             |
| Спільнота чартів | -                              | Велика (artifacthub.io)           |
| State            | Stateless (kubectl)            | Має state (releases у k8s)       |

Можна комбінувати -- Helm генерує YAML, Kustomize його патчить.

---

## Типові пастки

**1. Secret у git:**
Навіть base64 -- не шифрування. Для git-based підходу використовуйте `sealed-secrets` або `SOPS`.

**2. PVC delete → дані пропадають:**
Якщо `reclaimPolicy: Delete` -- видалення PVC видалить диск. Критичні дані -- `Retain` + снапшоти.

**3. StatefulSet scale down не видаляє PVCs:**
Це фіча -- дані збережені на випадок scale up назад. Треба чистити вручну: `kubectl delete pvc data-mysql-2`.

**4. Resize volume:**
Збільшити PVC можна, зменшити -- ні. Перевірте, що StorageClass має `allowVolumeExpansion: true`.

**5. ConfigMap занадто великий:**
Ліміт 1 MiB. Великі файли -- у PVC або зовнішній storage (S3).

**6. Два Pods пишуть в один RWO volume:**
Буде помилка монтування на другому Pod. Або використовуйте RWX (EFS), або StatefulSet з PVC per Pod.

**7. imagePullSecrets з іншого namespace:**
Secrets прив'язані до namespace. Якщо Deployment в одному namespace, а Secret в іншому -- не спрацює. Копіюйте Secret або використовуйте ESO.
