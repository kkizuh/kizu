# Dockerfile Best (multi-stage, non-root, healthcheck, labels)

> цель: собрать **маленький**, **безопасный** и **воспроизводимый** образ.  
> копируй куски и не страдай.

---

## 0) Базовые принципы

- **Multi-stage**: собираем в “тяжёлом” слое → переносим только артефакт в “лёгкий”.
- **Non-root**: не запускаем под root. Ставим `USER` на фиксированный uid/gid.
- **Маленькая база**: `*-slim` / distroless / UBI-micro. Alpine — ок, но осторожно с glibc/native.
- **Кэш**: lock-файлы копируем первыми, чтобы реже ломать кэш.
- **HEALTHCHECK**: быстрый локальный ping, короткие таймауты.
- **LABEL’ы** (OCI): кто собрал, когда, из какого коммита.
- **.dockerignore** обязателен: не тащи `.git`, `node_modules`, тесты и мусор.

---

## 1) Скелет идеального Dockerfile (язык-агностично)

```Dockerfile
# ---------- build stage: ставим SDK, собираем артефакт ----------
FROM ghcr.io/canonical/ubuntu:24.04 AS build
WORKDIR /src

# 1) зависимости сборки (если нужны)
RUN apt-get update && apt-get install -y --no-install-recommends build-essential ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*

# 2) сначала lock-файлы → устойчивый кэш
COPY go.mod go.sum* package.json package-lock.json* requirements.txt* pyproject.toml* ./
# (скопируется только то, что реально есть)

# 3) установка зависимостей (пример для разных менеджеров)
# RUN npm ci --fund=false --audit=false
# RUN pip install --no-cache-dir -r requirements.txt
# RUN go mod download

# 4) код и сборка
COPY . .
# Пример: Go бинарь без мусора в символах:
# RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /src/app ./cmd/app

# ---------- runtime stage: минимальный рантайм ----------
FROM gcr.io/distroless/base-debian12
# или: FROM debian:bookworm-slim / eclipse-temurin:21-jre / python:3.12-slim / node:20-slim

# 5) нефривольный пользователь (фиксированный uid/gid для совместимости с томами)
USER 10001:10001
WORKDIR /app

# 6) копируем только нужное
COPY --from=build /src/app /app/app

# 7) порты/healthcheck
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["/app/app","-health"]  # подставь свою проверку

# 8) точка входа
ENTRYPOINT ["/app/app"]
```

> Не нужен shell? Distroless — 🔥. Нужен дебаг (sh/curl)? Бери `*-slim`.

---

## 2) .dockerignore (минимум)

```
.git
.gitignore
node_modules
**/__pycache__/
*.pyc
dist
build
coverage
tests
.env
Dockerfile*
docker-compose*.yml
```

> Меньше контекст → быстрее build и меньше случайных секретов внутрь.

---

## 3) Non-root правильно

```Dockerfile
# В slim-образах часто нет пользователя — создадим с фиксированным uid/gid
RUN addgroup --system --gid 10001 app && adduser --system --uid 10001 --ingroup app app
USER 10001:10001
```

**Почему фиксированный uid/gid?**  
Тома и хостовые файлы будут иметь согласованные права без сюрпризов.

---

## 4) HEALTHCHECK — короткий и локальный

```Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/health || exit 1
```

- Делай быстрый endpoint или встроенную проверку (`/app --health`).
- Не делай curl наружу — сеть может быть недоступна при старте.    

---

## 5) LABEL’ы (OCI стандарт)

```Dockerfile
ARG VCS_REF=unknown
ARG BUILD_DATE=unknown
ARG VERSION=dev

LABEL org.opencontainers.image.source="https://github.com/you/repo" \
      org.opencontainers.image.revision="$VCS_REF" \
      org.opencontainers.image.created="$BUILD_DATE" \
      org.opencontainers.image.version="$VERSION" \
      org.opencontainers.image.title="your-app" \
      org.opencontainers.image.description="Short description"
```

> Подставь через CI: `--build-arg VCS_REF=$(git rev-parse HEAD)` и т.п.

---

## 6) ENTRYPOINT vs CMD (не путать)

- **ENTRYPOINT** — «что запускать всегда».
- **CMD** — значения по умолчанию (аргументы), можно переопределить.

```Dockerfile
ENTRYPOINT ["your-binary"]
CMD ["--serve","--port=8080"]
```

