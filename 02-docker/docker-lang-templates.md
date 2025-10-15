# Шаблоны Dockerfile: Python / Node / Go / Java / Ruby

> Правила: **минимум базы** (`*-slim` или distroless), **не root** (`USER`), **.dockerignore** обязателен.

---

## Быстрые пояснения

**`USER 10001` — что это и зачем**

- Запускает процесс **не под root**, а под UID **10001** (просто число ≠ 0).
- Безопаснее и предсказуемее с томами (права файлов).
- `USER` не создаёт пользователя и не меняет владельцев. Если нужны права — создай юзера и `chown`.

```Dockerfile
# Debian/Ubuntu slim
RUN addgroup --system --gid 10001 app \
 && adduser --system --uid 10001 --ingroup app app
USER 10001:10001
```

Distroless часто уже имеет непривилегированного `65532`:

```Dockerfile
USER 65532:65532
```

**Alpine — когда да/когда нет**

- - Крошечный, быстрый. − libc=**musl**, нативные модули (Node/Python/Java JNI) могут падать.
        
- **Да:** простые сервисы, статический Go. **Нет:** Java, много Python/Node native deps → бери `*-slim` или distroless.
    

---

## Python (Flask/FastAPI/скрипт)

**Dockerfile**

```Dockerfile
FROM python:3.12-slim
WORKDIR /app
ENV PIP_NO_CACHE_DIR=1 PYTHONDONTWRITEBYTECODE=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN adduser --system --uid 10001 app && chown -R 10001:10001 /app
USER 10001
EXPOSE 8000
CMD ["python","app.py"]   # Flask: ["python","-m","flask","run","--host=0.0.0.0"]
```

**.dockerignore**

```
__pycache__/
*.pyc
.env
.git
```

---

## Node.js (Express/Nest без нативной боли)

**Dockerfile**

```Dockerfile
FROM node:20-bookworm-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
USER node
EXPOSE 3000
CMD ["node","server.js"]
```

**Если нужны native модули на Alpine**

```Dockerfile
FROM node:20-alpine
RUN apk add --no-cache make g++ python3
# дальше как обычно: npm ci / COPY / USER node
```

**.dockerignore**

```
node_modules
npm-debug.log
.git
.env
```

---

## Go (статический бинарь → distroless)

**Dockerfile**

```Dockerfile
# build
FROM golang:1.22 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o app ./cmd/app

# runtime
FROM gcr.io/distroless/base-debian12
WORKDIR /app
COPY --from=build /src/app /app/app
USER 65532:65532   # nonroot в distroless
EXPOSE 8080
ENTRYPOINT ["/app/app"]
```

---

## Java (Spring Boot — jar)

**Dockerfile**

```Dockerfile
# build
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY . .
RUN ./mvnw -DskipTests package

# runtime
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /src/target/*.jar /app/app.jar
RUN adduser --system --uid 10001 app && chown -R 10001:10001 /app
USER 10001
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

**.dockerignore**

```
target
.git
.env
```

---

## Ruby (Puma/Rails — минимально)

**Dockerfile**

```Dockerfile
FROM ruby:3.3-slim
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*
COPY Gemfile Gemfile.lock ./
RUN gem install bundler \
 && bundle config set without 'development test' \
 && bundle config set deployment 'true' \
 && bundle install
COPY . .
RUN adduser --system --uid 10001 app && chown -R 10001:10001 /app
USER 10001
EXPOSE 3000
CMD ["bundle","exec","puma","-C","config/puma.rb"]
```

**.dockerignore**

```
log
tmp
.git
.env
```

---

## Памятка (в любой язык)

- `.dockerignore` всегда (меньше контекста → быстрее build, меньше шансов утечь секретам).
    
- `USER` не root; для томов — фиксированный uid/gid.
    
- `*-slim`/distroless для совместимости; Alpine — осторожно с нативными deps.
    
- По возможности добавь `HEALTHCHECK` (локальный `/health`).
    
- Сначала копируй **lock-файлы** (кэш), потом код.
    

хочешь — добавлю сюда мини-блок для **React статик + nginx** с конфигом кэша.