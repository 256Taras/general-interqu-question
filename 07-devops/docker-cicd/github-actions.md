# GitHub Actions

## Що таке GitHub Actions і з яких будівельних блоків складається?

GitHub Actions -- це вбудована в GitHub платформа для CI/CD. Pipelines описуються YAML-файлами в директорії `.github/workflows/`.

**Основні поняття:**

- **Workflow** -- один YAML-файл, який описує pipeline. Запускається на тригер (push, PR, schedule).
- **Job** -- набір кроків, виконуються в одному runner. Jobs йдуть паралельно за замовчуванням.
- **Step** -- окрема дія (команда або action).
- **Action** -- переюзний компонент (наприклад, `actions/checkout`).
- **Runner** -- машина, де виконується job (GitHub-hosted або self-hosted).

```
Workflow
├── Job: test
│   ├── Step: checkout
│   ├── Step: setup-node
│   ├── Step: npm install
│   └── Step: npm test
└── Job: build (needs: test)
    ├── Step: checkout
    ├── Step: docker build
    └── Step: docker push
```

---

## Мінімальний приклад workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm test
```

---

## Як тригерити workflow?

```yaml
on:
  # При пуші в конкретні гілки / з певними файлами
  push:
    branches: [main, 'release/**']
    paths:
      - 'src/**'
      - 'package*.json'
    tags:
      - 'v*'

  # На Pull Request
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

  # За розкладом (cron у UTC)
  schedule:
    - cron: '0 2 * * *'          # щодня о 02:00 UTC

  # Вручну з UI
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]

  # Від іншого workflow
  workflow_run:
    workflows: ["CI"]
    types: [completed]

  # Від зовнішньої події (repository_dispatch API)
  repository_dispatch:
    types: [deploy-request]
```

---

## Як правильно налаштувати кеш?

**Кеш npm / залежностей:**
```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'          # автоматично кешує ~/.npm
    cache-dependency-path: 'package-lock.json'

- run: npm ci
```

**Загальний кеш через `actions/cache`:**
```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

**Docker layer cache через Buildx + GitHub Cache:**
```yaml
- uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

Це дає persistent кеш між білдами (набагато швидше, ніж кожен раз з нуля).

---

## Matrix strategy: запуск на кількох версіях/платформах

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false     # не скасовувати інші варіанти при фейлі
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
        exclude:
          - os: windows-latest
            node: 18          # не запускати цю комбінацію
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

3 OS × 3 версії Node - 1 exclude = 8 паралельних jobs.

---

## Як передавати дані між jobs?

Jobs виконуються у **різних runner-ах** -- файли не діляться. Два способи:

### 1. Artifacts (для файлів)
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      - run: ./deploy.sh
```

### 2. Outputs (для рядків)
```yaml
jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.bump.outputs.new_tag }}
    steps:
      - id: bump
        run: echo "new_tag=v1.2.3" >> "$GITHUB_OUTPUT"

  deploy:
    needs: version
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.version.outputs.tag }}"
```

---

## Робота з секретами та OIDC

**Статичні секрети** через Repository/Organization Secrets:
```yaml
steps:
  - run: ./deploy.sh
    env:
      API_KEY: ${{ secrets.DEPLOY_API_KEY }}
```

Секрети автоматично маскуються в логах.

**OIDC для хмарних провайдерів** (рекомендовано):
Замість статичних ключів, GitHub Actions генерує короткоживучий OIDC-токен, який обмінюється на хмарні credentials.

**AWS приклад:**
```yaml
permissions:
  id-token: write        # ← обов'язково для OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: us-east-1
      - run: aws s3 sync ./dist s3://my-bucket/
```

Переваги:
- Немає статичних ключів, які можна витягти.
- Токен живе ~1 годину.
- Ролі можна обмежити конкретним repo/гілкою.

---

## Reusable workflows і composite actions

**Composite action** -- переюзний набір steps (зберігається в repo або публічно):
```yaml
# .github/actions/setup/action.yml
name: 'Setup project'
description: 'Node + deps install'
runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    - run: npm ci
      shell: bash
```

Використання:
```yaml
- uses: ./.github/actions/setup
```

**Reusable workflow** -- цілий workflow, що викликається з іншого:
```yaml
# .github/workflows/deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy-key:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh ${{ inputs.environment }}
        env:
          KEY: ${{ secrets.deploy-key }}
```

```yaml
# В іншому workflow
jobs:
  staging:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
    secrets:
      deploy-key: ${{ secrets.STAGING_KEY }}
```

---

## Умови та контроль потоку

```yaml
jobs:
  deploy:
    # Запустити тільки на main push (не на PR)
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Крок тільки якщо попередній успішний і це не скасування
      - if: success() && !cancelled()
        run: ./migrate.sh

      # Завжди виконати (навіть якщо фейл)
      - if: always()
        run: ./cleanup.sh

      # Тільки якщо попередні фейлили
      - if: failure()
        run: ./notify-slack.sh
```

**Continue-on-error** -- не зупиняти job при фейлі кроку:
```yaml
- run: npm run flaky-test
  continue-on-error: true
```

---

## Environments і required reviewers

Environments -- спосіб додати ручні approval-gate або environment-specific секрети:

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.API_KEY }}   # специфічний для env
```

У Settings → Environments можна:
- Додати required reviewers (person must approve).
- Обмежити, які гілки можуть деплоїти.
- Встановити wait timer.
- Зберегти environment-specific secrets.

---

## Повний приклад: CI + Docker + deploy

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true   # скасувати старі запуски на тому ж PR

jobs:
  # ---------- Test ----------
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm test
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/postgres

  # ---------- Security scan ----------
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  # ---------- Build & push Docker ----------
  build:
    needs: [test, security]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write        # для ghcr.io push
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest,enable={{is_default_branch}}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ---------- Deploy ----------
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/deploy
          aws-region: us-east-1

      - run: |
          aws ecs update-service \
            --cluster prod \
            --service api \
            --force-new-deployment
```

---

## Типові пастки

**1. `${{ github.token }}` замість PAT:**
`GITHUB_TOKEN` має обмежений scope. Для крос-repo операцій потрібен PAT або GitHub App token.

**2. `on: push` без обмеження гілок:**
Workflow буде запускатися на кожну feature-гілку -- зайві runner хвилини.

**3. Немає `concurrency`:**
Новий push в PR не скасовує старий run -- CI черга забивається.

**4. Забутий `fetch-depth: 0`:**
`actions/checkout` за замовчуванням робить shallow clone. Якщо потрібна git-історія (для changelog, semantic-release) -- вкажіть `fetch-depth: 0`.

**5. Pinning actions за тегом, а не SHA:**
```yaml
- uses: actions/checkout@v4             # ← теги можуть переміщатися
- uses: actions/checkout@b4ffde65...    # ✅ pinning за SHA (для security-sensitive CI)
```

**6. Логування секретів через `echo ${{ secrets.X }}`:**
GitHub маскує, але не у всіх випадках (наприклад, після `base64`). Не виводьте секрети у stdout взагалі.
