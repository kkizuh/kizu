# Reverse Proxy: Caddy vs Traefik vs Nginx

|Критерий|**Caddy**|**Traefik**|**Nginx (OSS)**|
|---|---|---|---|
|**Авто-TLS (Let’s Encrypt)**|✅ из коробки, 1 строка в Caddyfile|✅ через встроенный ACME (HTTP-01/DNS-01)|❌ вручную (certbot/скрипты/cron)|
|**Автодискавер контейнеров**|❌ (нужен плагин caddy-docker-proxy)|✅ через Docker/K8s провайдеры и labels|❌ вручную писать конфиги|
|**Конфиг формат**|`Caddyfile` (очень простой) / JSON|YAML labels / static+dynamic yaml|`nginx.conf` (гибко, но многословно)|
|**Hot-reload без даунтайма**|✅|✅|✅ (через `-s reload`)|
|**HTTP/2 / HTTP/3 (QUIC)**|✅ / ✅|✅ / ✅|✅ / ⚠️ (HTTP/3 только в новых сборках/дистрибутивах)|
|**WebSockets / gRPC**|✅ / ✅|✅ / ✅|✅ / ⚠️ (gRPC требует настройку)|
|**Rate-limit / Middleware**|Базово, плагинами расширяется|✅ богато (headers, ratelimit, ipWhiteList, forwardAuth и т.д.)|Частично (модули/3rd party/NGINX Plus)|
|**Dashboard / UI**|❌ (нет встроенного)|✅ web-dashboard|❌|
|**Metrics / Prometheus**|Плагины|✅ встроено|Через `stub_status`/exporters|
|**Ролевая модель / Multi-tenant**|Больше «simple server»|✅ удобно через labels и роутеры|Вручную секции/включения|
|**Статика (file server)**|✅ из коробки (gzip/zstd, кеш)|❌ (не файловый сервер)|✅ отлично (sendfile, cache, gzip)|
|**Производительность**|Высокая|Высокая|Очень высокая (тонкая настройка)|
|**Кривая обучения**|Низкая (минимум магии)|Средняя (нужно понять провайдеры/labels)|Средняя/Высокая (директивы, тонкости)|
|**Лучше всего подходит**|Быстрое авто-HTTPS, простые прокси/статики|Многосервисные стеки, Docker/K8s, динамика|Тонкий тюнинг, edge-кейсы, старая школа|
|**Секьюрити заголовки out-of-the-box**|✅ удобно|✅ через middleware|❌ руками|
|**DNS-01 (wildcard)**|✅ плагинами провайдеров DNS|✅ встроено|❌ внешние скрипты|
|**Лицензия/стоимость**|OSS|OSS|OSS (NGINX Plus — платно)|

---

## Быстрый выбор (что брать?)

- **Хочу за 5 минут HTTPS и прокси без боли** → **Caddy**.
- **У меня куча контейнеров/домены/стейджи, люблю всё в labels** → **Traefik**.
- **Нужен предельный контроль и производительность, сложные директивы** → **Nginx**.

---

## Короткие шаблоны

**Caddy (auto-HTTPS):**

```caddy
app.example.com {
  encode gzip zstd
  reverse_proxy app:8080
}
```

**Traefik (labels в compose):**

```yaml
labels:
  traefik.enable: "true"
  traefik.http.routers.app.rule: "Host(`app.example.com`)"
  traefik.http.routers.app.entrypoints: "websecure"
  traefik.http.routers.app.tls.certresolver: "le"
```

**Nginx (reverse proxy):**

```nginx
server {
  listen 443 ssl;
  ssl_certificate     /etc/letsencrypt/live/app/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/app/privkey.pem;

  location / {
    proxy_pass http://app:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

---

## Рекомендации по прод-минимуму

- Открой наружу **только proxy**; бэкенды — в внутренней сети.
- TLS: **HTTP-01** (80 открыт) или **DNS-01** (Cloudflare/Route53) для `*.domain`
- Логи — в stdout/stderr; собирай агентом.
- Healthcheck у сервисов; в Traefik завяжи `depends_on.condition: healthy`.
- HSTS/Headers включай (в Caddy/Traefik — middleware, в Nginx — руками).

