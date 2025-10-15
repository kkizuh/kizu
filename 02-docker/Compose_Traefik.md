# Compose + Traefik (HTTPS auto, HSTS, редиректы)

## 1) Файлы и структура

```
project/
├─ docker-compose.yml
├─ .env
├─ traefik/
│  ├─ traefik.yml          # статическая конфигурация
│  └─ dynamic.yml          # динамика: middleware/headers/rate-limit
└─ acme/                   # для acme.json (сертификаты)
```

## 2) `.env`

```env
TRAEFIK_EMAIL=admin@example.com       # почта для Let's Encrypt
TRAEFIK_DOMAIN=example.com            # базовый домен (опц.)
TRAEFIK_ACME_STAGING=false            # true = тестовый CA (без лимитов)
```

## 3) `docker-compose.yml`

```yaml
version: "3.9"

networks:
  web:
    external: false    # внутренняя сеть для traefik + сервисы

volumes:
  traefik_state:       # acme.json и прочее состояние
  traefik_logs:

services:
  traefik:
    image: traefik:v3.1
    container_name: traefik
    command:
      # провайдеры
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false" # только с label=enable=true
      - "--providers.file.filename=/traefik/dynamic.yml"

      # entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"

      # редирект http -> https
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"

      # ACME (Let's Encrypt)
      - "--certificatesresolvers.le.acme.email=${TRAEFIK_EMAIL}"
      - "--certificatesresolvers.le.acme.storage=/acme/acme.json"
      - "--certificatesresolvers.le.acme.httpchallenge=true"
      - "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web"
      # для тестов (не ловить rate limit LE)
      - "--certificatesresolvers.le.acme.caserver=${TRAEFIK_ACME_STAGING:-false}"
    # NOTE: для staging включи переменную как:
    # TRAEFIK_ACME_STAGING=https://acme-staging-v02.api.letsencrypt.org/directory

    ports:
      - "80:80"      # http
      - "443:443"    # https
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/traefik.yml:/traefik/traefik.yml:ro
      - ./traefik/dynamic.yml:/traefik/dynamic.yml:ro
      - traefik_state:/acme
      - traefik_logs:/logs
    networks:
      - web
    restart: unless-stopped
    labels:
      traefik.enable: "true"
      # dashboard (закрываем базовой auth/whitelist в dynamic.yml)
      traefik.http.routers.api.rule: "Host(`traefik.${TRAEFIK_DOMAIN}`)"
      traefik.http.routers.api.entrypoints: "websecure"
      traefik.http.routers.api.tls.certresolver: "le"
      traefik.http.routers.api.service: "api@internal"
      traefik.http.routers.api.middlewares: "secHeaders@file,auth@file,whitelist@file"

  # Демка: кто я (проверка TLS и маршрутизации)
  whoami:
    image: traefik/whoami:v1.10
    networks: [web]
    restart: unless-stopped
    labels:
      traefik.enable: "true"
      traefik.docker.network: "web"

      # HTTP router (не публикуем, т.к. уже редирект в traefik)
      traefik.http.routers.whoami.rule: "Host(`whoami.${TRAEFIK_DOMAIN}`)"
      traefik.http.routers.whoami.entrypoints: "websecure"
      traefik.http.routers.whoami.tls.certresolver: "le"
      traefik.http.routers.whoami.middlewares: "secHeaders@file,gzip@file"

  # Твой backend (пример)
  app:
    image: myorg/app:1.2.3
    networks: [web]
    environment:
      - APP_ENV=prod
    restart: unless-stopped
    labels:
      traefik.enable: "true"
      traefik.docker.network: "web"

      traefik.http.routers.app.rule: "Host(`app.${TRAEFIK_DOMAIN}`)"
      traefik.http.routers.app.entrypoints: "websecure"
      traefik.http.routers.app.tls.certresolver: "le"
      traefik.http.routers.app.middlewares: "secHeaders@file,gzip@file,ratelimit@file" # пример
```

## 4) `traefik/traefik.yml` (статическая)

