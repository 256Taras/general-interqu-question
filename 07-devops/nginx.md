# Nginx як Reverse Proxy та Load Balancer

## Що таке nginx і чому він такий швидкий?

Nginx -- це високопродуктивний HTTP-сервер, reverse proxy, load balancer та TCP/UDP proxy. Його було створено Ігорем Сисоєвим у 2004 році як відповідь на проблему C10K -- обробку 10 тисяч одночасних з'єднань на одному сервері. На відміну від Apache, який використовує модель "один процес (або потік) на з'єднання", nginx побудований на event-driven асинхронній архітектурі.

Ключова ідея: nginx запускає невелику кількість worker-процесів (зазвичай по одному на ядро CPU), і кожен worker обробляє **тисячі з'єднань одночасно** в одному потоці за допомогою неблокуючого I/O (epoll на Linux, kqueue на BSD/macOS, IOCP на Windows). Коли приходить запит, worker не створює новий потік -- він просто реєструє файловий дескриптор у системному механізмі подій і продовжує обробляти інші з'єднання.

```
Apache (prefork/worker):              Nginx (event-driven):

  Request 1 ──▶ Process/Thread 1       Request 1 ──┐
  Request 2 ──▶ Process/Thread 2       Request 2 ──┤
  Request 3 ──▶ Process/Thread 3       Request 3 ──┼──▶ Worker 1 (epoll)
  ...                                  ...         ┤
  Request N ──▶ Process/Thread N       Request N ──┘

  Пам'ять: O(N)                        Пам'ять: O(1) на запит
  Контекст-світчі: багато              Контекст-світчі: мінімум
```

Архітектура nginx складається з **master-процесу** (керує конфігурацією, слухає сигнали, розподіляє сокети) та **worker-процесів** (безпосередньо обробляють запити). Workers не розділяють стан -- кожен має свій event loop. Це дозволяє робити graceful reload: master запускає нових workers, старі дообслуговують активні з'єднання і завершуються.

```nginx
# Базова оптимізація worker-процесів
worker_processes auto;              # = кількість ядер CPU
worker_rlimit_nofile 65535;         # Ліміт відкритих файлів на worker

events {
    worker_connections 10240;        # Макс. з'єднань на worker
    use epoll;                       # Linux (kqueue для macOS/BSD)
    multi_accept on;                 # Worker приймає всі нові з'єднання одразу
}
```

Максимальна кількість одночасних з'єднань = `worker_processes × worker_connections`. Але для reverse proxy це число треба ділити на 2 (одне з'єднання до клієнта + одне до backend). Саме тому nginx часто називають "C10K solved" -- на скромному сервері він легко тримає 50-100к активних з'єднань, чого майже неможливо досягти з thread-per-request моделлю.

---

## Reverse proxy vs forward proxy -- в чому різниця?

Обидва типи proxy -- це посередник між клієнтом і сервером, але вони розв'язують протилежні задачі. **Forward proxy** діє від імені клієнта: коли ви налаштовуєте корпоративний proxy в браузері, ваш запит до `google.com` спочатку йде на proxy, а вже він іде в інтернет. Сервер (Google) бачить IP proxy, а не ваш. Класичні приклади: Squid, корпоративний фільтр контенту, Tor.

**Reverse proxy** діє від імені сервера: клієнт думає, що спілкується безпосередньо з вашим сервером, але насправді nginx приймає запит і пересилає його на один із backend-серверів (Node.js, Python, Java). Клієнт не знає про існування backend. Саме тому nginx -- це reverse proxy.

```
Forward Proxy (клієнт-центричний):

  [Client A] ──┐
  [Client B] ──┼──▶ [Forward Proxy] ──▶ [Internet]
  [Client C] ──┘     (Squid, Tor)         (будь-який сайт)

  Сервер бачить IP proxy, не клієнтів.


Reverse Proxy (сервер-центричний):

  [Internet] ──▶ [Reverse Proxy] ──┬──▶ [Backend 1: Node.js]
                 (Nginx, HAProxy)  ├──▶ [Backend 2: Node.js]
                                   └──▶ [Backend 3: Python]

  Клієнт бачить тільки nginx (один IP/домен).
```

Use cases для reverse proxy в реальних проєктах:

1. **Load balancing** -- розподіл трафіку між кількома інстансами Node.js (бо Node -- single-threaded, на 16-ядерному сервері треба 16 процесів + nginx попереду).
2. **SSL termination** -- nginx обробляє HTTPS, backend приймає простий HTTP (швидше, простіше сертифікати).
3. **Caching** -- кешування статичних відповідей, щоб не навантажувати backend.
4. **Compression** -- gzip/brotli стискання на рівні proxy.
5. **Rate limiting** -- захист від DDoS і зловживання API.
6. **Security** -- приховування внутрішньої архітектури, WAF, фільтрація headers.
7. **A/B testing & canary deploys** -- маршрутизація частини трафіку на нову версію.

