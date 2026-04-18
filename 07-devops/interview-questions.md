# DevOps: Зведений список питань для співбесіди

Швидкий прогін питань за темами з посиланнями на детальні відповіді в інших файлах. Використовуй цей файл для самоперевірки перед співбесідою.

---

## Docker

**Основи:**
1. Чим контейнер відрізняється від VM? (namespaces, cgroups, спільне ядро)
2. Що таке Docker image, чим відрізняється від контейнера?
3. Як працює шарова файлова система Docker? Що таке OverlayFS?
4. Які Linux-механізми забезпечують ізоляцію контейнерів? (namespaces: pid, net, mnt, uts, ipc, user)
5. Що відбувається коли виконується `docker run`? Опиши послідовність.
6. Як Docker ізолює CPU і пам'ять? (cgroups, `--cpus`, `--memory`)
7. Як працює Docker networking за замовчуванням? Bridge, host, none.
8. Різниця між bind mount і named volume. Коли що використовувати.

**Dockerfile і білд:**
9. Що таке multi-stage build і яку проблему вирішує?
10. Як оптимізувати кеш Docker-шарів? Правильний порядок інструкцій.
11. Чим `CMD` відрізняється від `ENTRYPOINT`? Коли що використовувати. Exec-form vs shell-form.
12. Як зменшити розмір образу? (alpine/distroless, multi-stage, .dockerignore)
13. Як правильно обробляти SIGTERM у контейнері? Що таке `tini` і навіщо.
14. Як передати секрет у білд, не зберігаючи його в шарах? (BuildKit secrets)
15. Яка різниця між `COPY` і `ADD`? Що краще використовувати і чому.
16. Що робить `HEALTHCHECK` і чи потрібен він в Kubernetes?

**Практичні кейси:**
17. Запустили `docker run`, контейнер миттєво виходить. Як дебажити?
18. Docker image 2GB замість 200MB. Кроки оптимізації.
19. Контейнер не отримує SIGTERM. Причини і рішення.
20. Чому `docker build` повільний, як прискорити?

[→ detailed docker-basics.md](docker-cicd/docker-basics.md) | [→ dockerfile-best-practices.md](docker-cicd/dockerfile-best-practices.md)

---

## Docker Compose

1. Коли використовувати Compose, а коли Kubernetes?
2. Як працює `depends_on`? Чому проста форма не чекає готовності сервісу?
3. Як правильно зробити wait for PostgreSQL before starting API?
4. Чим відрізняється `docker compose` від `docker-compose`?
5. Як розділити конфігурацію між dev і prod у Compose?
6. Що таке profiles і як їх використовувати?
7. Як запустити тести в CI, використовуючи Compose?
8. Дані в named volume -- як зробити backup?

[→ docker-compose.md](docker-cicd/docker-compose.md)

---

## CI/CD

**Концепції:**
1. Чим CI відрізняється від CD? (Continuous Integration / Delivery / Deployment)
2. Які стадії має типовий CI/CD pipeline?
3. Що таке trunk-based development і чому це краще для CI/CD, ніж Git Flow?
4. Які є стратегії деплою? (Recreate, Rolling, Blue-Green, Canary) Коли яку використовувати.
5. Що таке feature flags і як вони співвідносяться зі стратегіями деплою?
6. Що таке DORA metrics? (Deployment Frequency, Lead Time, Change Failure Rate, MTTR)

**Практика:**
7. Як зробити zero-downtime deployment для Node.js застосунку з БД?
8. Як робити rollback, коли нова версія має міграції БД? (forward-compatible migrations)
9. Як убезпечити secrets у CI? (OIDC, vault, не в env logs)
10. Як організувати pipeline для monorepo? (affected tests, nx/turborepo)
11. Тест флакує в CI, але локально працює. Кроки діагностики.
12. Build pipeline став займати 30 хвилин. Оптимізації.

**GitHub Actions:**
13. Як передати дані між jobs? (artifacts, outputs)
14. Що таке matrix strategy? Як виключити комбінації.
15. Як кешувати залежності (npm, Docker layers) у GitHub Actions?
16. Як налаштувати OIDC federation з AWS без статичних ключів?
17. Чим reusable workflow відрізняється від composite action?
18. Як зробити, щоб нові push у PR скасовував попередній run? (concurrency)

