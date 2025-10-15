## 1) Установка

Официальная страница: _Install or update to the latest version of the AWS CLI_ (amazon.com). Поставьте **AWS CLI v2** для вашей ОС.

Проверка:

```bash
aws --version
```

---

## 2) Базовая настройка (профиль `rgw`)

**Важно для Ceph RGW:** часто нужен **path‑style** и явный **endpoint**.

```bash
aws configure --profile rgw
# Введите:
# AWS Access Key ID: <ваш_access_key>
# AWS Secret Access Key: <ваш_secret_key>
# Default region name: us-east-1 (или оставьте пустым)
# Default output format: json
```

### Файлы конфигурации

`~/.aws/credentials`

```ini
[rgw]
aws_access_key_id = <ACCESS_KEY>
aws_secret_access_key = <SECRET_KEY>
```

`~/.aws/config`

```ini
[profile rgw]
region = us-east-1
# ВАЖНО: endpoint вашего RGW
s3 =
    endpoint_url = https://s3.example.com
    addressing_style = path    # для Ceph обычно нужен path‑style
# (Опционально) отключить SSL‑проверку, если самоподписанный сертификат
# ca_bundle = /etc/ssl/certs/ca-bundle.crt   # лучше добавить свой CA, чем отключать
```

> Альтернатива: указывать `--endpoint-url` в каждой команде.

Также можно использовать переменную окружения (на один сеанс):

```bash
export AWS_PROFILE=rgw
export AWS_ENDPOINT_URL=https://s3.example.com
```

---

## 3) Быстрый старт: команды

С профилем `rgw` (либо добавляйте `--profile rgw`).

```bash
# Список бакетов
aws s3api list-buckets --profile rgw

# Создание бакета
aws s3api create-bucket --bucket demo-bkt --profile rgw

# Загрузка файла
aws s3 cp ./12345.txt s3://demo-bkt/ --profile rgw

# Листинг в бакете
aws s3 ls s3://demo-bkt/ --profile rgw

# Скачивание
aws s3 cp s3://demo-bkt/12345.txt ./ --profile rgw

# Удаление объекта
aws s3 rm s3://demo-bkt/12345.txt --profile rgw

# Удаление пустого бакета
aws s3api delete-bucket --bucket demo-bkt --profile rgw
```

> Если получаете ошибки — попробуйте добавить `--endpoint-url https://s3.example.com` прямо в команду.

---

## 4) Полезные приёмы

### 4.1 Presigned URL (временная ссылка на скачивание)

```bash
aws s3 presign s3://demo-bkt/12345.txt \
  --expires-in 3600 \
  --profile rgw
```

### 4.2 Мультичасть‑загрузка больших файлов (надёжнее)

AWS CLI делает это автоматически при `aws s3 cp` для крупных файлов. Можно задать пороги:

```bash
aws s3 cp big.iso s3://demo-bkt/ \
  --expected-size 20GB \
  --profile rgw
```

### 4.3 Синхронизация директорий

```bash
aws s3 sync ./backup/ s3://demo-bkt/backup/ --delete --profile rgw
```

### 4.4 Версионирование и жизненный цикл

```bash
# Включить версионирование
aws s3api put-bucket-versioning \
  --bucket demo-bkt \
  --versioning-configuration Status=Enabled \
  --profile rgw

# Посмотреть статусы версий
aws s3api list-object-versions --bucket demo-bkt --profile rgw
```

(Политики lifecycle обычно задаются JSON‑файлом; храним их рядом с проектом.)

---

## 5) Типичные грабли и их фиксы

- **SignatureDoesNotMatch / 403** — проверьте ключи, время на хосте (NTP!), `addressing_style=path`, регион `us-east-1`.
- **Could not connect / SSL** — верный ли `endpoint_url`? Установлен ли ваш корпоративный/самоподписанный CA в систему и в AWS CLI (`ca_bundle`)?
- **NoSuchBucket** — бакет не создан или опечатка; в Ceph RGW имена бакетов глобально уникальны на эндпоинте.
- **ListObjectsV2 странно медленный** — проверьте latency пула метаданных RGW, индекс‑операции; избегайте «миллионов» мелких файлов в одном префиксе, используйте префиксирование.

---

## 6) Короткая памятка: s3 vs s3api

- `aws s3` — удобные high‑level команды (cp/sync/ls/rm), автоконвертируют multipart.
- `aws s3api` — низкоуровневый доступ к API (list-buckets, put-bucket-versioning, policies и т. д.).

---

## 7) Отладка

Добавляйте `--debug` к командам, чтобы увидеть HTTP‑запросы/подписи.  
Пример:

```bash
aws s3 ls s3://demo-bkt/ --profile rgw --debug 2>debug.log
```

---

## 8) Безопасность

- Никогда не коммитьте `~/.aws/credentials` в репозитории.
- Для скриптов/CI используйте отдельные ключи и ограниченные политики.
- Ротируйте ключи, включайте версионирование/логирование в бакетах.

---

## 9) Ссылки

- Офиц. AWS CLI v2 (страница установки) - https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- Документация Ceph RGW (Reef) - https://docs.ceph.com/en/latest/mgr/rgw/
