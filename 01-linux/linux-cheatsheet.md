https://www.geeksforgeeks.org/linux-unix/linux-commands-cheat-sheet/
> Легенда: `<svc>` — имя сервиса, угловые скобки не печатаем.

---

## Сервисы и логи

```bash
systemctl status <svc>
# status — показать состояние сервиса (активен, PID, последние логи)

sudo systemctl restart <svc>
# restart — перезапустить (если завис — это первое, что пробуем)

sudo systemctl enable --now <svc>
# enable — включить автозапуск при загрузке
# --now  — сразу запустить (не ждать ребута)

journalctl -u <svc> -n 200 -f
# -u <svc>   — логи конкретного юнита (сервиса)
# -n 200     — показать последние 200 строк
# -f         — “follow”, следить в реальном времени (как tail -f)

journalctl -p err -S -2h
# -p err     — только сообщения уровня error и выше (err, crit, alert, emerg)
# -S -2h     — “since”, за последние 2 часа (можно -S "2025-10-05 08:00")
```

---

## Нагрузка / память / диск

```bash
top      # (или htop/btop)
# топ-процессы по CPU/памяти. Стрелками/клавишами меняешь сортировку.

ps aux --sort=-%mem | head
# a — все процессы
# u — с именами пользователей
# x — без привязки к TTY (демоны)
# --sort=-%mem — сортировка по памяти, по убыванию

iostat -xz 1 3
# -x  — расширенная статистика по устройствам
# -z  — не показывать устройства без активности (нулевые)
# 1 3 — каждые 1 секунду, вывести 3 раза (интервал и число итераций)

free -h
# -h — human readable (М/ГБ)

uptime
# нагрузка (load average) за 1/5/15 минут
```

---

## Диски и ФС

```bash
lsblk -f
# -f — сразу показать ФС, метки, UUID

df -hT
# -h — удобно читаемые размеры
# -T — тип файловой системы (ext4/xfs и т.п.)

du -sh * | sort -h
# -s — суммарно по каждому элементу каталога (без проваливания внутрь)
# -h — human readable
# sort -h — числовая сортировка по человеку-читабельным единицам

sudo fdisk -l
# -l — перечислить таблицы разделов (какие диски/разделы видит система)

sudo mkfs.ext4 /dev/sdb1
# создать файловую систему ext4 на разделе (ОСТОРОЖНО: затирает данные)

sudo mkdir -p /data && sudo mount /dev/sdb1 /data
# -p — создать все недостающие каталоги в пути

sudo blkid /dev/sdb1
# показать UUID — пригодится для fstab

echo 'UUID=<uuid> /data ext4 defaults 0 2' | sudo tee -a /etc/fstab && sudo mount -a
# defaults — стандартные опции монтирования
# последняя “2” — порядок проверки fsck (0 — не проверять)
# mount -a — примонтировать всё из fstab не ребутясь
```

---

## Поиск и правки

```bash
grep -RIn "ERROR" /var/log
# -R — рекурсивно по подкаталогам
# -I — пропускать бинарные файлы
# -n — показывать номер строки

sed -i 's/old/new/g' /etc/app.conf
# -i — править файл “на месте”
# s/old/new/g — заменить “old” на “new”, g — глобально в строке

awk -F, '{sum+=$2} END{print sum}' data.csv
# -F, — разделитель полей — запятая
# $2 — второе поле; END{print sum} — вывести сумму после прохода
```

---

## Права / владельцы / ACL

```bash
ls -lAh
# -l — длинный формат (права/владелец/группа)
# -A — все, кроме . и ..
# -h — human readable для размеров

sudo chown user:group file
# сменить владельца и группу

chmod 0644 file
# 0 — спецбиты; 6 (rw-) для владельца; 4 (r--) для группы; 4 (r--) для остальных

getfacl file
# показать ACL (доп. права помимо chmod)

setfacl -m u:alice:rwx /srv/share
# -m — изменить/добавить правило
# u:alice:rwx — пользователю alice дать rwx на каталог
```

По-простому:

- **`chmod`** — базовые права Unix.
    - Управляет: `rwx` для **owner / group / others** + спецбиты (`setuid/setgid/sticky`).
    - Пример: `chmod 2770 dir` → owner:rwx, group:rwx, others:—; **setgid** = новые файлы унаследуют группу.
    - Ограничение: **никаких адресных правил**; новые файлы всё равно режет **umask**.
- **`setfacl`** — расширенные права (ACL).
    - Может выдавать права **конкретным юзерам/группам**: `u:vasya:r-x`, `g:devs:rwx`.
    - Есть **default ACL** для наследования внутри каталогов: `setfacl -dm g:devs:rwx dir` — шаблон прав на всё новое.
    - Важный нюанс: действует **ACL MASK** (как верхняя планка для всех group-like записей). `chmod g=...` меняет MASK и может “подрезать” ACL.

Когда что брать:
- **Просто общая папка** и дружелюбный `umask 002` → хватит `chmod 2770 dir`.
- **Гарантированно одинаковые права на всё внутри**, независимо от umask и автора → `chmod 2770 dir && setfacl -dm g:GROUP:rwx dir`.