[→ ci-cd-pipelines.md](docker-cicd/ci-cd-pipelines.md) | [→ github-actions.md](docker-cicd/github-actions.md)

---

## Kubernetes: Основи

1. Яка архітектура Kubernetes-кластера? (control plane + workers)
2. Які компоненти у Control Plane і що робить кожен? (API Server, etcd, Scheduler, Controller Manager)
3. Що таке Pod і чому це мінімальна одиниця, а не контейнер?
4. Як працює desired state і reconciliation loop?
5. Як Scheduler вирішує, на який Node планувати Pod? (filtering + scoring)
6. Що таке requests і limits? Які є QoS класи (Guaranteed, Burstable, BestEffort)?
7. Що таке CrashLoopBackOff і як з цим розбиратися?
8. Різниця між labels, selectors і annotations.
9. Чому Pod не треба запускати напряму? Завжди через контролер.
10. Що таке namespace і що в нього входить / не входить?

[→ k8s-basics.md](kubernetes/k8s-basics.md)

---

## Kubernetes: Workloads

1. Як Deployment керує Pods? Через що проміжний шар?
2. Як працює Rolling Update? Що таке maxSurge і maxUnavailable?
3. Як зробити rollback Deployment до попередньої версії?
4. Коли використовувати StatefulSet замість Deployment?
5. Як StatefulSet забезпечує стабільні імена Pods і persistent storage?
6. Коли потрібен DaemonSet? Приклади use cases.
7. Різниця між Job і CronJob. Що таке concurrencyPolicy у CronJob?
8. Різниця між liveness, readiness і startup probes. Коли яку використовувати?
9. Що таке graceful shutdown у k8s? Навіщо preStop hook і як його налаштувати?
10. Що таке init containers і sidecars? Native sidecars у k8s 1.29+.
11. Що таке PodDisruptionBudget і чому він важливий?
12. Як зробити anti-affinity, щоб replicas були на різних Nodes?

[→ k8s-workloads.md](kubernetes/k8s-workloads.md)

---

## Kubernetes: Networking

1. Які базові принципи мережі в k8s? (Pod IPs, no NAT, CNI)
2. Що робить CNI plugin? Порівняй Calico, Cilium, Flannel.
3. Чому не звертаються до Pod за IP напряму? Що таке Service.
4. Які типи Services? (ClusterIP, NodePort, LoadBalancer, ExternalName, Headless)
5. Як працює ClusterIP під капотом? (kube-proxy, iptables/IPVS)
6. Як CoreDNS резолвить service names? Що всередині `/etc/resolv.conf` у Pod?
7. Що таке Ingress, чому це Controller + rules? Різниця з LoadBalancer.
8. Ingress vs Gateway API -- чому переходять на Gateway API?
9. Як обмежити трафік між Pods через NetworkPolicy? Default-deny pattern.
10. Коли варто використовувати Service Mesh (Istio/Linkerd)? Коли оверкілл.
11. Як дебажити "Service не маршрутизує трафік"? (endpoints, selector, probes)

[→ k8s-networking.md](kubernetes/k8s-networking.md)

---

## Kubernetes: Storage & Config

1. Різниця між PersistentVolume, PersistentVolumeClaim і StorageClass.
2. Як працює dynamic provisioning?
3. Які access modes у PVC? Чому не всі drivers підтримують RWX?
4. Що таке reclaimPolicy? Retain vs Delete -- коли що.
5. Як працює ConfigMap -- монтування як env vs volume? Що оновлюється автоматично?
6. Secret -- це шифрування чи кодування? Що треба для реальної безпеки.
7. Що таке External Secrets Operator і навіщо він потрібен?
8. Kustomize vs Helm -- коли що використовувати?
9. Як оновлювати конфіг у Pods без downtime? (reloader, checksum annotation)

[→ k8s-storage-config.md](kubernetes/k8s-storage-config.md)

---

## Kubernetes: Scaling & Operations

