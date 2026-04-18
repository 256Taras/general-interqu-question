# Networking Fundamentals для Backend/DevOps

## Що таке OSI та TCP/IP моделі, та чому на практиці використовується саме TCP/IP?

OSI (Open Systems Interconnection) -- це теоретична 7-шарова модель, розроблена ISO у 1984 році. Вона описує, як мережеві протоколи мали б взаємодіяти. TCP/IP -- це реальний, практичний стек, який виріс з ARPANET і став основою сучасного Інтернету. TCP/IP має 4 (іноді 5) шари, і саме він реально реалізований у всіх ОС, мережевому обладнанні та бібліотеках.

```
OSI Model (теорія)          TCP/IP Model (практика)     Приклади
┌─────────────────────┐     ┌─────────────────────┐
│ 7. Application       │     │                     │     HTTP, DNS, SSH, FTP,
├─────────────────────┤     │                     │     SMTP, gRPC, WebSocket
│ 6. Presentation      │ ──▶ │  Application (L7)    │
├─────────────────────┤     │                     │     TLS часто сюди
│ 5. Session           │     │                     │     приписують
├─────────────────────┤     ├─────────────────────┤
│ 4. Transport         │ ──▶ │  Transport (L4)      │     TCP, UDP, QUIC
├─────────────────────┤     ├─────────────────────┤
│ 3. Network           │ ──▶ │  Internet (L3)       │     IP, ICMP, routing
├─────────────────────┤     ├─────────────────────┤
│ 2. Data Link         │ ──▶ │  Link (L2)           │     Ethernet, ARP, MAC,
├─────────────────────┤     │                     │     Wi-Fi, VLAN
│ 1. Physical          │     │                     │     Кабелі, радіосигнали
└─────────────────────┘     └─────────────────────┘
```

На практиці використовується TCP/IP, бо:
- Він існував до OSI та реально працював (DoD, університети)
- Прості та прагматичні RFC замість "ідеальних" специфікацій
- Розробники мислять у термінах TCP/IP: "L4 balancer", "L7 proxy", "L3 routing"

OSI досі корисна як словник: коли кажуть "L7 load balancer", мають на увазі саме шар OSI.

**На якому шарі працює типове обладнання та софт:**

| Компонент | Шар | Що бачить |
|---|---|---|
| Switch (звичайний) | L2 | MAC-адреси |
| Router | L3 | IP-адреси |
| Firewall (stateless) | L3/L4 | IP + порт |
| Firewall (stateful, напр. AWS SG) | L4 | IP + порт + стан з'єднання |
| AWS NLB | L4 | TCP/UDP-з'єднання, не розуміє HTTP |
| HAProxy в mode tcp | L4 | TCP-з'єднання |
| AWS ALB | L7 | HTTP-заголовки, path, host |
| Nginx / HAProxy в mode http | L7 | HTTP-запити |
| WAF (Web Application Firewall) | L7 | HTTP body, cookies, параметри |
| VPN (IPSec) | L3 | Інкапсулює IP-пакети |
| VPN (OpenVPN, WireGuard) | L3/L4 | IP-пакети над UDP/TCP |

**Ключова відмінність L4 vs L7:** L4-балансер не читає payload, тільки приймає рішення на основі IP:порт -- тому швидший, але не вміє routing'у за path чи host. L7-балансер парсить HTTP і може робити `path-based routing`, додавати `X-Forwarded-For`, термінувати TLS.

---

## Чим TCP відрізняється від UDP? Що таке 3-way handshake?

**TCP (Transmission Control Protocol)** -- надійний, з встановленням з'єднання, з гарантією доставки та порядку. **UDP (User Datagram Protocol)** -- простий, без з'єднання, "fire and forget", без гарантій.

| Критерій | TCP | UDP |
|---|---|---|
| З'єднання | Встановлюється (handshake) | Немає |
| Надійність | Гарантована доставка, retransmit | Немає |
| Порядок | Гарантований | Не гарантований |
| Контроль перевантажень | Є (congestion control) | Немає |
| Overhead | ~20-60 байт заголовок | 8 байт заголовок |
| Швидкість | Повільніший | Швидший |
| Use cases | HTTP, SSH, бази даних | DNS, VoIP, ігри, відеотрансляції |

**TCP 3-way handshake:**

```
Client                              Server
  │                                   │
  │ ──────── SYN (seq=x)  ─────────▶ │   Клієнт: "хочу з'єднатись,
  │                                   │             мій initial seq = x"
  │                                   │
  │ ◀──── SYN-ACK (seq=y, ack=x+1) ── │   Сервер: "ок, мій seq = y,
  │                                   │             підтверджую твій x"
  │                                   │
  │ ──────── ACK (ack=y+1) ─────────▶ │   Клієнт: "підтверджую y"
  │                                   │
  │ ═══════ CONNECTION ESTABLISHED ══════════
  │                                   │
  │ ◀───── DATA обидва напрямки ────▶ │
```

Після handshake йде обмін даними. Кожен сегмент має seq number, і приймач надсилає ACK з очікуваним наступним номером.

**Congestion control (перевантаження):** TCP автоматично зменшує швидкість передачі при втраті пакетів. Алгоритми: Reno, CUBIC (за замовчуванням у Linux), BBR (Google, новий). Починається зі slow start: congestion window подвоюється кожен RTT, поки не сталася втрата пакета -- тоді window зменшується.

**Retransmit:** якщо ACK не прийшов за RTO (Retransmission Timeout), TCP повторно надсилає сегмент. Також є fast retransmit: 3 дубльовані ACK -> негайний retransmit без очікування таймера.

