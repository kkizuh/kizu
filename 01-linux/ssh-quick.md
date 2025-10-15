# SSH / SCP / RSYNC — шпора с комментами

## 0) Подключение (база)

```bash
ssh user@host              # стандартное подключение по ключу/паролю
ssh -p 2222 user@host      # указать порт (если не 22)
ssh -i ~/.ssh/id_ed25519 user@host   # использовать конкретный ключ
```

---

## 1) Ключи: создать, установить, права

```bash
ssh-keygen -t ed25519 -C "you@company"    # сгенерить ключ современного типа ED25519
# ключи: ~/.ssh/id_ed25519 (приватный, НИКОМУ) и id_ed25519.pub (публичный)

ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host   # докинуть ключ на сервер в authorized_keys
# если нет ssh-copy-id:
# cat ~/.ssh/id_ed25519.pub | ssh user@host 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'

# корректные права (иначе ssh откажет)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

**Агент ключей (чтобы не вводить пароль ключа каждый раз):**

```bash
eval "$(ssh-agent -s)"           # стартовать агент
ssh-add ~/.ssh/id_ed25519        # загрузить ключ в агент
ssh-add -l                       # проверить, что ключ там
```

---

## 2) ~/.ssh/config — один раз настроил и живёшь спокойно

`~/.ssh/config`

```sshconfig
# Бастион (jump host): задаём алиас, юзера и ключ
Host bastion
  HostName bastion.example.com
  User ops
  IdentityFile ~/.ssh/id_ed25519

# Шаблон для всех прод-хостов (маски можно)
Host app-*.prod
  User deploy                       # юзер по умолчанию
  IdentityFile ~/.ssh/id_ed25519    # какой ключ
  ServerAliveInterval 30            # пинги, чтобы коннект не падал
  ServerAliveCountMax 3
  StrictHostKeyChecking accept-new  # доверять новым хостам 1-й раз
  ControlMaster auto                # мультиплексирование (быстрые повторные коннекты)
  ControlPersist 5m
  ControlPath ~/.ssh/cm-%r@%h:%p
  ProxyJump bastion                 # ходим через бастион

# Простой алиас
Host app1
  HostName 10.0.1.23
  User deploy
```

Теперь достаточно `ssh app1` или `ssh app-01.prod`.

---

## 3) Bastion (джамп-хост)

```bash
ssh -J user@bastion user@internal      # разовый прыжок через бастион
# лучше через ~/.ssh/config → ProxyJump (см. выше)
```

---

## 4) Туннели (портфорвардинг)

### Local forward (локальный порт → удалённый сервис)

```bash
ssh -L 127.0.0.1:8080:127.0.0.1:80 user@host
# теперь localhost:8080 у тебя → host:80 на сервере
```

### Remote/reverse (удалённый порт → твой локальный)

```bash
ssh -R 0.0.0.0:8080:127.0.0.1:80 user@host
# откроет порт 8080 на сервере, трафик уйдёт в твой локальный :80
# нужно: на сервере в sshd_config -> GatewayPorts yes (иначе слушает только 127.0.0.1)
```

### SOCKS-прокси (динамический)

```bash
ssh -D 1080 user@host
# в браузере прокси SOCKS5 -> 127.0.0.1:1080
```

### Держать туннель живым

```bash
# apt/dnf install autossh
autossh -M 0 -fN -L 8080:127.0.0.1:80 user@host
# -fN = фон, не запускаем команду; -M 0 = без monitor-порта
```

---

## 5) Копирование файлов

### scp (просто, быстро)

```bash
scp file.txt user@host:/tmp/             # файл → сервер
scp -r dir/ user@host:/srv/dir/          # каталог рекурсивно
scp -P 2222 -i ~/.ssh/id_ed25519 file user@host:/tmp/  # кастомный порт/ключ
```

### rsync (умно, надёжно, умеет инкременты)

```bash
# локально → удалённо по SSH (сохраняем права/владельцев/ACL/xattr)
rsync -aHAX --numeric-ids --info=progress2 \
  -e "ssh -i ~/.ssh/id_ed25519 -p 22" \
  /src/ user@host:/dst/
# ВАЖНО: хвостовой слэш у /src/ копирует СОДЕРЖИМОЕ, без него — каталог целиком.

# зеркало (удалить лишнее на приёмнике) — сначала dry-run!
rsync -an --delete -e ssh /src/ user@host:/dst/   # dry-run (ничего не тронет)
rsync -a  --delete -e ssh /src/ user@host:/dst/   # боевой запуск

