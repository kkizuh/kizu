# Docker: Build + Publish + Security (с нуля, по-человечески)

Коротко и по делу: как собрать образ, запушить его в реестр и не выстрелить себе в ногу. Примеры минимальны, комментариев много.

---

## 1) Базовая сборка и теги

```bash
# Собрать в текущей папке (ищет Dockerfile)
docker build -t myapp:1.0 .

# Добавить удобный ярлык latest
docker tag myapp:1.0 myapp:latest

# Проверить слои и размер
docker image ls
docker history myapp:1.0
```

**Что важно понимать**

- Образ = слои. Каждый `RUN/COPY/ADD` в Dockerfile создаёт слой. Меньше слоёв → быстрее тянется/кешируется.
- Тег — это просто имя для ссылки на конкретный образ. Один образ может иметь несколько тегов.

---

## 2) Buildx: multi-arch и кеш

Почему нужно: у тебя есть amd64 (x86_64) и arm64 (Apple Silicon/Graviton). Хочешь один тег, который «знает» обе архитектуры.

```bash
# 1) Включаем buildx (один раз)
docker buildx create --use

# 2) Сборка multi-arch + пуш сразу в реестр
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/ORG/myapp:1.0 \
  -t ghcr.io/ORG/myapp:latest \
  --push .
```

Кеш между сборками (ускоряет CI):

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from=type=registry,ref=ghcr.io/ORG/myapp:buildcache \
  --cache-to=type=registry,ref=ghcr.io/ORG/myapp:buildcache,mode=max \
  -t ghcr.io/ORG/myapp:1.0 --push .
```

---

## 3) SBOM и скан уязвимостей

**SBOM** = список пакетов внутри образа (что именно ты в мир выкатываешь).

```bash
# Показать SBOM (Software Bill of Materials)
docker sbom ghcr.io/ORG/myapp:1.0
```

Скан уязвимостей — используй любой сканер, который есть под рукой в CI (например, `docker scout`, Trivy, Grype). Суть: прогнать и зафейлить пайплайн при критах.

---

## 4) Registry: login / tag / push / digest

```bash
# Логин в реестр
docker login ghcr.io         # или registry.gitlab.com / your-registry:5000

# Тегирование под реестр
docker tag myapp:1.0 ghcr.io/ORG/myapp:1.0

# Публикация
docker push ghcr.io/ORG/myapp:1.0
docker push ghcr.io/ORG/myapp:latest
```

**Digest (неподвижная ссылка)**

```bash
# Узнать digest опубликованного образа
docker inspect --format='{{index .RepoDigests 0}}' ghcr.io/ORG/myapp:1.0
# Пример ссылки по digest: ghcr.io/ORG/myapp@sha256:abcd...
# Это всегда один и тот же образ, в отличие от «latest».
```

**Тянуть по digest (надёжно для прода)**

```bash
docker pull ghcr.io/ORG/myapp@sha256:ABCD...
```

---

## 5) Паблиш-чеклист (перед push в прод)

- **База образа**: `*-slim` или distroless (если умеешь). Меньше пакетов → меньше атак.
- **Версии зафиксированы**: не «latest» в Dockerfile/CI. Повторяемость сборки решает.
- **Многостадийная сборка** (multi-stage): билды/компиляторы — в первом, итог — минимальный.
- **Не root**: `USER 10001`/`USER app`.
- **Healthcheck**: контейнер сам сигналит, что жив/болен.
- **Labels (OCI)**: `org.opencontainers.image.source`, `version`, `revision`, `created`.
- **Нет секретов в `ENV/ARG`** и в слоях (`RUN echo "secret"` — зло).
- **Размер образа**: цель <300 MB (для бэкендов Go/Node норма — десятки мегов).
- **SBOM+scan** пройдены.
- **Read-only rootfs** (если сервис готов к этому).

---

## 6) Безопасность образа и рантайма

### Dockerfile (минимум)

```dockerfile
# пример на Python
FROM python:3.12-slim

# Создаём юзера 10001 и рабочую директорию
RUN useradd -u 10001 -m app && mkdir -p /app && chown -R 10001:10001 /app
WORKDIR /app

# Копим зависимости отдельно (лучше кешируется)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копим код
COPY . .

# Не root
USER 10001

# Healthcheck (опционально можно в Compose)
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health').read()" || exit 1

CMD ["python", "app.py"]  # или gunicorn/uvicorn
```

### Запуск (жёстче, но безопаснее)

```bash
docker run -d --name myapp -p 8000:8000 \
  --read-only \                      # файловая система только для чтения
  --tmpfs /tmp \                     # временная запись в tmpfs
  --cap-drop ALL \                   # убрать все Linux-capabilities
  --cap-add NET_BIND_SERVICE \       # если надо слушать порт <1024
  --security-opt no-new-privileges \ # запрет повышения привилегий
  ghcr.io/ORG/myapp:1.0
