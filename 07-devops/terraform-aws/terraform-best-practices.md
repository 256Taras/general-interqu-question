# Terraform: best practices

## Як структурувати Terraform-репозиторій?

**Монорепо approach (рекомендовано для невеликих команд):**
```
infra/
├── modules/                      # переюзні модулі
│   ├── vpc/
│   ├── eks/
│   └── rds/
├── envs/                         # середовища
│   ├── dev/
│   │   ├── main.tf               # викликає модулі
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
└── global/                       # cluster-wide: IAM, DNS
    └── main.tf
```

**Плюси:**
- Версії модулів синхронізовані між середовищами.
- Один git repo -- простіше orchestrate changes.

**Multi-repo approach:**
- Окремий repo для кожного модуля (`tf-module-vpc`, `tf-module-eks`).
- Окремий repo для середовищ.
- Модулі версіонуються через Git tags.

**Плюси:**
- Модулі можна переюзати в інших командах.
- Чіткіший ownership.

**Порада:** починайте з монорепо, виносьте модулі коли стає тісно.

---

## Як переюзати конфігурацію між середовищами?

**Підхід 1: окремі root-модулі для кожного середовища** (рекомендовано).

```
envs/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

```hcl
# envs/prod/main.tf
module "vpc" {
  source = "../../modules/vpc"

  cidr_block = "10.0.0.0/16"
  environment = "prod"
}

module "eks" {
  source = "../../modules/eks"