# через bastion (прозрачно)
rsync -a -e "ssh -J bastion" /src/ app@web01:/dst/
```

### sftp (интерактивно)

```bash
sftp user@host
sftp> put file ; get file ; lcd ; cd ; mput ; mget ; ls ; pwd ; !
```

---

## 6) Безопасность (минимум, но обязательно)

### На сервере: /etc/ssh/sshd_config

```
PasswordAuthentication no            # только ключи
PermitRootLogin prohibit-password    # или no (если root не нужен интерактивно)
PubkeyAuthentication yes
AllowGroups sshusers                 # белый список группы
# (Перезапуск)
```

```bash
sudo systemctl reload sshd
```

### Ограничить ключ (fine-grained) в authorized_keys

```
from="10.0.0.0/8",command="/usr/local/bin/backup.sh",no-pty,no-port-forwarding ssh-ed25519 AAAAC3...
# доступ только с подсети, запускает ровно 1 команду, без интерактива/туннелей
```

### Capabilities вместо root для портов <1024

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/myapp
# бинарник сможет слушать 80/443 без sudo/root
```

### Fail2ban (анти-брутфорс)

```bash
sudo apt/dnf install fail2ban
# включить стандартный jail для sshd — 90% достаточно
```

---

## 7) Траблшут: быстро найти причину

### Включи подробный дебаг клиента

```bash
ssh -vvv user@host
# смотри, какой ключ выбирается, какой порт, почему отказ
```

### Типовые баги и фиксы

- **Permission denied (publickey)**  
    → проверь права:  
    `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519`  
    → правильный юзер? правильный host? ключ добавлен в агент? `ssh-add -l`
- **REMOTE HOST IDENTIFICATION HAS CHANGED**  
    → ключ сервера изменился. Проверь out-of-band; затем: `ssh-keygen -R host && ssh host`
- **Сессия отваливается**  
    → в `~/.ssh/config`: `ServerAliveInterval 30` и `ServerAliveCountMax 3`
- **Agent forwarding не едет**  
    → `ForwardAgent yes` у клиента, `AllowAgentForwarding yes` на сервере; `ssh-add -l` показывает ключ?
- **Reverse-туннель не слушает внешку**  
    → на сервере `GatewayPorts yes` либо используйте `ssh -R 0.0.0.0:...`
- **Rsync копирует “не туда”**  
    → проверь слэши: `/src/` vs `/src` и ключ `--delete` сначала с `-n`

---

## 8) Быстрые рецепты (копипаст)

### Проброс БД через бастион

```bash
ssh -J bastion -L 5432:db.internal:5432 app@web01
psql -h 127.0.0.1 -U dbuser -p 5432
# теперь локальный 5432 ходит на внутренний db.internal:5432
```

### Архив через ssh (без промежуточных файлов)

```bash
# забрать
ssh user@host "tar -C /var/www -czf - ." > www-$(date +%F).tgz
# отправить
tar -C /var/www -czf - . | ssh user@host "cat > /tmp/www-$(date +%F).tgz"
```

### Массовая доставка файла на несколько хостов

```bash
for h in app01 app02 app03; do scp -q file.conf $h:/etc/myapp/; done
```

### AutoSSH systemd-сервис (держит туннель)

```ini
# /etc/systemd/system/autossh-tunnel.service
[Unit]
Description=Persistent SSH tunnel
After=network-online.target

[Service]
User=deploy
ExecStart=/usr/bin/autossh -M 0 -N -L 127.0.0.1:8080:127.0.0.1:80 user@host
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now autossh-tunnel
```

---

## 9) Серверный hardening (чтобы спать спокойнее)

- Включи **UFW/firewalld** и открой только нужный SSH-порт.
- Лимит логинов: `MaxAuthTries 3`, `LoginGraceTime 30`.
- Логи смотри регулярно: `journalctl -u ssh -n 200 | less`.
- Доступ к прод — через **bastion** + `AllowGroups`.

---

## 10) Мини-FAQ (на собесе и в бою)

- **Зачем ProxyJump vs ProxyCommand?**  
    ProxyJump проще и стандартнее в новых OpenSSH, под капотом тот же ProxyCommand.
- **В чём разница scp/rsync?**  
    scp — тупо копирует; rsync — синхронизирует, экономит трафик/время, есть `--delete`, инкременты (`--link-dest`).
- **Как ограничить ключи в authorized_keys?**  
    Префиксы `from=`, `command=`, `no-pty`, `no-port-forwarding`, `no-agent-forwarding`.
- **Почему ssh отказывается из-за прав?**  
    ~/.ssh должен быть 700, authorized_keys/id_ed25519 — 600. Любая “дырка” — отказ.

---