|Восьмеричная цифра|Биты|rwx|Что даёт (операции)|
|--:|:-:|:-:|:--|
|7|4 + 2 + 1|rwx|чтение, запись и исполнение|
|6|4 + 2|rw-|чтение и запись|
|5|4 + 1|r-x|чтение и исполнение|
|4|4|r--|только чтение|
|3|2 + 1|-wx|запись и исполнение|
|2|2|-w-|только запись|
|1|1|--x|только исполнение|
|0|0|---|нет доступа|

Перевод заметки:
- Команда `stat` может показать права в **восьмеричном** виде.  
```bash
stat -c %a findPhoneNumbers.sh
    Вывод: 754.
```

Расшифровка `754`:
- владелец: **7** → `rwx`
- группа: **5** → `r-x`
- прочие: **4** → `r--`

Мини-шпаргалка:
```bash
# Базовые права
chmod 2770 dir           # setgid + rwx,rwx,---
# ACL на каталог
setfacl -m g:devs:rwx dir
# Default ACL (наследование)
setfacl -dm g:devs:rwx dir
# Смотреть / снять ACL
getfacl dir
setfacl -bk dir          # b: убрать обычные, k: убрать default
```
---

## Сеть (ip/ss/DNS/HTTP)

```bash
ip a
# адреса интерфейсов

ip r
# таблица маршрутизации

ss -tulpn
# -t — TCP
# -u — UDP
# -l — только слушающие
# -p — показать процесс/PID
# -n — не пытаться резолвить имена (быстрее, чистые порты/IP)

ss -tpn dst :443
# фильтрация по TCP-соединениям, у которых dest port = 443

ping -c3 1.1.1.1
# -c3 — отправить 3 пакета и остановиться

traceroute 8.8.8.8
# маршрут до узла по hop-ам

curl -I https://example.com
# -I — HEAD-запрос (только заголовки: код ответа, Location и т.д.)

dig +short A example.com @1.1.1.1
# +short — короткий вывод (только ответы)
# A — тип записи (IPv4)
# @1.1.1.1 — опрашивать именно этот DNS-сервер
```

---

## Время / DNS-система

```bash
timedatectl status
# текущее время, timezone, синхронизация NTP

timedatectl list-timezones
# Лист timezone
sudo timedatectl set-timezone Europe/Moscow
# сменить TZ (подставь свой)

resolvectl status
# какой DNS сейчас активен, для каких интерфейсов
```

|Что|Ubuntu (18.04+)|RHEL 8/9|
|---|---|---|
|По умолчанию|**systemd-resolved ВКЛ**|**systemd-resolved ВЫКЛ**|
|`/etc/resolv.conf`|symlink → `/run/systemd/resolve/stub-resolv.conf` (127.0.0.53)|чаще **файл/линк, управляет NetworkManager**|
|Кто рулит DNS|`systemd-resolved`|`NetworkManager` (или dhclient)|
|Править DNS быстро|`resolvectl dns eth0 1.1.1.1`|обычно через `nmcli` или включить resolved|
|Net config|**netplan** (server) / NetworkManager (desktop)|**NetworkManager** (nmcli, keyfiles)|
|Включить DoT/DNSSEC|в `/etc/systemd/resolved.conf`|то же, но **сначала включи resolved**|

#### Включить `systemd-resolved` на RHEL (если хочешь “как в Ubuntu”)

```bash
# 1) Попросить NM использовать resolved
echo -e "[main]\ndns=systemd-resolved" | sudo tee /etc/NetworkManager/conf.d/00-use-resolved.conf
sudo systemctl restart NetworkManager

# 2) Запустить resolved
sudo systemctl enable --now systemd-resolved

# 3) Отдать /etc/resolv.conf под resolved
sudo rm -f /etc/resolv.conf
sudo ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
# (можно /run/systemd/resolve/resolv.conf, но stub — стандарт для локального 127.0.0.53)

# 4) Проверка
resolvectl status
```

#### Настройка фич (одинаково в обеих)

```ini
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1 9.9.9.9
FallbackDNS=8.8.8.8 1.0.0.1
LLMNR=no
MulticastDNS=no
DNSOverTLS=yes
# DNSSEC=yes   # если сеть/резолвер поддерживают
```

```bash
sudo systemctl restart systemd-resolved
```

#### Быстрые команды (везде одинаковы)

```bash
resolvectl status
resolvectl dns eth0 1.1.1.1 9.9.9.9
resolvectl domain eth0 ~corp.local
resolvectl query example.com
resolvectl flush-caches
```
###### **Итого:** на Ubuntu уже “всё готово”; на RHEL сначала переключи NetworkManager на `systemd-resolved` и сделай symlink для `/etc/resolv.conf`. Потом всё одинаково.
---

## Firewall и NAT

### nftables (современный вариант)