  vpc_id         = module.vpc.vpc_id
  cluster_name   = "prod"
  node_instance_type = "m5.large"
  min_nodes      = 3
  max_nodes      = 10
}
```

```hcl
# envs/prod/backend.tf
terraform {
  backend "s3" {
    bucket = "my-tf-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

Плюси:
- Прозоро видно, що є в кожному середовищі.
- Середовища можуть еволюціонувати окремо.
- Один apply чіпає тільки одне середовище.

**Підхід 2: Terraform workspaces** (не рекомендується для середовищ).

```bash
terraform workspace new prod
terraform workspace select prod
terraform apply
```

**Мінуси:** один backend на всі середовища, легко помилитися (не той workspace вибрали). Workspaces корисні для тимчасових/feature-branch середовищ.

---

## Модулі: як писати хороші модулі?

**Ознаки хорошого модуля:**
- Один фокус -- робить одну річ (VPC, EKS кластер, RDS інстанс).
- Мінімальний required input -- решта з sensible defaults.
- Output-и -- все, що може знадобитись споживачам (IDs, ARNs, endpoints).
- Документація в `README.md` + приклади в `examples/`.
- Тести (terratest, kitchen-terraform).
- Versioned via git tags.

**Структура модуля:**
```
modules/rds-postgres/
├── main.tf          # основні ресурси
├── variables.tf     # вхідні параметри з описами
├── outputs.tf
├── versions.tf      # required_providers
├── README.md        # опис, приклади
└── examples/
    └── complete/
        └── main.tf
```

**Приклад хорошого variable:**
```hcl
variable "instance_class" {
  description = "RDS instance class (e.g., db.t3.medium). See https://..."
  type        = string
  default     = "db.t3.medium"
}

variable "allocated_storage" {
  description = "Allocated storage in GB"
  type        = number
  default     = 20
  validation {
    condition     = var.allocated_storage >= 20
    error_message = "Minimum 20 GB required."
  }
}
```

**Приклад output:**
```hcl
output "endpoint" {
  description = "RDS endpoint for connections"
  value       = aws_db_instance.main.endpoint
}

output "db_password" {
  description = "Database password (sensitive)"
  value       = random_password.db.result
  sensitive   = true
}
```

---

## Версіонування модулів

**Pinning версій (обов'язково):**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"              # ✅ точна версія

  # або
  version = "~> 5.1"             # ⚠️ будь-який 5.1.x (обережно з minor updates)
}
```

**Для git-sourced модулів:**
```hcl
module "vpc" {
  source = "git::https://github.com/org/tf-modules.git//vpc?ref=v1.2.3"
  #                                                        ^^^^^^
  #                                                        git tag
}
```

**НЕ використовуйте `ref=main`** -- будь-яка зміна в main ламає ваш деплой.

**Providers теж pin:**
```hcl
terraform {
  required_version = ">= 1.6, < 2.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }
}
```

`.terraform.lock.hcl` фіксує точні версії provider binaries -- обов'язково коммітьте в git.

---

## Теги: як і навіщо?

Теги -- для:
- **Billing** -- бачити, скільки коштує кожен застосунок / команда / середовище.
- **Ownership** -- хто відповідальний за ресурс.
- **Auditing** -- хто створив, коли, через який PR.
- **Автоматизація** -- cleanup неактуальних ресурсів.

**Default tags на рівні provider:**
```hcl
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Project     = "myapp"
      Owner       = "platform-team"
      CostCenter  = "engineering"
    }
  }
}
```

Тепер всі ресурси автоматично отримують ці теги. Можна додавати resource-specific теги:
```hcl
resource "aws_instance" "web" {
  # ...
  tags = {
    Name      = "web-server"
    Component = "frontend"
  }
}
```

**Governance через AWS Config / SCP:** можна забороняти створення ресурсів без потрібних тегів.

---

## Як уникнути дрифтів (drift)?

**Drift** -- розходження між Terraform state і реальною інфраструктурою. Виникає, коли хтось змінює щось вручну через консоль.

**Профілактика:**

**1. Заборонити ручні зміни через IAM:**
- Production ролі тільки для read. Write -- тільки через CI.
- Уникайте `AdministratorAccess` для людей.

**2. Регулярний drift detection:**
```bash
# Nightly cron або CI job
terraform plan -detailed-exitcode
# Exit code 0: no changes
# Exit code 1: error
# Exit code 2: changes detected (drift!)
```

При exit code 2 -- алерт у Slack.

**3. `ignore_changes` для очікуваних змін:**
```hcl
resource "aws_autoscaling_group" "main" {
  # ...
  lifecycle {
    ignore_changes = [
      desired_capacity,    # HPA або scheduled scaling змінює
      tag,                 # Auto-tagger додає теги
    ]
  }
}
```

**4. Import-ити ручні зміни:**
Якщо drift корисний (хтось додав щось потрібне вручну) -- імпортуйте в state:
```bash
terraform import aws_s3_bucket.logs my-logs-bucket
```
Потім додайте у HCL, `plan` має показати "no changes".

---

## CI/CD для Terraform

**Типовий pipeline:**

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths: ['infra/**']
  push:
    branches: [main]
    paths: ['infra/**']

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra/envs/prod
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/tf-deploy
          aws-region: us-east-1

      - run: terraform fmt -check -recursive
      - run: terraform init
      - run: terraform validate

      - name: Plan
        id: plan
        run: terraform plan -out=tfplan -no-color
        continue-on-error: true

      - name: Post Plan to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `\`\`\`${{ steps.plan.outputs.stdout }}\`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      - name: Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

**Альтернативи:**
- **Terraform Cloud** (HashiCorp) -- SaaS, керує state, runs, policies.
- **Atlantis** -- self-hosted bot, `terraform plan/apply` через PR коментарі.
- **Env0, Spacelift, Scalr** -- комерційні альтернативи.

**Policy as Code:**
- **Sentinel** (Terraform Cloud) або **OPA / Conftest** -- автоматична перевірка policies (теги обов'язкові, instance types з allowlist, шифрування увімкнено).

---

## Секрети у Terraform

**Правила:**
1. Секрети **ніколи не в коді**.
2. Секрети **можуть бути в state** -- захистіть backend (S3 encryption, restricted IAM).
3. Для вихідних значень -- `sensitive = true`.

**Джерела секретів:**

**1. Змінні середовища:**
```bash
export TF_VAR_db_password=supersecret
terraform apply
```

**2. `.tfvars` файли (поза git):**
```hcl
# secrets.auto.tfvars -- у .gitignore
db_password = "supersecret"
```

**3. SSM Parameter Store / Secrets Manager:**
```hcl
data "aws_ssm_parameter" "db_password" {
  name = "/prod/db/password"
  with_decryption = true
}

