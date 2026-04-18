# Terraform: основи

## Що таке Infrastructure as Code (IaC) і навіщо Terraform?

**IaC** -- підхід, коли інфраструктура описується в коді (declarative або imperative), а не конфігурується вручну через UI. Код коммітиться в git, проходить review, тестується, деплоїться через CI/CD.

**Переваги IaC:**
- **Відтворюваність** -- розгорнути dev/staging/prod з одних вихідних даних.
- **Version control** -- `git blame`, історія змін, rollback.
- **Code review** -- зміна в інфраструктурі проходить PR.
- **Тестування** -- `plan` перед `apply`, validate, policy checks.
- **Document-as-code** -- код сам є документацією.

**Terraform** -- декларативний IaC інструмент від HashiCorp (тепер OpenTofu -- open-source форк після зміни ліцензії). Ви описуєте **бажаний стан**, а Terraform рахує різницю між поточним і бажаним станом і застосовує її.

**Чому Terraform, а не CloudFormation / ARM / Pulumi?**
- Мультихмарний (AWS, GCP, Azure, DigitalOcean, Kubernetes, Datadog...) -- єдина мова.
- Велика спільнота модулів.
- HCL -- простий для читання/писання.

**Pulumi** -- альтернатива, пише мовою програмування (TypeScript, Go, Python). Краще для складної логіки, але складніший для базових кейсів.

---

## Базові концепції: providers, resources, data, variables, outputs

**Provider** -- плагін, що спілкується з API провайдера:
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**Resource** -- об'єкт, яким керує Terraform (EC2, S3, VPC):
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcd1234"
  instance_type = "t3.micro"
  tags = {
    Name = "web-server"
  }
}
```

**Data source** -- читає існуючі ресурси (не керує ними):
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
  # ...
}
```

**Variable** -- вхідний параметр:
```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

resource "aws_instance" "web" {
  tags = {
    Environment = var.environment
  }
}
```

**Output** -- значення, яке експортується з модуля:
```hcl
output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true    # не показувати у plan output
}
```

---

## Як працює Terraform state?

**State** (`terraform.tfstate`) -- це JSON-файл, у якому Terraform тримає мапінг між HCL-описом і реальними ресурсами в хмарі. Без state Terraform не знає, які ресурси вже існують.

```
HCL (код)           State                       AWS
──────────          ─────                       ───
aws_instance.web → i-abc123 → живий EC2
aws_s3_bucket.x  → my-bucket → живий bucket
```

**Що робить Terraform при `terraform apply`:**
1. Читає HCL.
2. Читає state (поточна відома інфраструктура).
3. Робить **refresh** -- звіряє state з реальністю (API calls).
4. Обчислює diff -- що створити/змінити/видалити.
5. Показує план (`terraform plan`).
6. Застосовує (create/update/destroy).
7. Оновлює state.

**Важливо:**
- **State -- джерело істини для Terraform.** Якщо ресурс створено поза Terraform -- він не в state.
- **State може містити секрети** -- пароль БД, ключі API. Треба захищати.
- **Не редагуйте state вручну** (є команди `terraform state` для цього).

---

## Remote state і state locking

**Local state** (за замовчуванням) -- файл на диску. Проблеми:
- Команда не бачить змін один одного.
- Якщо два інженери запускають apply одночасно -- race condition.
- Ризик втратити файл.

**Remote state** -- state зберігається в backend (S3, Azure Blob, GCS, Terraform Cloud). З підтримкою locking.

**Приклад S3 backend:**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/network/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"      # для state locking
    encrypt        = true
  }
}
```

DynamoDB table для locking:
```hcl
resource "aws_dynamodb_table" "tf_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Як працює locking:**
1. `terraform apply` бере lock у DynamoDB.
2. Якщо хтось інший запустить apply -- отримає помилку "state locked".
3. Після завершення lock знімається.

**Організація state:**
- Одна state-file на середовище і компонент: `prod/network/tfstate`, `prod/app/tfstate`, `staging/network/tfstate`.
- Не одна мега-state на все (повільно, ризиковано).
- Можна ділити через `workspaces` (один backend, різні префікси) або окремі backend configs (надійніше).