```bash
sudo nft list ruleset
# показать текущие таблицы/цепочки/правила

sudo nft add table inet filter
# создать таблицу "filter" в семействе inet (общая для IPv4/IPv6)

sudo nft add chain inet filter input { type filter hook input priority 0; policy drop; }
# chain input — обрабатывает входящие пакеты на локальную машину
# hook input — точка “врезки” в стек
# policy drop — по умолчанию всё дропаем

sudo nft add rule inet filter input ct state established,related accept
# разрешить установленные/связанные соединения (иначе сломаешь всё)

sudo nft add rule inet filter input iif lo accept
# iif lo — разрешить loopback (localhost)

sudo nft add rule inet filter input tcp dport {22,80,443} accept
# открыть нужные TCP-порты разом
```

### iptables (классика/нат)

```bash
sudo iptables -L -n -v
# -L — показать правила
# -n — не резолвить имена (быстрее/чище)
# -v — подробная статистика

# DNAT: 1.2.3.4:80 -> 10.0.0.10:8080
sudo iptables -t nat -A PREROUTING -d 1.2.3.4 -p tcp --dport 80 \
  -j DNAT --to-destination 10.0.0.10:8080
# -t nat        — таблица NAT
# -A PREROUTING — цепочка до маршрутизации (входящий трафик)
# -d            — целевой публичный адрес
# -p tcp --dport 80 — протокол/порт
# -j DNAT --to-destination — куда пробрасывать внутрь

# SNAT: исходящий с 10.0.0.0/24 через 1.2.3.4
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 \
  -j SNAT --to-source 1.2.3.4
# -A POSTROUTING — цепочка после маршрутизации (исходящий трафик)
# -s            — исходная подсеть
# -j SNAT --to-source — подменить исходный адрес
```

> Памятка: DNAT = «снаружи → внутрь», POSTROUTING SNAT = «изнутри → наружу». Нужен корректный **обратный маршрут** до клиента.

```bash
# Статус/перезагрузка
sudo ufw status verbose
sudo ufw reload
sudo ufw reset

# Безопасный старт (не отрежь SSH)
sudo ufw allow OpenSSH
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable

# Базовые правила
sudo ufw allow 22/tcp
sudo ufw limit 22/tcp           # защита от брута
sudo ufw allow 80,443/tcp       # несколько портов
sudo ufw allow 3000:3010/tcp    # диапазон

# По IP/сети/интерфейсу/направлению
sudo ufw allow from 10.0.0.5 to any port 22 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 5432
sudo ufw deny from 203.0.113.0/24
sudo ufw allow in  on eth0 to any port 80
sudo ufw allow out on eth0 to any port 53 proto udp

# Профили приложений
sudo ufw app list
sudo ufw app info "Nginx Full"
sudo ufw allow "Nginx Full"

# Удаление
sudo ufw status numbered
sudo ufw delete <N>
# или:
sudo ufw delete allow 80/tcp

# IPv6
# /etc/ufw/ufw.conf -> IPV6=yes ; затем sudo ufw reload

```
---

## SSH и копирование

```bash
ssh-keygen -t ed25519 -C "me@host"
# -t ed25519 — тип ключа (быстрый и безопасный)
# -C         — комментарий (метка), просто подпись

ssh-copy-id user@host
# залить публичный ключ на сервер (в authorized_keys)

ssh -J jump user@target
# -J — ProxyJump (через бастион-сервер "jump")

scp file user@host:/path/
# простой копир “как в ssh”

rsync -avP dir/ user@host:/dst/
# -a — archive (права, ссылки, рекурсивно)
# -v — подробно (verbose)
# -P — прогресс + partial (докачка при обрыве)
```

---

## Пакеты

```bash
# Debian/Ubuntu
sudo apt update && sudo apt upgrade -y
# update — обновить кеш репозиториев
# upgrade -y — обновить пакеты (да, ответить “yes” автоматически)

apt search <pkg> ; apt show <pkg>
# поиск и подробности пакета

sudo apt install <pkg> ; sudo apt remove <pkg>
# установка/удаление

# RHEL/Alma/Rocky
sudo dnf makecache && sudo dnf upgrade -y
# makecache — обновить метаданные реп
# upgrade -y — обновить пакеты

dnf search <pkg> ; dnf info <pkg>
sudo dnf install <pkg> ; sudo dnf remove <pkg>
```

---

## Systemd: юнит за минуту

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target

[Service]
User=myuser
ExecStart=/usr/local/bin/myapp --config /etc/myapp.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
# перечитать новые/изменённые юниты

sudo systemctl enable --now myapp
# включить автозапуск и сразу запустить

journalctl -u myapp -f
# логи сервиса в реальном времени
```

---

## Мини-паттерны Bash

```bash
# прогнать команду по списку хостов (hosts.txt — один хост на строке)
for h in $(cat hosts.txt); do ssh "$h" 'uptime'; done

# массовая замена текста по проекту
grep -Rl 'old' . | xargs sed -i 's/old/new/g'

# быстро прочитать JSON эндпоинт
curl -s http://localhost:8080/health | jq .
# -s — тихо (без прогресс-бара)
```

По