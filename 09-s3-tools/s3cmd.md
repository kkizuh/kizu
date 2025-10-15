**Зачем:** минимальный и удобный CLI для S3/Swift‑совместимых хранилищ. Идеален для админ‑скриптов и разовых операций. Ниже — настройки под **Ceph RGW** (path‑style, свой endpoint).

---

## 1) Установка

Официально: _Amazon S3 Tools: s3cmd_. https://s3tools.org/s3cmd

### Ubuntu/Debian

```bash
sudo apt update && sudo apt install -y s3cmd
```

### RHEL/Alma/Rocky

```bash
sudo dnf install -y s3cmd
```

### Проверка

```bash
s3cmd --version
```

---

## 2) Быстрая настройка (Ceph RGW, профиль по умолчанию)

Запустите мастер и укажите ключи/endpoint:

```bash
s3cmd --configure
# Access Key: <ACCESS_KEY>
# Secret Key: <SECRET_KEY>
# Default Region: us-east-1 (или оставьте пустым)
# S3 Endpoint: https://s3.example.com
# DNS-style bucket+host: No   # для Ceph нужен path-style
# Use HTTPS: Yes
# HTTP Proxy server name: (пусто)
```

Это создаст `~/.s3cfg`. Для Ceph RGW важно явно зафиксировать path‑style и сигнатуру v4:

```ini
# ~/.s3cfg — пример под Ceph RGW
[default]
access_key = <ACCESS_KEY>
secret_key = <SECRET_KEY>
host_base = s3.example.com
host_bucket = s3.example.com      # path-style: без %(bucket)s. ...
use_https = True
check_ssl_certificate = True
check_ssl_hostname = True
signature_v2 = False              # v4‑подпись (рекомендуется)
bucket_location = us-east-1
# Если свой корпоративный CA:
# ca_certs_file = /etc/ssl/certs/your-ca.pem
```

> Альтернатива: не трогать файл и передавать `--host`, `--host-bucket`, `--signature-v2` в командах.

---

## 3) Базовые операции

```bash
# Список бакетов
s3cmd ls

# Создать бакет
aws_bucket=my-bkt
s3cmd mb s3://$aws_bucket

# Загрузить файл
s3cmd put ./file.txt s3://$aws_bucket/

# Скачать файл
s3cmd get s3://$aws_bucket/file.txt ./

# Листинг внутри бакета
s3cmd ls s3://$aws_bucket/

# Удалить объект
s3cmd del s3://$aws_bucket/file.txt

# Удалить пустой бакет
s3cmd rb s3://$aws_bucket
```

**sync** — зеркалирование каталога (внимание на `--delete-removed`):

```bash
s3cmd sync ./data/ s3://$aws_bucket/data/ --delete-removed --acl-private --no-preserve
```

Полезные флаги:

- `--acl-private` / `--acl-public` — явные ACL при загрузке.
- `--multipart-chunk-size-mb=64` — размер частей для больших файлов.
- `--dry-run` — «показать, что сделает», без изменений.
- `--no-progress` — тише в логах.

---

## 4) Явный endpoint в команде (если не настроен в .s3cfg)

```bash
s3cmd ls \
  --host=s3.example.com \
  --host-bucket=s3.example.com \
  --signature-v2=0 \
  --ssl
```

> Для path‑style `--host-bucket` должен совпадать с `--host`.

---

## 5) Метаданные, ACL, policy (минимум)

```bash
# Узнать информацию об объекте
s3cmd info s3://$aws_bucket/file.txt

# Поставить публичный доступ ТОЛЬКО на конкретный объект
s3cmd setacl s3://$aws_bucket/public.png --acl-public

# Политика бакета из JSON (пример файла policy.json рядом)
s3cmd setpolicy policy.json s3://$aws_bucket
```

> Публичные ACL/политики используйте осознанно. Для Ceph RGW предпочтительно закрыто по умолчанию (private).

---

## 6) Траблшутинг

- **ERROR: S3 error: 403 (SignatureDoesNotMatch)** — проверьте время на хосте (NTP), `signature_v2 = False`, правильный endpoint, токены.
- **SSL: certificate verify failed** — добавьте ваш CA в систему и укажите `ca_certs_file`.
- **Bucket '...' does not exist** — создайте бакет, проверьте имя/регион.
- **Очень медленный листинг** — это RGW‑метаданные/индекс: избегайте «миллионов» мелких файлов в одном префиксе, шардируйте путь.

---

## 7) Шпаргалка

```bash
# Рекурсивная загрузка каталога с минимальными правами
s3cmd put -r ./folder s3://$aws_bucket/folder/ --acl-private

# Синхронизация только новых/изменённых за 7 дней
find ./src -type f -mtime -7 -print0 | xargs -0 -I{} s3cmd put "{}" s3://$aws_bucket/src/

# «Сухой» прогон sync перед продовой заливкой
s3cmd sync ./data/ s3://$aws_bucket/data/ --dry-run
```

---

## 8) Безопасность

- Не храните `.s3cfg` в общедоступных репозиториях; права 600.
- Для CI/CD — отдельные ключи, минимальные политики, ротация.
- Всегда используйте HTTPS; свой CA — в доверенные.