**Де UDP доречніший:**
- **DNS** -- один запит/відповідь, handshake зайвий, якщо відповідь вміщається в 512 байт
- **QUIC / HTTP/3** -- UDP як транспорт, але надійність робиться у user-space
- **VoIP, відеодзвінки** -- краще втратити пакет, ніж чекати retransmit (затримка гірша за артефакт)
- **Онлайн-ігри** -- актуальні дані важливіші за старі втрачені
- **NTP, SNMP, DHCP** -- короткі broadcast/unicast повідомлення
- **Відеотрансляції** (UDP-based протоколи типу SRT, RTP)

---

## Що таке TCP SYN, ACK, FIN, RST? Що таке TIME_WAIT і CLOSE_WAIT?

TCP-сегменти мають біти-прапори (flags) у заголовку. Основні:
- **SYN** -- synchronize, початок з'єднання
- **ACK** -- acknowledgement, підтвердження
- **FIN** -- finish, ввічливе закриття з'єднання
- **RST** -- reset, примусове скидання (без graceful shutdown)
- **PSH** -- push, передати дані вгору без буферизації
- **URG** -- urgent pointer (майже не використовується)

**Graceful close (FIN):**

```
Client                              Server
  │                                   │
  │ ───────── FIN ────────────────▶ │   "я все надіслав"
  │                                   │
  │ ◀──────── ACK ─────────────────── │   "прийняв твій FIN"
  │                                   │
  │ ◀──────── FIN ─────────────────── │   "я теж все надіслав"
  │                                   │
  │ ────────── ACK ───────────────▶ │   "прийняв"
  │                                   │
  │     TIME_WAIT (2 * MSL ~ 60s)    │
  │                                   │
```

**Повний life cycle стану TCP:**

```
        CLOSED
          │
          │ (відкриття клієнтом)
          ▼
        SYN_SENT ──────▶ ESTABLISHED ◀──── SYN_RECEIVED
                            │    │              ▲
                            │    │              │
                  (active)  │    │  (passive)   │ LISTEN
                   close    │    │   close      │
                            ▼    ▼              │
                     FIN_WAIT_1  CLOSE_WAIT ────┘
                            │         │
                            ▼         ▼
                     FIN_WAIT_2    LAST_ACK
                            │         │
                            ▼         │
                       TIME_WAIT      │
                            │         │
                            ▼         ▼
                          CLOSED   CLOSED
```

**TIME_WAIT** -- стан на стороні, яка ініціювала закриття (active close). Триває `2 * MSL` (Maximum Segment Lifetime, ~ 60 секунд у Linux). Потрібен щоб:
1. Гарантувати, що останній ACK дійшов до peer (якщо загубиться -- peer ретранслює FIN, ми можемо відповісти)
2. Уникнути, щоб "заблукалі" старі сегменти з цього ж (IP:port, IP:port) не втрутились у нове з'єднання

**Проблема:** на сервері, який відкриває багато вихідних з'єднань (наприклад, до БД, до API), TIME_WAIT може вичерпати ephemeral ports. Рішення: connection pooling, `SO_REUSEADDR`, збільшення діапазону ефемерних портів.

**CLOSE_WAIT** -- стан на стороні, яка отримала FIN, але ще не закрила з'єднання. Зазвичай означає **баг у коді**: сервер отримав FIN, але не викликав `close()` на сокеті. Велика кількість CLOSE_WAIT -- сигнал що треба шукати leak'и.

