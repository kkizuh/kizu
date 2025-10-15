# Docker Compose: базовый стек

> TL;DR: один `docker-compose.yml` описывает весь мини-кластер. `docker compose up -d` — и всё поднялось.

## Быстрый старт (10 строк)

```yaml
# docker-compose.yml
services:
  app:
    image: nginx:alpine
    ports: ["8080:80"]           # наружу 8080 → контейнеру 80
    volumes: ["./site:/usr/share/nginx/html:ro"]  # локальный код внутрь
```

```bash
docker compose up -d     # запустить
docker compose ps        # что запущено
docker compose logs -f   # логи
docker compose down      # стоп + убрать сеть
```

---

## Стек: Web + App + DB (+ Redis)

```yaml
version: "3.9"

services:
  web:                               # обратный прокси
    image: nginx:alpine
    ports: ["80:80"]
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      app:
        condition: service_healthy    # ждём готовности app (ниже healthcheck)

  app:                               # ваш backend (пример: Node)
    build: ./app
    environment:
      - PORT=3000
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379/0
    expose: ["3000"]                 # доступно ТОЛЬКО внутри compose-сети
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:                      # web не дёргает, пока app не жив
      test: ["CMD", "curl", "-fsS", "http://localhost:3000/health"]
      interval: 10s
      timeout: 3s
      retries: 6

  db:                                 # Postgres с именованным томом
    image: postgres:16
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass        # лучше .env или secrets (см. ниже)
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes: ["redis_data:/data"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 6

volumes:
  db_data:
  redis_data:
```

**Почему так:**

- **`depends_on.condition: service_healthy`** — дождись `healthcheck`, а не просто «контейнер запустился».
- **`expose`** — порт доступен внутри сети, но не наружу; наружу торчит только `web`.
- Именованные **volumes** → данные переживают `down` (без `-v`).

---

## Команды — 90% рутины

```bash
docker compose up -d                    # поднять
docker compose up -d --build            # пересобрать и поднять
docker compose down                     # стоп + удалить сеть
docker compose down -v                  # + снести volumes (данные)

docker compose logs -f app              # хвост логов сервиса
docker compose exec app sh              # интерактив внутрь
docker compose run --rm app bash -lc 'npm run migrate'  # одноразовый

docker compose ps                       # статусы
docker compose top                      # процессы

docker compose up -d --scale app=3      # масштабировать сервис
```

---

## .env и переменные

**`.env` рядом с compose-файлом** подхватывается автоматически:

```
POSTGRES_PASSWORD=supersecret
APP_IMAGE=myorg/app:1.2.3
```

```yaml
services:
  app:
    image: ${APP_IMAGE}
  db:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

> **Не коммить секреты.** Для прод — см. **secrets** ниже.

---

## Сети (когда нужно разделить трафик)

```yaml
services:
  web:
    networks: [edge, backend]   # виден извне и к бэкенду
  app:
    networks: [backend]         # только внутренняя сеть
  db:
    networks: [backend]

networks:
  edge:
    driver: bridge
  backend:
    driver: bridge
```

- По умолчанию Compose создаёт одну сеть `<dir>_default`. Этого обычно хватает.
- Имена сервисов — это DNS-имена: `app` обращается к `db:5432`.

---

## Секреты (безопаснее, чем ENV)

**Вариант 1: встроенные secrets (лучше с Swarm, но и локально ок):**

```
# ./secrets/db_pass.txt  (не коммитим)
```

```yaml
services:
  db:
    image: postgres:16
    secrets: [db_pass]
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_pass

secrets:
  db_pass:
    file: ./secrets/db_pass.txt
```

**Вариант 2: ENV из `.env`** (просто, но аккуратно с Git).  
**Kubernetes**: используй `Secret`, не ENV.

---

## Профили (вкл/выкл сервисы наборами)

```yaml
services:
  grafana:
    image: grafana/grafana
    profiles: ["obs"]   # включится только с профилем obs
```

```bash
docker compose --profile obs up -d
```

---

## Nginx reverse-proxy (минимум)

```nginx
# ./nginx.conf
server {
  listen 80;
  location / {
    proxy_pass         http://app:3000;
    proxy_set_header   Host $host;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-Proto $scheme;
}
}
```

---

## Best practices (коротко)

- **Образы**: конкретные теги (`:3.12-slim`), без `latest`.
- **Healthcheck** у критичных сервисов; `depends_on.condition: healthy`.
- **Открывай наружу только один слой** (обычно `web`/proxy). Остальные — `expose`.
- **Volumes** именованные; бекапни перед `down -v`.
- **Секреты** — не в `docker-compose.yml`. Либо `.env` (локально), либо `secrets`/K8s.
- **Логи**: `docker compose logs -f <svc>`; для прод — логируй в stdout/stderr и собирай агентом
- **Миграции**: отдельным `run --rm` джобом, НЕ в основном контейнере при старте (чтобы не блокировать health).
- **Ресурсы** (ограничения, если нужно):
```yaml
    deploy:
      resources:
        limits: {cpus: "1.0", memory: "512M"}
        reservations: {cpus: "0.25", memory: "128M"}
    ```
    
> 	В Docker Desktop/Compose v2 `deploy` частично игнорится вне Swarm, но полезно для совместимости.
    

---

## Частые боли → фиксы

- **Сервис «запустился», но не готов** → добавь `healthcheck` и `depends_on.condition: healthy`.
- **Порт занят** → `docker ps` → найди, кто слушает; или поменяй `host:container` порт.
- **Данные пропали** → ты делал `down -v`. Держи бэкап томов (`docker run --rm -v db_data:/data alpine tar czf - /data > db.tgz`).
- **Секреты в Git** → вынеси в `.env` (в `.gitignore`), либо `secrets`.
- **Нужно 3 экземпляра app** → `up -d --scale app=3` + proxy на `app:3000`.

---

## Мини-compose для PHP+Nginx+MySQL

```yaml
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes:
      - ./code:/var/www/html:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      php: { condition: service_started }
      db:  { condition: service_healthy }

  php:
    image: php:8.3-fpm
    volumes: ["./code:/var/www/html"]

  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: mydb
      MYSQL_USER: user
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes: ["db_data:/var/lib/mysql"]
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u $$MYSQL_USER -p$$MYSQL_PASSWORD --silent"]
      interval: 10s
      retries: 10
volumes:
  db_data:
```

`.env`:

```
MYSQL_ROOT_PASSWORD=rootpass
MYSQL_PASSWORD=userpass
```