---

## Які є алгоритми балансування навантаження в nginx?

Nginx підтримує кілька стратегій розподілу запитів між backend-серверами через директиву `upstream`. Вибір алгоритму залежить від того, чи є у вас stateful-сесії, чи всі backends однакові за потужністю, і які метрики ви хочете оптимізувати.

**1. Round-robin (за замовчуванням)** -- запити йдуть по колу: 1→A, 2→B, 3→C, 4→A... Простий і працює добре, коли всі backends однакові та запити приблизно одного "розміру".

```nginx
upstream node_backend {
    # Round-robin за замовчуванням, нічого не треба вказувати
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}

server {
    listen 80;
    location / {
        proxy_pass http://node_backend;
    }
}
```

**2. Weighted round-robin** -- для гетерогенних backend. Сервер з `weight=3` отримає втричі більше запитів, ніж із `weight=1`. Корисно, коли додаєте новіший потужніший сервер до старого парку.

```nginx
upstream node_backend {
    server 10.0.0.1:3000 weight=5;   # Нова машина, 16 CPU
    server 10.0.0.2:3000 weight=2;   # Стара машина, 8 CPU
    server 10.0.0.3:3000 weight=1;   # Зовсім стара, 4 CPU
}
```

**3. Least connections (`least_conn`)** -- новий запит іде на сервер із найменшою кількістю активних з'єднань. Оптимально для довгих запитів різної тривалості (наприклад, API з різними SLA або WebSocket). Round-robin тут несправедливо перевантажить один сервер.

```nginx
upstream api_backend {
    least_conn;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}
```

**4. IP hash (`ip_hash`)** -- гарантує, що запити від одного клієнта (за IP) завжди йдуть на один і той самий backend. Використовується для session affinity, коли сесія зберігається в пам'яті процесу (не в Redis). Мінус: якщо одні IP дають багато трафіку (NAT корпоративної мережі), буде дисбаланс.

```nginx
upstream legacy_backend {
    ip_hash;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}
# Усі запити з IP 192.168.1.50 підуть на один сервер постійно
```

**5. Hash (за ключем)** -- hash за довільним ключем, наприклад `$request_uri` для consistent caching або `$cookie_user_id` для session affinity через cookies.

```nginx
upstream cache_backend {
    hash $request_uri consistent;     # consistent hashing -- менше ремапінгів при змінах
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}
```

**6. Random with two choices (nginx 1.15.1+)** -- випадково вибирає 2 сервери, а потім із них той, у якого менше з'єднань. Краще розподіляє навантаження в кластерах із багатьма nginx-інстансами перед одним пулом backend (уникає "herd effect" від least_conn).

```nginx
upstream backend {
    random two least_conn;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}
```

На практиці для stateless Node.js API найчастіше використовують `least_conn` або звичайний round-robin. `ip_hash` -- останній засіб, бо за архітектурою краще виносити стан у Redis/БД.

---

## Як працюють health checks? Passive vs active

Health checks -- це механізм, який дозволяє nginx виключити "мертвий" backend із ротації, щоб клієнти не отримували 502/504. Nginx open source підтримує тільки **passive health checks**, а active -- тільки в nginx Plus (комерційна версія). Але на практиці passive checks достатньо для більшості задач.

**Passive health check** працює так: nginx помічає помилки, коли сам пробує зробити запит на backend. Якщо з'єднання відмовило, timeout, або повернулася помилка -- nginx рахує це як "failure". Після `max_fails` невдач за `fail_timeout` секунд сервер позначається як `unavailable` і на нього не йдуть нові запити протягом `fail_timeout`.

```nginx
upstream node_backend {
    # max_fails=3 невдалих спроб за fail_timeout=30s
    # → сервер виключається на 30 секунд
    server 10.0.0.1:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:3000 max_fails=3 fail_timeout=30s;

    # Backup-сервер -- використовується, тільки коли всі основні мертві
    server 10.0.0.99:3000 backup;
}

server {
    location / {
        proxy_pass http://node_backend;

        # Вважати помилкою не тільки 5xx, а й timeout/invalid headers
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 3;       # Не більше 3 спроб на запит
        proxy_next_upstream_timeout 10s;   # Загальний таймаут retry

        proxy_connect_timeout 5s;          # Таймаут на встановлення TCP
        proxy_send_timeout 10s;            # Таймаут на відправку запиту
        proxy_read_timeout 30s;            # Таймаут на читання відповіді
    }
}
```