---

## Життєвий цикл ресурсу: create, update, replace, destroy

Terraform вирішує, що робити, порівнюючи HCL зі state:

- **Create (+)** -- ресурс у HCL, немає в state.
- **Destroy (-)** -- ресурс у state, немає в HCL (або змінили `count` / `for_each`).
- **Update (~)** -- ресурс є в обох, змінилися атрибути, які можна оновити без re-create.
- **Replace (-/+)** -- атрибут, який не можна змінити "на льоту" (наприклад, AMI у EC2, або ім'я БД). Terraform видаляє і створює новий.

```bash
terraform plan
# Plan: 2 to add, 1 to change, 1 to destroy.
#   + aws_security_group.new
#   + aws_instance.web
#   ~ aws_db_instance.main
#       engine_version: "15.4" → "15.5"
#   - aws_s3_bucket.old
```

**Lifecycle rules:**
```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true      # створити заміну, потім видалити старий
    prevent_destroy       = true      # забороняє destroy (захист від випадковостей)
    ignore_changes        = [tags["LastModified"]]   # не реагувати на зміни цього поля
  }
}
```

`prevent_destroy` корисний для критичних ресурсів (prod БД, бакет з даними).

---

## Основні команди Terraform

```bash
# ---- Ініціалізація ----
terraform init                      # завантажити providers, налаштувати backend
terraform init -upgrade             # оновити provider версії

# ---- Планування і застосування ----
terraform plan                      # показати diff
terraform plan -out=tfplan          # зберегти план
terraform apply tfplan              # застосувати збережений план
terraform apply -auto-approve       # без запиту підтвердження (для CI)

# ---- Цільові операції ----
terraform plan -target=aws_instance.web    # тільки цей ресурс
terraform apply -replace=aws_instance.web  # форсувати replace

# ---- Знищення ----
terraform destroy                   # видалити всю керовану інфраструктуру
terraform destroy -target=aws_instance.web

# ---- State ----
terraform state list                # всі ресурси у state
terraform state show aws_instance.web
terraform state mv aws_instance.web aws_instance.api   # перейменувати
terraform state rm aws_instance.old # видалити зі state (не з хмари)
terraform import aws_instance.web i-abc123             # додати існуючий ресурс у state

# ---- Utility ----
terraform fmt                       # форматування
terraform validate                  # перевірка синтаксису
terraform show                      # показати state або plan
terraform output                    # показати outputs
terraform output -json              # для автоматизації
```

---

## Модулі: як структурувати код

**Модуль** -- переюзний блок Terraform-коду. Є:
- **Root module** -- директорія, в якій запускається `terraform`.
- **Child modules** -- викликаються через `module "..." { source = ... }`.

```hcl
# main.tf
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"

  name = "main"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}

module "ec2" {
  source = "./modules/ec2"

  vpc_id    = module.vpc.vpc_id
  subnet_id = module.vpc.private_subnets[0]
}
```

**Source може бути:**
- Локальна директорія: `./modules/ec2`.
- Terraform Registry: `terraform-aws-modules/vpc/aws`.
- Git repo: `git::https://github.com/org/repo.git//modules/vpc?ref=v1.2.3`.
- HTTP ZIP: `https://.../archive.zip`.
- S3: `s3::...`.

**Структура модуля:**
```
modules/ec2/
├── main.tf          # ресурси
├── variables.tf     # вхідні параметри
├── outputs.tf       # вихідні значення
├── versions.tf      # required_providers
└── README.md
```

---

## for_each vs count

**count** -- список з індексами. Простий, але небезпечний при змінах.

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0abc"
  instance_type = "t3.micro"
}
# Створить aws_instance.web[0], web[1], web[2]
```

**Проблема count:** якщо видалити `web[1]`, всі подальші зсуваються в state -- Terraform захоче replace всіх.

**for_each** -- map або set. Стабільний.

```hcl
resource "aws_instance" "web" {
  for_each = {
    api    = "t3.medium"
    worker = "t3.small"
  }
  ami           = "ami-0abc"
  instance_type = each.value
  tags          = { Name = each.key }
}
# Створить aws_instance.web["api"] і aws_instance.web["worker"]
```

**Рекомендація:** використовуйте `for_each` майже завжди. `count` -- тільки для умовного створення (`count = var.enable ? 1 : 0`).

---

## Функції та вирази

HCL має багато вбудованих функцій:

```hcl
# Рядки
upper("hello")                    # "HELLO"
format("Server-%03d", 5)          # "Server-005"
join(",", ["a", "b", "c"])        # "a,b,c"

