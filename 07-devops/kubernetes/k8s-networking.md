# Kubernetes: Networking (Services, Ingress, NetworkPolicy)

## Як влаштована мережа в Kubernetes?

**Базові правила Kubernetes networking:**
1. Кожен Pod має свій унікальний IP у межах кластера.
2. Контейнери всередині Pod діляться одним мережевим namespace (localhost).
3. Pod може напряму звернутися до будь-якого іншого Pod за його IP -- **без NAT**.
4. Ноди бачать Pod IPs напряму.

```
Node 1 (10.0.1.1)            Node 2 (10.0.1.2)
┌──────────────────┐         ┌──────────────────┐
│  Pod (10.244.0.3)│◄────────┤  Pod (10.244.1.5)│
│  Pod (10.244.0.4)│         │  Pod (10.244.1.6)│
└──────────────────┘         └──────────────────┘
       │                            │
       └────── Overlay network ─────┘
            (Calico / Cilium / Flannel)
```

Реалізація цього -- **CNI plugin** (Container Network Interface). Найпопулярніші:
- **Calico** -- BGP, підтримує NetworkPolicy.
- **Cilium** -- eBPF, висока продуктивність, advanced policies.
- **Flannel** -- простий overlay, без NetworkPolicy.
- **AWS VPC CNI, Azure CNI** -- хмарні рідні.

**Проблема:** Pod IP нестабільний -- при рестарті Pod отримує новий IP. Тому ніхто не звертається до Pod за IP напряму. Для стабільного доступу -- використовуються **Services**.

---

## Service: навіщо і які типи?

Service -- це стабільна точка доступу до набору Pods. Вона має стабільний IP (ClusterIP) і DNS-ім'я.

```
       Client
         │
         ▼
   Service (10.96.1.10)           ← стабільний IP
         │
     ┌───┴───┬─────┐
     ▼       ▼     ▼
   Pod     Pod   Pod              ← динамічні IPs, знаходить через selector
  (:3000) (:3000)(:3000)
```

**Як Service знаходить Pods:** за **label selector**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api                    # ← знаходить всі Pods з цим label
  ports:
    - port: 80                  # порт Service
      targetPort: 3000          # порт контейнера
      protocol: TCP
```

### Типи Services

**1. ClusterIP** (дефолт) -- внутрішній IP, доступний тільки з кластера.
```yaml
spec:
  type: ClusterIP
```
Використання: service-to-service всередині кластера.

**2. NodePort** -- відкриває порт на кожному Node (30000-32767).
```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 3000
      nodePort: 30080    # Node:30080 → Service
```
Використання: простий external доступ без LoadBalancer. Не рекомендується для production.

**3. LoadBalancer** -- запитує у хмарного провайдера зовнішній балансувальник (AWS ALB/NLB, GCP LB).
```yaml
spec:
  type: LoadBalancer
```
Хмара створить public IP і направить трафік у кластер.

**4. ExternalName** -- DNS CNAME на зовнішній ресурс, без selector.
```yaml
spec:
  type: ExternalName
  externalName: api.example.com
```
Use case: абстрагувати зовнішній сервіс.

**5. Headless Service** -- без ClusterIP, повертає IPs Pods напряму через DNS.
```yaml
spec:
  clusterIP: None
  selector:
    app: mysql
```
Use case: StatefulSet (отримати `mysql-0.mysql`, `mysql-1.mysql` як окремі IP).

---

## Як працює kube-proxy і ClusterIP під капотом?

Коли ви робите запит на ClusterIP `10.96.1.10:80`:

1. Пакет йде з Pod.
2. **iptables / IPVS** правила на Node (встановлені kube-proxy) перехоплюють.
3. Перенаправляють на IP одного з Pods за алгоритмом балансування (round-robin для iptables, більше варіантів для IPVS).

```
Pod → 10.96.1.10:80 → (iptables DNAT) → 10.244.0.3:3000 (конкретний Pod)
```

Нема реального "Service процесу" -- це просто набір мережевих правил. Тому `curl 10.96.1.10` працює -- але `ping` не працює (нема ICMP обробника).

**kube-proxy modes:**
- **iptables** (дефолт) -- швидко, але O(n) правил.
- **IPVS** -- краще для великих кластерів, більше алгоритмів балансування.
- **nftables** (новіше) -- заміна iptables.

---

## DNS у кластері: як Service знаходять один одного?

Кожен Service отримує DNS-запис у вигляді:
```
<service>.<namespace>.svc.cluster.local
```

**CoreDNS** (типовий DNS сервер k8s) резолвить ці імена.

```bash
# З Pod в namespace "default"
curl http://api                              # api.default.svc.cluster.local
curl http://api.payments                     # api.payments.svc.cluster.local
curl http://api.payments.svc.cluster.local   # повне ім'я
```

`/etc/resolv.conf` в Pod автоматично налаштовано:
```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10                        # CoreDNS
```

Тому коротке `api` розгортається у `api.default.svc.cluster.local`.

**Headless Service** DNS:
```
mysql.default.svc.cluster.local    → повертає IPs всіх Pods
mysql-0.mysql.default.svc          → IP конкретного Pod StatefulSet
```

---

## Що таке Ingress і для чого він?

**Ingress** -- шар L7 (HTTP/HTTPS) routing для зовнішнього доступу до Services. Дозволяє мати **один балансувальник** для багатьох застосунків, з route-правилами.

```
Internet
   │
   ▼
LoadBalancer (один для всіх)
   │
   ▼
