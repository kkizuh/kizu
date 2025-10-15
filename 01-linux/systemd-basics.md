Прежде всего нужно упомянуть, что службы Systemd можно разделить на две категории:
1) системные службы, которые запускаются от суперпользователя. Для управления такими службами нужно использовать sudo. 
2) пользовательские службы, которые запускаются от определённого юзера, который имеет над ними полный контроль без необходимости в sudo.

Системные службы лежат в:
1) /usr/lib/systemd/system;
2) /lib/systemd/system;
3) /etc/systemd/system;

Пользовательские служы лежат в:
1) /usr/lib/systemd/user;
2) /lib/systemd/user;
3) /etc/systemd/user;
4) /home/user/.config/systemd/user;
## База

```bash
systemctl status nginx          # статус + логи хвостом
systemctl is-active nginx       # active|inactive|failed
systemctl is-enabled nginx      # enabled|disabled
systemctl start|stop|restart nginx
systemctl reload nginx          # мягкая перезагрузка
systemctl reload-or-restart nginx
```

## Автозапуск

```bash
systemctl enable nginx          # в автозагрузку
systemctl disable nginx
systemctl enable --now nginx    # включить + сразу старт
systemctl disable --now nginx
```

## Логи сервиса (journalctl)

```bash
journalctl -u nginx             # все логи
journalctl -u nginx -f          # follow (как tail -f)
journalctl -u nginx -b          # с последней загрузки
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -p err..crit
```

## Списки и фильтры

```bash
systemctl list-units --type=service              # активные сервисы
systemctl list-units --type=service --all        # все (в т.ч. inactive)
systemctl list-units --type=service --state=failed
systemctl list-unit-files --state=enabled
```

## Маскирование (жёстко запретить)

```bash
systemctl mask bluetooth.service
systemctl unmask bluetooth.service
```

## Зависимости

```bash
systemctl list-dependencies nginx
systemctl list-dependencies --reverse nginx
```

## Юниты: где лежат и как править

Порядок поиска:  
`/etc/systemd/system` (override) → `/run/systemd/system` → `/usr/lib/systemd/system`

```bash
systemctl cat nginx             # показать юнит
systemctl show -p FragmentPath nginx.service  # где лежит файл
systemctl edit nginx            # drop-in override (/etc/systemd/system/nginx.service.d/override.conf)
systemctl edit --full nginx     # редактировать полный юнит (копия в /etc)
systemctl daemon-reload         # перечитать юниты (после правок!)
systemctl restart nginx
```

**Пример override (добавить лимиты/хардениг):**

```ini
# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65536
Restart=always
RestartSec=5s
TimeoutStopSec=30s
AmbientCapabilities=CAP_NET_BIND_SERVICE
ProtectSystem=full
PrivateTmp=true
```

## Типы сервисов (когда пишешь свои)

`Type=simple|forking|oneshot|notify|dbus`  
Частое: `Restart=on-failure` / `always`, `RestartSec=5`

**Скелет юнита:**

```ini
[Unit]
Description=My App
After=network.target
Requires=postgresql.service

[Service]
Type=simple
User=app
WorkingDirectory=/opt/app
ExecStart=/opt/app/run.sh
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
EnvironmentFile=/etc/app/env

[Install]
WantedBy=multi-user.target
```

## Ресурсные лимиты (cgroups)

```ini
[Service]
CPUQuota=50%
MemoryLimit=512M
LimitNOFILE=65535
# IO приоритет и прочее:
IOSchedulingClass=best-effort
IOSchedulingPriority=5
```

## Таймеры (замена cron)

```ini
# backup.timer
[Timer]
OnCalendar=*-*-* 02:30:00
RandomizedDelaySec=30m
Persistent=true
Unit=backup.service
```

```bash
systemctl enable --now backup.timer
systemctl list-timers --all
```

## Таргеты (ранлевелы)

```bash
systemctl get-default
systemctl set-default multi-user.target
systemctl isolate multi-user.target    # переключиться прямо сейчас
```

## Питание/перезагрузка

```bash
systemctl reboot
systemctl poweroff
systemctl suspend
systemctl hibernate
```

## Траблшутинг быстро

```bash
systemctl --failed                     # что упало
systemctl reset-failed                 # сбросить фейлы
systemctl status <svc> -n50            # больше логов в статусе
journalctl -u <svc> -p err..crit -b    # ошибки за текущий бут
```

## Анализ загрузки

```bash
systemd-analyze
systemd-analyze blame                  # кто дольше стартует
systemd-analyze critical-chain
```

## Шорткаты/лайфхаки

```bash
systemctl restart apache*              # по маске
systemctl status nginx -n5             # урезать хвост логов в статусе
systemctl show -p ActiveState nginx    # одно свойство
systemctl restart nginx && systemctl status nginx
systemctl reload nginx || systemctl restart nginx
```

## Кросс-дистро нюансы

- Ubuntu/Debian: `apache2`, firewall — `ufw`
- RHEL/Fedora: `httpd`, firewall — `firewalld`  
    Пакеты ставятся по-разному, но `systemctl` везде одинаков.