# Колекції
length([1, 2, 3])                 # 3
keys({ a = 1, b = 2 })            # ["a", "b"]
merge({ a = 1 }, { b = 2 })       # { a = 1, b = 2 }
flatten([[1, 2], [3]])            # [1, 2, 3]

# Файли
file("./user-data.sh")            # зчитати файл
templatefile("./template.tpl", { env = "prod" })
jsondecode(file("config.json"))

# Мережа
cidrsubnet("10.0.0.0/16", 8, 1)   # "10.0.1.0/24"
cidrhost("10.0.1.0/24", 5)        # "10.0.1.5"

# Умовні
condition ? true_val : false_val

# Локальні значення
locals {
  tags = merge(var.common_tags, {
    Environment = var.environment
    ManagedBy   = "Terraform"
  })
}
```

**Вирази:**
```hcl
# for expression
[for s in var.subnets : s.id]     # список
{ for s in var.subnets : s.name => s.id }   # map

# splat
aws_instance.web[*].id            # список IDs усіх instances
```

---

## Типовий workflow з Terraform

**1. Локально:**
```bash
# Клонувати repo
git checkout -b feature/add-bucket

# Внести зміни
vim main.tf

# Формат і валідація
terraform fmt
terraform validate

# Переглянути план
terraform plan
```

**2. Pull Request:**
- CI запускає `terraform fmt -check`, `validate`, `plan`.
- Plan-output постить бот у PR.
- Review, approve, merge.

**3. Apply (CI):**
- На merge до main -- CI запускає `terraform apply`.
- Або через Terraform Cloud / Atlantis -- коментар у PR тригерить apply.

**4. Drift detection:**
- Регулярний `terraform plan` в CI виявляє ручні зміни.

---

## Типові пастки

**1. `terraform apply` без `plan`:**
- Завжди спочатку `plan`, дивіться, що зміниться.
- У CI -- `plan -out=tfplan` + `apply tfplan`, щоб гарантувати, що застосовується саме те, що побачили.

**2. State у git:**
- ❌ НЕ коммітьте `terraform.tfstate` -- там секрети, і конфлікти merge жахливі.
- Додайте у `.gitignore`:
  ```
  *.tfstate
  *.tfstate.*
  .terraform/
  .terraform.lock.hcl    # цей -- КОММІТЬТЕ
  ```

**3. Секрети у HCL:**
- Не пишіть `password = "supersecret"` у коді.
- Використовуйте змінні без default і передавайте через env: `TF_VAR_db_password`.
- Або SSM Parameter Store / Secrets Manager + data source.

**4. Одна мега-state на все:**
- Розбивайте на модулі / окремі state-files по компонентах.
- Один великий apply -- ризиковано і повільно.

**5. Ігнорування `.terraform.lock.hcl`:**
- Файл lock фіксує версії providers. Має коммітитись у git для reproducibility.

**6. Destroy на prod state:**
- `terraform destroy` видалить все. Для prod -- `prevent_destroy = true` на критичні ресурси.
- Краще мати окремі ролі для apply vs destroy.

**7. `taint` як пошкоджений шлях:**
- Застаріла команда. Замість `terraform taint` -- `terraform apply -replace=...`.

**8. Terraform не для всього:**
- Для деплою Docker-образів / кодів -- використовуйте CI/CD, не Terraform.
- Terraform для інфраструктури (мережа, БД, bucket). Код та ActiveRecord migrations -- окремо.