**Active health check** (nginx Plus, або через `nginx_upstream_check_module` від Taobao) -- nginx сам періодично стукає на `/health` endpoint і вимикає backend, навіть якщо туди ніхто не ходить. Це корисно для раннього виявлення проблем.

```nginx
# Nginx Plus синтаксис (не працює в open source)
upstream node_backend {
    zone backend 64k;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}

server {
    location / {
        proxy_pass http://node_backend;
        health_check interval=5s fails=3 passes=2 uri=/health match=ok;
    }
}

match ok {
    status 200;
    header Content-Type = application/json;
    body ~ '"status":"ok"';
}
```

На практиці для open source nginx достатньо грамотних passive checks + `proxy_next_upstream`, який автоматично ретраїть на інший сервер, якщо поточний повернув 502/504 або timeout. Це дає майже active behaviour, бо реальний user-трафік працює як health probes.

Важливий момент: endpoint `/health` в Node.js має бути **справжнім** health check -- перевіряти з'єднання з БД, Redis, чергою. Не просто `res.json({ok: true})`, бо такий сервер може "брехати", що живий, коли БД вже лежить.

---

## Що таке upstream, proxy_pass і важливі headers?

`upstream` -- це іменована група backend-серверів, до якої можна звертатися через `proxy_pass`. Без upstream можна теж робити proxy, але тільки на один сервер.

```nginx
# Варіант 1: без upstream, на один сервер
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}

# Варіант 2: з upstream, пул серверів + load balancing
upstream api {
    least_conn;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    keepalive 32;   # Пул persistent з'єднань до backend
}

location /api/ {
    proxy_pass http://api;
}
```

**Keepalive до backend** -- критично важливо для performance. Без keepalive nginx на кожен запит відкриває новий TCP-з'єднання до Node.js, що додає ~1мс latency (3-way handshake + TLS handshake якщо HTTPS). З keepalive nginx тримає пул з'єднань і перевикористовує їх.

```nginx
upstream api {
    server 10.0.0.1:3000;
    keepalive 32;                    # 32 з'єднання в пулі на worker
    keepalive_requests 1000;         # Макс запитів на одне з'єднання
    keepalive_timeout 60s;           # Idle timeout
}

server {
    location /api/ {
        proxy_pass http://api;

        # ОБОВ'ЯЗКОВО для keepalive:
        proxy_http_version 1.1;      # HTTP/1.0 не підтримує keepalive
        proxy_set_header Connection ""; # Прибрати "Connection: close" від клієнта
    }
}
```

**Proxy headers** -- коли nginx пересилає запит на backend, оригінальна інформація про клієнта втрачається. Backend бачить IP nginx (`127.0.0.1` або internal), а не реальний IP юзера. Щоб це виправити, nginx додає стандартні headers:

```nginx
location /api/ {
    proxy_pass http://api;

    # Оригінальний Host, який запросив клієнт (наприклад, api.example.com)
    proxy_set_header Host $host;

    # Реальний IP клієнта
    proxy_set_header X-Real-IP $remote_addr;

    # Ланцюжок proxy: якщо запит пройшов через кілька proxy, всі IP тут
    # Наприклад: "1.2.3.4, 5.6.7.8" (клієнт, перший proxy)
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # Оригінальний протокол (http чи https), бо backend приймає http
    proxy_set_header X-Forwarded-Proto $scheme;

    # Оригінальний порт
    proxy_set_header X-Forwarded-Port $server_port;

    # Оригінальний хост (для Forwarded header стандарту)
    proxy_set_header X-Forwarded-Host $host;
}
```

У Node.js/Express це треба правильно обробити через `trust proxy`:

```javascript
// Express: довіряти nginx як proxy, щоб req.ip брав із X-Forwarded-For
app.set('trust proxy', 'loopback'); // або конкретний CIDR "10.0.0.0/8"

app.get('/whoami', (req, res) => {
  // req.ip тепер реальний IP клієнта, не 127.0.0.1
  // req.protocol буде "https", якщо nginx зробив SSL termination
  res.json({ ip: req.ip, protocol: req.protocol });
});
```

**Типова пастка**: якщо не довіряти `X-Forwarded-For`, можна дістати спуфлений IP від клієнта. Зловмисник просто надсилає header `X-Forwarded-For: 1.1.1.1` і в логах буде цей IP. Тому в Express треба явно казати `trust proxy` тільки довіреним IP (IP nginx).

---

