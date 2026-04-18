# AWS: Serverless і контейнери (Lambda, ECS, EKS, Fargate)

## Як обрати між Lambda, ECS, EKS, EC2?

Спрощена матриця:

| Потреба                           | Варіант                       |
|-----------------------------------|-------------------------------|
| Event-driven, короткі функції     | **Lambda**                    |
| Контейнерний застосунок, простий  | **ECS on Fargate**            |
| Kubernetes або multi-cloud        | **EKS**                       |
| Legacy / specific kernel / sticky | **EC2**                       |
| Static site / SPA                 | **S3 + CloudFront**           |
| Container CRON / batch            | **ECS Scheduled Tasks**       |
| Background worker                 | **ECS Service** або **Lambda (SQS)** |

Шкала абстракції:
```
EC2          → ти керуєш ОС, runtime, scaling
ECS + EC2    → ти керуєш EC2 cluster, AWS керує оркестрацією
ECS + Fargate→ AWS керує все окрім контейнера
EKS + EC2    → Kubernetes + керуєш вузлами
EKS + Fargate→ Kubernetes + AWS керує вузлами
Lambda       → AWS керує все, ти пишеш функцію
```

Більше абстракції = менше гнучкості, але менше operational burden.

---

## Lambda: serverless функції

**AWS Lambda** -- запускає функцію у відповідь на подію. Ви не керуєте серверами, AWS масштабує автоматично.

**Характеристики:**
- Максимум **15 хвилин** на виконання.
- **128 MB - 10 GB** memory (CPU пропорційний memory).
- **10 GB ephemeral /tmp** storage (або EFS mount).
- **Cold start** -- перший виклик після idle = затримка (100ms для Node.js, до кількох секунд для JVM).
- Biлинг per millisecond виконання + кількість invocations.

**Тригери:**
- API Gateway (HTTP requests)
- S3 (object created/deleted)
- SQS / SNS
- DynamoDB Streams / Kinesis
- EventBridge (cron, events)
- CloudWatch Logs
- SDK invocation

**Приклад Node.js функції:**
```javascript
// handler.js
export const handler = async (event) => {
  const body = JSON.parse(event.body);

  // бізнес-логіка

  return {
    statusCode: 200,
    body: JSON.stringify({ ok: true })
  };
};
```

**Terraform:**
```hcl
resource "aws_lambda_function" "api" {
  function_name = "api"
  role          = aws_iam_role.lambda.arn
  handler       = "handler.handler"
  runtime       = "nodejs20.x"

  filename         = "function.zip"
  source_code_hash = filebase64sha256("function.zip")

  memory_size = 512
  timeout     = 30

  environment {
    variables = {
      NODE_ENV = "production"
      DB_URL   = aws_ssm_parameter.db_url.value
    }
  }

  tracing_config {
    mode = "Active"   # X-Ray tracing
  }
}
```

**Best practices:**
- **Keep it small** -- функція має робити одну річ. Швидше cold start, простіше тестувати.
- **Reuse SDK clients поза handler** -- вони переюзяться між invocations у теплому контейнері.
  ```javascript
  import { S3Client } from '@aws-sdk/client-s3';
  const s3 = new S3Client({});  // ← поза handler

  export const handler = async (event) => {
    const response = await s3.send(...);
  };
  ```
- **Provisioned Concurrency** -- тримає N warm instances (платно), зменшує cold start.
- **Layers** -- спільні залежності для кількох функцій.
- **Timeout** -- встановлюйте realistic, не 15 хв для всього.
- **Reserved concurrency** -- обмежує параллелізм (захист БД від перенавантаження).

**Типові патерни:**

**API (Lambda + API Gateway):**
```
User → API Gateway → Lambda → DynamoDB
```

**Async processing (SQS + Lambda):**
```
Producer → SQS → Lambda (до 1000 concurrent) → DB
```

**Fan-out (SNS + Lambda):**
```
Event → SNS → Lambda-1, Lambda-2, Lambda-3 (паралельно)
```

**Scheduled job:**
```
EventBridge (cron) → Lambda
```