1. Які є рівні масштабування? (HPA, VPA, Cluster Autoscaler)
2. Як працює HPA? Формула розрахунку replicas.
3. Які вимоги до HPA? (metrics-server, requests)
4. Що дає KEDA, чого немає у HPA? (scale to zero, event-driven)
5. Як VPA і HPA конфліктують? Як уникнути.
6. Cluster Autoscaler vs Karpenter -- різниця.
7. Як працює RBAC? Role vs ClusterRole vs RoleBinding.
8. Що таке ServiceAccount і як Pod автентифікується до API?
9. Що таке IRSA (AWS) / Workload Identity (GCP)? Навіщо.
10. Як безпечно оновлювати кластер (drain, PDB, version skew)?
11. Як дебажити Pod у стані Pending? ImagePullBackOff? CrashLoop?
12. Чотири Golden Signals моніторингу.

[→ k8s-scaling-operations.md](kubernetes/k8s-scaling-operations.md)

---

## Terraform

**Базове:**
1. Що таке IaC і чому саме Terraform? (vs CloudFormation, Pulumi)
2. Чим відрізняються resource, data source, provider, module?
3. Що таке Terraform state? Що буде, якщо його втратити?
4. Як працює remote state і state locking? Чому DynamoDB потрібна для S3 backend?
5. Коли Terraform робить replace замість update? ForceNew.
6. Що робить `create_before_destroy`? Коли без нього не можна.
7. Різниця між `count` і `for_each`. Чому `for_each` краще.
8. Як переюзати Terraform-код між середовищами? (окремі root-модулі)
9. Як писати хороші модулі? Structure, inputs, outputs, versions.

**Практика:**
10. Що означає "Terraform хоче replace" на ресурсі, якого ти не міняв? (drift)
11. Як зробити drift detection в CI?
12. Як перейменувати resource без destroy? (`terraform state mv`)
13. Як імпортувати існуючий ресурс у state?
14. Як убезпечити критичні ресурси від випадкового destroy? (`prevent_destroy`)
15. Як зберігати секрети в Terraform? Що робити з секретами в state?
16. Як тестувати Terraform? (validate, tflint, terratest, OPA)
17. Чому не варто робити один великий state на все?
18. Що таке `.terraform.lock.hcl` і чи коммітити його?

[→ terraform-basics.md](terraform-aws/terraform-basics.md) | [→ terraform-best-practices.md](terraform-aws/terraform-best-practices.md)

---

## AWS: Core

**IAM:**
1. Різниця між User, Group, Role, Policy.
2. Identity-based vs Resource-based policies.
3. Що таке Permission Boundaries і SCP?
4. Як працює assume-role? Trust policy vs permission policy.
5. Як правильно давати AWS-права застосунку на EC2 / ECS / Lambda / EKS Pods?
6. Чому MFA на root критично важлива?

**VPC:**
7. Як побудувати HA VPC? (3 AZ, public/private subnets)
8. Різниця між Internet Gateway і NAT Gateway.
9. Security Group vs NACL -- stateful vs stateless.
10. Чому RDS завжди в private subnet?
11. VPC Endpoints -- для чого? Як економлять на NAT Gateway.
12. Peering vs Transit Gateway -- коли що.

**S3:**
13. Які storage classes є і коли що вибирати?
14. Як захистити S3 bucket від випадкового public exposure?
15. Чим pre-signed URL відрізняється від IAM access?
16. Versioning + lifecycle policy для backup-стратегії.
17. Як зробити CDN поверх S3 без public bucket? (OAC)

**RDS:**
18. Multi-AZ vs Read Replicas -- різниця.
19. Aurora vs RDS Postgres -- коли що.
20. Як зробити zero-downtime major version upgrade у RDS?
21. Як убезпечити RDS password? (Secrets Manager, rotation)

[→ aws-core-services.md](terraform-aws/aws-core-services.md)

---

## AWS: Serverless & Containers

1. Як обрати між Lambda, ECS Fargate, EKS, EC2?
2. Коли Lambda не підходить? (>15 хв, >10GB memory, потрібна stickyness)
3. Що таке cold start і як з ним боротись? (Provisioned Concurrency, keep warm)
4. Чому клієнти AWS SDK треба оголошувати поза handler?
5. ECS vs EKS -- коли який вибрати.
6. Fargate: плюси і коли дорого.
7. Як масштабувати ECS service? Target tracking на CPU vs custom metrics.
8. Як працює ALB target group health check з ECS?
9. Що таке API Gateway? HTTP API vs REST API -- різниця.
10. Коли SQS, коли SNS, коли EventBridge?
11. SQS FIFO vs Standard -- коли яка.
12. Як будувати fan-out pattern? (SNS + SQS)
13. Як зменшити AWS bill? (Savings Plans, right-sizing, Spot, lifecycle)
14. Як налаштувати alerts на несподівані витрати?