resource "aws_db_instance" "main" {
  password = data.aws_ssm_parameter.db_password.value
}
```

**4. HashiCorp Vault:**
```hcl
data "vault_generic_secret" "db" {
  path = "secret/prod/db"
}

resource "aws_db_instance" "main" {
  password = data.vault_generic_secret.db.data["password"]
}
```

**5. Random password + store в SSM:**
```hcl
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_ssm_parameter" "db_password" {
  name  = "/prod/db/password"
  type  = "SecureString"
  value = random_password.db.result
}

resource "aws_db_instance" "main" {
  password = random_password.db.result
}
```

---

## Тестування Terraform

**1. Static checks:**
- `terraform fmt -check` -- форматування.
- `terraform validate` -- синтаксис і посилання.
- **tflint** -- linter (знаходить deprecated атрибути, неіснуючі instance types).
- **Checkov / tfsec / trivy** -- security scan (незашифрований bucket, public security group).

**2. Unit tests (Terraform 1.6+ native):**
```hcl
# tests/main.tftest.hcl
run "vpc_creates" {
  command = plan

  assert {
    condition     = module.vpc.vpc_id != null
    error_message = "VPC ID should not be null"
  }
}
```

```bash
terraform test
```

**3. Integration tests (terratest):**
Go-бібліотека, яка реально розгортає інфру, тестує через API, потім знищує:
```go
func TestVPC(t *testing.T) {
  options := &terraform.Options{TerraformDir: "../examples/complete"}
  defer terraform.Destroy(t, options)
  terraform.InitAndApply(t, options)

  vpcId := terraform.Output(t, options, "vpc_id")
  assert.NotEmpty(t, vpcId)
}
```

**4. Policy tests (OPA / Sentinel):**
```rego
# policy.rego
deny[msg] {
  resource := input.resource.aws_s3_bucket[_]
  not resource.server_side_encryption_configuration
  msg := "S3 buckets must have encryption enabled"
}
```

---

## Типові помилки

**1. Terraform керує тим, що керувати не треба:**
Не керуйте Terraform-ом IAM користувачами, яких створюють зовнішні системи (наприклад, деплоями Okta). Визначте межі відповідальності.

**2. Занадто великі root-модулі:**
Один `apply` на 500 ресурсів -- повільно (5-10 хв refresh), ризиковано (щось одне може зламати все).
Ділить на менші state-files.

**3. Не pin-ите version провайдерів:**
Новий minor release провайдера може змінити поведінку. Завжди `~>` з конкретною minor.

**4. Копіпаста замість модулів:**
Якщо ви створюєте 5 однакових VPC з невеликими відмінностями -- це модуль.

**5. Зміна resource name:**
```hcl
# Було
resource "aws_instance" "web" { ... }
# Стало
resource "aws_instance" "api" { ... }
```
Terraform бачить як destroy + create, а не переіменування! Використовуйте `terraform state mv`:
```bash
terraform state mv aws_instance.web aws_instance.api
```

**6. Destroy-then-create замість rename:**
Якщо міняєте атрибут, який форсує replace (`ForceNew`), -- prod буде simply destroy+create. Для stateful ресурсів (RDS) -- катастрофа. Використовуйте `create_before_destroy`:
```hcl
lifecycle {
  create_before_destroy = true
}
```

**7. Забутий `-lock-timeout`:**
Якщо хтось інший тримає lock -- `apply` одразу фейлиться. Додайте:
```bash
terraform apply -lock-timeout=10m
```

**8. Plan, що показує зміни, яких не хотіли:**
Це означає drift. Не ігноруйте -- розберіться.
