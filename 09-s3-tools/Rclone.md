## 1) Установка

Официальная страница: _Rclone downloads_. Рекомендуемые варианты:

### Linux (скрипт от rclone)

```bash
# Ubuntu/Debian/Alma/Rocky и т. п.
sudo -v
curl -fsSL https://rclone.org/install.sh | sudo bash
```

> В проде предпочтительнее использовать пакеты дистрибутива или собственный репозиторий артефактов.

### Проверка

```bash
rclone version
```

---

## 2) Настройка подключения к Ceph RGW

Запустите мастер:

```bash
rclone config
```

Ответьте на вопросы (пример для удалённого имени `rgw`):

```
n) New remote
name> rgw
Storage> s3    # Amazon S3 Compatible Storage
provider> Ceph  # если нет пункта Ceph — выберите 'Other' и задайте endpoint вручную
env_auth> false # "No"
access_key_id> <ACCESS_KEY>
secret_access_key> <SECRET_KEY>
region> us-east-1
endpoint> https://s3.example.com
location_constraint> (пусто)
acl> private
server_side_encryption> none
storage_class> (по умолчанию)

y) Yes this is OK
```

Итоговый фрагмент в `~/.config/rclone/rclone.conf` будет выглядеть так:

```ini
[rgw]
type = s3
provider = Ceph
env_auth = false
access_key_id = <ACCESS_KEY>
secret_access_key = <SECRET_KEY>
region = us-east-1
endpoint = https://s3.example.com
acl = private
force_path_style = true      # важно для Ceph RGW
# (если свой CA)
# ca_cert = /etc/ssl/certs/your-ca.pem
```

> Критично: для Ceph почти всегда нужен **force_path_style = true** (или флаг `--s3-force-path-style`).

---

## 3) Базовые команды

> В примерах `rgw:` — это имя вашего remote, созданного на шаге выше.

```bash
# Список бакетов
rclone lsd rgw:

# Листинг содержимого бакета
rclone ls rgw:mybucket
rclone lsl rgw:mybucket        # подробный формат

# Загрузка файла в бакет
rclone copy ./test.txt rgw:mybucket

# Скачивание файла из бакета в текущую папку
rclone copy rgw:mybucket/test.txt ./

# Синхронизация директории → бакет/префикс
rclone sync ./backup/ rgw:mybucket/backup/ --progress

# Миграция между двумя S3-совместимыми хранилищами (две конфигурации)
rclone sync srcRemote:bucket/prefix dstRemote:bucket/prefix --progress
```

Полезные флаги:

- `--dry-run` — показать, что будет сделано, ничего не меняя.
- `--checksum` — сравнение по хешу (дороже, но точнее),  
    `--size-only` — только по размеру (быстрее, но осторожно).
- `--transfers N` и `--checkers N` — параллелизм.
- `--s3-chunk-size 64M` — размер частей для multipart (ускоряет большие файлы).
- `--s3-force-path-style` — если не прописали в конфиге.
- `--progress` — прогресс‑бар.

---

## 4) Фильтрация по возрасту файлов

Для выборочной копии по времени модификации используйте длительности:

```bash
# Копировать только файлы СТАРШЕ 24 часов
rclone copy ./data rgw:mybucket/data --min-age 24h

# Копировать только файлы МОЛОЖЕ 24 часов
rclone copy ./data rgw:mybucket/data --max-age 24h
```

> Для более тонкой фильтрации используйте `--include`/`--exclude` по маскам, или предварительно отберите файлы локальными инструментами и передайте rclone список через `--files-from`.

---

## 5) Типичные сценарии

### 5.1. Резервная копия каталога с верификацией

```bash
rclone sync /srv/data rgw:backup/data \
  --checksum --transfers 8 --checkers 16 --progress
```

### 5.2. Горячая миграция бакета между площадками

```bash
rclone sync old:project/ rgw:project/ \
  --dry-run
# если список операций устраивает — уберите --dry-run
```

### 5.3. Однонаправленное «заливание» без удаления на стороне S3

```bash
rclone copy /mnt/export rgw:ingest/ --transfers 16 --progress
```

---

## 6) Траблшутинг

- **AccessDenied/SignatureDoesNotMatch/403** — проверьте ключи, системное время (NTP), endpoint и `force_path_style`.
- **Could not connect / SSL** — убедитесь, что используете правильную схему (`https://`) и, при собственном CA, укажите `ca_cert`.
- **NoSuchBucket** — бакет не создан или опечатка; создайте бакет заранее.
- **Очень медленный листинг/заливка** — увеличьте `--transfers/--checkers`, проверьте RTT/полосу до RGW, храните мелкие файлы пакетно (архив/тар), держите метаданные RGW на быстрых дисках.

---

## 7) Памятка по операциям

- **copy** — копирует (не удаляет на приёмнике то, чего нет у источника).
- **sync** — делает приёмник зеркалом источника (удаляет лишнее).
- **move** — перемещает (после успешной передачи удаляет на источнике).

---

## 8) Безопасность

- Не храните конфиг с ключами в общедоступных местах; ограничьте права файла `rclone.conf`.
- Используйте отдельные ключи/политики под каждую интеграцию; ротируйте их.
- На проде — всегда `https`, свой CA добавляйте в доверенные.

---

## 9) Короткие шпаргалки

```bash
# Принудительно path-style и явный endpoint через флаги
rclone ls rgw: --s3-force-path-style --s3-endpoint https://s3.example.com

# Ограничить скорость
rclone copy ./big rgw:big --bwlimit 50M --progress

# Копировать только новые/изменённые последние 7 дней
rclone copy ./src rgw:dst --max-age 7d
```