```yaml
api:
  dashboard: true

log:
  level: INFO
  filePath: /logs/traefik.log

accessLog:
  filePath: /logs/access.log
  bufferingSize: 0

# можно включить metrics (prometheus), если надо
# metrics:
#   prometheus: {}
```

## 5) `traefik/dynamic.yml` (динамика: хедеры, gzip, auth, whitelist)

```yaml
http:
  middlewares:
    # Security headers (HSTS включён!)
    secHeaders:
      headers:
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
        contentTypeNosniff: true
        browserXssFilter: true
        frameDeny: true
        referrerPolicy: "no-referrer"
        permissionsPolicy: "geolocation=(), microphone=(), camera=()"

    # gzip/deflate (авто)
    gzip:
      compress: {}

    # Базовая auth на дашборд (создай htpasswd: `htpasswd -nb user pass`)
    auth:
      basicAuth:
        users:
          - "admin:$apr1$NAbCDe12$QwErTyUiOpasdfghjk"   # пример, замени!

    # Белый список по IP (доп. защита дашборда)
    whitelist:
      ipWhiteList:
        sourceRange:
          - "10.0.0.0/8"
          - "192.168.0.0/16"
          - "127.0.0.1/32"

    # Пример лимита запросов для app
    ratelimit:
      rateLimit:
        average: 50
        burst: 100
```

## 6) Инициализация и запуск

```bash
mkdir -p acme
# acme.json должен быть 600, иначе Traefik ругается
touch acme/acme.json && chmod 600 acme/acme.json

docker compose up -d
```

Дальше открой:

- `https://traefik.example.com` — дашборд (с базовой auth + whitelist)
- `https://whoami.example.com` — демо сервис
- `https://app.example.com` — твой сервис

> DNS: добавь A/AAAA записи `whoami.example.com`, `app.example.com`, `traefik.example.com` → на IP сервера. Порты 80/443 должны быть доступны извне.

---

## Как работает авто-TLS

- Используем **HTTP-01 challenge**: Traefik слушает `:80` (entrypoint `web`) и отвечает ACME-валидатору по `/.well-known/acme-challenge/...`.
- Let’s Encrypt выдаёт сертификат → Traefik хранит в `acme.json` и **сам продлевает**.
- Для тестов включай staging CA (переменная `TRAEFIK_ACME_STAGING` с URL из коммента), чтобы не ловить rate-limit.

### Альтернатива: DNS-01 (например, Cloudflare)

Если 80/443 закрыты или нужен wildcard `*.example.com`:

```yaml
# ДОБАВЬ в traefik: command:
- "--certificatesresolvers.le.acme.dnschallenge=true"
- "--certificatesresolvers.le.acme.dnschallenge.provider=cloudflare"
# И передай CF_API_TOKEN через env/secret
```

Докинешь токен как:

```yaml
environment:
  - CF_DNS_API_TOKEN=${CF_DNS_API_TOKEN}
```

(и поставь в `.env`)

---

## Полезные приёмы

- **Отключи auto-expose по умолчанию** (мы уже сделали), и мети `traefik.enable=true` только тем, что реально наружу.
- **Проверка конфигов**: `docker logs traefik` — сразу видно, что не так.
- **Rate-limit/Headers** держи в `dynamic.yml`, а маршрутизацию — в labels у сервисов.
- **IPv6**: проверь, что хост слушает и по v6, и что DNS AAAA есть (если надо).
- **gRPC/WS**: Traefik понимает автоматически (HTTP/2, `Connection: Upgrade` не нужен).

---

## Частые ошибки → быстрые фиксы

- **`acme.json` wrong permissions** → `chmod 600 acme/acme.json`.
- **Сертификат не выдаётся** → проверь DNS, что порты 80/443 доступны снаружи, staging выключен, домен указывает на сервер.
- **Dashboard 404** → проверь правило `Host(traefik.${TRAEFIK_DOMAIN})`, и что `TRAEFIK_DOMAIN` в `.env`.
- **Несколько сервисов на один домен/подпуть** — настрой роутер с правилом `PathPrefix(`/api`)` и middleware `StripPrefix`.
