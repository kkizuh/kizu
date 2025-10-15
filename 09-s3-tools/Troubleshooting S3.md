# Troubleshooting S3 (Ceph RGW) — чеклист и рецепты

> Коротко, по делу. От симптома к причине, с готовыми командами. Фокус на Ceph RGW и S3‑совместимых клиентах (awscli/s3cmd/rclone, SDK).

---

## 0) Быстрый чеклист (перед паникой)

1. **Время на хосте**: `ntpstat || timedatectl` — разъезд >5 мин → 403/SignatureDoesNotMatch.
2. **Endpoint/стиль адресации**: для RGW почти всегда нужен **path‑style**.
3. **Ключи и права**: верный `access/secret`, нет ли опечаток, действуют ли политики/ACL.
4. **TLS**: корректный сертификат/CA.
5. **Сеть**: пинги/RTT, MTU/фрагментация, балансировщик (L4/L7, sticky), DNS.
6. **RGW/пулы метаданных**: нет ли проблем с latency/шардированием индекса.

---

## 1) Аутентификация / подпись

### Симптомы

- `403 Forbidden`, `SignatureDoesNotMatch`, `InvalidAccessKeyId`, `AccessDenied`.
### Причины и фикс

- **Clock skew**: синхронизируйте время (NTP/chrony).
- **Signer v2/v4**: используйте **v4**. В awscli это дефолт; в s3cmd выключите `signature_v2`.
- **Path‑style**: включите `addressing_style=path` (awscli), `host_bucket=s3.example.com` (s3cmd), `force_path_style=true` (rclone).
- **Неправильный endpoint**: явно укажите `--endpoint-url` / `--host`.
- **Проблемные символы в ключе/секрете**: проверьте кавычки/экранирование в конфиге.

### Диагностика

```bash
# awscli: детальный дебаг в файл
aws s3 ls --endpoint-url https://s3.example.com --debug 2>debug.log

# s3cmd: verbose и тест подписи
s3cmd ls --host=s3.example.com --host-bucket=s3.example.com -d 2>debug.log
```

---

## 2) DNS / endpoint / редиректы

### Симптомы

- `301 Moved Permanently`, странные редиректы на `bucket.s3…`.
### Причины и фикс

- Клиент пытается **virtual‑hosted style** (bucket в хосте), а вы слушаете только **path‑style**. Принудите path‑style.
- Неверный `host_bucket` (s3cmd) или не тот `endpoint_url` (awscli).

---

## 3) TLS/SSL

### Симптомы

- `SSL: certificate verify failed`, ругань на CA/hostname.

### Причины и фикс

- Самоподписанный/корпоративный CA: добавьте в trust store системы и клиента (awscli `ca_bundle`, s3cmd `ca_certs_file`, rclone `ca_cert`).
- Несовпадение CN/SAN хоста и фактического имени — чините сертификат или используйте правильный hostname.

---

## 4) Бакеты и права

### Симптомы

- `NoSuchBucket`, `AccessDenied` при `mb`/`ls`/`put`.
### Причины и фикс

- Бакет не создан/опечатка — создайте/проверьте имя.
- Политики/ACL запрещают операцию — проверьте `radosgw-admin` (права пользователя), bucket policy.

### Команды

```bash
# Проверить бакеты от имени пользователя
radosgw-admin bucket list --uid=<user>

# Статистика бакета
radosgw-admin bucket stats --bucket=<name>
```

---

## 5) Медленный листинг / операции на мелких объектах

### Симптомы

- `aws s3 ls`/`ListObjectsV2` и `sync` ползут.

### Причины и фикс

- Индекс/метаданные RGW на медленных дисках — держите их на быстрых (NVMe).
- «Миллионы» файлов в одном префиксе — используйте префиксирование/папки.
- Недостаточный параллелизм клиента — увеличьте `--max-concurrent-requests` (awscli через конфиг), `--transfers`/`--checkers` (rclone), `--multipart-chunk-size`.
- RGW под L7 LB без keepalive/буферов — настройте keepalive, увеличьте буферы.

