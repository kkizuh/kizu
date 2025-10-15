# Docker Cheatsheet: build / run / exec / logs / cleanup

> цель: за 5 минут вспомнить основы и не сломать прод.

## 0) Базовые термины

- **image** — шаблон (слои).
- **container** — запущенный экземпляр образа.
- **registry** — хранилище образов (Docker Hub, GHCR, ECR и т.д.).

---

## 1) Build (сборка образа)

```bash
# собрать из текущей папки (где лежит Dockerfile)
docker build -t app:dev .

# указать другой Dockerfile/контекст
docker build -f Dockerfile.prod -t app:prod ./src

# подробный лог сборки (удобно дебажить)
docker build --progress=plain -t app:dbg .

# если кэш мешает
docker build --no-cache -t app:nocache .
```

**Памятка:** положи мусор в `.dockerignore` (node_modules, .git, tests).

---

## 2) Run (запуск контейнера)

```bash
# запустить в фоне, пробросить порт 8080 наружу
docker run -d --name app -p 8080:8080 app:dev

# проброс env и volume (локальная папка внутрь контейнера)
docker run -d --name app \
  -e ENV=prod --env-file .env \
  -v $(pwd)/data:/data \
  -p 8080:8080 app:dev

# запустить и сразу попасть в shell (для отладки)
docker run --rm -it --entrypoint sh app:dev
```

- `-d` — в фоне,
- `--rm` — удалить после выхода,
- `-it` — интерактивный терминал.

---

## 3) Exec (зайти в уже работающий контейнер)

```bash
# bash (если есть)
docker exec -it app bash

# sh (минимальный образ)
docker exec -it app sh

# выполнить команду внутри
docker exec app ls -la /app
```

---

## 4) Logs (логи контейнера)

```bash
# “хвост” логов (как tail -f)
docker logs -f app

# за последний час
docker logs --since 1h app

# последние 200 строк
docker logs --tail 200 app
```

**Ротация логов** (чтобы диск не забили):

```bash
docker run --log-opt max-size=10m --log-opt max-file=5 ...
```

---

## 5) Стоп/удаление

```bash
docker stop app          # мягко остановить
docker kill app          # жёстко прибить
docker rm app            # удалить контейнер
docker rm -f app         # остановить и удалить
```

---

## 6) Список/поиск

```bash
docker ps                # запущенные контейнеры
docker ps -a             # все (включая остановленные)
docker images            # локальные образы
docker image ls          # то же
```

---

## 7) Копирование файлов

```bash
# с хоста в контейнер
docker cp ./local.conf app:/etc/app.conf

# из контейнера на хост
docker cp app:/var/log/app.log ./app.log
```

---

## 8) Inspect (полезная инфа)

```bash
# IP адрес контейнера (bridge-сеть)
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' app

# env контейнера
docker inspect app | jq '.[0].Config.Env'

# порт-маппинги
docker port app
```

---

## 9) Сети (минимум)

```bash
docker network ls
docker network create mynet              # своя bridge-сеть
docker run -d --name api --network mynet app:dev
docker run -it --network mynet alpine ping api
```

---

## 10) Registry (login / tag / push / pull)

```bash
docker login ghcr.io

docker tag app:dev ghcr.io/user/app:1.2.3
docker push ghcr.io/user/app:1.2.3

docker pull ghcr.io/user/app:1.2.3

# получить digest (для детерминизма в деплое)
docker inspect ghcr.io/user/app:1.2.3 --format '{{index .RepoDigests 0}}'
# пример: ghcr.io/user/app@sha256:abcdef...
```

---

## 11) Compose (80% кейсов)

```yaml
# docker-compose.yml
services:
  app:
    image: ghcr.io/user/app:1.2.3
    ports: ["8080:8080"]
    env_file: [.env]
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: example
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL","pg_isready -U postgres"]
volumes: { pgdata: {} }
```

```bash
docker compose up -d
docker compose logs -f app
docker compose down -v     # снять и снести тома (осторожно!)
```

---

## 12) Cleanup (не сломать, но почистить)

```bash
docker system df                          # сколько места занято
docker image prune -f                     # удалить “висячие” образы (dangling)
docker container prune -f                 # удалить остановленные контейнеры
docker volume prune -f                    # удалить неиспользуемые тома
docker system prune -af --volumes         # ЖЕСТКО: всё неиспользуемое + тома
```

> ⚠️ `system prune -af --volumes` может снести данные в томах. Думай!

---

## 13) Быстрый троблшут

```bash
docker logs app                           # что говорит приложение?
docker inspect app | jq '.[0].State'      # почему упал/перезапускается?
docker exec -it app sh                    # глянуть изнутри (конфиги/порты)
ss -ltnp | grep 8080                      # слушает ли порт внутри?
cat /etc/resolv.conf                      # DNS в контейнере
```

---

## 14) Мини-советы (чтобы не страдать)

- Делай `.dockerignore` — меньше контекста → быстрее build.
- В Dockerfile: **multi-stage**, **USER не root**, **HEALTHCHECK**.
- В compose: `depends_on + healthcheck`, `restart: unless-stopped`.
- Логи ротируй (см. п.4) или используй journald/fluentd.
- Для прод деплоев фиксируй **digest** `@sha256:...`, не только `:tag`.
# **Кеш Docker:**  
Dockerfile идёт **сверху вниз**. Любое изменение в строке **№N ломает кеш с неё и ниже**; всё, что **выше**, берётся из кеша. Поэтому **стабильное — вверх** (база, apt, deps по lock-файлам), **частое — вниз** (код, сборка).