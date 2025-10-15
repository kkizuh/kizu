# Квоты и бэкапы: нулевой вход

## Что это и зачем (в 2 строках)
- **Квоты** — ограничение места для пользователей/групп (чтобы один не забил весь диск).
- **Бэкапы** — копия важных файлов на другой диск/каталог (чтобы было что восстановить).

---

## Часть 1 — Квоты (ext4 на `/`)

### 1) Включаем квоты в `/etc/fstab`

Добавь в строку с корнем `usrquota,grpquota` (через запятую к `defaults`):

```
/dev/vda2  /  ext4  defaults,usrquota,grpquota  0  1
```

Применить без ребута:

```bash
sudo mount -o remount /
mount | grep ' on / '          # убедись, что видишь usrquota,grpquota
```

### 2) Устанавливаем и инициализируем

```bash
sudo apt install -y quota                 # (dnf install quota — на RHEL)
sudo quotacheck -cum /
sudo quotacheck -cgm /
sudo quotaon -vug /
```

### 3) Выдать лимит пользователю (пример)

Лимит для `user1`: мягкий 10 ГБ, жёсткий 12 ГБ (иноды не ограничиваем):

```bash
sudo setquota -u user1 10G 12G 0 0 /
```

Проверка:

```bash
quota -u user1          # что видит юзер
sudo repquota -a        # общий отчёт
```

> Если надо снять квоту: `sudo setquota -u user1 0 0 0 0 /`

**Коротко о терминах:**  
_soft_ — предупреждение; _hard_ — дальше писать нельзя. Иноды = «кол-во файлов» (чаще не трогаем).

---

## Часть 2 — Простой бэкап (rsync-снимки)

### Идея (очень простая)

- Каждый запуск создаёт папку-снимок `YYYY-MM-DD`.
- Новое копируется, старое — хардлинки (экономит место).
- Можно удалить старые папки по дате — и всё.

### 1) Готовим каталог бэкапа

Выбери **другой диск/раздел** (лучше), либо временно системный:

```bash
sudo mkdir -p /mnt/backup/daily /mnt/backup/logs
```

### 2) Файл исключений (чтобы не тащить мусор)

```bash
sudo tee /root/rsync-exclude.txt >/dev/null <<'EOF'
/proc/*
/sys/*
/dev/*
/run/*
/tmp/*
/mnt/*
/media/*
/swapfile
/var/tmp/*
/var/cache/*
/var/lib/docker/*
*.iso
EOF
```

### 3) Скрипт бэкапа (одним файлом)

```bash
sudo tee /root/backup-daily.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
SRC="/"
DST_BASE="/mnt/backup/daily"
TODAY="$DST_BASE/$(date +%F)"
LAST="$DST_BASE/last"
LOG="/mnt/backup/logs/rsync-$(date +%F).log"

mkdir -p "$TODAY"
LINK_ARG=""
[ -L "$LAST" ] && LINK_ARG="--link-dest=$LAST"

rsync -aHAX --numeric-ids --delete --delete-excluded \
  --exclude-from=/root/rsync-exclude.txt \
  $LINK_ARG "$SRC" "$TODAY" | tee "$LOG"

rm -f "$LAST"
ln -s "$TODAY" "$LAST"
EOF
sudo chmod +x /root/backup-daily.sh
```

Запуск вручную:

```bash
sudo /root/backup-daily.sh
```

### (Опционально) Автоматически каждую ночь в 02:30

```bash
sudo tee /etc/systemd/system/backup-daily.service >/dev/null <<'EOF'
[Unit]
Description=Daily rsync snapshot backup
[Service]
Type=oneshot
ExecStart=/root/backup-daily.sh
EOF

sudo tee /etc/systemd/system/backup-daily.timer >/dev/null <<'EOF'
[Unit]
Description=Run daily backup at 02:30
[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now backup-daily.timer
systemctl list-timers | grep backup-daily
```

### Ротация (держим 14 дней)

```bash
sudo find /mnt/backup/daily -maxdepth 1 -type d -name '20*' -mtime +14 -exec rm -rf {} +
```

### Восстановление файла

```bash
# посмотреть доступные снимки
ls /mnt/backup/daily

# вернуть файл из снимка за конкретную дату
sudo rsync -aHAX /mnt/backup/daily/2025-10-01/path/to/file /restore/path/
```

---

## Быстрый чек-лист, чтобы не облажаться

- Перед правкой `fstab` — **сделай копию**: `sudo cp /etc/fstab /etc/fstab.bak`.
- После правки `fstab` — **всегда** `sudo mount -a` (если тихо — ок, если ошибка — починить до ребута).
- Квоты: видишь `usrquota,grpquota` в `mount` — значит включены.
- `rsync --delete` опасен, если перепутать `SRC/DST`. Не меняй пути в скрипте без нужды.
- Бэкап храни **на другом диске/VM/сервере** — тогда он спасёт.

---

## Мини-FAQ (только нужное)

- **Как снять лимит?** `setquota -u user 0 0 0 0 /`
- **Как посмотреть общий отчёт?** `repquota -a`
- **Как проверить, что бэкап прошёл?** смотри новые папки в `/mnt/backup/daily` и логи в `/mnt/backup/logs`, либо `journalctl -u backup-daily.service -n 50`.
- **Можно ли бэкапить на SSH-сервер?** Да, меняешь `DST` на `backup@host:/path/$(date +%F)` и добавляешь `--link-dest=/path/last` на той стороне (нужен доступ по ключу).
    

---

Если хочешь, оформлю это как страницу `[[quota-backup]]` для твоей базы знаний, чтобы всегда было «под рукой», и добавлю короткую «версию на одну страницу» для печати.