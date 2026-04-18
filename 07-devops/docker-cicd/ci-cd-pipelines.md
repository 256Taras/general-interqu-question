# CI/CD Pipelines

## Що таке CI/CD і навіщо воно потрібно?

**Continuous Integration (CI)** -- практика, коли розробники часто (декілька разів на день) інтегрують свої зміни в спільну гілку. Кожна інтеграція автоматично перевіряється білдом і тестами.

**Continuous Delivery (CD)** -- автоматичне підготування кожної зміни до релізу. Код завжди у стані, готовому до деплою, але рішення деплоїти приймається людиною.

**Continuous Deployment (CD)** -- повністю автоматизований деплой у production після проходження всіх перевірок. Людина не втручається.

```
Developer pushes commit
        │
        ▼
┌─────────────────────────┐
│   1. Checkout код        │
├─────────────────────────┤
│   2. Lint + Format      │    ← CI
├─────────────────────────┤
│   3. Unit tests          │
├─────────────────────────┤
│   4. Build              │
├─────────────────────────┤
│   5. Integration tests   │
├─────────────────────────┤
│   6. Security scan       │
├─────────────────────────┤
│   7. Build Docker image  │
├─────────────────────────┤
│   8. Push to registry    │
├─────────────────────────┤
│   9. Deploy to staging  │    ← Continuous Delivery
├─────────────────────────┤
│   10. E2E tests          │
├─────────────────────────┤
│   11. Deploy to prod     │    ← Continuous Deployment
│       (manual або auto)  │
└─────────────────────────┘
```

**Навіщо:**
- Швидкий feedback -- баги ловляться за хвилини, не за тижні.
- Менший batch size змін -- легше відловити, що саме зламало.
- Впевненість у якості -- автоматизовані перевірки перед мерджем.
- Часті релізи -- менший ризик на кожен реліз.

---

## З яких етапів зазвичай складається pipeline?

**1. Build stage:**
- Checkout коду.
- Встановлення залежностей (з кешем).
- Компіляція (TypeScript, Go, Java тощо).
- Lint / format check (ESLint, Prettier, gofmt).

**2. Test stage:**
- Unit tests.
- Integration tests (з реальними залежностями -- БД, Redis через Docker).
- Coverage report.
- Contract tests (Pact, OpenAPI).

**3. Security stage:**
- Dependency scan (Snyk, npm audit, Dependabot, Trivy).
- SAST (Semgrep, SonarQube) -- статичний аналіз коду.
- Secret scan (GitLeaks, TruffleHog).
- Container image scan (Trivy, Docker Scout).

**4. Package stage:**
- Docker build.
- Push у registry з тегом (semver або git SHA).
- Генерація SBOM (Software Bill of Materials).

**5. Deploy stage:**
- Staging deploy (автоматичний).
- Smoke/E2E tests на staging.
- Production deploy (manual approval або automated).

**6. Post-deploy:**
- Health checks.
- Моніторинг метрик (error rate, latency).
- Автоматичний rollback при аномаліях.

---

## Які є стратегії деплою?

### 1. Recreate (downtime)
```
Stop old version → Deploy new version → Start
     │              │                    │
     ▼              ▼                    ▼
  users down    users down            users up
```
Просто, але є downtime. Підходить для внутрішніх інструментів.

### 2. Rolling update
```
Instance 1: v1 → v2
Instance 2: v1 → v2
Instance 3: v1 → v2
```
Замінюємо екземпляри поступово. Zero downtime за умови що v1 і v2 сумісні (і API, і схема БД). Дефолт у Kubernetes.

### 3. Blue-Green
```
Blue (v1) -- serves traffic
Green (v2) -- deployed, tested

Switch traffic: Blue → Green (instant)

Keep Blue as rollback for some time
```
Швидкий rollback (перемкнути назад). Потребує 2x ресурсів.

### 4. Canary
```
95% traffic → v1
 5% traffic → v2 (canary)

Monitor metrics. If OK → gradually shift to v2.
If errors → rollback.
```
Мінімізує blast radius -- зміни бачить тільки частина користувачів. Потребує складнішого routing (service mesh, Argo Rollouts, Flagger).

