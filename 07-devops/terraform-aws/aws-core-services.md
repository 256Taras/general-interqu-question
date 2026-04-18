# AWS: Core services (VPC, EC2, S3, IAM, RDS)

## AWS account, Regions, Availability Zones

**Account** -- ізольований контейнер для ресурсів і рахунків. Для великих організацій -- **AWS Organizations** з кількома акаунтами (production, staging, dev, security, logging).

**Region** -- географічно ізольована група data centers. Наприклад, `us-east-1` (Virginia), `eu-west-1` (Ireland). Ресурси одного region не бачать інших, якщо явно не налаштовано.

**Availability Zone (AZ)** -- один або кілька data centers у межах region, з незалежним живленням, мережею, охолодженням. `us-east-1` має AZs `us-east-1a`, `us-east-1b`, і т.д.

**Правило:** для high availability -- деплойте у мінімум 2 AZ. Для disaster recovery -- multi-region.

**Edge locations** -- для CloudFront CDN, Route 53 DNS, Global Accelerator. Їх сотні по світу.

---

## IAM: ідентифікація та доступ

**Концепції:**
- **User** -- людина / система з login/password або access keys.
- **Group** -- набір users зі спільними правами.
- **Role** -- тимчасова identity. Приймається сервісами (EC2, Lambda) або users через `assume-role`.
- **Policy** -- JSON-документ, що описує дозволи.

**Приклад policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "IpAddress": { "aws:SourceIp": "10.0.0.0/16" }
      }
    }
  ]
}
```

**Типи policies:**
- **Identity-based** -- прикріплена до user/group/role ("що може робити").
- **Resource-based** -- прикріплена до ресурсу (S3 bucket policy) ("хто може звертатись").
- **Permission boundaries** -- максимум, що може дозволити role/user.
- **SCP (Service Control Policy)** -- на рівні Organizations, обмежує всі акаунти.

### Best practices
- **Root account** -- тільки для initial setup. MFA обов'язково. Не використовуйте щоденно.
- **MFA** для всіх IAM users з console access.
- **Least privilege** -- дозволити тільки необхідне. Починайте з нуля, додавайте.
- **Roles замість access keys** для сервісів:
  - EC2 → IAM Instance Profile.
  - Lambda → Execution Role.
  - EKS Pods → IRSA (IAM Roles for Service Accounts).
  - GitHub Actions → OIDC federation (не статичні ключі).
- **Access Analyzer** -- виявляє публічні ресурси, cross-account access.
- **Credential rotation** -- ключі ротируйте регулярно.
- **CloudTrail** -- аудит всіх API викликів.

---

## VPC: мережева основа

**VPC (Virtual Private Cloud)** -- ізольована віртуальна мережа в межах AWS account і region. Ви контролюєте IP range, subnets, routing.

```
┌─────────────── VPC (10.0.0.0/16) ─────────────────┐
│                                                   │
│  ┌──── Public subnet ──────┐  ┌──── Public subnet┐│
│  │  10.0.1.0/24 (AZ-1a)   │  │  10.0.2.0/24 (AZ-1b)││
│  │  ┌─────┐ ┌─────┐        │  │  ┌─────┐         ││
│  │  │ ALB │ │ NAT │        │  │  │ ALB │         ││
│  │  └─────┘ └─────┘        │  │  └─────┘         ││
│  └────────────────────────┘  └───────────────────┘│
│                                                   │
│  ┌──── Private subnet ─────┐  ┌──── Private subnet┐│
│  │  10.0.11.0/24 (AZ-1a)  │  │  10.0.12.0/24 (AZ-1b)│
│  │  ┌─────┐                │  │  ┌─────┐         ││
│  │  │ EC2 │                │  │  │ EC2 │         ││
│  │  └─────┘                │  │  └─────┘         ││
│  └────────────────────────┘  └───────────────────┘│
│                                                   │
│  Route Tables, Internet Gateway, NAT GW, etc.    │
└───────────────────────────────────────────────────┘
```

**Компоненти:**
- **CIDR block** -- IP range (`10.0.0.0/16` -- 65k адрес).
- **Subnet** -- частина VPC в одному AZ.
  - **Public subnet** -- route до Internet Gateway.
  - **Private subnet** -- без прямого internet access. Для isходящого -- через NAT Gateway.
- **Internet Gateway (IGW)** -- з'єднує VPC з Інтернетом (для public subnets).
- **NAT Gateway** -- egress для private subnets (Pod → Internet). Managed, платний.
- **Route Table** -- правила маршрутизації per subnet.
- **Security Group** -- stateful firewall на рівні instance (default deny inbound).
- **NACL** -- stateless firewall на рівні subnet (менш популярно).

**Приклад Terraform:**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "main"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false    # по NAT на кожну AZ (HA)
  enable_dns_hostnames   = true
  enable_dns_support     = true
}
```