### Быстрое сравнение/синк

```bash
# rclone быстрее для миграций
rclone sync src:bucket dst:bucket --transfers 16 --checkers 32 --s3-chunk-size 64M --progress
```

---

## 6) Multipart upload / большие файлы

### Симптомы

- Обрывы/таймауты на >5–10 ГБ, `IncompleteBody`.

### Причины и фикс

- MTU/фрагментация в сети — проверьте `tracepath`/`mturoute`, настройте MTU end‑to‑end.
- Слишком маленькие части — поднимите `--multipart-chunk-size` (s3cmd), `--s3-chunk-size` (rclone).
- Балансировщик рвёт длительные коннекты — увеличьте `timeout client/server` (HAProxy/Nginx), включите keepalive.

---

## 7) Версионирование / lifecycle / delete‑markers

### Симптомы

- «Удалил файл, но он всё ещё есть», «не очищается бакет».
### Причины и фикс

- Включено **versioning** — удаление создаёт delete‑marker, а не удаляет версии.
- Lifecycle не подхватывается — проверьте JSON и права.

### Команды

```bash
# Показать версии
aws s3api list-object-versions --bucket <bkt> --endpoint-url https://s3.example.com

# Удалить все версии (осторожно)
aws s3api delete-objects --bucket <bkt> --delete file://del.json
# где del.json сформирован из list-object-versions
```

---

## 8) CORS / браузерные клиенты

### Симптомы

- Ошибки в консоли браузера при прямом доступе (JS/SPA), `CORS`.

### Фикс

- Настройте CORS на бакете (разрешённые origin/методы/заголовки).

```bash
aws s3api put-bucket-cors --bucket <bkt> --cors-configuration file://cors.json \
  --endpoint-url https://s3.example.com
```

---

## 9) Логи, метрики и где смотреть

- **RGW логи:** `/var/log/ceph/ceph-client.rgw.*.log` или journald (`journalctl -u ceph-* | rg rgw`).
- **radosgw-admin:** пользователи, бакеты, индексы, reshard.
- **Ceph health:** `ceph -s`, `ceph health detail`, `ceph df`, `ceph osd perf`.
- **Prometheus/mgr‑dashboard:** latency, ops, 5xx/4xx.

---

## 10) Индекс бакетов и reshard

### Симптомы

- Листинг/операции на бакете очень медленные, особенно с большим числом ключей.
### Фикс

- Проверьте шардирование индекса и при необходимости увеличьте (в простоe):

```bash
# Текущие шары индекса
radosgw-admin bucket stats --bucket <bkt> | jq .num_shards

# План reshard
radosgw-admin bucket reshard --bucket <bkt> --num-shards 64 --yes-i-really-mean-it
```

> Делайте в окно обслуживания; следите за нагрузкой.

---

## 11) Балансировщики/прокси

- Включите **keepalive** и увеличьте таймауты.
- Пробрасывайте нужные заголовки (`Host`, `Date`, `Authorization`).
- Не ломайте `Expect: 100-continue` — это влияет на multipart.

---

## 12) Быстрые шаблоны команд

```bash
# awscli с явным endpoint и профилем
aws s3 ls s3://<bkt>/ --profile rgw --endpoint-url https://s3.example.com --no-sign-request=false

# s3cmd «path-style на жёстко»
s3cmd ls --host=s3.example.com --host-bucket=s3.example.com --ssl

# rclone листинг с path-style
rclone lsd rgw: --s3-force-path-style --s3-endpoint https://s3.example.com
```

---

## 13) Чек‑лист перед инцидентом (для runbook)

- Записать: время/эндпоинт/клиент/ошибку/размеры/RTT.
- Сохранить `--debug`/логи RGW, одну проблемную операцию воспроизвести curl‑ом.
- Проверить ceph health и RGW daemons (число инстансов, балансировка LBs).
- Проверить индексы бакета и pgs у пулов метаданных.