### 5. Feature flags
```
if (featureFlag('new-checkout').isEnabled(user)) {
  newCheckout();
} else {
  oldCheckout();
}
```
Деплой і реліз розділені. Код з фічею вже в production, але ввімкнений тільки для певних користувачів. Це не стратегія деплою, а доповнення -- працює в парі з іншими.

**Порівняння:**

| Стратегія        | Downtime | Rollback     | Складність | Ресурси |
|------------------|----------|--------------|------------|---------|
| Recreate         | Так      | Повторний deploy | Низька | 1x      |
| Rolling update   | Ні       | Повторний rolling | Середня | 1x   |
| Blue-Green       | Ні       | Миттєвий     | Середня    | 2x      |
| Canary           | Ні       | Швидкий      | Висока     | 1-1.2x  |

---

## Як керувати версіями і тегами?

**Семантичне версіонування** (SemVer): `MAJOR.MINOR.PATCH`
- MAJOR -- breaking changes (API не сумісне).
- MINOR -- нові фічі, зворотньо сумісне.
- PATCH -- bug fixes.

**Теги Docker-образів:**
```
myapp:1.2.3              # конкретна версія (production)
myapp:1.2                # останній патч у minor (обережно з автоматикою)
myapp:1                  # останній minor у major
myapp:latest             # ❌ не використовуйте в production
myapp:git-a1b2c3d        # git SHA -- 100% відтворювано
myapp:1.2.3-a1b2c3d      # комбінація
```

**Рекомендація:**
- Для production -- прив'язуйтесь до `1.2.3` або SHA.
- `latest` -- тільки для dev/прототипування.

---

## Що таке artifact і навіщо він?

Artifact -- це результат білду, який передається між стадіями або зберігається для деплою. Приклади:
- Docker image в registry.
- Скомпільовані JARs / DLLs.
- Zip з фронтенд-асетами.
- SBOM, test reports, coverage.

**Принцип:** артефакт білдиться **один раз**, а далі переміщується між середовищами (dev → staging → prod). Не можна робити `npm run build` окремо для кожного середовища -- це порушує відтворюваність.

```
                    ┌──────────┐
                    │  Build   │  → artifact (image v1.2.3)
                    └────┬─────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
         ┌────┐       ┌─────┐     ┌──────┐
         │Dev │       │Stage│     │ Prod │
         └────┘       └─────┘     └──────┘
     (той самий образ деплоїться скрізь)
```

Середовищезалежну конфігурацію виносимо у змінні середовища (`DATABASE_URL`, `API_KEY`), не в код.

---

## Як тестувати на різних рівнях у pipeline?

**Піраміда тестів:**
```
         ▲
        ╱ ╲
       ╱E2E╲         ← мало (10%), повільні, крихкі
      ╱─────╲
     ╱ Integr ╲      ← помірно (20-30%)
    ╱─────────╲
   ╱   Unit     ╲    ← багато (70-80%), швидкі
  ╱_____________╲
```

**Де що запускати в CI:**
- **Unit + Lint** -- на кожен push, має бути < 5 хвилин.
- **Integration** -- на кожен PR, з Docker-залежностями.
- **E2E** -- на merge в main або перед deploy, з реальним браузером / API.
- **Load / performance** -- періодично (nightly), не на кожен PR.

**Приклад integration test з Docker у CI:**
```yaml
# GitHub Actions приклад
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_PASSWORD: test
    options: --health-cmd="pg_isready" --health-interval=5s
  redis:
    image: redis:7
```

---

## Як обробляти секрети у pipeline?

**Ніколи:**
- ❌ Не коммітьте секрети в код.
- ❌ Не пишіть секрети в логи (`echo $API_KEY`).
- ❌ Не передавайте як `ARG` у Dockerfile (зберігається в історії).