```

---

## 7) Секреты: как правильно

**Никогда** не клади секреты в `ENV`, `ARG`, `COPY`, `.dockerignore` с дырками. Варианты:

### В рантайме через env-file (простенько, не идеально)

```bash
# .env (НЕ коммитить!)
DB_URL=postgres://user:pass@db/mydb

docker run --env-file .env ghcr.io/ORG/myapp:1.0
```

### Docker Compose — secrets (лучше)

```yaml
# docker-compose.yml
services:
  app:
    image: ghcr.io/ORG/myapp:1.0
    secrets: [db_pass]
    environment:
      DB_PASS_FILE: /run/secrets/db_pass   # приложение читает файл, не ENV
secrets:
  db_pass:
    file: ./secrets/db_pass.txt            # права 0400, .gitignore
```

Внутри контейнера секрет доступен как файл `/run/secrets/db_pass`.

> Ещё варианты: внешние секреты (Vault, AWS/GCP/Azure secrets manager) — уже по месту.

---

## 8) Мини-траблшут

- **ImagePullBackOff** — проверь `docker login`, имя реестра, права на репозиторий, DNS.
- **CrashLoopBackOff** — `docker logs <name>`, код выхода `docker inspect -f '{{.State.ExitCode}}'`.
- **Порты** — `-p 8080:8080` и сервис действительно слушает `0.0.0.0:8080` внутри.
- **Права на volume** — UID процесса в контейнере должен владеть файлами на хосте:
```bash
    id -u  # в контейнере
    sudo chown -R 10001:10001 /data
    ```
- **Размер** — `docker image ls`, чистка: `docker system prune -af --volumes`.

---

## 9) Шаблон минимального Dockerfile (multi-stage, Go/Node — как пример)

**Go**

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app ./cmd/app

FROM gcr.io/distroless/static:nonroot
WORKDIR /app
COPY --from=build /src/app /app/app
USER nonroot
EXPOSE 8080
HEALTHCHECK CMD ["/app/app","-healthcheck"]
ENTRYPOINT ["/app/app"]
```

**Node**

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
USER node
EXPOSE 3000
HEALTHCHECK CMD node healthcheck.js
CMD ["node","server.js"]
```

---

## 10) Быстрый план «как выкатывать по-умному»

1. **Собери** multi-arch с кешем → `buildx build --push`.
2. **Проверь** SBOM и скан уязвимостей.
3. **Запусти** контейнер локально с жёсткими флагами (`--read-only`, `--cap-drop ALL`).
4. **Пушни** в реестр и **запини** деплой по digest.
5. **Мониторь** `HEALTHCHECK`, логи, размер образа и частоту скачиваний.



**multi-arch — это один тег образа для разных архитектур (amd64/arm64 и т.п.)**
### Зачем
- Один `myapp:1.0` работает и на x86_64 (серверы/CI), и на ARM64 (Mac M-чипы, Graviton).
- Регистр хранит **manifest list**: каталог, где под одним тегом лежат несколько вариаций образа.
- Клиент сам тянет «свой» вариант по архитектуре хоста. Магии нет — это просто правильные манифесты.

### Как сделать

```bash
# Включить buildx (один раз)
docker buildx create --use

# Собрать и запушить для двух архитектур
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/ORG/myapp:1.0 \
  -t ghcr.io/ORG/myapp:latest \
  --push .
```

> `--platform` перечисляет целевые архитектуры. `--push` обязателен, чтобы собрать manifest list в реестре.

### Как это выглядит

- Тег `myapp:1.0` в реестре → **manifest list** → записи на:
    - `myapp@sha256:<digest-amd64>` (слои для amd64)
    - `myapp@sha256:<digest-arm64>` (слои для arm64)
- При `docker pull myapp:1.0` клиент читает список и скачивает нужный вариант.

### Проверка

```bash
# Показать манифест (должен быть "manifest list")
docker buildx imagetools inspect ghcr.io/ORG/myapp:1.0
```

### Когда пригодится

- Разработчики на Mac M-чипах, прод на x86_64.
- K8s-кластеры c узлами разных архитектур.
- Паблик-образы, которые должны «заводиться везде».

### Нюансы/подводные

- Нативные сборки быстрее. Кросс-сборка через QEMU может тормозить.
- Библиотеки: для `alpine` (musl) и `debian/ubuntu` (glibc) иногда нужны разные зависимости; нативные аддоны (`node-gyp`, `pip` с C-расширениями) должны собираться под каждую архитектуру.
- Фиксируй версии базовых образов: `node:20-alpine`, `python:3.12-slim` и т.д., не `latest`.

### Compose/K8s

```yaml
# docker-compose.yml
services:
  app:
    image: ghcr.io/ORG/myapp:1.0
    # обычно platform НЕ указывают — докер сам выберет
    # platform: linux/arm64    # принудительно, если очень надо