---

## ECS: керовані контейнери

**ECS (Elastic Container Service)** -- AWS-нативний оркестратор контейнерів. Простіший за Kubernetes.

**Концепції:**
- **Cluster** -- логічне групування.
- **Task Definition** -- шаблон для контейнерів (image, CPU, RAM, мережа, IAM).
- **Task** -- запущена копія Task Definition (аналог Pod).
- **Service** -- підтримує desired count tasks, інтегрується з LB (аналог Deployment).

**Launch types:**
- **Fargate** -- serverless. AWS керує вузлами. Сплачуєте за vCPU + RAM × час.
- **EC2** -- ви керуєте cluster з EC2 instances. Дешевше при високому завантаженні, але треба управляти.

**Приклад Fargate service через Terraform:**
```hcl
# 1. Cluster
resource "aws_ecs_cluster" "main" {
  name = "prod"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# 2. Task Definition
resource "aws_ecs_task_definition" "api" {
  family                   = "api"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 512   # 0.5 vCPU
  memory                   = 1024  # 1 GB

  execution_role_arn = aws_iam_role.ecs_execution.arn
  task_role_arn      = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = "api"
    image = "123.dkr.ecr.us-east-1.amazonaws.com/api:v1.2.3"

    portMappings = [{
      containerPort = 3000
      protocol      = "tcp"
    }]

    environment = [
      { name = "NODE_ENV", value = "production" }
    ]

    secrets = [{
      name      = "DB_PASSWORD"
      valueFrom = aws_ssm_parameter.db_password.arn
    }]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = "/ecs/api"
        awslogs-region        = "us-east-1"
        awslogs-stream-prefix = "ecs"
      }
    }

    healthCheck = {
      command = ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
    }
  }])
}

# 3. Service
resource "aws_ecs_service" "api" {
  name            = "api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = module.vpc.private_subnets
    security_groups  = [aws_security_group.api.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api"
    container_port   = 3000
  }

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }

  health_check_grace_period_seconds = 60
}
```

**Auto Scaling:**
```hcl
resource "aws_appautoscaling_target" "api" {
  max_capacity       = 20
  min_capacity       = 3
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "api_cpu" {
  name               = "cpu-70"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  service_namespace  = aws_appautoscaling_target.api.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value = 70

    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}
```

**ECS vs EKS: коли який вибрати?**

| Аспект                  | ECS                          | EKS                               |
|-------------------------|------------------------------|-----------------------------------|
| Складність              | Нижча                        | Вища                              |
| Тoolchain               | AWS-специфічний              | Kubernetes ecosystem              |
| Multi-cloud portability | Ні                           | Так                               |
| Ціна control plane      | Безкоштовно                  | $0.10/hr (~$73/міс)               |
| Community / ecosystem   | Менше                        | Величезне (Helm, operators...)    |
| Коли обирати            | Простий AWS-only workload    | K8s know-how, складні потреби    |

---

## EKS: Kubernetes on AWS

**EKS (Elastic Kubernetes Service)** -- managed Kubernetes control plane. AWS керує master nodes, ви платите $0.10/год за cluster + за worker nodes.

**Worker nodes options:**
- **Managed Node Groups** -- AWS керує EC2 instances (ASG).
- **Self-managed nodes** -- ви керуєте повністю.
- **Fargate** -- serverless Pods (один Pod = один Fargate task).
- **Karpenter** -- smart autoscaler, заміна Cluster Autoscaler.

**Ключові інтеграції:**
- **IRSA (IAM Roles for Service Accounts)** -- Pods автентифікуються до AWS без статичних ключів.
- **AWS Load Balancer Controller** -- створює ALB/NLB для Kubernetes Services та Ingress.
- **EBS CSI / EFS CSI drivers** -- PersistentVolumes на AWS storage.
- **CloudWatch Container Insights** -- моніторинг.
- **VPC CNI** -- кожен Pod отримує IP з VPC (можна напряму досягати з EC2).