┌───────────── Ingress Controller (NGINX, Traefik, ALB) ──────────┐
│                                                                  │
│  api.example.com     → Service: api                              │
│  admin.example.com   → Service: admin                            │
│  example.com/app     → Service: web                              │
│  example.com/static  → Service: cdn                              │
└──────────────────────────────────────────────────────────────────┘
```

**Ingress сам по собі нічого не робить** -- це декларація правил. Треба встановити **Ingress Controller** (NGINX Ingress, Traefik, AWS ALB Controller), який читає Ingress і налаштовує реальний proxy.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin
                port:
                  number: 80
```

**LoadBalancer vs Ingress:**

| Аспект            | LoadBalancer Service     | Ingress                       |
|-------------------|--------------------------|-------------------------------|
| Рівень            | L4 (TCP/UDP)              | L7 (HTTP/HTTPS)               |
| Routing           | Немає                      | За host, path, headers        |
| TLS               | Треба вручну               | Вбудована підтримка           |
| Кількість на застосунок | По одному              | Один для багатьох             |
| Вартість (AWS)    | LB per Service            | Один ALB на всі               |

---

## Gateway API -- наступник Ingress

Ingress -- старий API з обмеженнями (багато функцій реалізовано через annotations, не стандартизовано). **Gateway API** -- сучасна заміна, з більш гнучкою моделлю:

- **GatewayClass** -- тип Gateway (ким надається -- NGINX, Envoy тощо).
- **Gateway** -- слухач (порти, TLS).
- **HTTPRoute** / **TCPRoute** / **GRPCRoute** -- правила маршрутизації.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
    - name: main-gateway
  hostnames: ["api.example.com"]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /v1 }
      backendRefs:
        - name: api-v1
          port: 80
    - matches:
        - path: { type: PathPrefix, value: /v2 }
      backendRefs:
        - name: api-v2
          port: 80
```

Gateway API краще розділяє відповідальності (Platform team -- Gateway, App team -- Routes) і стандартизує advanced features (traffic splitting, headers modification, mirroring).

---

## NetworkPolicy: як обмежити трафік між Pods?

За замовчуванням **всі Pods можуть звертатися до всіх Pods** у кластері. NetworkPolicy дозволяє обмежити це.

**Важливо:** NetworkPolicy працює тільки якщо CNI її підтримує (Calico, Cilium). Flannel -- ні.

**Приклад: дозволити трафік до `api` тільки з `frontend`:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 3000
```

**Default deny all** (рекомендовано як базу):
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}           # всі Pods у namespace
  policyTypes: [Ingress, Egress]
  # ніяких правил = все заборонено
```

Потім додаєте selective allow policies.

**Egress правила** -- обмежити, куди Pods можуть звертатися:
```yaml
spec:
  egress:
    - to:
        - podSelector:
            matchLabels: { app: db }
      ports:
        - port: 5432
    - to:                              # дозволити DNS
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
```

**Patterns:**
- Ізолювати production namespaces від dev.
- Заборонити Pods звертатися до metadata сервісу хмари (AWS IMDS).
- Frontend не може звертатися напряму до БД, тільки через API.

---

## Service Mesh: коли і навіщо?

Service Mesh (Istio, Linkerd, Cilium Service Mesh) -- шар L7 керування трафіком через **sidecar proxies** (Envoy) у кожному Pod.

**Що додає:**
- **mTLS** між Pods автоматично.
- **Traffic splitting** -- canary, A/B.
- **Retry / timeout / circuit breaker** на рівні мережі (без змін у коді).
- **Observability** -- метрики, traces, логи для кожного запиту.
- **Authorization** -- хто може звертатися до кого на рівні HTTP методів.

**Ціна:**
- Додатковий контейнер у кожному Pod (CPU, memory, затримка).
- Складність debugging.
- Стрімкий learning curve.

**Коли варто:** велика мікросервісна архітектура (50+ сервісів), необхідність mTLS, advanced traffic management. Для 5-10 сервісів -- зазвичай надлишок.

---

## Типові проблеми з Networking

**1. Service не маршрутизує трафік:**
- Перевірте `selector` -- чи збігається з Pod labels.
- `kubectl get endpoints <service>` -- якщо endpoints порожні, значить selector нікого не знаходить.
- Чи є Pods з `Ready` станом? Якщо readinessProbe fail -- Pod не в endpoints.

**2. DNS не резолвить:**
- CoreDNS Pods живі? `kubectl get pods -n kube-system -l k8s-app=kube-dns`.
- `/etc/resolv.conf` правильний у Pod?
- Спробуйте повне ім'я: `api.default.svc.cluster.local`.

**3. Ingress 502/503:**
- Ingress Controller запущений?
- Backend Service має endpoints?
- Порт у Ingress збігається з `service.spec.ports.port`?
- TLS секрет у тому ж namespace, що й Ingress?

**4. NetworkPolicy блокує трафік:**
- CNI підтримує NetworkPolicy?
- Egress policy може блокувати DNS (UDP :53) -- завжди дозволяйте окремо.
- Тестувати політики перед ввімкненням `default-deny`.

**5. `hostNetwork: true` обминає всю мережу k8s:**
Pod використовує мережу Node напряму. Зазвичай не треба -- використовується для system DaemonSets.

---

## Приклад: повний стек networking

```yaml
# 1. Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: myapp:1.0
          ports:
            - containerPort: 3000
---
# 2. Service (ClusterIP)
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 3000
---
# 3. Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
---
# 4. NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { name: ingress-nginx }
      ports:
        - port: 3000
```

Потік запиту:
```
Internet → LoadBalancer (AWS NLB)
         → ingress-nginx controller
         → Service "api"
         → kube-proxy (iptables)
         → один з Pods
         → container на :3000
```