```

### Быстрый чек-лист

-  `buildx` включён
-  `--platform` перечислены (`linux/amd64,linux/arm64`)
-  `--push` используется
-  `imagetools inspect` показывает manifest list
-  Зависимости/аддоны собираются на обеих архитектурах

Коротко: **multi-arch → один тег, много архитектур, меньше боли.**
# SBOM (Software Bill of Materials) — по-простому

**Что это:** список всего «софта» внутри образа/приложения: пакеты, версии, хэши, источники. Как накладная для ПО.  
**Зачем:** прозрачность, безопасность, лицензии, быстрый отклик на 0-day (“нашли уязвимость в X — где у нас X?”).

## Форматы

- **SPDX** (`*.spdx.json`) — де-факто стандарт Linux Foundation.
- **CycloneDX** (`*.cdx.json`) — от OWASP, тоже очень популярен.

## Минимум, что должно быть в SBOM

- Идентификатор образа/билда (digest), дата, генератор.
- Список пакетов + версии + источник (apk/deb/pip/npm/…).
- Хэши файлов (желательно).
- Лицензии (если есть).

---

## Быстрый старт (Docker)

### 1) Сгенерировать SBOM для уже собранного образа

```bash
# Встроенная команда Docker (под капотом Syft)
docker sbom myrepo/myapp:1.2 \
  -o spdx-json > sbom.spdx.json
# альтернатива: CycloneDX
docker sbom myrepo/myapp:1.2 \
  -o cyclonedx-json > sbom.cdx.json
```

### 2) Генерировать SBOM при сборке (Buildx/BuildKit)

```bash
# Создать/включить buildx один раз
docker buildx create --use

# Сборка + публикация SBOM как OCI-аттестации
docker buildx build \
  --attest=type=sbom \
  --provenance=true \
  -t ghcr.io/org/myapp:1.2 \
  --push .
```

> `--attest=type=sbom` — прикрепит SBOM к образу в реестре,  
> `--provenance=true` — добавит SLSA-provenance (кто/как собрал).

### 3) Посмотреть SBOM у образа в реестре

```bash
# Тянет SBOM из OCI-аттестации
docker sbom ghcr.io/org/myapp:1.2
```

---

## Инструменты

- **Syft** (Anchore): генерация SBOM локально/в CI.
```bash
    syft packages ghcr.io/org/myapp:1.2 -o spdx-json > sbom.spdx.json
    ```
- **Trivy** (Aqua): может делать и SBOM, и скан уязвимостей.
```bash
    trivy image --format spdx-json -o sbom.spdx.json ghcr.io/org/myapp:1.2
    ```
- **Grype/Trivy**: скан уязвимостей по SBOM/по образу.
```bash
    grype sbom:sbom.spdx.json
    # или
    trivy sbom sbom.spdx.json
    ```

---

## Где хранить

- **В реестре** как **OCI attestation** (лучше всего).    
- В артефакт-хранилище (S3/Artifacts).
- В релизе репозитория (GitHub/GitLab Releases).

---

## Подпись и доверие (опционально, но круто)

Подписать SBOM/аттестацию:

```bash
# Подписать образ (cosign keyless OIDC или ключом)
cosign sign ghcr.io/org/myapp:1.2

# Прикрепить подписанную SBOM как attestation
cosign attest \
  --predicate sbom.spdx.json \
  --type spdx \
  ghcr.io/org/myapp:1.2
```

---

## Включаем в CI (скелет)

```yaml
# GitHub Actions (пример)
- name: Build & push with SBOM
  run: |
    docker buildx create --use
    docker login ghcr.io -u $GH_USER -p $GH_TOKEN
    docker buildx build \
      --attest=type=sbom --provenance=true \
      --platform linux/amd64,linux/arm64 \
      -t ghcr.io/org/myapp:${GITHUB_SHA::7} \
      -t ghcr.io/org/myapp:latest \
      --push .

- name: Verify SBOM exists
  run: docker sbom ghcr.io/org/myapp:latest -o spdx-json | jq . > sbom.spdx.json

- name: Scan vulnerabilities from SBOM
  run: trivy sbom --exit-code 1 sbom.spdx.json
```

---

## Бытовые советы

- Генерируй **всегда** (дёшево и бесценно при инцидентах).
- Один формат на проект (напр. SPDX), чтобы не плодить зоопарк.
- Закрепляй версии базовых образов/зависимостей — SBOM будет стабильнее.
- Храни SBOM рядом с образом (attestation) — «приезжает» вместе с ним.
- Автоматическая проверка SBOM в CI (лицензии/вульны) — must-have.

**Итог:** SBOM = инвентаризация ПО. Помогает быстро отвечать на «где у нас уязвимый пакет?», ускоряет аудит и даёт прозрачность сборок.