Запуск с переопределением:  
`docker run image --port=9090` → заменит CMD, ENTRYPOINT останется.

---

## 7) Кэш: делай слои стабильными

Плохo:

```Dockerfile
COPY . .            # меняется любой файл → ломается весь кэш
RUN npm install     # всегда долго
```

Хорошо:

```Dockerfile
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
RUN npm run build
```

**Go:** сначала `go.mod`/`go.sum` → `go mod download` → `COPY .` → `go build`.

---

## 8) Языковые мини-шаблоны

### Go (distroless runtime)

```Dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o app ./cmd/app

FROM gcr.io/distroless/base-debian12
USER 10001:10001
WORKDIR /app
COPY --from=build /src/app /app/app
EXPOSE 8080
HEALTHCHECK CMD ["/app/app","-health"]
ENTRYPOINT ["/app/app"]
```

### Python (slim, без кеша pip)

```Dockerfile
FROM python:3.12-slim
ENV PIP_NO_CACHE_DIR=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER 10001
EXPOSE 8000
HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')" >/dev/null || exit 1
CMD ["python","app.py"]
```

### Node.js (prod deps, ci + omit dev)

```Dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
USER node
EXPOSE 3000
HEALTHCHECK CMD node health.js || exit 1
CMD ["node","server.js"]
```

### Java (Spring Boot, JRE slim)

```Dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY . .
RUN ./mvnw -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /src/target/app.jar /app/app.jar
USER 10001
EXPOSE 8080
HEALTHCHECK CMD wget -qO- http://127.0.0.1:8080/actuator/health || exit 1
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

### Ruby (Bundler deployment)

```Dockerfile
FROM ruby:3.3-slim
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*
COPY Gemfile Gemfile.lock ./
RUN bundle config set deployment 'true' \
 && bundle config set without 'development test' \
 && bundle install
COPY . .
USER 10001
EXPOSE 3000
CMD ["bundle","exec","puma","-C","config/puma.rb"]
```

---

## 9) Alpine? да, но осторожно

- Плюсы: маленький. Минусы: `musl` вместо `glibc`, **некоторые нативные модули/библиотеки ломаются**.
- Если нужны `glibc`-зависимости (Java/JNI, некоторые Python wheels, Node native) — лучше `*-slim` на Debian/Ubuntu или distroless.

---

## 10) Size/без мусора

- Удаляй списки пакетов: `rm -rf /var/lib/apt/lists/*`.
- Не оставляй build-tools в runtime (multi-stage!).
- Сжимай Go/бинарники: `-ldflags "-s -w"`, `upx` — только если допустимо.
- Цель: **runtime < 300 MB** (реально достигается почти всегда).

---

## 11) Secrets/конфиг — не в образ

- Секреты через **env/файлы/volume**, а не COPY.
- Для прод: менеджер секретов (Vault/KMS/…).
- В compose/k8s: `secrets:`/ConfigMap/Secrets.
- Никогда не коммить `.env` и не COPY его в образ.

---

## 12) BuildKit/кэш-маунты (ускорение сборки)

```Dockerfile
# пример для npm с BuildKit
# syntax=docker/dockerfile:1.7
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev
```

Запуск: `DOCKER_BUILDKIT=1 docker build ...`

---

## 13) Частые грабли и фиксы

- **CrashLoop**: ENTRYPOINT завершается → проверь команду/права/порты/env.
- **Пермишены томов**: non-root не видит файлы → фиксируй uid/gid, `chown` в build, на SELinux используй `:z`/`:Z`.
- **Timezones/locale**: не ставь лишнего. Обычно достаточно `TZ=UTC` и работать в UTC.
- **SIGTERM**: твоё приложение должно ловить SIGTERM и завершаться корректно (graceful). Иначе `docker stop` превратится в `kill`.

---

## 14) Публикация: минимальный чек-лист

- [ ]  Multi-stage, минимальная база
- [ ]  `USER 10001`, `HEALTHCHECK`
- [ ]  `.dockerignore` заполнен
- [ ]  Размер runtime < 300MB
- [ ]  LABEL’ы OCI (`source`, `revision`, `version`, `created`)
- [ ]  Зависимости pinned (lock-files)
- [ ]  Секреты/конфиг **вне** образа
- [ ]  В деплое используем **digest** (`repo/app@sha256:...`)