## Як налаштувати SSL/TLS termination, HTTP/2, HTTP/3?

SSL termination -- це коли TLS-шифрування закінчується на nginx, а далі в приватній мережі йде звичайний HTTP. Це дає кілька переваг: централізоване керування сертифікатами (один Let's Encrypt замість N на кожному backend), нижче навантаження на Node.js (TLS handshake -- дорога операція), простіше debugging.

```nginx
server {
    listen 443 ssl http2;              # HTTPS + HTTP/2
    listen [::]:443 ssl http2;         # IPv6
    listen 443 quic reuseport;         # HTTP/3 (QUIC over UDP), nginx 1.25+
    listen [::]:443 quic reuseport;

    server_name api.example.com;

    # Сертифікат і приватний ключ (Let's Encrypt або інший CA)
    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    # Тільки сучасні протоколи
    ssl_protocols TLSv1.2 TLSv1.3;

    # Безпечні шифри (Mozilla Intermediate config)
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;     # TLS 1.3 обирає шифр клієнт

    # Session caching для швидшого повторного підключення
    ssl_session_cache shared:SSL:10m;  # 10MB = ~40k сесій
    ssl_session_timeout 1d;
    ssl_session_tickets off;           # Безпечніше без session tickets

    # OCSP stapling -- nginx сам перевіряє revocation статус сертифіката
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/api.example.com/chain.pem;
    resolver 1.1.1.1 8.8.8.8 valid=300s;

    # HSTS -- форсує HTTPS на рік (тільки після того, як впевнені в HTTPS)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Оголошує HTTP/3 для клієнтів (alt-svc)
    add_header Alt-Svc 'h3=":443"; ma=86400' always;

    location / {
        proxy_pass http://api_backend;
        proxy_set_header X-Forwarded-Proto https;
    }
}

# Редірект з HTTP на HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name api.example.com;
    return 301 https://$host$request_uri;
}
```

**HTTP/2** дає multiplexing (кілька запитів в одному TCP-з'єднанні без head-of-line blocking як у pipelining HTTP/1.1), header compression (HPACK), server push. Включається тривіально через `listen 443 ssl http2`.

**HTTP/3** (QUIC) -- новий протокол поверх UDP, а не TCP. Ключові переваги: 0-RTT handshake (підключення без затримки для повторних клієнтів), відсутність TCP head-of-line blocking (втрата одного пакета не блокує інші стріми), краща робота при зміні мережі (перемикання WiFi → 4G без перевстановлення з'єднання). Потрібен nginx 1.25+ з підтримкою QUIC.

**Типова пастка**: `proxy_set_header X-Forwarded-Proto $scheme` -- якщо backend робить редіректи (наприклад, Express redirect), він має знати оригінальний протокол, щоб не зробити `https → http → https` infinite loop.

---

## Як працює кешування в nginx (proxy_cache)?

Nginx може кешувати відповіді від backend на диску або в RAM, щоб не навантажувати backend повторними запитами. Це особливо корисно для GET-запитів із рідко змінюваними даними (списки товарів, профілі, конфіги).

Налаштування відбувається у два етапи: оголошення cache zone на рівні `http`, і використання в `location`:

```nginx
http {
    # keys_zone=api_cache:10m -- 10MB для метаданих кешу (~80k ключів)
    # max_size=1g -- ліміт розміру кешу на диску
    # inactive=60m -- видалити з кешу, якщо не зверталися 60 хв
    # levels=1:2 -- структура підкаталогів (щоб не мати 1М файлів в одному)
    proxy_cache_path /var/cache/nginx/api
                     levels=1:2
                     keys_zone=api_cache:10m
                     max_size=1g
                     inactive=60m
                     use_temp_path=off;

    server {
        location /api/products {
            proxy_cache api_cache;

            # Ключ кешу -- по чому групуємо запити
            proxy_cache_key "$scheme$request_method$host$request_uri";

            # Кешуємо 200/301/302 на 10хв, 404 на 1хв
            proxy_cache_valid 200 301 302 10m;
            proxy_cache_valid 404 1m;

            # Розблокувати повторні запити одним backend call
            # (без цього 100 паралельних запитів зроблять 100 calls)
            proxy_cache_lock on;
            proxy_cache_lock_timeout 5s;

            # Використовувати застарілий кеш, якщо backend лежить
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_background_update on;  # Оновлювати кеш у фоні

            # Кешувати тільки якщо Cache-Control дозволяє
            proxy_cache_revalidate on;         # Перевіряти If-Modified-Since

            # Не кешувати для авторизованих юзерів
            proxy_cache_bypass $http_authorization $cookie_session;
            proxy_no_cache $http_authorization $cookie_session;

            # Додати header із статусом кешу (дуже корисно для debug)
            add_header X-Cache-Status $upstream_cache_status;
            # Значення: HIT, MISS, BYPASS, EXPIRED, STALE, UPDATING, REVALIDATED

            proxy_pass http://api_backend;
        }
    }
}
```

**Cache invalidation** -- одна з двох складних задач у комп'ютерних науках. Є три підходи:

1. **TTL-based** -- просто чекаємо, поки `proxy_cache_valid` не сплине. Просто, але клієнти бачать застарілі дані.

2. **Active purge** -- видаляти кеш вручну при зміні даних. В open source nginx немає команди purge, але можна використати `ngx_cache_purge` модуль або трюк із `proxy_cache_bypass`:

```nginx
# Якщо запит має header X-Purge-Cache: 1, перечитати з backend і оновити кеш
location /api/products {
    proxy_cache_bypass $http_x_purge_cache;
    # ...
}
```

3. **Versioned keys** -- включати версію даних у ключ кешу, наприклад `/api/products?v=42`. Коли дані змінюються, інкрементуємо `v` -- старий кеш не використовується. Найпростіший і найнадійніший підхід.

Типова пастка: забути `proxy_cache_lock on` -- отримаєте thundering herd, коли при expiration тисячі паралельних запитів підуть на backend замість одного.

---

## Як зробити rate limiting у nginx?

Nginx має два механізми: `limit_req_zone` (rate limit -- обмеження запитів на секунду) і `limit_conn_zone` (connection limit -- обмеження одночасних з'єднань).

**limit_req_zone** використовує leaky bucket алгоритм. Ключ (зазвичай IP) має "відро" певного розміру; запити "капають" у відро зі швидкістю rate, і нові запити приймаються, тільки якщо відро не переповнене.

```nginx
http {
    # Зона пам'яті "api" розміром 10MB
    # rate=10r/s -- 10 запитів на секунду на один ключ ($binary_remote_addr)
    # $binary_remote_addr компактніший за $remote_addr (4 vs 15 байт для IPv4)
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    # Більш жорсткий ліміт для login endpoint
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;

    server {
        location /api/ {
            # burst=20 -- дозволити сплеск до 20 запитів
            # nodelay -- не затримувати burst (відповідати одразу, але не >30 разом)
            limit_req zone=api burst=20 nodelay;

            # Код відповіді при перевищенні ліміту (429 Too Many Requests)
            limit_req_status 429;

            proxy_pass http://backend;
        }

        location /api/login {
            # Жорстко обмежуємо login, burst=5, з затримкою (без nodelay)
            limit_req zone=login burst=5;
            proxy_pass http://backend;
        }
    }
}
```

Різниця між `burst` без і з `nodelay`:
- **Без nodelay**: запити в burst затримуються, щоб йти зі швидкістю `rate`. Клієнт чекає.
- **З nodelay**: запити в burst обслуговуються одразу, але загальна кількість обмежена.

**limit_conn_zone** -- обмеження кількості одночасних з'єднань. Корисно проти slowloris-атак або для обмеження download-потоків.

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
    limit_conn_zone $server_name zone=conn_per_server:10m;

    server {
        # Максимум 10 одночасних з'єднань з одного IP
        limit_conn conn_per_ip 10;

        # Максимум 1000 з'єднань на весь сервер
        limit_conn conn_per_server 1000;

        location /downloads/ {
            # Обмежити швидкість завантаження одного з'єднання
            limit_rate 500k;
            limit_rate_after 10m;   # Перші 10MB без обмеження (для HTML)
        }
    }
}
```

**Типова пастка**: використовувати `$remote_addr` як ключ, коли клієнт за CDN/proxy. Тоді всі запити приходять з IP Cloudflare, і ви обмежуєте весь Cloudflare. Треба використовувати `$http_x_forwarded_for` або правильно налаштований `real_ip_header` + `set_real_ip_from`.

```nginx
http {
    # Якщо nginx за Cloudflare, довіряти їхнім IP
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 103.21.244.0/22;
    # ... усі CF диапазони
    real_ip_header CF-Connecting-IP;

    # Тепер $binary_remote_addr -- реальний IP юзера, а не CF
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
}
```

---

## Як працює compression (gzip, brotli)?

Стискання на льоту економить трафік і прискорює завантаження. Для текстових форматів (JSON, HTML, CSS, JS) економія 70-90%. Nginx стискає відповідь перед відправкою клієнту, якщо той підтримує compression (header `Accept-Encoding: gzip, br`).

```nginx
http {
    # GZIP -- підтримується всіма браузерами
    gzip on;
    gzip_vary on;                    # Додати "Vary: Accept-Encoding" header
    gzip_proxied any;                # Стискати навіть при наявності proxy headers
    gzip_comp_level 6;               # 1 (швидко, великий) - 9 (повільно, малий)
    gzip_min_length 1024;            # Не стискати відповіді <1KB (overhead)
    gzip_types
        application/json
        application/javascript
        application/xml
        text/css
        text/plain
        text/xml
        image/svg+xml;

    # BROTLI -- кращий за gzip на 15-25%, потребує модуля ngx_brotli
    brotli on;
    brotli_comp_level 4;             # 1-11, 4 -- хороший баланс
    brotli_min_length 1024;
    brotli_types
        application/json
        application/javascript
        text/css
        text/plain
        image/svg+xml;
}
```

Чому `gzip_comp_level 6` а не 9? Рівень 9 стискає на ~2-3% краще, але витрачає в 2-3 рази більше CPU. На практиці 4-6 -- золота середина. Для статичних файлів краще використовувати **pre-compressed** варіант (`gzip_static` модуль):

```nginx
# Якщо є file.js.gz поряд із file.js, віддавати його напряму
location /static/ {
    gzip_static on;                  # Використати .gz файли, якщо існують
    brotli_static on;                # Аналогічно для .br
}
```

**Типова пастка**: стискати вже стиснуті формати (JPEG, PNG, MP4, PDF) -- це додає CPU-навантаження без жодної вигоди (а інколи робить файл більшим).

---

## Static files serving: root vs alias і try_files

Nginx спочатку створювався для віддачі статики і робить це в рази швидше за Node.js. Для Node.js API типовий підхід -- віддавати статику через nginx, а всі інші запити proxy'ти на backend.

**root** -- додає location path до шляху. **alias** -- повністю замінює location path.

```nginx
server {
    # root: /var/www/site + /images/logo.png → /var/www/site/images/logo.png
    root /var/www/site;

    location /images/ {
        # Шукає файли в /var/www/site/images/
    }

    # alias: /static/logo.png → /var/www/assets/logo.png
    # (префікс /static/ ЗАМІНЮЄТЬСЯ на /var/www/assets/)
    location /static/ {
        alias /var/www/assets/;
    }
}
```

**try_files** -- спроба віддати файл з кількох можливих шляхів, з фолбеком. Класичний приклад: SPA (React/Vue), де всі URL мають віддавати `index.html`.

```nginx
server {
    root /var/www/spa/dist;
    index index.html;

    # SPA fallback: якщо файл не знайдено, віддати index.html
    location / {
        try_files $uri $uri/ /index.html;
        # Пробує:
        # 1. $uri -- точний шлях (/about → /about)
        # 2. $uri/ -- директорія з index.html
        # 3. /index.html -- fallback для client-side routing
    }

    # API запити йдуть на Node.js
    location /api/ {
        proxy_pass http://node_backend;
    }

    # Кешування статики на рік (immutable assets із hash у назві)
    location ~* \.(js|css|png|jpg|jpeg|gif|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;              # Не логувати статику (шум)
    }
}
```

**Типова пастка**: `alias` без trailing slash. Якщо `location /static/ { alias /var/www/assets; }` (без `/` в кінці alias), при запиті `/static/file.png` nginx буде шукати `/var/www/assetsfile.png` -- помилка. Слеші з обох боків мають збігатися.

---

## Як пропроксити WebSocket через nginx?

WebSocket починається як HTTP/1.1 GET із спеціальними headers (`Upgrade: websocket`, `Connection: Upgrade`), які запускають protocol upgrade на той самий TCP-сокет. За замовчуванням nginx обрізає ці headers як hop-by-hop, тому WebSocket не працює без явної конфігурації.

```nginx
# Map для коректної обробки Connection header
# Якщо клієнт надсилає "Upgrade", передаємо "upgrade"; інакше "close"
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream ws_backend {
    # ip_hash корисний для WebSocket, щоб клієнт не перемикався між серверами
    ip_hash;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}

server {
    listen 443 ssl http2;
    server_name ws.example.com;

    location /socket.io/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;                    # WebSocket потребує HTTP/1.1

        # КРИТИЧНО для WebSocket:
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Стандартні proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket з'єднання довгі -- великий таймаут
        proxy_read_timeout 3600s;                  # 1 година idle
        proxy_send_timeout 3600s;

        # Вимкнути buffering для real-time повідомлень
        proxy_buffering off;
    }
}
```

**Чому `proxy_read_timeout` важливий**: за замовчуванням 60 секунд. Якщо WebSocket не надсилає повідомлень хвилину (idle), nginx закриє з'єднання. Для чатів/ігор треба або великий таймаут, або application-level ping/pong frames.

**Чому `ip_hash`**: якщо backend зберігає WebSocket-з'єднання в пам'яті (Map<userId, socket>), і клієнт реконектиться на інший backend, повідомлення не дійдуть. Альтернатива -- sticky sessions + Redis pub/sub для cross-backend комунікації.

---

## Що робити з 502 Bad Gateway і 504 Gateway Timeout?

**502 Bad Gateway** -- nginx отримав невалідну відповідь від backend (або не зміг підключитися). Найчастіші причини:

1. **Backend не запущений** -- `connect() failed (111: Connection refused)`. Перевірити `systemctl status node-app`, чи правильний порт.
2. **Backend впав на запиті** -- Node.js кинув uncaught exception і порвав з'єднання. Перевірити логи backend.
3. **Buffer overflow** -- відповідь backend не влізла в buffer nginx.
4. **Ліміт file descriptors** -- `worker_connections are not enough`.

```nginx
# Збільшити буфери для великих відповідей
location /api/ {
    proxy_pass http://backend;
    proxy_buffer_size 16k;           # Буфер для headers відповіді
    proxy_buffers 8 16k;             # 8 буферів по 16k для тіла
    proxy_busy_buffers_size 32k;     # Буфери, що відправляються клієнту

    # Якщо backend повертає великі файли, можна вимкнути buffering
    # proxy_buffering off;
}
```

**504 Gateway Timeout** -- backend не встиг відповісти за `proxy_read_timeout` (60s за замовчуванням). Причини:

1. **Повільний запит** -- важкий SQL, зовнішній API, обробка великого файлу.
2. **Deadlock/блокування в backend** -- Node.js event loop заблокований синхронною операцією.
3. **Backend перевантажений** -- запити в черзі, не встигає.

```nginx
location /api/export {
    proxy_pass http://backend;

    # Для тривалих операцій (експорт, аналітика) збільшити таймаут
    proxy_connect_timeout 5s;
    proxy_send_timeout 300s;         # 5 хв на відправку запиту
    proxy_read_timeout 300s;         # 5 хв на очікування відповіді
}
```

**worker_connections + open files limit** -- частий прихований source 502. Кожне з'єднання = 2 file descriptors (клієнт + backend). Якщо ліміт FD в системі = 1024, реально nginx не витягне і 512 одночасних запитів.

```bash
# Системний ліміт (часто за замовчуванням 1024)
ulimit -n

# Перевірити, скільки FD відкрито у worker
ls /proc/$(pgrep -f 'nginx: worker' | head -1)/fd | wc -l

# Підняти ліміт у systemd unit /etc/systemd/system/nginx.service.d/override.conf:
# [Service]
# LimitNOFILE=65535
```

```nginx
# nginx.conf
worker_rlimit_nofile 65535;          # Підняти ліміт FD для worker

events {
    worker_connections 16384;         # Тепер можна більше
}
```

**Практичний чек-лист для 502/504**:

```
1. Backend живий?              → curl -v http://127.0.0.1:3000/health
2. Що в error.log nginx?       → tail -f /var/log/nginx/error.log
3. Що в логах backend?         → journalctl -u node-app -f
4. Скільки FD використовується? → lsof -u nginx | wc -l
5. CPU/пам'ять на backend?     → htop
6. Мережа/fail2ban?            → ss -tan | grep :3000
```

---

## Альтернативи nginx: HAProxy, Envoy, Traefik -- коли який?

**Nginx** -- універсальний солдат. HTTP server + reverse proxy + static files + SSL + basic TCP/UDP. Ідеальний для 90% задач: web app із Node.js backend, SSL termination, caching, rate limiting. Мінуси: конфіг синтаксис специфічний, перезавантаження конфігу потребує reload (хоч і graceful), open source версія не має повноцінних active health checks.

**HAProxy** -- load balancer на стероїдах. Найкраща в класі продуктивність для L4/L7 балансування, розширені health checks, stick tables для rate limiting і session persistence, детальний stats dashboard. Гірше за nginx як HTTP-сервер (не віддає статику), але краще як чистий LB. Використовується, коли треба балансувати не тільки HTTP, але й БД (PostgreSQL, MySQL), Redis, Kafka.

**Envoy** -- modern service proxy, створений у Lyft. Серце service mesh (Istio, Consul Connect). Підтримує gRPC first-class, dynamic configuration через xDS API (можна змінювати роутинг без reload), детальна observability (Prometheus metrics, tracing). Складніший у конфігурації, але must-have для Kubernetes-based мікросервісів.

**Traefik** -- cloud-native reverse proxy з автоматичним service discovery. Читає конфігурацію з Docker labels, Kubernetes Ingress, Consul, etcd. Додали новий контейнер -- Traefik сам створив route, згенерував Let's Encrypt сертифікат. Ідеальний для Docker Swarm і K8s, де сервіси з'являються/зникають динамічно. Гірше за nginx у performance під екстремальним навантаженням.

```
Коли що обирати:

┌─────────────────────┬────────────────────────────────────────────┐
│ Задача              │ Рекомендація                               │
├─────────────────────┼────────────────────────────────────────────┤
│ Web app + Node.js   │ Nginx (стандарт де-факто)                 │
│ Статика             │ Nginx (найшвидший)                        │
│ Pure L4/L7 LB       │ HAProxy (найкраща продуктивність)         │
│ Service mesh / K8s  │ Envoy + Istio                             │
│ Docker / K8s Ingress│ Traefik (auto-discovery) або nginx-ingress│
│ gRPC-heavy          │ Envoy (native HTTP/2 і gRPC)              │
│ TCP LB для БД       │ HAProxy                                   │
│ Простий setup       │ Nginx                                     │
│ Динамічна конфіг    │ Envoy/Traefik (API-driven, без reload)    │
└─────────────────────┴────────────────────────────────────────────┘
```

На практиці багато компаній використовують комбінацію: nginx на краю (SSL termination, static, rate limiting), HAProxy/Envoy всередині для internal service-to-service комунікації.

---

## Типові помилки та пастки

**1. Не перезавантажувати nginx через `restart`** -- використовуйте `nginx -s reload` (або `systemctl reload nginx`), це graceful reload без дропа активних з'єднань. Перед reload обов'язково `nginx -t` для перевірки синтаксису.

**2. Забувати `proxy_http_version 1.1` + `proxy_set_header Connection ""`** для keepalive. Без цього nginx робить HTTP/1.0 без keepalive, що дає +1мс latency на кожен запит і спалює ephemeral ports на бекенді.

**3. Не налаштовувати `client_max_body_size`** -- за замовчуванням 1MB. При завантаженні файлів отримаєте `413 Request Entity Too Large`. Потрібно явно вказати `client_max_body_size 50M;` в location для upload.

**4. Використовувати `if` там, де не треба** -- "if is evil" (офіційна документація nginx). `if` усередині `location` часто ламає конфіг нетривіальним чином. Замість `if` використовуйте `map`, `try_files`, або окремі `location` блоки.

**5. Не обробляти trailing slashes в `proxy_pass`** -- `proxy_pass http://backend;` і `proxy_pass http://backend/;` (зі слешем) поводяться по-різному. Зі слешем nginx замінює location-префікс на URI з proxy_pass. Без слеша передає URI as-is. Це джерело тонн годин налагодження.

**6. `access_log` в hot loop** -- якщо ви обробляєте 50к rps, write до access_log стає bottleneck. Використовуйте `access_log off` для health checks і статики, або буферизуйте: `access_log /var/log/nginx/access.log buffer=32k flush=5s;`.

**7. Не моніторити `nginx_status`** -- увімкніть `stub_status` endpoint і збирайте метрики: active connections, accepts, requests per second. Без моніторингу ви дізнаєтесь про проблему від користувачів.

```nginx
server {
    listen 127.0.0.1:8080;           # Тільки з localhost!
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }
}
```

**8. Ігнорувати `worker_cpu_affinity`** -- на багатоядерних серверах прив'язка workers до CPU-ядер дає +10-15% performance через cache locality:

```nginx
worker_processes auto;
worker_cpu_affinity auto;            # Nginx сам розподілить по ядрах
```

**9. Використовувати `$host` там, де треба `$http_host`** (або навпаки). `$host` -- з request line або Host header, lowercase, без порту. `$http_host` -- точне значення Host header. Для proxy_pass зазвичай `$host`.

**10. Забувати про DNS caching для upstream** -- якщо `upstream` вказує на домен (не IP), nginx резолвить його раз при старті і кешує назавжди. Якщо IP змінився (наприклад, AWS ELB), nginx цього не побачить. Рішення: комерційний nginx Plus із `resolve` директивою, або в open source -- використання `resolver` + змінної:

```nginx
resolver 8.8.8.8 valid=30s;
set $backend "api.example.com";
proxy_pass http://$backend;          # Тепер резолвиться динамічно
```
