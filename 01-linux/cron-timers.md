# Cron и systemd-timers: периодические задачи

> цель: быстро поставить периодичку, понимать где логи, и как мигрировать с cron на systemd timers.

---

## TL;DR (когда что брать)

- **cron** — простые задачи, везде есть.
- **systemd timers** — логи в journal, зависимости от сервисов, “догоняет” пропущенные задачи (`Persistent=true`). **В проде — лучше таймеры.**

---

## 1) Cron — базовое

### Где лежит

- Пользовательские задания: `crontab -e` → хранится в `/var/spool/cron/crontabs/<user>` (Deb) / `/var/spool/cron/<user>` (RHEL).
- Системные каталоги:
    - `/etc/crontab` (с полем USER в строке)
    - `/etc/cron.hourly/` `/etc/cron.daily/` `/etc/cron.weekly/` `/etc/cron.monthly/`

### Формат строки в crontab

```
# * * * * *  команда
# ┬ ┬ ┬ ┬ ┬
# │ │ │ │ └─ день недели (0-7, где 0/7=вс)
# │ │ │ └─── месяц (1-12)
# │ │ └───── день месяца (1-31)
# │ └─────── час (0-23)
# └───────── минута (0-59)
```

### Примеры

```crontab
# Каждый день в 02:30
30 2 * * * /usr/local/bin/backup.sh

# Будни в 09:00
0 9 * * 1-5 /usr/local/bin/report.sh

# Каждые 15 минут
*/15 * * * * /usr/local/bin/job.sh

# Первое число месяца в 01:10
10 1 1 * * /usr/local/bin/billing.sh
```

### Важные нюансы

- PATH у cron **минимальный** — прописывай полный путь к бинарям или задай `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` вверху crontab.
- Редирект логов:
```crontab
    */5 * * * * /usr/local/bin/check.sh >> /var/log/check.log 2>&1
```
- Среда:
```crontab
    SHELL=/bin/bash
    MAILTO=""
```
- Тест: запусти команду вручную, затем эмулируй через `run-parts` (для cron.*) или подставь дату в скрипт.

### Логи cron

- Debian/Ubuntu: `/var/log/syslog` (ищи `CRON`) или `/var/log/cron.log` (если настроено)
- RHEL/Fedora: `/var/log/cron`
- journald (иногда): `journalctl -u cron` / `-u crond`

### anacron (если ПК не 24/7)

- Выполняет ежедневные/еженедельные задачи, если машина была выключена.
- Конфиг: `/etc/anacrontab`

---

## 2) systemd timers — как это устроено

Два файла:
1. `.service` — что делать
2. `.timer` — когда делать

Ложим **в `/etc/systemd/system/`**.

### Пример: ежедневный бэкап в 02:30

`/etc/systemd/system/backup.service`

```ini
[Unit]
Description=Nightly backup
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

`/etc/systemd/system/backup.timer`

```ini
[Unit]
Description=Run backup daily at 02:30
[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true          # “догнать”, если машина была off
AccuracySec=1min         # ок, если +/- минута
RandomizedDelaySec=5min  # случайная задержка (анти-шторм)
Unit=backup.service
[Install]
WantedBy=timers.target
```

Включение/проверка:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers --all
journalctl -u backup.service -b
```

### Частые шаблоны `OnCalendar`

```ini
# Каждые 15 минут
OnCalendar=*:0/15

# Будни 09:00
OnCalendar=Mon..Fri 09:00

# Первое число месяца 01:10
OnCalendar=*-*-01 01:10:00

# Каждые 30 минут с момента последнего запуска (не OnCalendar):
# вместо этого используем:
OnBootSec=5min
OnUnitActiveSec=30min
```

### Таймеры “от события”

```ini
[Timer]
OnBootSec=5min           # через 5 минут после загрузки
OnUnitActiveSec=1h       # затем каждый час после предыдущего ВЫПОЛНЕНИЯ
```

### User-timers (без sudo)

- Файлы: `~/.config/systemd/user/my.timer` и `my.service`
- Команды:  
    `systemctl --user daemon-reload`  
    `systemctl --user enable --now my.timer`  
    `systemctl --user list-timers`

---

## 3) Миграция cron → systemd timers (шпаргалка соответствий)

|Cron|systemd timer|
|---|---|
|`30 2 * * *`|`OnCalendar=*-*-* 02:30:00`|
|`0 9 * * 1-5`|`OnCalendar=Mon..Fri 09:00`|
|`*/15 * * * *`|`OnCalendar=*:0/15`|
|`10 1 1 * *`|`OnCalendar=*-*-01 01:10:00`|
|(запуск через N после старта)|`OnBootSec=Ns`|
|каждые N минут от последнего запуска|`OnUnitActiveSec=Nmin`|

**Главное:** команда/скрипт описывается в `.service`, а расписание — в `.timer`.

---

## 4) Траблшут таймеров

```bash
# Активные/следующие срабатывания
systemctl list-timers --all

# Логи сервиса (то, что выполняется)
journalctl -u backup.service -b -n 100

# Проверить юнит на ошибки синтаксиса
systemd-analyze verify /etc/systemd/system/backup.service

# Принудительно запустить сейчас
systemctl start backup.service

# Таймер “не видит сеть” — сервису нужны зависимости:
# в backup.service:
[Unit]
After=network-online.target
Wants=network-online.target
```

Частые грабли:

- Забыл `daemon-reload` после создания/правки.
- Скрипт требует переменные окружения → задай в `[Service]`:
```
    Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    EnvironmentFile=-/etc/myenv
    WorkingDirectory=/opt/task
```
- Нет прав/путей → всегда пиши абсолютные пути `/usr/bin/rsync` и т.п.

---

## 5) Безопасность и ресурсы

В `.service` пользуйся cgroups/санбоксом:

```ini
[Service]
User=backup
Group=backup
PrivateTmp=true
NoNewPrivileges=yes
ProtectSystem=full
ProtectHome=true
ReadWritePaths=/backup /var/log/backup
CPUQuota=50%
MemoryMax=500M
Nice=10
```

---

## 6) Быстрые карточки (копипаста)

**Каждые 10 мин, лог в journald:**

```ini
# /etc/systemd/system/job.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/job.sh

# /etc/systemd/system/job.timer
[Timer]
OnUnitActiveSec=10min
Persistent=true
[Install]
WantedBy=timers.target
```

**Будни в 19:30, с рандом-джиттером:**

```ini
[Timer]
OnCalendar=Mon..Fri 19:30
RandomizedDelaySec=10min
```

**Ежедневно, но без запуска при оффлайне (как чистый cron):**

```ini
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=false
```

---

## 7) Где логи

- **Cron:**
    - Debian/Ubuntu: `/var/log/syslog` (строки с `CRON`) или `/var/log/cron.log`, если настроено
    - RHEL/Fedora: `/var/log/cron`
- **systemd timers:** логи выполняемого `.service` в journald  
    `journalctl -u <service> -f`  
    Сделай persist журналов:
```bash
    sudo mkdir -p /var/log/journal
    sudo systemctl restart systemd-journald
```
    

---

## 8) Выбор: cron или timers?

Беру **timers**, если нужно:
- видеть логи и статусы сразу в `journalctl`;
- зависимости от сети/служб (`After=/Wants=` в `.service`);
- гарантии “догонялки” (`Persistent=true`);
- единая модель через systemd (меньше зоопарка).

Оставляю **cron**, если:
- древняя система без systemd;
- простая команда и не важны логи/зависимости.

https://crontab.guru/