**Базовий кластер через Terraform:**
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "prod"
  cluster_version = "1.29"

  cluster_endpoint_public_access = true
  cluster_addons = {
    coredns                = {}
    kube-proxy             = {}
    vpc-cni                = {}
    aws-ebs-csi-driver     = {}
  }

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = {
      desired_size = 3
      min_size     = 3
      max_size     = 10

      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"
    }

    spot = {
      desired_size = 0
      min_size     = 0
      max_size     = 20

      instance_types = ["m5.large", "m5a.large", "m5d.large"]
      capacity_type  = "SPOT"

      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }
  }

  enable_irsa = true
}
```

**IRSA приклад:**
```hcl
module "s3_reader_irsa" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name = "s3-reader"

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["default:s3-reader"]
    }
  }

  role_policy_arns = {
    s3 = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
  }
}
```

У Kubernetes:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/s3-reader
```

Тепер Pod з цим SA може звертатись до S3 без статичних credentials.

---

## Fargate: коли використовувати?

**Fargate (для ECS або EKS)** -- serverless контейнери. Немає вузлів, про які треба думати.

**Pros:**
- Немає управління вузлами.
- Pod-per-VM ізоляція (безпечніше за shared EC2).
- Auto-scaling на рівні AWS.

**Cons:**
- **Дорожче** -- ~20-40% дорожче, ніж EC2 при високому завантаженні.
- **Холодний старт** повільніший, ніж на готовому EC2.
- **Деякі обмеження:** немає hostPath, немає privileged containers, нема GPU (поки що).
- **Мінімальний розмір** -- 0.25 vCPU, 512 MB RAM. Дорого для дрібних workloads.

**Коли брати Fargate:**
- Малі команди без SRE capacity.
- Spiky workloads (батчі, CRONs).
- Isolation requirements.

**Коли не брати:**
- Постійне високе навантаження -- EC2 дешевше.
- Потрібні specific kernel features.
- Потрібні DaemonSets (Fargate не підтримує).

---

## API Gateway: HTTP entrypoint

**API Gateway** -- managed API endpoint. Підтримує REST, HTTP, WebSocket.

**HTTP API vs REST API:**
- **HTTP API** -- новіший, дешевший, простіший, швидший. Рекомендується.
- **REST API** -- старіший, більше фіч (API keys, usage plans, custom auth integrations).

**Інтеграції:**
- Lambda (найпопулярніше).
- HTTP backend (proxy).
- AWS services (DynamoDB, SQS напряму).

**Ключові фічі:**
- **Throttling** -- rate limiting per API або per key.
- **Authorizers** -- JWT, Lambda, Cognito.
- **CORS** -- вбудована підтримка.
- **Request/response transformation** -- VTL або Lambda.
- **Stages** -- dev/prod з окремим URL.
- **Custom domain** + ACM certificate.

**Альтернатива:** **Application Load Balancer (ALB)** для контейнерних застосунків. Дешевший при високому трафіку, але менше фіч.

---

## CloudFront: CDN і edge

**CloudFront** -- CDN від AWS. Кешує статику, переносить обчислення на edge (Lambda@Edge, CloudFront Functions).

**Use cases:**
- Static hosting (S3 + CloudFront).
- API acceleration.
- SSL termination у 400+ локаціях.
- Захист від DDoS (з AWS Shield / WAF).

```
User (Київ) → CloudFront edge (Warsaw) → Origin (S3 / ALB у us-east-1)
             │
             └── кеш на edge, ~5ms до користувача
```

**Lambda@Edge** -- запуск Node.js/Python на edge. Для modifications запитів, A/B testing, авторизації.

**CloudFront Functions** -- простіші і дешевші за Lambda@Edge, але тільки JS і обмежені (< 1 мс виконання, без мережі).

---

## Messaging: SQS, SNS, EventBridge

**SQS (Simple Queue Service):**
- Point-to-point queue.
- **Standard** (at-least-once, unordered) або **FIFO** (exactly-once, ordered).
- Messages до 256 KB, retention до 14 днів.
- **Visibility timeout** -- messages hidden after read; якщо consumer не зробив `delete` за цей час, повертаються в чергу.
- **Dead-letter queue (DLQ)** -- повідомлення після N failed attempts переходять сюди.