**Правильно:**
- ✅ CI-провайдер зберігає секрети як encrypted variables (GitHub Secrets, GitLab CI variables).
- ✅ Використовуйте secret manager (AWS Secrets Manager, HashiCorp Vault, Doppler).
- ✅ OIDC для короткоживих токенів замість статичних ключів:
  ```yaml
  # GitHub Actions → AWS без статичних ключів
  permissions:
    id-token: write
  steps:
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123:role/github-actions
        aws-region: us-east-1
  ```

**Secret scanning:**
Підключіть GitLeaks / TruffleHog як pre-commit hook і в CI, щоб запобігти коміту секретів.

---

## Trunk-based development vs Git Flow

**Git Flow** (classic):
- `main` -- production.
- `develop` -- integration.
- `feature/*`, `release/*`, `hotfix/*` -- гілки.
- Довгоживучі feature-гілки, рідкі релізи.

**Trunk-based** (сучасний підхід для CI/CD):
- Одна основна гілка (`main` / `trunk`).
- Короткоживучі feature-гілки (< 1-2 днів).
- Мерджимо часто, деплоїмо часто.
- Незакінчені фічі ховаються за feature flags.

**Чому trunk-based краще для CI/CD:**
- Менший merge conflict хаос.
- Менші PR → простіший review.
- Кожен merge → potential release.
- Feedback loop коротший.

Довгоживучі гілки -- ознака того, що фічі треба розбивати на менші інкременти.

---

## Як робити rollback?

Сценарії:
1. **Docker-деплой на один сервер:**
   ```bash
   docker pull myapp:1.2.2           # попередня версія
   docker stop myapp && docker rm myapp
   docker run -d --name myapp myapp:1.2.2
   ```

2. **Kubernetes:**
   ```bash
   kubectl rollout undo deployment/api
   # або до конкретної ревізії
   kubectl rollout undo deployment/api --to-revision=5
   kubectl rollout history deployment/api
   ```

3. **Blue-Green:** перемкнути router назад на Blue.

4. **Canary:** зменшити trafic на canary до 0%.

**Важливо:** міграції БД ускладнюють rollback. Правила:
- Робіть **forward-compatible** схеми: спочатку додаємо колонку (opt), деплоїмо код, який її пише і читає, потім робимо NOT NULL.
- Ніколи не видаляйте колонки одразу -- deprecate на релізі N, видаляйте на N+2.
- Міграції мають бути backward-compatible між N-1 і N версіями коду (для rolling updates).

---

## Метрики здоров'я pipeline

**DORA metrics** (від Google DevOps Research):
1. **Deployment Frequency** -- як часто деплоїте в prod. Elite: on-demand (кілька разів/день).
2. **Lead Time for Changes** -- від коміту до production. Elite: < 1 год.
3. **Change Failure Rate** -- % деплоїв, що спричинили інцидент. Elite: 0-15%.
4. **Mean Time to Recovery (MTTR)** -- як швидко відновлюєтеся після інциденту. Elite: < 1 год.

**Практичні сигнали здорового pipeline:**
- Pipeline runs < 15 хвилин загалом.
- Flaky tests < 1% (перезапуски).
- `main` завжди green.
- Rollback можливий за < 5 хвилин.

---

## Типові пастки

**1. Повільний pipeline:**
Розробник чекає 30+ хвилин на CI -- починає заходити в задачу інакше. Оптимізації:
- Паралельні jobs (тести по шардах).
- Docker layer cache.
- Package manager cache (npm, Go modules).
- Test selection -- запускати тільки affected tests (Nx, Turborepo, Bazel).

**2. Flaky tests:**
Тест фейлить випадково. Люди починають робити "rerun" -- втрачають довіру до CI. Треба або чинити, або видаляти.

**3. "Works in CI, fails in production":**
Причини: різні середовища, різні змінні, різні версії залежностей. Рішення -- той самий Docker-образ іде з dev у prod.

**4. Секрети в логах:**
Особливо у `set -x` bash-скриптах або коли `echo $VAR`. Маскуйте у CI налаштуваннях, ніколи не виводьте в stdout.

**5. Ручні кроки між автоматизованими:**
Якщо після автоматизованих тестів є ручний крок "скопіюй файл на сервер" -- це не CD, а плутанина. Автоматизуйте все або нічого.
