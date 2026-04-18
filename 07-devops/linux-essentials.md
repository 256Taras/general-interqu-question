# Linux Essentials для DevOps / Backend інженера

## Як працюють процеси в Linux: fork, exec, PID, PPID?

У Linux кожен процес має унікальний ідентифікатор -- **PID** (Process ID), і батьківський ідентифікатор -- **PPID** (Parent PID). Усі процеси утворюють дерево з коренем у процесі `init` (зазвичай PID 1, це може бути `systemd` або `/sbin/init`). У контейнерах PID 1 -- це ваш процес (наприклад, `node server.js`).

Новий процес створюється через системний виклик `fork()`, який створює точну копію батьківського процесу (копія пам'яті працює через copy-on-write). Після цього дочірній процес часто викликає `exec()`, який замінює образ процесу на нову програму. Саме комбінація `fork + exec` стоїть за запуском будь-якої команди в shell.

```
┌─────────────────┐
│   bash (PID 100)│
└────────┬────────┘
         │ fork()
         ▼
┌─────────────────┐         ┌─────────────────┐
│ bash copy       │  exec() │   ls (PID 101)  │
│ (PID 101)       │────────▶│   PPID=100      │
└─────────────────┘         └─────────────────┘
```

```bash
# Показати дерево процесів поточного користувача
pstree -p $USER

# Подивитися ланцюг батьків для конкретного процесу
ps -o pid,ppid,user,cmd -p 12345

# Знайти усі процеси Node.js із сортуванням за пам'яттю
ps aux --sort=-%mem | grep -E "node|PID" | head -20

# Показати повну командну лінію (не обрізану)
ps -ef -ww | grep node

# Тільки PID за назвою процесу
pgrep -fl node

# Надіслати сигнал усім процесам із певною назвою
pkill -TERM -f "node server.js"
```

**`top` vs `htop`:** `top` -- завжди є на будь-якому Linux; `htop` -- інтерактивний, із деревом процесів (F5), пошуком (F3), сортуванням. На продакшені корисно мати `htop`, але треба бути готовим до `top` (він точно буде встановлений).

---

## Що таке zombie і orphan процеси?

**Zombie (Z)** -- це процес, який уже завершився, але його батько ще не викликав `wait()` щоб забрати його exit status. Zombie займає лише запис у таблиці процесів, не має пам'яті, але якщо їх накопичується багато -- це баг у батьківському процесі (він не reap-ить дочірні).

**Orphan** -- це процес, батько якого помер раніше за нього. Ядро переприсвоює orphan процесу PPID=1 (init/systemd), і саме init відповідає за збирання його exit-коду, коли процес завершиться.

```
До смерті батька:                Після смерті батька:
  init (1)                         init (1)
    └── parent (100)                └── child (101)  ← PPID став 1
          └── child (101)               (orphan, але це ОК)
```

```bash
# Знайти зомбі (status Z)
ps axo stat,pid,ppid,cmd | awk '$1 ~ /^Z/ { print }'

# Або коротше
ps aux | awk '$8=="Z"'

# Хто батько зомбі? Це він винен -- треба або перезапустити, або фіксити reap
ps -o pid,ppid,stat,cmd -p <zombie_pid>
```

**Чому це важливо у Docker:** якщо ви запускаєте Node.js напряму як PID 1 у контейнері, він не очікує бути init-ом і не reap-ить дочірні процеси. Саме тому існує `tini` / `--init` flag у Docker:

```bash
# Запустити контейнер з правильним init, який reap-ає zombies
docker run --init -d myapp

# Або в Dockerfile
# ENTRYPOINT ["/sbin/tini", "--", "node", "server.js"]
```

---

## SIGTERM vs SIGKILL vs SIGHUP: як робити graceful shutdown?

Сигнали -- це механізм асинхронного сповіщення процесу. Найважливіші для backend/DevOps:

| Сигнал    | Номер | Що робить                                         | Ловиться?      |
|-----------|-------|---------------------------------------------------|----------------|
| `SIGTERM` | 15    | Ввічливе прохання завершитися (за замовчуванням у `kill`) | Так            |
| `SIGKILL` | 9     | Ядро вбиває процес миттєво, без прибирання          | **Ні, ніколи**  |
| `SIGHUP`  | 1     | Термінал закрився / reload config (nginx, haproxy)  | Так            |
| `SIGINT`  | 2     | Ctrl+C у терміналі                                  | Так            |
| `SIGUSR1` | 10    | Користувацький (часто -- log rotate)               | Так            |
| `SIGSTOP` | 19    | Призупинити процес                                  | **Ні**         |
| `SIGCONT` | 18    | Продовжити                                          | Так            |

**Правильний graceful shutdown flow:**

```
docker stop / kubectl delete pod
         │
         ▼ надсилає SIGTERM
    ┌────────────┐
    │ Ваш сервіс  │  ── ловить SIGTERM
    └─────┬──────┘
          │ 1. Закрити HTTP listener (stop accepting)
          │ 2. Дочекатися поточних requests (drain)
          │ 3. Закрити DB pool, Redis, message queues
          │ 4. process.exit(0)
          ▼
  Якщо за N сек (30 у Docker, terminationGracePeriodSeconds у k8s)
  процес ще живий -- надсилається SIGKILL.
```

```bash
# Надіслати SIGTERM (за замовчуванням)
kill 12345

# SIGKILL -- крайня міра
kill -9 12345
kill -SIGKILL 12345

# Ребутнути конфіг nginx без рестарту (SIGHUP)
kill -HUP $(cat /var/run/nginx.pid)

# Trap у shell-скрипті -- ловимо сигнали й робимо cleanup
#!/bin/bash
cleanup() {
  echo "Отримано сигнал, прибираю..."
  rm -f /tmp/mylock
  exit 0
}

# Ловимо TERM, INT та EXIT (EXIT завжди спрацьовує при виході)
trap cleanup SIGTERM SIGINT EXIT

# Основна робота
sleep 1000 &
wait
```

У Node.js:

```javascript
// Приклад graceful shutdown для Express/Fastify
const server = app.listen(3000);

const shutdown = async (signal) => {
  console.log(`Отримано ${signal}, завершуюсь...`);
  server.close(() => console.log("HTTP закритий"));
  await db.end();          // закрити pool
  await redis.quit();      // закрити Redis
  process.exit(0);
};

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

---

## Файлова система: inodes, hard links vs symbolic links

У Linux файл -- це не ім'я у директорії, а **inode**. Inode зберігає метадані (розмір, права, timestamps, номери блоків даних), але **не ім'я**. Ім'я живе у директорії, яка є мапінгом `name → inode`.

```
┌──────────────────┐           ┌──────────────────┐
│ Dir: /home/taras │           │     inode 42     │
│                  │           │  size: 1024       │
│  config.json ───▶│──────────▶│  blocks: [...]    │
│  backup.json ───▶│──────────▶│  nlink: 2         │
└──────────────────┘           └──────────────────┘
 (два імені, один файл -- hard link)
```

**Hard link** -- ще одне ім'я для того ж inode. Файл видаляється тільки коли всі hard links пропали І жоден процес його не відкрив (nlink=0 AND ref_count=0).

**Symbolic link (symlink)** -- окремий файл, у якого вміст -- це шлях. Може вказувати на неіснуючий файл, на інший файловий пристрій, на директорію.

```bash
# Hard link -- той самий inode (перевіряємо через ls -i)
ls -li original.txt
# 12345 -rw-r--r-- 1 user user 100 ... original.txt
ln original.txt hardlink.txt
ls -li original.txt hardlink.txt
# 12345 -rw-r--r-- 2 user user 100 ... original.txt   ← nlink=2
# 12345 -rw-r--r-- 2 user user 100 ... hardlink.txt   ← той самий inode

# Symbolic link
ln -s /etc/nginx/sites-available/site.conf /etc/nginx/sites-enabled/site.conf

# Показати куди вказує symlink
readlink -f /etc/nginx/sites-enabled/site.conf

# Знайти усі symlinks у директорії
find /etc -type l -ls
```

**inodes можуть закінчитися навіть якщо місце є** -- поширений баг у мейл-серверах чи кешах із мільйонами дрібних файлів:

```bash
# Скільки inodes використано / вільно
df -i

# Knowing what eats inodes: рахуємо файли по директоріях
du --inodes -s /var/* 2>/dev/null | sort -n
```

**Чому файл після `rm` може продовжувати займати місце:** якщо процес тримає файл відкритим, inode живе. Типово для лог-файлів, які ротейтили через `rm` замість `logrotate`.

```bash
# Знайти "deleted but still held" файли
lsof +L1 | head

# Результат: node 1234 ... /var/log/app.log (deleted)
# Правильний fix -- перезапустити процес або reopen логи (SIGUSR1/SIGHUP)
```

---

## Permissions: chmod, chown, octal notation, setuid/setgid/sticky

Класичні Unix-дозволи: три групи (owner / group / other) × три дії (read=4, write=2, execute=1). Octal -- сума.

```
rwxr-xr-- → 111 101 100 → 7 5 4 → 754
```

Типові значення:
- `644` -- файли для читання (rw-r--r--)
- `755` -- виконувані / директорії (rwxr-xr-x)
- `600` -- приватні (SSH keys!) (rw-------)
- `700` -- приватна директорія (rwx------)

```bash
# Змінити права
chmod 644 config.json
chmod u+x script.sh         # додати execute власнику
chmod -R g-w /var/www        # рекурсивно прибрати write для групи

# Власник / група
chown user:group file
chown -R www-data:www-data /var/www

# Подивитися права у числовому вигляді
stat -c "%a %U:%G %n" ~/.ssh/id_rsa
# 600 taras:taras /home/taras/.ssh/id_rsa
```

**Special bits:**

| Біт        | Octal | Де ставиться      | Що робить                                                     |
|------------|-------|-------------------|---------------------------------------------------------------|
| `setuid`   | 4000  | Виконувані файли   | Запуск із правами власника (напр. `/usr/bin/passwd`)           |
| `setgid`   | 2000  | Файли / директорії | Файли -- з правами групи; директорії -- нові файли наслідують групу |
| `sticky`   | 1000  | Директорії         | Тільки власник файлу може його видалити (напр. `/tmp`)         |

```bash
# setuid на passwd (ядро дозволяє змінювати /etc/shadow)
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd  ← 's' = setuid

# setgid на директорії -- корисно для shared-директорій команди
chmod 2775 /srv/shared
# Тепер новостворені файли матимуть групу директорії

# Sticky bit на /tmp
ls -ld /tmp
# drwxrwxrwt ... /tmp  ← 't' = sticky

# Знайти всі setuid бінарники (аудит безпеки!)
find / -perm -4000 -type f 2>/dev/null
```

**ACL (Access Control Lists)** -- коли трьох груп мало. Напр., додати доступ одному user без зміни групи:

```bash
# Додати acl для user "deploy" (read+execute)
setfacl -m u:deploy:rx /var/log/app/

# Подивитися ACL
getfacl /var/log/app/

# Видалити ACL
setfacl -x u:deploy /var/log/app/

# ACL за замовчуванням для нових файлів у директорії
setfacl -d -m u:deploy:rx /var/log/app/
```

Примітка: `ls -l` показує `+` у кінці прав, якщо на файлі є ACL: `drwxr-x---+`.

---

## Networking: як подивитися з'єднання, DNS, маршрути

**Хто слухає порт / хто куди підключений:**

```bash
# ss (socket statistics) -- сучасна заміна netstat, швидша
ss -tulnp
#  -t TCP, -u UDP, -l listening, -n numeric, -p process

# Усі встановлені з'єднання на порт 443
ss -tn state established '( dport = :443 or sport = :443 )'

# Скільки з'єднань у стані TIME_WAIT (часта причина "Cannot bind")
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c

# lsof -- "який процес тримає цей файл/порт"
lsof -i :3000              # хто слухає 3000
lsof -iTCP -sTCP:LISTEN    # усі TCP listeners
lsof -p 12345              # всі файли процесу (сокети, файли, pipes)

# netstat старий, але все ще часто в docs
netstat -tulnp
```

**DNS:**

```bash
# dig -- повний вивід, включно з серверами та TTL
dig example.com
dig +short example.com           # тільки IP
dig @8.8.8.8 example.com         # питати конкретний DNS
dig example.com MX               # MX-записи
dig example.com +trace           # пройти всю DNS-ієрархію (root → TLD → authoritative)

# nslookup -- простіший, але менш детальний
nslookup example.com

# Що каже /etc/resolv.conf (який DNS в системі)
cat /etc/resolv.conf

# hosts-файл (перекриває DNS)
cat /etc/hosts
```

**Маршрути й інтерфейси:**

```bash
# Сучасні команди (iproute2)
ip addr                 # інтерфейси та IP
ip -br addr            # коротко
ip route               # таблиця маршрутизації
ip route get 8.8.8.8   # через який маршрут піде пакет

# Перевірити шлях до хосту
traceroute example.com
mtr example.com         # комбінація ping + traceroute, дуже корисно для debug

# Перевірити HTTP-запит із купою деталей (TLS handshake, timings)
curl -v https://api.example.com/health
curl -w "@-" -o /dev/null -s https://api.example.com <<'EOF'
    dns:    %{time_namelookup}s
    connect:%{time_connect}s
    tls:    %{time_appconnect}s
    ttfb:   %{time_starttransfer}s
    total:  %{time_total}s
EOF
```

**tcpdump -- коли треба точно подивитися, що йде по мережі:**

```bash
# Всі пакети на порт 3000
sudo tcpdump -i any -nn 'port 3000'

# HTTP-запити до зовнішнього API (побачимо тіло)
sudo tcpdump -i any -A -s0 'tcp port 80'

# Записати у файл для аналізу у Wireshark
sudo tcpdump -i eth0 -w /tmp/capture.pcap 'host 10.0.0.5'
```

---

## Firewall: iptables, nftables, ufw. Що робить Docker із iptables?

Linux-фаєрвол працює на рівні ядра (netfilter). Утиліти:

- **iptables** -- класична, ще всюди, але deprecated
- **nftables** (`nft`) -- сучасна заміна iptables, єдиний синтаксис для IPv4/IPv6
- **ufw** -- user-friendly обгортка над iptables (Ubuntu)
- **firewalld** -- те ж саме для RHEL/CentOS

```bash
# ufw -- простий спосіб на Ubuntu
sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw deny from 1.2.3.4
sudo ufw enable
sudo ufw status numbered

# iptables -- подивитися поточні правила
sudo iptables -L -n -v            # filter table (за замовч.)
sudo iptables -t nat -L -n -v     # NAT
sudo iptables -S                  # у форматі команд

# Заблокувати IP
sudo iptables -I INPUT -s 1.2.3.4 -j DROP

# Зберегти правила на RHEL/Debian (інакше зникнуть після ребуту)
sudo iptables-save > /etc/iptables/rules.v4
```

**Docker і iptables:** Docker активно модифікує iptables, додаючи ланцюжки `DOCKER`, `DOCKER-USER`, `DOCKER-ISOLATION`. Коли ви пишете `-p 3000:3000`, Docker додає DNAT-правило у NAT-таблицю, яке перенаправляє трафік на контейнер.

```
  Зовнішній запит :3000
         │
         ▼
  iptables PREROUTING (nat)
    DNAT → 172.17.0.5:3000
         │
         ▼
  Docker bridge → контейнер
```

**Важливо:** якщо у вас є ufw, він **не блокує** трафік до контейнерів за замовчуванням, бо Docker обходить його через низько-рівневі DNAT. Використовуйте ланцюжок `DOCKER-USER` для власних правил:

```bash
# Дозволити доступ до Docker-контейнерів лише з певної мережі
sudo iptables -I DOCKER-USER -i eth0 ! -s 10.0.0.0/8 -j DROP
```

---

## Resource limits: ulimit, systemd, cgroups

**ulimit** -- ліміти на процес у рамках сесії. Дві категорії: soft (поточний) і hard (максимум, до якого soft можна піднімати).

```bash
# Подивитися всі ліміти
ulimit -a

# Найважливіше для бекенду:
ulimit -n             # open files (сокети теж!)
ulimit -u             # max user processes

# Типова проблема: "Too many open files" -- Node.js впав під навантаженням
# Підняти open files до 65535
ulimit -n 65535
```

**Постійні ліміти через `/etc/security/limits.conf`:**

```
# user / group / *   type   item    value
*                    soft   nofile  65535
*                    hard   nofile  65535
node                 soft   nproc   4096
```

**systemd перекриває ulimit -- якщо ваш сервіс запускається через systemd, limits.conf не застосовується!** Треба прописувати у unit-файлі:

```ini
[Service]
LimitNOFILE=65535
LimitNPROC=4096
# Обмеження пам'яті -- викличе OOM-kill при перевищенні
MemoryMax=2G
# CPU quota (200% = 2 ядра)
CPUQuota=200%
```

**cgroups (control groups)** -- механізм ядра, який обмежує ресурси (CPU, memory, I/O, мережа) для групи процесів. Це фундамент контейнерів. Docker/Kubernetes використовують cgroups під капотом.

```bash
# Подивитися cgroup процесу
cat /proc/self/cgroup

# Docker контейнер: CPU та memory обмеження
docker run --cpus=1.5 --memory=512m myapp

# Перевірити ліміти поточного cgroup
cat /sys/fs/cgroup/memory.max      # cgroup v2
cat /sys/fs/cgroup/cpu.max

# Поточне споживання
cat /sys/fs/cgroup/memory.current
```

---

## Пошук і логи: find, grep, awk, journalctl, tail

**find -- швидкий пошук файлів за критеріями:**

```bash
# Файли, змінені за останні 10 хвилин
find /var/log -mmin -10 -type f

# Файли більші за 100MB
find / -size +100M -type f 2>/dev/null

# Видалити логи старші за 30 днів (sec-safe -- спочатку -print!)
find /var/log/app -name "*.log" -mtime +30 -print
find /var/log/app -name "*.log" -mtime +30 -delete

# Виконати команду на кожен файл
find . -name "*.js" -exec wc -l {} +
```

**grep -- пошук у файлах з regex:**

```bash
# Recursive пошук у коді
grep -rn "TODO" src/

# Case-insensitive, з контекстом (3 рядки до і після)
grep -i -C 3 "error" /var/log/app.log

# Виключити певний патерн
grep -v "healthcheck" access.log

# Extended regex (підтримка {n,m}, |, +)
grep -E "error|warn|fatal" app.log

# Підрахувати кількість збігів по файлах
grep -c "5[0-9]{2}" access.log
```

**awk -- маніпуляція таблицями/колонками:**

```bash
# Вивести 7-му колонку (URL з nginx access log)
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head

# Сума розмірів відповідей (колонка 10 у nginx)
awk '{sum+=$10} END {print sum/1024/1024 " MB"}' access.log

# Рядки, де статус >= 500
awk '$9 >= 500 {print}' access.log

# CSV -- роздільник -F
awk -F',' '$3 > 100 {print $1, $3}' data.csv
```

**journalctl -- логи systemd-сервісів:**

```bash
# Логи сервісу
journalctl -u myapp.service

# Останні 100 рядків + follow
journalctl -u myapp.service -n 100 -f

# За часом
journalctl -u myapp --since "2 hours ago"
journalctl -u myapp --since "2026-04-17 10:00" --until "2026-04-17 12:00"

# Тільки помилки (priority >= err)
journalctl -u myapp -p err

# Логи з попереднього ребуту
journalctl -u myapp -b -1

# Дисковий розмір журналу
journalctl --disk-usage
```

**tail / less для великих файлів:**

```bash
# Slідкувати за файлом
tail -f /var/log/app.log

# Слідкувати навіть якщо файл ротейтиться
tail -F /var/log/app.log

# less із live-режимом (Shift+F щоб увімкнути, Ctrl+C щоб вийти в normal)
less +F /var/log/app.log

# Подивитися великий файл без завантаження у пам'ять
less /var/log/huge.log    # / для пошуку, n/N для наступного/попереднього
```

---

## Диск та I/O: df, du, iostat, LVM

```bash
# Вільне місце на файлових системах
df -h              # human readable
df -hT             # + тип ФС
df -i              # inodes

# Що займає місце у директорії (top-level)
du -h --max-depth=1 /var | sort -h

# Top-10 найбільших директорій
du -h /var 2>/dev/null | sort -rh | head -10

# Розмір конкретної директорії сумарно
du -sh /var/log

# ncdu -- інтерактивний (треба встановити окремо)
ncdu /
```

**I/O-навантаження:**

```bash
# iostat -- utilization, await, IOPS по дисках
# -x extended, 2 -- інтервал, 5 -- кількість samples
iostat -xz 2 5

# Ключові колонки:
# %util     -- наскільки диск зайнятий (>80% = bottleneck)
# await     -- середній час I/O в мс
# r/s, w/s  -- IOPS
# rkB/s     -- throughput

# iotop -- топ процесів за I/O (потрібен root)
sudo iotop -oPa

# pidstat -- I/O по процесах
pidstat -d 1
```

**LVM (Logical Volume Manager)** -- гнучке управління дисками. Замість фіксованих партицій -- пул фізичних томів → volume group → logical volumes, які можна розширювати на льоту.

```
  Physical Volumes (PV)    /dev/sdb, /dev/sdc
         │
         ▼
  Volume Group (VG)        vg0 (об'єднаний простір)
         │
         ▼
  Logical Volumes (LV)     /dev/vg0/data, /dev/vg0/logs
```

```bash
# Огляд
sudo pvs                    # physical volumes
sudo vgs                    # volume groups
sudo lvs                    # logical volumes

# Розширити LV (додати 10G) -- БЕЗ ДОВНТАЙМУ
sudo lvextend -L +10G /dev/vg0/data
sudo resize2fs /dev/vg0/data     # ext4
# або для xfs:
sudo xfs_growfs /mnt/data
```

---

## SSH: ключі, ProxyJump, tunneling

**Ключі -- основа:**

```bash
# Згенерувати ed25519 (краще за RSA)
ssh-keygen -t ed25519 -C "taras@work" -f ~/.ssh/id_ed25519

# Розкласти публічний ключ на сервер
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Перевірити fingerprint
ssh-keygen -lf ~/.ssh/id_ed25519.pub
```

**authorized_keys** -- файл `~/.ssh/authorized_keys` на сервері зі списком публічних ключів, яким дозволено вхід. Права **критично важливі:**

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
# SSH відмовиться використовувати ключ, якщо права занадто ліберальні
```

**Bastion host / ProxyJump** -- коли прод-сервери не доступні напряму:

```
Ваш ноут ──SSH──▶ bastion.example.com ──SSH──▶ prod-db-01 (в приватній мережі)
```

```bash
# Старий спосіб: два хопи вручну
ssh user@bastion ssh user@prod-db

# Правильний: -J (ProxyJump)
ssh -J user@bastion user@prod-db-01

# У ~/.ssh/config -- найзручніше
cat >> ~/.ssh/config <<'EOF'
Host bastion
  HostName bastion.example.com
  User taras

Host prod-*
  ProxyJump bastion
  User deploy

# тепер працює: ssh prod-db-01
EOF
```

**Agent forwarding -- небезпечно! Замість нього використовуйте ProxyJump.** Agent forwarding (`-A`) дає root-у на bastion доступ до вашого агента.

**SSH tunneling:**

```bash
# Local forward (-L): на локальному порту 5433 → віддалений prod-db:5432
# Зручно для debug-у продакшн-БД через psql локально
ssh -L 5433:prod-db.internal:5432 user@bastion
# Тепер: psql -h localhost -p 5433

# Remote forward (-R): віддалений сервер бачить ваш локальний порт
# Напр., webhook із публічного сервера на ваш dev-локальний :3000
ssh -R 8080:localhost:3000 user@public-server

# SOCKS proxy (-D): динамічний -- проксі для цілого браузера
ssh -D 1080 user@bastion
# У браузері: SOCKS5 localhost:1080 -- увесь трафік через bastion
```

---

## systemd: unit files, сервіси, таймери

**systemd** -- init-система у більшості сучасних дистрибутивів. Керує сервісами, монтуванням, сокетами, таймерами.

**Мінімальний unit-файл для Node.js сервісу:**

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node.js application
# Стартувати після мережі та Postgres
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
Type=simple                       # процес не forк-ається, PID 1 -- наш
User=node
Group=node
WorkingDirectory=/opt/myapp
Environment="NODE_ENV=production"
EnvironmentFile=-/etc/myapp/env   # "-" = ОК якщо файла немає
ExecStart=/usr/bin/node server.js
ExecReload=/bin/kill -HUP $MAINPID

# Перезапуск при падінні
Restart=on-failure
RestartSec=5s

# Ліміти (перекривають limits.conf!)
LimitNOFILE=65535
MemoryMax=1G

# Безпека
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/myapp /var/lib/myapp

[Install]
WantedBy=multi-user.target
```

```bash
# Після змін перечитати
sudo systemctl daemon-reload

# Керувати сервісом
sudo systemctl start myapp
sudo systemctl stop myapp
sudo systemctl restart myapp
sudo systemctl reload myapp          # SIGHUP через ExecReload
sudo systemctl status myapp
sudo systemctl enable myapp          # автозапуск при boot
sudo systemctl enable --now myapp    # enable + start одним рухом

# Логи (systemd автоматично захоплює stdout/stderr)
journalctl -u myapp -f
```

**Таймери -- кращі за cron:**

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup DB

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup at 3am

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true                 # запустити, якщо пропустили (comp був вимкнений)

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers           # всі активні таймери
```

---

## Troubleshooting високого навантаження: USE method

**USE (Utilization, Saturation, Errors) від Brendan Gregg** -- систематичний підхід. Для кожного ресурсу перевіряємо три показники.

```
                     Первинна діагностика
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
      uptime               vmstat 1             dmesg -T | tail
  (load average)       (CPU/mem/swap)       (помилки ядра/OOM)
```

**1. CPU -- високе споживання:**

```bash
# Load average -- скільки процесів хочуть CPU (1, 5, 15 хв)
uptime
# load average: 2.15, 1.85, 1.60
# Правило: якщо load > кількість ядер → система saturated

# Скільки ядер
nproc

# Який процес їсть CPU?
top          # натиснути P -- sort по CPU
htop

# Детально по ядрах
mpstat -P ALL 1

# vmstat: колонки us (user), sy (system), wa (iowait), id (idle)
vmstat 1

# Якщо sy >> us -- щось дивне в kernel space
# Якщо wa високий -- диск bottleneck (див. нижче)
```

**2. Memory -- OOM ризик:**

```bash
# Free -- але важливо розуміти "available"
free -h
#               total  used  free  shared buff/cache available
# Mem:           16G    8G   1G     500M    7G          7.5G
# "available" -- реально доступно без swap

# Топ по пам'яті
ps aux --sort=-%rss | head

# Чи був OOM-kill?
dmesg -T | grep -i "killed process"
journalctl -k | grep -i oom

# Swap активний?
vmstat 1     # si/so колонки -- swap in/out
# Якщо не нульові -- система свопить, це ДУЖЕ повільно
```

**3. Load average високий, а CPU idle?** Значить процеси у стані `D` (uninterruptible sleep, зазвичай диск).

```bash
ps axo stat,pid,cmd | awk '$1 ~ /D/'

# Iowait у vmstat
vmstat 1
# wa: %

# Який процес генерує I/O?
sudo iotop -oPa
```

**4. High iowait -- диск bottleneck:**

```bash
iostat -xz 1

# %util = 100 -- диск завантажений на 100%
# await >> svctm -- черга запитів на диск
```

**Чекліст для production-інциденту "сайт тормозить":**

```
1. uptime                       → load average
2. free -h                      → чи є пам'ять
3. df -h                        → чи не переповнений диск
4. iostat -xz 1 3               → диск
5. ss -s                        → скільки з'єднань / TIME_WAIT
6. journalctl -p err -n 100     → помилки в системі
7. docker ps / systemctl status → чи всі сервіси живі
8. curl -v localhost:port/health → чи відповідає сервіс
```

---

## Що таке /proc і /sys? Як читати стан процесу?

`/proc` і `/sys` -- віртуальні файлові системи, які ядро експонує у вигляді файлів. Це не справжні файли, а інтерфейс до структур ядра.

- **`/proc`** -- інформація про процеси та систему (legacy, починаючи з Linux 1.x)
- **`/sys`** (sysfs) -- структурований інтерфейс до пристроїв і cgroups (новіший)

```bash
# /proc/PID -- усе про процес
ls /proc/$(pgrep -f "node server.js" | head -1)/

# Найкорисніше:
cat /proc/PID/status         # memory, threads, UID, capabilities
cat /proc/PID/cmdline        # повна командна лінія (null-separated!)
ls -l /proc/PID/cwd          # working directory (symlink)
ls -l /proc/PID/exe          # бінарник (symlink)
ls /proc/PID/fd              # відкриті файлові дескриптори
cat /proc/PID/limits         # ulimits саме цього процесу
cat /proc/PID/io             # скільки прочитано/записано
cat /proc/PID/stack          # kernel stack -- де процес "застряг"
cat /proc/PID/environ        # env vars (null-separated)
```

```bash
# Приклад: скільки open file descriptors у процесу зараз
ls /proc/12345/fd | wc -l
cat /proc/12345/limits | grep "open files"

# Який сервіс під конкретним PID?
readlink /proc/12345/exe
cat /proc/12345/cmdline | tr '\0' ' '; echo

# Що читає/пише процес прямо зараз? (strace -- тут не treat файлово, але /proc допомагає)
sudo strace -p 12345 -e trace=openat,read,write
```

**Глобальні системні параметри:**

```bash
# CPU
cat /proc/cpuinfo
cat /proc/loadavg

# Пам'ять
cat /proc/meminfo

# Мережеві з'єднання (звідси ss і читає)
cat /proc/net/tcp
cat /proc/net/sockstat

# Версія ядра
cat /proc/version

# Час роботи, idle time
cat /proc/uptime
```

**`/sys` -- більш структурований, використовується для cgroups v2 і пристроїв:**

```bash
# Блочні пристрої
ls /sys/block/
cat /sys/block/sda/queue/scheduler    # який I/O scheduler

# cgroups v2 -- ліміти контейнера
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/cpu.max

# Мережеві інтерфейси
ls /sys/class/net/
cat /sys/class/net/eth0/speed         # швидкість лінку
```

**`sysctl` -- змінити параметри ядра:**

```bash
# Подивитися
sysctl net.ipv4.tcp_tw_reuse
sysctl -a | grep somaxconn

# Змінити на льоту (до ребуту)
sudo sysctl -w net.core.somaxconn=4096

# Постійно -- у /etc/sysctl.d/99-custom.conf
# net.core.somaxconn = 4096
# net.ipv4.tcp_tw_reuse = 1
sudo sysctl -p /etc/sysctl.d/99-custom.conf
```

---

## Bonus: корисні однолайнери для прод-debug

```bash
# Топ-10 IP за кількістю запитів у nginx log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Топ-10 URL, що повертають 5xx
awk '$9 ~ /^5/ {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# P95 latency (якщо 11-та колонка -- response time)
awk '{print $11}' access.log | sort -n | awk 'BEGIN{c=0} {a[c++]=$1} END{print a[int(c*0.95)]}'

# Швидкий check: чи відкриті критичні порти
for port in 22 80 443 5432; do
  nc -zv -w 2 example.com $port 2>&1 | grep -E "succeeded|failed"
done

# Скільки пам'яті реально їсть Node (без shared)
ps -o pid,rss,cmd -p $(pgrep node) | awk 'NR>1 {sum+=$2} END {print sum/1024 " MB"}'

# Показати розмір кожного контейнера Docker
docker ps --size --format "table {{.Names}}\t{{.Size}}"

# Знайти процеси, що відкривають певний файл
sudo fuser -v /var/log/app.log
sudo lsof /var/log/app.log
```

---

## Підсумкова шпаргалка: що запам'ятати

| Задача                          | Команда                                      |
|---------------------------------|----------------------------------------------|
| Хто слухає порт?                | `ss -tulnp` або `lsof -i :PORT`              |
| Хто їсть CPU?                   | `top` / `htop`, `pidstat 1`                  |
| Хто їсть пам'ять?               | `ps aux --sort=-%rss \| head`                |
| Хто їсть диск (I/O)?            | `iotop -oPa`, `iostat -xz 1`                 |
| Хто їсть місце?                 | `du -h --max-depth=1 \| sort -h`             |
| Чи OOM-killed?                  | `dmesg -T \| grep -i oom`                    |
| Логи сервісу                    | `journalctl -u NAME -f`                      |
| Подивитися trace              | `strace -p PID` / `tcpdump`                  |
| Graceful reload                 | `kill -HUP PID` / `systemctl reload`         |
| Перевірити DNS                  | `dig +short domain`                           |
| Перевірити HTTP timing          | `curl -w "%{time_total}" -o /dev/null -s URL` |
| Тунель до прод-БД               | `ssh -L 5433:db:5432 bastion`                 |