**Важливо:**
- Не використовуйте `172.17.0.0/16` -- дефолтний Docker bridge, буде конфлікт.
- Залиште запас на зростання. `/16` -- 65k IPs, `/24` subnet -- 251 usable.
- NAT Gateway -- дорогий (~$0.045/год + data transfer). Для dev -- NAT Instance або VPC Endpoints.

---

## Security Groups vs NACLs

**Security Group (SG):**
- **Stateful** -- якщо дозволили вихідний трафік, відповідь приходить автоматично.
- На рівні **instance / ENI**.
- Тільки **allow** правила.
- Default: deny all inbound, allow all outbound.

**NACL (Network ACL):**
- **Stateless** -- треба явно дозволяти і в одну, і в другу сторону.
- На рівні **subnet**.
- **Allow і deny** правила з пріоритетом (rule number).
- Default: дефолтна NACL -- allow all. Custom -- deny all.

**Приклад SG:**
```hcl
resource "aws_security_group" "web" {
  name   = "web"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    cidr_blocks     = ["0.0.0.0/0"]
  }

  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # тільки з ALB
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**Правило:** використовуйте SG для найбільшої ізоляції, NACL -- тільки для additional defense (наприклад, заблокувати діапазон IP).

---

## EC2: віртуальні машини

**EC2 (Elastic Compute Cloud)** -- VM у хмарі.

**Instance types:**
- **General purpose** -- `t3`, `t4g`, `m5`, `m6i` -- збалансовані.
- **Compute optimized** -- `c5`, `c6i` -- CPU-heavy (коди, ML).
- **Memory optimized** -- `r5`, `r6i`, `x2` -- великі БД.
- **Storage optimized** -- `i3`, `d3` -- багато NVMe диску.
- **GPU** -- `p3`, `g5` -- ML, відео.

`t` family -- **burstable** (credits для короткотривалих CPU-пічних задач). Добре для API з низьким середнім навантаженням.

**Billing models:**
- **On-Demand** -- pay-per-hour, найдорожче, без зобов'язань.
- **Reserved Instances** -- 1/3 роки commitment, -30-75% знижка.
- **Savings Plans** -- гнучкіша альтернатива RI.
- **Spot Instances** -- до -90%, але AWS може їх забрати з 2-хв warning. Для batch / stateless workloads.

**Приклад:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  subnet_id              = module.vpc.private_subnets[0]
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.web.name

  root_block_device {
    volume_size = 30
    volume_type = "gp3"
    encrypted   = true
  }

  user_data = templatefile("./user-data.sh", {
    env = var.environment
  })

  tags = { Name = "web-01" }
}
```

**Auto Scaling Groups (ASG):** автоматично підтримує N instances. З Launch Template або Launch Config.

**EC2 vs ECS/EKS/Lambda:** для нових застосунків -- рідко потрібен EC2 напряму. Переважно йдуть з ECS/Fargate (простіше), EKS (для Kubernetes) або Lambda (серверleless).

---

## S3: object storage

**S3** -- масштабований object storage. Ієрархія: **bucket → objects (keys)**.

**Storage classes:**
- **Standard** -- гаряче сховище, частий доступ.
- **Standard-IA** / **One Zone-IA** -- Infrequent Access, дешевше, за retrieve фe̶кo.
- **Glacier Instant/Flexible/Deep Archive** -- архіви, від секунд до 12 год retrieval.
- **Intelligent-Tiering** -- автоматично переміщує між класами залежно від patterns.

**Характеристики:**
- **Durability** -- 11 дев'яток (99.999999999%) -- дані практично не губляться.
- **Availability** -- від 99.5% (One Zone-IA) до 99.99% (Standard).
- **Eventual consistency** (до 2020) → **Strong consistency** (тепер) для read-after-write.