**SNS (Simple Notification Service):**
- Pub/sub -- один publisher, багато subscribers.
- Subscribers: SQS, Lambda, HTTP, email, SMS.
- Використовується для fan-out.

**EventBridge:**
- Event bus з filtering.
- Інтегрований з AWS services (events від 100+ AWS сервісів + 3rd party: Datadog, Zendesk, Shopify).
- Routing rules за event pattern.
- Заміна CloudWatch Events.

**Патерни:**
```
SNS fan-out:
Publisher → SNS → SQS-queue-1 → Consumer-1
                → SQS-queue-2 → Consumer-2
                → Lambda

SQS + Lambda:
Producer → SQS → Lambda (batch processing)

EventBridge:
User signs up → EventBridge → Lambda (welcome email)
                            → SQS (provision resources)
                            → Kinesis (analytics)
```

---

## Cost optimization: типові важелі

**1. Правильний розмір instances:**
- Використовуйте **Compute Optimizer** -- рекомендує правильні розміри.
- `t3/t4g` (ARM) замість `m5` для невимогливих workloads.
- Graviton (ARM) -- на 20-40% дешевше і часто швидше.

**2. Spot instances** для stateless workloads:
- ASG з mixed instances policy (70% on-demand + 30% spot).
- EKS з Karpenter + spot node pool.

**3. Reserved Instances / Savings Plans:**
- Compute Savings Plan покриває EC2, Lambda, Fargate.
- 1-year no upfront -- ~30% знижка без commitment.

**4. Видалення зайвого:**
- **Unattached EBS volumes** -- платите за storage навіть якщо не прикріплено.
- **Unused Elastic IPs** -- $0.005/год якщо не прив'язаний.
- **Old snapshots** -- застарілі EBS snapshots накопичуються.
- **CloudWatch Logs без retention** -- ростуть вічно.

**5. Data transfer costs:**
- Inter-AZ traffic платний ($0.01/GB). Розміщуйте пов'язані сервіси в одній AZ (з trade-off на HA).
- CloudFront cache hit rate -- менше egress з origin.
- VPC Endpoints замість NAT Gateway для AWS-service traffic.

**6. S3 lifecycle:**
- Переміщення старих об'єктів в Glacier / Deep Archive.
- Incomplete multipart uploads -- автоматично видаляти через 7 днів.

**7. Моніторинг витрат:**
- **AWS Budgets** з alerts.
- **Cost Explorer** для аналізу.
- **Cost and Usage Reports (CUR)** -- detailed billing.
- Tag-based cost allocation.

---

## Типові помилки на AWS

**1. Public S3 bucket з даними користувачів:**
Завжди Public Access Block. Якщо треба публічна роздача -- CloudFront з OAC, а сам bucket private.

**2. RDS у public subnet:**
Ніколи. RDS -- тільки private. Доступ через bastion / Session Manager / VPN.

**3. Security Group з `0.0.0.0/0` на всі порти:**
Витік очевидний. SG по portу і source.

**4. Hardcoded IAM keys у коді:**
Використовуйте IAM roles (EC2 instance profile, ECS task role, Lambda execution role, IRSA). Для CI -- OIDC.

**5. Один AWS account на все:**
Змішування prod/dev/security в одному акаунті -- великий ризик. AWS Organizations з окремими акаунтами.

**6. Немає MFA на root:**
Найкритичніше. MFA + root credentials у сейфі, не використовувати в повсякденній роботі.

**7. Немає CloudTrail:**
Без неї не знаєте, хто що робив. Централізований CloudTrail у Security акаунт.

**8. Немає cost alerts:**
Одного дня побачите $10k bill за leaked key, що майнив криптовалюту. Budgets + billing alarms.

**9. Неправильні лімити:**
AWS має default limits (number of EIPs, EC2 instances). Перевірте та підніміть заздалегідь для prod.

**10. "Works in console, broken in Terraform":**
Console часто ставить різні defaults. Завжди перевіряйте `terraform plan` уважно.
