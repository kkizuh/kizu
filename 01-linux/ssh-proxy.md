## SOCKS-прокси (динамический)

**Что это:** SSH становится локальным SOCKS5-прокси. Любой трафик приложений гонится через сервер.

```bash
ssh -D 1080 user@host
# теперь в браузере: SOCKS5 proxy -> 127.0.0.1:1080
# чтобы не зависал — добавь keepalive в ~/.ssh/config:
#   ServerAliveInterval 30
#   ServerAliveCountMax 3
```

---

## Держать туннель живым (autossh)

**Зачем:** чтобы SSH-туннель авто-переподнимался при обрывах.

```bash
# установка: apt/dnf install autossh
autossh -M 0 -fN -L 8080:127.0.0.1:80 user@host
```

Разбор ключей:

- `-M 0` — без отдельного монитор-порта (исп. только SSH keepalive).
- `-f` — в фон.
- `-N` — не выполнять команду (просто туннель).
- `-L 8080:127.0.0.1:80` — локальный порт 8080 → удалённый 127.0.0.1:80.

**Сервисом (systemd) лучше:**

```ini
# /etc/systemd/system/autossh-L8080.service
[Unit]
Description=Persistent SSH L-forward 8080->host:80
After=network-online.target

[Service]
User=deploy
Environment=AUTOSSH_GATETIME=0
ExecStart=/usr/bin/autossh -M 0 -N -L 127.0.0.1:8080:127.0.0.1:80 user@host
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## Jump host / Bastion — что это и какие бывают

**Bastion (jump host)** — “промежуточный” хост в публичке, через который прыгаешь во внутреннюю сеть.

Виды/паттерны:

1. **Одинарный прыжок (classic bastion)**  
    Путь: `ноут → bastion → внутренний`.
```bash
    ssh -J user@bastion user@internal
    ```
2. **Много хопов (chain)**  
    Путь: `ноут → bastion → jump2 → internal`.
```bash
    ssh -J user@bastion,user@jump2 user@internal
    ```
    
3. **Бастион + прокси-команды (старый стиль)**
```bash
    ssh -o ProxyCommand="ssh -W %h:%p user@bastion" user@internal
    ```
    
4. **Постоянный bastion в ~/.ssh/config (удобно)**
```sshconfig
    Host bastion
      HostName bastion.example.com
      User ops
      IdentityFile ~/.ssh/id_ed25519
    
    Host *.int
      User dev
      ProxyJump bastion6. 
   ```
    Теперь: `ssh app01.int`.

**Для чего:** не открывать все внутренние сервера наружу, аудит, MFA, тонкие правила.

---

## sftp (интерактивная сессия по SSH)

```bash
sftp user@host
sftp> help         # список команд
sftp> pwd          # где я на сервере
sftp> lpwd         # где я локально
sftp> ls /var/log
sftp> lcd ~/Downloads
sftp> put file.txt                 # загрузить файл на сервер
sftp> mput *.log                   # массовая загрузка
sftp> get /etc/hosts               # скачать файл с сервера
sftp> mget *.tar.gz                # массовая скачка
sftp> mkdir /tmp/upload
sftp> rm /tmp/upload/old.log
sftp> !pwd                         # выполнить локальную команду
```

---

## Ограничить ключ в `authorized_keys` (fine-grained)

**Зачем:** привязать ключ к подсети/одной команде/запретить TTY/туннели.
Строка в `~/.ssh/authorized_keys`:

```
from="10.0.0.0/8",command="/usr/local/bin/backup.sh",no-pty,no-port-forwarding \
ssh-ed25519 AAAAC3... user@laptop
```

Разбор:

- `from="10.0.0.0/8"` — входить только с этой сети.
- `command="..."` — при логине всегда выполняется эта команда (интерактив запрещён).
- `no-pty` — без интерактивной сессии.
- `no-port-forwarding` / `no-agent-forwarding` / `no-X11-forwarding` — запретить туннели/агент/X11.

Полезные варианты:

- **только rsync-бэкап:**
```
	command="/usr/local/bin/backup.sh",no-pty,no-port-forwarding,no-agent-forwarding ssh-ed25519 AAAA...
```
    
- **только один сервер (source IP):**
```
    from="203.0.113.10" ssh-ed25519 AAAA...
```

---

## Capabilities вместо root для портов <1024

**Проблема:** слушать 80/443 может только root.  
**Решение:** выдать бинарнику capability `CAP_NET_BIND_SERVICE`.

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/myapp
# теперь myapp может слушать 80/443 без sudo/root
getcap /usr/local/bin/myapp             # проверка
```

Ещё полезные cap’ы (редко, аккуратно):

- `CAP_CHOWN`, `CAP_DAC_OVERRIDE` — доступ к файлам, **опасно**.
- `CAP_NET_ADMIN` — сетевые настройки, **опасно**.
- `CAP_SYS_PTRACE` — трейсить процессы, **опасно**.

Лучше **минимум**: выдать ровно то, что нужно (часто только `NET_BIND_SERVICE`).

---

## Короткие рецепты (копипаст)

**SOCKS через bastion:**

```bash
ssh -J user@bastion -D 1080 user@internal
# настроить браузер на SOCKS5 127.0.0.1:1080
```

**Rsync через bastion:**

```bash
rsync -a -e "ssh -J user@bastion" /src/ user@internal:/dst/
```

**Локальный туннель к БД за bastion:**

```bash
ssh -J bastion -L 5432:db.internal:5432 app@web01
psql -h 127.0.0.1 -p 5432 -U dbuser
```

**Reverse-туннель (дать доступ к локалке извне):**

```bash
# на своей машине
ssh -R 0.0.0.0:8080:127.0.0.1:3000 user@server
# на server теперь доступен твой локальный 3000 по порту 8080
# (на server: GatewayPorts yes)
```

---

## Мини анти-грабли

- В `authorized_keys` опции идут **перед** ключом, через запятую, без лишних пробелов.
- Для autossh поставь keepalive: в `~/.ssh/config` `ServerAliveInterval 30`, `ServerAliveCountMax 3`.
- При rsync внимательно со слешем: `/src/` (содержимое) vs `/src` (каталог).
- Для ProxyJump нужен OpenSSH ≥ 7.3 (почти везде уже есть).