**Ключові фічі:**
- **Versioning** -- зберігати старі версії об'єктів.
- **Lifecycle policies** -- автоматичне переміщення між класами / видалення.
- **Server-side encryption (SSE)** -- SSE-S3, SSE-KMS, SSE-C.
- **Pre-signed URLs** -- тимчасовий доступ до об'єктів без IAM.
- **Event notifications** -- тригерити Lambda / SQS / SNS при змінах.
- **Transfer Acceleration** -- швидше завантаження через CloudFront edge.
- **Cross-Region Replication** -- для DR.

```hcl
resource "aws_s3_bucket" "uploads" {
  bucket = "myapp-uploads-prod"
}

resource "aws_s3_bucket_public_access_block" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
  }
}

resource "aws_s3_bucket_versioning" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_lifecycle_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  rule {
    id     = "archive-old"
    status = "Enabled"
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
    expiration { days = 2555 }    # 7 years
  }
}
```

**Типові помилки:**
- ❌ Public bucket без Public Access Block -- витоки даних.
- ❌ Bucket name з sensitive info (компанія, env) -- глобально унікальні.
- ❌ Немає encryption -- compliance issue.
- ❌ Немає versioning -- випадкове видалення = втрата.

---

## RDS: реляційні БД

**RDS** -- managed реляційні БД: MySQL, PostgreSQL, MariaDB, SQL Server, Oracle, Amazon Aurora.

**Aurora** -- AWS-нативний варіант MySQL/PostgreSQL. Краща продуктивність, автомасштабування storage, 6 копій даних в 3 AZs.

**Ключові фічі:**
- **Automated backups** -- щодня + point-in-time recovery.
- **Multi-AZ** -- synchronous standby replica в іншій AZ. Auto-failover ~30-60 сек при падінні primary.
- **Read replicas** -- до 5 (15 для Aurora) asynchronous replicas для read scale.
- **Encryption at rest** -- через KMS.
- **Parameter groups** -- конфігурація БД.

**Важливо:**
- **Не використовуйте RDS у публічному subnet.** Private only, доступ через bastion / VPN / Session Manager.
- **Backup retention** -- мінімум 7 днів для prod. Max 35.
- **Monitoring** -- Enhanced Monitoring + Performance Insights.
- **Мінімальний instance** для prod -- `db.t3.medium` або вище. `db.t3.micro` -- тільки dev.

```hcl
resource "aws_db_instance" "main" {
  identifier     = "myapp-prod"
  engine         = "postgres"
  engine_version = "16.1"
  instance_class = "db.r6g.large"

  allocated_storage     = 100
  max_allocated_storage = 500      # auto-scaling storage
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn

  db_name  = "myapp"
  username = "postgres"
  password = random_password.db.result    # з Secrets Manager

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  multi_az               = true
  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"

  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "myapp-prod-final-${formatdate("YYYY-MM-DD", timestamp())}"

  performance_insights_enabled = true
  monitoring_interval          = 60
  enabled_cloudwatch_logs_exports = ["postgresql"]
}
```

---

## CloudWatch: monitoring і logs

**CloudWatch** -- observability platform.

**Компоненти:**
- **Metrics** -- числові значення (CPU, RAM, latency). Вбудовані + custom.
- **Logs** -- log groups, streams. З retention policy.
- **Alarms** -- алерти на метрики.
- **Dashboards** -- візуалізація.
- **Events / EventBridge** -- обробка подій (cron, pattern matching).
- **Contributor Insights** -- аналіз логів у реальному часі.

**Приклад alarm:**
```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "rds-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 80

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.id
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

**Log retention** -- default "Never expire" -- обов'язково поставте retention (14-90 днів для application logs).

---

## Маленький checklist для production VPC на AWS

- [ ] VPC з CIDR блоком, який не конфліктує з onprem і Docker.
- [ ] 3 AZs, public + private subnets у кожній.
- [ ] NAT Gateway у кожній AZ (для HA) або VPC Endpoints для AWS services.
- [ ] Всі бази в private subnets.
- [ ] Security Groups з least privilege (not `0.0.0.0/0` inbound).
- [ ] VPC Flow Logs ввімкнено (для аудиту мережі).
- [ ] Public Access Block на всіх S3 buckets.
- [ ] Всі RDS з Multi-AZ + encryption + deletion protection.
- [ ] CloudTrail в окремий logging-акаунт.
- [ ] GuardDuty (threat detection) ввімкнено.
- [ ] AWS Config для compliance.
- [ ] Cost alerts (Budgets).