**RST (Reset)** -- надсилається коли:
- Порт закритий (SYN на порт без listener'а -> RST)
- Протокол порушено
- Застосунок викликав `SO_LINGER` з timeout=0 і закрив з'єднання
- Firewall/балансер примусово обірвав з'єднання

**Дебаг через `ss`:**

```bash
# Всі з'єднання з статистикою
ss -tanp

# Рахуємо з'єднання за станом
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c
# Приклад виводу:
#    150 ESTAB
#   1523 TIME-WAIT
#      4 CLOSE-WAIT
#      2 LISTEN

# Багато TIME_WAIT -> проблема з connection reuse
# Багато CLOSE_WAIT -> баг, застосунок не закриває з'єднання

# З'єднання до конкретного сервіса
ss -tanp '( dport = :5432 or sport = :5432 )'

# Слухаючі порти
ss -tlnp

# Детальна інфа про конкретне з'єднання (RTT, cwnd, etc.)
ss -tin
```

---

## Чим відрізняються HTTP/1.1, HTTP/2, HTTP/3?

Всі три версії HTTP мають однакову семантику (GET, POST, headers, status codes), але різняться транспортом і оптимізаціями.

**HTTP/1.1 (1997)** -- текстовий протокол над TCP. Особливості:
- **Keep-alive** (persistent connections): одне TCP-з'єднання для декількох запитів
- **Pipelining** (теоретично): надсилати кілька запитів не чекаючи відповідей -- але на практиці майже не використовується через head-of-line blocking
- Один запит за раз на з'єднанні -> браузери відкривають 6-8 паралельних з'єднань до одного хосту
- Head-of-line blocking на рівні HTTP: повільний запит блокує наступні у черзі на тому ж з'єднанні

**HTTP/2 (2015, RFC 7540)** -- бінарний протокол, все ще над TCP+TLS.
- **Multiplexing**: один TCP-зв'язок, багато логічних streams паралельно
- **Header compression** (HPACK): повторювані заголовки стискаються
- **Server Push** (майже не використовується, виключено з багатьох імплементацій)
- **Stream prioritization**: клієнт вказує пріоритети (критичний CSS важливіший за картинку)
- Проблема: head-of-line blocking переноситься на рівень TCP. Один втрачений TCP-сегмент блокує ВСІ stream'и поки не прийде retransmit.

```
HTTP/1.1:                           HTTP/2:
Conn 1: req1 ─▶ res1 ─▶ req2 ─▶... Conn 1: req1+req2+req3 (multiplexed)
Conn 2: req3 ─▶ res3 ─▶...                 res1+res2+res3 (multiplexed)
Conn 3: ...
(до 6 паралельних з'єднань)         (одне з'єднання)
```

**HTTP/3 (2022, RFC 9114)** -- над **QUIC** замість TCP. QUIC працює над UDP.
- Handshake об'єднаний: TCP+TLS = 2-3 RTT, QUIC = 1 RTT (або 0-RTT для повернення)
- Multiplexing без head-of-line blocking: кожен stream має власні sequence numbers, втрата в одному не блокує інші
- Connection migration: якщо клієнт змінює IP (Wi-Fi -> 4G) -- з'єднання не рветься
- Обов'язкова шифрація (TLS 1.3 вбудований)
- Мінуси: не всі firewalls/middleboxes пропускають UDP, складніший дебаг

**Порівняння handshake'ів (час до першого байту):**

```
HTTP/1.1 + TLS:
TCP: 1 RTT   TLS: 2 RTT   HTTP: 1 RTT   = 4 RTT

HTTP/2 + TLS 1.3:
TCP: 1 RTT   TLS 1.3: 1 RTT   HTTP: 1 RTT   = 3 RTT

HTTP/3 (QUIC):
QUIC (TLS 1.3 inside): 1 RTT   HTTP: 0 RTT   = 1 RTT
QUIC 0-RTT (resumption): 0 RTT                 = 0 RTT
```

---

## Які HTTP-методи існують, що таке idempotency та safe methods?

**HTTP verbs (методи):**

| Метод | Safe | Idempotent | Body | Типове призначення |
|---|---|---|---|---|
| GET | Так | Так | Ні | Отримати ресурс |
| HEAD | Так | Так | Ні | Отримати тільки headers (перевірити існування) |
| OPTIONS | Так | Так | Ні | Отримати дозволені методи (CORS) |
| POST | Ні | Ні | Так | Створити ресурс, generic action |
| PUT | Ні | Так | Так | Повна заміна ресурса |
| PATCH | Ні | Ні\* | Так | Часткове оновлення |
| DELETE | Ні | Так | Іноді | Видалити ресурс |

**Safe** = не змінює стан на сервері (тільки читає).
**Idempotent** = повторний виклик з тими ж параметрами дає той же результат. Важливо для retry: якщо запит таймаутнув, чи можна його безпечно повторити?

PATCH формально не idempotent (залежить від семантики), але якщо передавати повний набір полів -- стає idempotent.

POST не idempotent: два `POST /orders` створять два ордери. Якщо потрібна idempotency (наприклад, платежі) -- використовують **Idempotency-Key header**: клієнт генерує UUID, сервер зберігає результат за цим ключем і повертає кешовану відповідь при повторі.

**Status codes:**

- **1xx** -- Informational (рідко). `100 Continue`, `101 Switching Protocols` (WebSocket upgrade).

- **2xx** -- Success.
  - `200 OK` -- успіх, є body
  - `201 Created` -- створено, зазвичай з `Location: /resource/id`
  - `202 Accepted` -- прийнято для асинхронної обробки
  - `204 No Content` -- успіх, body порожній (наприклад, після DELETE)

- **3xx** -- Redirection.
  - `301 Moved Permanently` -- постійно, кешується браузером назавжди
  - `302 Found` / `307 Temporary Redirect` -- тимчасово
  - `304 Not Modified` -- кеш валідний (разом з ETag / If-None-Match)

- **4xx** -- Client error.
  - `400 Bad Request` -- невалідний синтаксис / дані
  - `401 Unauthorized` -- не автентифіковано (насправді "not authenticated")
  - `403 Forbidden` -- автентифіковано, але не авторизовано
  - `404 Not Found` -- ресурсу немає
  - `405 Method Not Allowed` -- метод не підтримується для URL
  - `409 Conflict` -- конфлікт стану (race condition, duplicate)
  - `422 Unprocessable Entity` -- синтаксично правильно, але семантично невалідно
  - `429 Too Many Requests` -- rate limit (разом з `Retry-After`)

- **5xx** -- Server error.
  - `500 Internal Server Error` -- загальний збій
  - `502 Bad Gateway` -- proxy/LB не зміг отримати відповідь від upstream
  - `503 Service Unavailable` -- сервіс перевантажений / на maintenance
  - `504 Gateway Timeout` -- proxy/LB не дочекався upstream

Типова плутанина на співбесідах: 401 vs 403. 401 = "я не знаю хто ти" (треба залогінитись), 403 = "я знаю хто ти, але тобі сюди не можна".

---

## Як працює TLS handshake? Що таке chain of trust, CA, Let's Encrypt, ACME?

**TLS (Transport Layer Security)** -- наступник SSL. Забезпечує:
1. **Конфіденційність** -- шифрація даних
2. **Цілісність** -- MAC / AEAD, ніхто не змінив пакет
3. **Автентифікацію** -- сервер (а іноді й клієнт) справжній

**TLS 1.3 handshake (спрощено, 1 RTT):**

```
Client                                                Server
  │                                                     │
  │ ── ClientHello ─────────────────────────────────▶ │
  │    (supported ciphers, random, key share)           │
  │                                                     │
  │ ◀── ServerHello ─────────────────────────────────── │
  │    (chosen cipher, random, key share,               │
  │     certificate, certificate verify,                │
  │     finished) [зашифровано з handshake keys]       │
  │                                                     │
  │ ── Finished ────────────────────────────────────▶ │
  │    [зашифровано з application keys]                 │
  │                                                     │
  │ ══════ ENCRYPTED APPLICATION DATA ══════════════════│
  │ ◀═══════════════════════════════════════════════▶  │
```

У TLS 1.2 було 2 RTT: ClientHello -> ServerHello+Certificate -> ClientKeyExchange+Finished -> Finished.

TLS 1.3 також підтримує **0-RTT resumption**: якщо раніше спілкувались, клієнт може надіслати дані разом з ClientHello. Мінус: вразливий до replay-атак, тому 0-RTT треба використовувати тільки для idempotent запитів.

**Certificates та chain of trust:**

```
                    Root CA
         (у trust store браузера/ОС)
                       │
                       │ підписує
                       ▼
               Intermediate CA
                       │
                       │ підписує
                       ▼
          Leaf (site) certificate
                 example.com
```

Браузер/клієнт має вбудований список Root CAs (Trust Store). Коли сервер надсилає свій сертифікат, клієнт:
1. Перевіряє підпис leaf-сертифіката через ланцюг до Root CA
2. Перевіряє дату (not before / not after)
3. Перевіряє CN або SAN (Subject Alternative Name) -- чи домен збігається
4. Перевіряє revocation (OCSP / CRL)

**Certificate Authority (CA)** -- організація, якій довіряють (DigiCert, Let's Encrypt, Sectigo). Вони підписують сертифікати після валідації, що ви володієте доменом.

**Let's Encrypt + ACME:**
- Безкоштовний публічний CA (від ISRG)
- Використовує протокол **ACME** (Automatic Certificate Management Environment, RFC 8555)
- Сертифікати дійсні 90 днів -> обов'язкова автоматизація (certbot, acme.sh, cert-manager в k8s)
- Валідація через HTTP-01 (файл у `/.well-known/acme-challenge/`) або DNS-01 (TXT-запис)

**OCSP (Online Certificate Status Protocol)** -- механізм перевірки, чи сертифікат відкликаний. Клієнт запитує CA "чи живий цей сертифікат?". Проблеми: privacy (CA бачить, хто куди ходить), повільно. Рішення: **OCSP stapling** -- сервер сам періодично отримує відповідь від CA і вкладає її в handshake.

**Certificate pinning** -- клієнт (зазвичай мобільний додаток) "запам'ятовує" конкретний сертифікат або публічний ключ сервера, і відмовляється приймати інші, навіть підписані довіреним CA. Захист від compromised CA. Мінус: rotation ускладнюється.

**Дебаг TLS:**

```bash
# Переглянути сертифікат сервера
openssl s_client -connect example.com:443 -servername example.com

# Тільки ланцюг сертифікатів
openssl s_client -connect example.com:443 -showcerts </dev/null

# Перевірити expiration
echo | openssl s_client -connect example.com:443 2>/dev/null \
    | openssl x509 -noout -dates

# Детальна інформація сертифіката
echo | openssl s_client -connect example.com:443 2>/dev/null \
    | openssl x509 -noout -text

# Тест конкретної версії TLS
openssl s_client -connect example.com:443 -tls1_2
openssl s_client -connect example.com:443 -tls1_3
```

---

## Що таке mTLS і де він використовується?

**mTLS (mutual TLS)** -- розширення TLS, де **обидві сторони** автентифікуються через сертифікати. У звичайному TLS сервер доводить свою автентичність клієнту, у mTLS -- і клієнт доводить серверу.

```
Звичайний TLS:                        mTLS:
  Client          Server                Client              Server
    │                │                    │                    │
    │ ───ClientHello─▶                   │ ────ClientHello───▶│
    │                │                    │                    │
    │ ◀──ServerCert──                     │ ◀──ServerCert──    │
    │                                     │    +CertRequest    │
    │                │                    │                    │
    │ ───Finished──▶                      │ ───ClientCert────▶│
    │                                     │    +CertVerify     │
    │                                     │    +Finished        │
```

**Де використовується:**

1. **Service Mesh** (Istio, Linkerd, Consul Connect) -- всі pod-to-pod комунікації автоматично шифруються mTLS. Sidecar proxy (Envoy) термінує TLS і видає сертифікат на основі SPIFFE ID.

2. **B2B API** -- коли партнер підключається до вашого API і ви не хочете покладатись лише на API key / токен. Приклад: платіжні API (Visa, банки), медичні системи.

3. **Zero-trust networks** -- принцип "never trust, always verify". Кожен запит всередині мережі -- mTLS, незалежно від того, внутрішній чи зовнішній.

4. **IoT devices** -- пристрій з фабричним сертифікатом довідує свою ідентичність серверу.

5. **Kubernetes API server** -- kubelet і control plane використовують mTLS.

Переваги: немає секретів (паролі, API-ключі) у trafiку. Мінуси: складність PKI, ротація сертифікатів, треба власний CA або використати спеціальний (Vault, cert-manager).

```bash
# Приклад mTLS клієнта через curl
curl --cert client.crt --key client.key --cacert ca.crt \
    https://api.example.com/resource

# Тест з openssl
openssl s_client -connect api.example.com:443 \
    -cert client.crt -key client.key -CAfile ca.crt
```

---

## Як працює DNS? Які є record types та що таке TTL?

DNS (Domain Name System) -- ієрархічна розподілена система, що перетворює імена (`example.com`) на IP-адреси.

**Ієрархія DNS:**

```
                       . (Root)
                       │
        ┌──────────────┼──────────────┐
        │              │              │
       com            org             ua         (TLD servers)
        │              │              │
        │              │              │
     example        wikipedia       gov.ua       (Authoritative)
     .com             .org           .ua
        │
     www.example.com
     api.example.com
```

**Resolution flow:**

```
  ┌─────────┐
  │ Browser │ "який IP у example.com?"
  └────┬────┘
       │ 1
       ▼
  ┌──────────────┐      2       ┌──────────┐
  │  OS / Stub   │ ─────────▶  │ Recursive │
  │  resolver    │              │ resolver  │  (напр. 8.8.8.8,
  │              │ ◀─────────── │           │   1.1.1.1, ISP DNS)
  └──────────────┘      7       └────┬──┬───┘
                                     │  │
                              3      │  │      4
                         ┌───────────┘  └──────────────┐
                         ▼                              ▼
                   ┌──────────┐                  ┌───────────┐
                   │ Root NS   │                  │ TLD NS (.com)│
                   │ (.)       │                  │            │
                   └──────────┘                  └───────┬────┘
                         │                                │
                         │  "запитай TLD .com"            │ 5
                         └────────────────────────────────┘
                                                         ▼
                                                ┌───────────────┐
                                                │ Authoritative NS│
                                                │ (example.com)   │
                                                │                 │
                                                │ A = 93.184.216.34│
                                                └───────────────┘
                                                         │ 6
                                                         ▼
                                                (повертає IP
                                                 recursive-у)
```

Recursive resolver кешує відповіді на час TTL.

**Типи DNS записів:**

| Тип | Призначення | Приклад |
|---|---|---|
| `A` | IPv4 адреса | `example.com A 93.184.216.34` |
| `AAAA` | IPv6 адреса | `example.com AAAA 2606:2800:220:1::` |
| `CNAME` | Alias на інше ім'я | `www.example.com CNAME example.com` |
| `MX` | Поштовий сервер з пріоритетом | `example.com MX 10 mail.example.com` |
| `TXT` | Довільний текст (SPF, DKIM, verification) | `example.com TXT "v=spf1 ..."` |
| `NS` | Authoritative name server | `example.com NS ns1.example.com` |
| `SOA` | Start of Authority (інфа про зону) | серійний номер, TTL |
| `SRV` | Сервіс+порт (XMPP, SIP, k8s) | `_sip._tcp.example.com SRV 10 5 5060 sip.example.com` |
| `PTR` | Reverse DNS (IP -> name) | `34.216.184.93.in-addr.arpa PTR example.com` |
| `CAA` | Які CA можуть видавати сертифікати | `example.com CAA 0 issue "letsencrypt.org"` |

**TTL (Time to Live)** -- скільки секунд DNS-клієнти можуть кешувати запис. Компроміс:
- Низький TTL (60-300s) -> швидке поширення змін, більше запитів до DNS
- Високий TTL (86400s = 1 день) -> повільне поширення, менше навантаження

**Чому іноді зміна DNS довго поширюється:**
1. **Кеш recursive resolvers** -- навіть після зменшення TTL, старі значення залишаються в кеші до закінчення попереднього TTL
2. **Кеш ОС** (`nscd`, `systemd-resolved`) -- окремий шар кешу
3. **Кеш браузера** (~60 секунд у Chrome)
4. **"Negative caching"** -- "запис не існує" теж кешується
5. **CDN та публічні DNS** ігнорують малий TTL (напр. Google може ставити мінімум 30 с)
6. **Stale records** -- деякі старі resolvers не дотримуються TTL

Правило: перед міграцією DNS заздалегідь (наприклад, за тиждень) знижуйте TTL до 60 секунд. Після міграції -- поверніть високий.

**Дебаг DNS:**

```bash
# Простий query
dig example.com

# Конкретний тип
dig example.com MX
dig example.com AAAA
dig example.com TXT

# Через конкретний resolver
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com

# Trace (повний шлях від root)
dig +trace example.com

# Тільки коротка відповідь
dig +short example.com

# Reverse DNS
dig -x 8.8.8.8

# Альтернатива
nslookup example.com
host example.com

# Подивитись на TTL
dig example.com | grep -A1 ANSWER
```

---

## L4 vs L7 Load Balancing. Що таке sticky sessions та connection draining?

**L4 (Transport) Load Balancing:**
- Працює на рівні TCP/UDP
- Бачить тільки IP і порт
- Дуже швидкий (часто на рівні ядра або апаратного рівня)
- Не може робити маршрутизацію за path/host
- Приклади: AWS NLB, HAProxy в mode tcp, IPVS, F5

**L7 (Application) Load Balancing:**
- Парсить HTTP
- Може робити routing за host, path, headers, cookies
- Може термінувати TLS, модифікувати заголовки, робити compression
- Повільніший, більше CPU
- Приклади: AWS ALB, Nginx, HAProxy mode http, Envoy, Traefik

```
L4 balancer (NLB):                   L7 balancer (ALB):
  Client                               Client
    │                                    │
    │ TCP connection to LB              │ HTTP request
    ▼                                    ▼
  ┌──────┐                             ┌──────┐
  │  LB  │── forward TCP ──▶          │  LB  │── parse HTTP ──▶ routing
  │ (L4) │── raw bytes ──▶            │ (L7) │── based on      ── ─▶
  └──────┘                             └──────┘   host/path
    │                                    │
    ▼                                    ▼
  backends                            backend pool A (/api/*)
                                       backend pool B (/static/*)
```

**Алгоритми балансування:**
- **Round-robin** -- по черзі
- **Least connections** -- на backend з найменшою кількістю активних з'єднань
- **IP hash** -- хеш IP клієнта визначає backend (стабільний mapping)
- **Weighted** -- різні ваги для різних backends
- **Random with two choices** (power of two choices) -- вибираємо 2 випадкових, з них з найменшим навантаженням

**Sticky sessions (session affinity)** -- гарантія, що запити одного клієнта йдуть на той самий backend. Способи:
1. **IP affinity** -- за source IP (ненадійно через NAT, CGN)
2. **Cookie-based** -- балансер підмішує свій cookie (AWSALB, `srv_id`)
3. **SSL session ID** -- за TLS session

Коли потрібні: local session state (у пам'яті), stateful WebSocket-з'єднання. Проблема: нерівномірне навантаження, складніше масштабування. Краще: stateless backends + зовнішній session store (Redis).

**Connection draining (deregistration delay)** -- при виведенні backend-а з ротації:
1. Балансер перестає надсилати нові з'єднання на цей backend
2. Існуючі з'єднання дотягують свої запити (наприклад, 30-300 сек)
3. Після таймауту залишкові з'єднання примусово закриваються

Критично для zero-downtime deployments. AWS ALB має "deregistration delay" (default 300 сек), Kubernetes -- `preStop` hook + `terminationGracePeriodSeconds`.

**DNS round-robin vs справжній LB:**

DNS round-robin: authoritative DNS повертає список IP-адрес у різному порядку. Клієнт вибирає перший. Проблеми:
- Немає health-checks: мертвий backend продовжує отримувати трафік
- TTL кешується -> відмова розподіляється повільно
- Нерівномірний розподіл (деякі клієнти агресивно кешують)
- Немає session affinity, connection draining, тощо

Використовується: коли є справжній LB, а DNS round-robin розподіляє між кількома LB-ами в різних регіонах (GeoDNS). Або для простих внутрішніх сервісів.

---

## Як працює CDN? Edge caching, cache invalidation, geo-routing.

**CDN (Content Delivery Network)** -- географічно розподілена мережа проксі-серверів (edge nodes), які кешують контент близько до користувачів.

```
Без CDN:                             З CDN:
  User (Ukraine)                       User (Ukraine)
      │                                     │
      │ ~200ms RTT                          │ ~10ms RTT
      ▼                                     ▼
  Origin (US-East)                     Edge (Kyiv / Frankfurt)
                                            │
                                            │ cache miss -> origin
                                            │ ~100ms RTT (тільки перший раз)
                                            ▼
                                        Origin (US-East)
```

Користувачі автоматично потрапляють на найближчий edge через anycast IP (один IP рутиться до різних POP в різних регіонах) або GeoDNS.

**Edge caching:**
- CDN кешує відповіді за URL (з урахуванням `Vary` header)
- Перший запит -> cache miss -> походить до origin -> кешує
- Наступні запити -> cache hit -> віддає з edge

Управління через HTTP headers:
- `Cache-Control: public, max-age=86400` -- кешувати 1 день
- `Cache-Control: private` -- не кешувати на CDN (тільки в браузері)
- `Cache-Control: no-store` -- взагалі не кешувати
- `ETag: "abc"` / `Last-Modified: ...` -- для перевалідації (If-None-Match / If-Modified-Since)
- `Vary: Accept-Encoding, Accept-Language` -- різний кеш для різних значень цих заголовків

**Cache invalidation:**
1. **TTL expiration** -- природний, найдешевший
2. **Purge / Invalidation API** -- примусово видалити ключ (Cloudflare Purge, CloudFront Invalidation). Зазвичай обмежено кількістю запитів на місяць.
3. **Cache key versioning** -- змінити URL (`/app-v1.23.js` замість `/app.js`). Надійний для статики
4. **Surrogate keys / Cache tags** -- прив'язка об'єктів до логічних груп, invalidate за тегом (Fastly, Varnish)

**Geo-routing** -- маршрутизація користувача до найближчого POP:
- **Anycast** -- один IP, різні сервери у різних локаціях (BGP вирішує, куди роутити). Використовує Cloudflare
- **GeoDNS** -- авторитетний DNS повертає різні IP залежно від IP resolver'а клієнта. Використовують AWS Route 53

**Коли CDN не допомагає:**
1. **Динамічний контент** -- персоналізований feed, unique для юзера. Без cache hits переваги немає.
2. **Запис/POST-запити** -- CDN зазвичай не кешує, вони проксять до origin (latency може навіть зрости)
3. **Authenticated pages** -- якщо cache key не враховує user session, буде витік даних. Треба `Cache-Control: private`.
4. **WebSocket / long-polling** -- потребують спеціальної підтримки, або повз CDN
5. **Single-region users** -- якщо всі юзери в одному регіоні і origin там же, CDN додає лише overhead
6. **Файли що часто змінюються** -- cache churn, постійні походи до origin

CDN також часто надає: DDoS protection, WAF, image optimization, edge workers (Cloudflare Workers, AWS Lambda@Edge).

---

## Що таке public vs private IP, NAT, CIDR, RFC 1918?

**Public vs Private IP:**

IPv4-адреси поділяють на публічні (routable в Інтернеті) та приватні (використовуються всередині мереж).

**RFC 1918 (приватні діапазони IPv4):**

```
10.0.0.0        - 10.255.255.255       (10.0.0.0/8)        16,777,216 адрес
172.16.0.0      - 172.31.255.255       (172.16.0.0/12)     1,048,576 адрес
192.168.0.0     - 192.168.255.255      (192.168.0.0/16)    65,536 адрес
```

Додатково:
- `127.0.0.0/8` -- loopback
- `169.254.0.0/16` -- link-local (APIPA; AWS metadata -- `169.254.169.254`)
- `100.64.0.0/10` -- CGN (Carrier-Grade NAT)

**NAT (Network Address Translation)** -- заміна src/dst IP при проходженні через роутер. Використовується бо IPv4-адрес не вистачає: мільярди пристроїв за приватними мережами ховаються за одною публічною IP.

**PAT (Port Address Translation)** або "NAT overload" -- розширення NAT, що використовує також порти. Роутер пам'ятає mapping: `(внутрішній IP:порт) <-> (зовнішній IP:зовнішній порт)`.

```
Внутрішня мережа 192.168.1.0/24          Інтернет
  ┌──────────────┐
  │ PC 192.168.1.10:54321
  │ ─────────────┐
  │              │                          Router (public 203.0.113.5)
  │              │                          ┌─────────────┐
  │              └───────SYN──────────────▶│             │
  │                                         │ NAT table:  │
  │                                         │ 192.168.1.10:54321 │
  │                                         │  <-> 203.0.113.5:40000│
  │                                         └──────┬──────┘
  │                                                │
  │                                                │ SYN (src=203.0.113.5:40000)
  │                                                ▼
  │                                         Server (198.51.100.10:443)
```

NAT "ламає" peer-to-peer (звідси техніки типу STUN, TURN, hole punching).

**CIDR (Classless Inter-Domain Routing)** -- запис `/prefix-length`.

| Нотація | Маска | Кількість хостів | Приклад використання |
|---|---|---|---|
| `/32` | 255.255.255.255 | 1 | окремий хост |
| `/30` | 255.255.255.252 | 2 корисних | point-to-point link |
| `/24` | 255.255.255.0 | 254 | типова офісна мережа |
| `/16` | 255.255.0.0 | 65534 | великий VPC segment |
| `/8` | 255.0.0.0 | 16M | клас A (дуже рідко) |

Як розрахувати хости: `2^(32 - prefix) - 2` (мінус broadcast та network address).

**Приклад:** `10.0.1.0/24`
- Network: 10.0.1.0
- Broadcast: 10.0.1.255
- Host range: 10.0.1.1 - 10.0.1.254
- 254 корисних адреси

**Subnetting приклад:** 10.0.0.0/16 можна розділити на:
- 10.0.0.0/24 (public subnet, AZ-a)
- 10.0.1.0/24 (public subnet, AZ-b)
- 10.0.10.0/24 (private subnet, AZ-a)
- 10.0.11.0/24 (private subnet, AZ-b)
...і так далі.

**IPv6 коротко:**
- 128-бітні адреси (vs 32 у IPv4), ~3.4 * 10^38
- Формат: 8 груп по 4 hex-цифри: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- Скорочення: нульові групи -> `::` (тільки раз на адресу), `2001:db8:85a3::8a2e:370:7334`
- Немає broadcast, замість нього -- multicast
- NAT не потрібен, кожен пристрій має публічну адресу
- Dual-stack: в сучасних системах IPv4 + IPv6 працюють паралельно

---

## Які інструменти використовувати для debugging мережі?

**curl -v** -- HTTP-рівень:

```bash
# Детальний HTTP-запит (headers, status)
curl -v https://example.com

# HTTP/2
curl -v --http2 https://example.com

# HTTP/3
curl -v --http3 https://example.com

# Конкретні headers
curl -v -H "Authorization: Bearer token" -H "X-Debug: 1" https://api.example.com

# Тільки headers, без body
curl -I https://example.com

# Таймінги (скільки часу на DNS, TLS, TTFB і т.д.)
curl -w "@-" -o /dev/null -s https://example.com <<'EOF'
  DNS lookup:    %{time_namelookup}s
  TCP connect:   %{time_connect}s
  TLS handshake: %{time_appconnect}s
  TTFB:          %{time_starttransfer}s
  Total:         %{time_total}s
EOF

# Resolve через конкретний IP (минути DNS)
curl --resolve example.com:443:93.184.216.34 https://example.com
```

**openssl s_client** -- TLS:

```bash
# Базовий handshake
openssl s_client -connect example.com:443 -servername example.com

# Показати повний ланцюг сертифікатів
openssl s_client -connect example.com:443 -showcerts </dev/null

# Перевірити конкретну версію TLS
openssl s_client -connect example.com:443 -tls1_3

# Локальний файл сертифіката -> інформація
openssl x509 -in cert.pem -text -noout

# Згенерувати self-signed
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
    -days 365 -nodes -subj "/CN=localhost"
```

**dig / nslookup** -- DNS:

```bash
# dig (перевага -- детальний вивід)
dig example.com
dig +short example.com
dig @8.8.8.8 example.com MX
dig +trace example.com

# nslookup (простіше, але менше інформації)
nslookup example.com
nslookup -type=MX example.com 8.8.8.8

# host (компактно)
host example.com
```

**traceroute / mtr** -- шлях пакета:

```bash
# traceroute (ICMP або UDP)
traceroute example.com
traceroute -T -p 443 example.com    # TCP traceroute

# mtr (combines traceroute + ping, realtime)
mtr example.com
mtr --report --report-cycles 100 example.com

# tracepath (не потребує root)
tracepath example.com
```

**ping / telnet / nc** -- доступність:

```bash
# ICMP ping
ping example.com
ping -c 4 example.com

# TCP port check (telnet застарілий, але простий)
telnet example.com 443

# Кращий варіант -- nc (netcat)
nc -zv example.com 443
nc -zv -u 8.8.8.8 53           # UDP

# З timeout
timeout 5 nc -zv example.com 443
```

**tcpdump / Wireshark** -- packet capture:

```bash
# Інтерфейс
sudo tcpdump -i any

# Конкретний порт
sudo tcpdump -i any port 443 -n

# Конкретний host
sudo tcpdump -i any host example.com -n

# Записати у файл для Wireshark
sudo tcpdump -i any -w capture.pcap port 443

# З деталями (пакет bodies)
sudo tcpdump -i any -A -s0 port 80
```

**ss / netstat** -- сокети та з'єднання:

```bash
# Усі TCP з'єднання (ss модерніший за netstat)
ss -tanp

# Слухаючі порти
ss -tlnp

# З ім'ям процесу
sudo ss -tanp

# Статистика
ss -s
```

---

## Типові мережеві проблеми: "timeout without error", "connection refused vs no route to host", MTU, split-brain DNS.

**"Timeout without error" / запит повисає:**

Клієнт надсилає SYN і не отримує відповіді. Причини:
- **Firewall** мовчки дропає пакет (`DROP` rule, а не `REJECT`). REJECT повернув би RST.
- **Security Group** у AWS блокує ingress (SG дропає, не reject'ить)
- Сервер перевантажений, черга SYN повна (SYN flood або реальне навантаження)
- Network partition між сегментами
- Неправильний route в таблиці маршрутизації

Симптом: `curl` висить секунд 30-120 поки не спрацює client timeout. Як дебажити: `tcpdump` на клієнті (бачимо SYN, не бачимо SYN-ACK -> проблема на шляху або у сервера).

**"Connection refused" vs "No route to host" vs "Host unreachable":**

```
Connection refused       ECONNREFUSED    Сервер існує, але на цьому порту
                                         ніхто не слухає. Отримали RST.
                                         Приклад: nc -zv 127.0.0.1 9999

No route to host         EHOSTUNREACH   Немає маршруту до IP.
                                         Отримали ICMP Host Unreachable
                                         (або немає відповіді і локальна
                                         таблиця маршрутизації підтверджує).

Network unreachable      ENETUNREACH    Немає маршруту до всієї мережі.
                                         ICMP Network Unreachable.

Connection timed out     ETIMEDOUT      SYN надіслано, відповіді немає
                                         за RTO. Firewall / сервер upper.
```

**MTU issues (Maximum Transmission Unit):**

MTU -- максимальний розмір пакета на мережевому інтерфейсі. Ethernet типово 1500 байт. Якщо пакет більший:
- З `DF` flag (Don't Fragment) -> ICMP Fragmentation Needed -> sender зменшує
- Без DF -> фрагментація (старіший підхід, викликає проблеми)

**Path MTU Discovery (PMTUD)** базується на ICMP. Якщо десь по дорозі firewall блокує ICMP -> **black hole**: з'єднання встановлюється (SYN маленькі), але великі пакети (POST-и, відповіді) губляться мовчки.

Симптоми:
- TCP handshake проходить
- Малі запити (GET без параметрів) працюють
- Великі запити (POST з body, великі відповіді) таймаутять

Типовий випадок: VPN/IPSec/GRE додає свій overhead -> ефективний MTU менший за 1500. Рішення:
- Увімкнути ICMP fragmentation-needed
- Використовувати **MSS clamping** на роутері: модифікувати TCP MSS у handshake
- Знизити MTU на інтерфейсі (наприклад, 1400 замість 1500)

```bash
# Перевірити MTU
ip link show eth0
ifconfig eth0

# Знайти реальний PMTU
tracepath example.com
ping -M do -s 1472 example.com     # don't fragment, 1472+28(ICMP+IP)=1500
ping -M do -s 1500 example.com     # буде помилка якщо MTU менший
```

**Split-brain / Split-horizon DNS:**

Один і той самий домен резолвиться в різні IP залежно від того, звідки питають:
- Всередині VPC: `api.example.com` -> приватний IP `10.0.1.5`
- Ззовні: `api.example.com` -> публічний IP `203.0.113.5`

Зручно для:
- Доступ з бекенд-сервісів до API минаючи public internet (швидше, дешевше)
- Spoofing prevention (внутрішні сервіси не ходять через Internet Gateway)

Реалізація: AWS Route 53 private hosted zone + public hosted zone з однаковим ім'ям. Або внутрішній DNS-сервер з власними записами.

Проблеми:
- Дивна поведінка при траблшутингу: з ноутбука вдома бачу IP X, з bastion бачу Y -- "то чий же результат правильний?"
- Docker-контейнери на laptop можуть використовувати host DNS -> інший результат
- VPN: при підключеній VPN -- внутрішній DNS, при відключеній -- публічний. Перемикання може кешуватись

Правило дебагу: завжди вказувати, через який resolver йде запит (`dig @10.0.0.2 api.example.com`), і розуміти, що клієнт бачить не те саме, що ви.

---

## Короткий чеклист для співбесіди

- Знай різницю L4/L7, коли який LB використовується
- Поясни TCP 3-way handshake і чому FIN двосторонній
- Знай типові TCP-стани (TIME_WAIT, CLOSE_WAIT) і що вони означають
- Поясни, чому HTTP/3 швидший (QUIC over UDP, no HOL blocking)
- HTTP idempotency: які методи, навіщо Idempotency-Key
- TLS: handshake спрощено, chain of trust, CA, Let's Encrypt / ACME
- DNS: ієрархія, типи записів, чому зміни довго поширюються
- CDN: edge cache, cache invalidation, коли не допомагає
- CIDR нотація: вмієш порахувати кількість хостів у /24, /16
- Розрізняй connection refused / timeout / no route to host
- Знай базові інструменти: curl, dig, ss, tcpdump, openssl s_client
