## TL;DR (сделай сразу)

```bash
# SSH: запрет root и паролей (оставь ключи)
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
LoginGraceTime 30
EOF
sudo systemctl restart sshd || sudo systemctl restart ssh

# Политика паролей (минимум)
sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/'  /etc/login.defs
sudo sed -i 's/^PASS_WARN_AGE.*/PASS_WARN_AGE   14/'  /etc/login.defs
echo -e "minlen = 12\nminclass = 3\nretry = 3" | sudo tee /etc/security/pwquality.conf >/dev/null

# Блокировка брутфорса (faillock)
sudo sed -i 's/^#\?deny.*/deny = 5/' /etc/security/faillock.conf 2>/dev/null || true
sudo sed -i 's/^#\?unlock_time.*/unlock_time = 600/' /etc/security/faillock.conf 2>/dev/null || true

# Постоянные журналы systemd
sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald

# Аудит
sudo apt-get install -y auditd 2>/dev/null || sudo dnf install -y auditd
sudo systemctl enable --now auditd
```

---

## 1) PAM и парольная политика

### Сроки паролей (всем по умолчанию)

- Файл: `/etc/login.defs`
    - `PASS_MAX_DAYS 90` — срок 90 дней
    - `PASS_MIN_DAYS 1` — минимум 1 день между сменами
    - `PASS_WARN_AGE 14` — предупреждать за 14 дней
- Применить пользователю:
```bash
    sudo chage -M 90 -m 1 -W 14 user
    ```
    

### Сложность паролей (pwquality)

- Файл: `/etc/security/pwquality.conf`
```ini
    minlen = 12          # длина
    minclass = 3         # классы: a-z, A-Z, 0-9, спец — минимум 3
    retry = 3
    dictcheck = 1
```
- Подключение модуля PAM:
    - **Debian/Ubuntu:** `/etc/pam.d/common-password` → должна быть строка с `pam_pwquality.so`  
    - **RHEL/Fedora:** `/etc/pam.d/system-auth` и `/etc/pam.d/password-auth` 

### Блокировка после неудачных входов (faillock)

- Конфиг: `/etc/security/faillock.conf`
    
    ```ini
    deny = 5            # 5 неверных попыток
    unlock_time = 600   # 10 минут
    fail_interval = 900 # окно 15 минут
    ```
    
- Сброс блокировки: `faillock --user <user> --reset`
    

> проверка: `sudo pam-auth-update` (Debian), `grep -R faillock /etc/pam.d` — модули активны.

---

## 2) Запретить root/пароли/SSH (hardening SSH)

**Где править:**  
основной файл `/etc/ssh/sshd_config`, лучше — дроп-ин `/etc/ssh/sshd_config.d/10-hardening.conf`.

**Минимум:**

```conf
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
LoginGraceTime 30
# (опц.) ограничить пользователей/сети:
# AllowUsers admin@1.2.3.4
# Match Address 10.0.0.0/8
#   PasswordAuthentication no
```

Применить:

```bash
sudo systemctl restart sshd || sudo systemctl restart ssh
```

**Полностью запретить SSH (если нужно):**

```bash
sudo systemctl disable --now sshd || sudo systemctl disable --now ssh
```

> Лайфхак: включи fail2ban/ufw/firewalld — уменьшит шум в логах и подрезает брут.

---

## 3) Что такое systemd и что за runtime

- **systemd** — менеджер системы: сервисы (`.service`), таймеры (`.timer`), сокеты (`.socket`), цели (`.target`), логи (`journald`), cgroups.
- **Runtime** — каталог `/run` (tmpfs). здесь PID-файлы, сокеты и прочee “живое” состояние. очищается при перезагрузке.
    - В юните можно попросить systemd создать runtime-директорию:
```ini
        RuntimeDirectory=myapp        # создаст /run/myapp с нужными правами
```
        

---

## 4) Запусти свою службу (юнит)

`/etc/systemd/system/myapp.service`

```ini
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
Environment=ENV=prod
ExecStart=/opt/myapp/bin/app --port=8080
Restart=on-failure
RestartSec=3
RuntimeDirectory=myapp
# Порты <1024 без root:
# AmbientCapabilities=CAP_NET_BIND_SERVICE
# CapabilityBoundingSet=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Запуск:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp.service
systemctl status myapp
journalctl -u myapp -f
```

---

## 5) Замени cron на systemd-timers

**Почему:** логируется в journal, зависимостей больше, `Persistent=true` добивает пропущенные задачи.

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
Persistent=true
AccuracySec=1min
[Install]
WantedBy=timers.target
```

Включить:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers --all
```

**Шаблоны:**
- будни 09:00 → `OnCalendar=Mon..Fri 09:00`
- каждые 30 мин → `OnUnitActiveSec=30min`
- через 5 мин после старта → `OnBootSec=5min`

---

## 6) Логи: где искать и как смотреть

### journald (systemd)

```bash
journalctl -xe                              # свежие ошибки и подсказки
journalctl -u nginx -b                      # логи сервиса за текущую загрузку
journalctl -u nginx -f                      # follow
journalctl --since "1 hour ago" -p err..alert
```

**Сделать постоянными (persist):**

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

### Классика (если rsyslog включён)

- **Debian/Ubuntu:** `/var/log/syslog`, `/var/log/auth.log`, `/var/log/kern.log`
- **RHEL/CentOS/Fedora:** `/var/log/messages`, `/var/log/secure`    

### Ядро

```bash
dmesg -T | tail -n 100
```

### Аудит (auditd)

```bash
sudo systemctl enable --now auditd
sudo less /var/log/audit/audit.log
ausearch -m USER_LOGIN,USER_AUTH      # события аутентификации
aureport -au                          # сводка по логинам
```

**Свои правила (персистентно):** `/etc/audit/rules.d/99-custom.rules`

```bash
-w /etc/shadow -p wa -k shadow-change
```

Загрузить: `sudo augenrules --load || sudo systemctl restart auditd`

---

## 7) Полезные проверки и быстрые фикс-коды

```bash
# SSH конфиг норм?
sshd -t && echo "sshd config OK" || echo "sshd config FAIL"

# Кто слушает 22 порт?
ss -ltnp | grep :22

# PAM pwquality активен?
grep -R pam_pwquality /etc/pam.d

# journald пишет на диск (persist)?
test -d /var/log/journal && echo "journald persistent" || echo "volatile"

# Таймер реально активен?
systemctl status backup.timer
systemctl list-timers | grep backup
```

---

## 8) Чеклист (минимум для прод)

-  `PermitRootLogin no`, `PasswordAuthentication no`
-  pwquality: `minlen=12`, `minclass=3` • chage: `MAX_DAYS=90`
-  faillock: `deny=5`, `unlock_time=600`
-  journald persistent (`/var/log/journal`)
-  auditd включён + хотя бы одно правило (shadow)
-  свои задачи — через `systemd timers`, не через cron
-  свои юниты в `/etc/systemd/system/`, `Restart=on-failure`
-  порты <1024 — через capabilities, а не root