[→ aws-serverless-containers.md](terraform-aws/aws-serverless-containers.md)

---

## nginx

1. Чому nginx швидкий? Event-driven architecture vs Apache prefork.
2. Reverse proxy vs forward proxy -- яка різниця.
3. Які є load balancing алгоритми? Коли який.
4. Як nginx робить active і passive health checks?
5. Навіщо `proxy_set_header X-Forwarded-For`, `X-Real-IP`, `X-Forwarded-Proto`?
6. Як зробити TLS termination на nginx?
7. Як налаштувати кеш у nginx? Cache zones, invalidation.
8. Як зробити rate limiting? (limit_req_zone, limit_conn)
9. Як проксувати WebSocket? Які headers потрібні.
10. Що робити при 502/504? Діагностика.
11. nginx vs HAProxy vs Envoy vs Traefik -- коли що.

[→ nginx.md](nginx.md)

---

## Linux для DevOps

1. Що таке процес-зомбі, як виникає, як уникнути в Docker?
2. SIGTERM vs SIGKILL -- чим відрізняються, як обробляти.
3. Як знайти процес, який слухає порт 8080? (`ss -ltnp`, `lsof -i`)
4. Як подивитися, скільки open files використовує процес? Ulimit.
5. Як працюють chmod/chown? Що означає 755, 644?
6. Що таке setuid, setgid, sticky bit?
7. Як користуватися `find` для пошуку великих файлів? Старих файлів?
8. journalctl для systemd: корисні флаги.
9. Як дебажити "high load average"? Що таке USE method.
10. Hard link vs symbolic link -- коли що.
11. Як написати простий systemd service unit?
12. SSH tunneling: -L, -R, -D -- що робить кожен.

[→ linux-essentials.md](linux-essentials.md)

---

## Networking Fundamentals

1. OSI vs TCP/IP модель. На якому рівні працюють nginx, ALB, firewall.
2. TCP vs UDP. 3-way handshake.
3. Що таке TIME_WAIT і чому виникає?
4. HTTP/1.1 vs HTTP/2 vs HTTP/3. Multiplexing, HOL blocking.
5. HTTP idempotency: які методи idempotent і чому.
6. Як працює TLS handshake? Що таке chain of trust.
7. Що таке mTLS, де використовується.
8. DNS: як ієрархічно резолвиться `api.example.com`.
9. DNS record types: A, AAAA, CNAME, MX, TXT, NS, SRV.
10. CDN: як працює edge caching? Коли CDN не допоможе.
11. L4 vs L7 load balancing -- різниця.
12. Debug HTTP: `curl -v`, TLS: `openssl s_client`, DNS: `dig`.

[→ networking-fundamentals.md](networking-fundamentals.md)

---

## Monitoring & Observability

1. Три стовпи observability: metrics, logs, traces -- різниця і коли що.
2. Monitoring vs Observability. Unknown unknowns.
3. Як працює Prometheus? Pull-based model.
4. Prometheus metric types: counter, gauge, histogram, summary -- приклади.
5. Що таке cardinality explosion? Чому не можна user_id як label.
6. Як написати запит на error rate за 5 хвилин? (PromQL)
7. Alertmanager: routing, inhibitions, silencing.
8. RED, USE, Four Golden Signals -- коли який.
9. SLO, SLI, error budget -- концепції і приклади.
10. Структуроване логування -- чому JSON logs, а не text.
11. Distributed tracing: spans, trace context, sampling.
12. Alert fatigue -- як уникнути. Alert on symptoms, not causes.

[→ monitoring-observability.md](monitoring-observability.md)

---

## GitOps / ArgoCD / Helm

1. Чотири принципи GitOps.
2. Push vs Pull model деплою. Чому pull безпечніший.
3. Як працює ArgoCD? Компоненти.
4. ArgoCD sync policies: automated, selfHeal, prune -- що кожна робить.
5. ArgoCD vs Flux -- різниця і коли що.
6. Helm: структура chart, основні команди.
7. Helm vs Kustomize -- коли що. Чи можна комбінувати.
8. Як керувати секретами в GitOps? (Sealed Secrets, SOPS, ESO)
9. Що таке app-of-apps pattern?
10. Progressive delivery: Argo Rollouts / Flagger.

[→ gitops-argocd-helm.md](gitops-argocd-helm.md)

---

## System Design з DevOps ухилом

Питання, які поєднують кілька тем:

1. **Спроектуй рівень деплою для SaaS застосунку.** (VPC → EKS → Ingress → Services → RDS → S3 + CloudFront. GitOps через ArgoCD. Секрети через ESO.)

2. **Як побудувати zero-downtime deploy для сервісу з базою даних?** (Forward-compatible migrations, rolling update, PDB, readiness probes, graceful shutdown з preStop hook.)

3. **Як захистити мультирентіалі кластер?** (Namespaces per tenant, NetworkPolicy default-deny, ResourceQuota, RBAC per tenant, Pod Security Standards.)

4. **Як розв'язати "наш сайт повільний під час пікового навантаження"?** (Спочатку -- виміряти: де затримка? Front-end / ALB / App / DB. Далі: CDN, HPA, БД read replicas, кешування, async processing через SQS.)

5. **Як зробити DR (disaster recovery)?** (RTO/RPO визначити. Multi-region з Route 53 failover, cross-region replication S3, Aurora Global Database, regular DR tests.)

6. **Як мігрувати застосунок з on-prem у хмару?** (Оцінка залежностей, lift-and-shift vs refactor, phased rollout, data migration strategy, rollback plan.)

7. **Compliance: як пройти SOC 2 з boxes-ми на AWS/k8s?** (CloudTrail + Config, encryption everywhere, key rotation, access reviews, audit logs with retention, incident response runbooks.)

---

## Lightning round (швидкі питання)

_Дай 1-2 речення на кожне._

1. Що робить `docker run -it`?
2. Що станеться, якщо видалити PVC у k8s?
3. Що таке `kubectl apply` vs `kubectl create`?
4. Чому не можна `docker system prune -a --volumes` на prod?
5. Навіщо `readinessProbe` окремо від `livenessProbe`?
6. Що таке `latest` тег і чому його уникати?
7. Скільки AZ потрібно для HA у production?
8. Що таке `terraform plan -out=tfplan`?
9. Як RDS Multi-AZ відрізняється від read replicas?
10. Що таке "shift left" у security?
11. Навіщо ECR image scanning?
12. Що таке Chaos Engineering?
13. Blue-green vs canary -- яка різниця?
14. Scale to zero -- де працює (Lambda, Fargate з KEDA), де ні (звичайний HPA)?
15. Хто такий SRE і чим відрізняється від DevOps engineer?

---

## Поведінкові / досвід

1. Розкажи про найскладніший production incident і як ти його вирішив.
2. Приклад, коли ти запровадив автоматизацію, яка зекономила значний час.
3. Як ти балансуєш швидкість деплоїв і стабільність?
4. Розкажи про migration / proof-of-concept, який ти робив.
5. Як ти організовуєш on-call у своїй команді?
6. Що ти змінив би в CI/CD на попередньому місці роботи, якби почав заново?
7. Опиши приклад, коли SLO був порушений. Як ви це проаналізували?
8. Як ти переконуєш менеджмент інвестувати в infrastructure/tech debt?

---

## Підказка: як відповідати на технічні питання

1. **Уточни контекст.** "Який розмір системи? Скільки користувачів? Хмара чи on-prem?" -- часто відповідь залежить від деталей.
2. **Дай загальну картину, потім деталі.** Спочатку "використаю X тому що Y", потім докладно.
3. **Розкажи про trade-offs.** Нема ідеального рішення -- покажи, що бачиш і плюси, і мінуси.
4. **Ділись досвідом.** "На попередньому проекті ми стикалися з X, і вибрали Y, бо..."
5. **Чесність:** "Не знаю точно, але припустив би..." краще, ніж вигадка.
