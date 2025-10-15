# Docker: хранение — образы/слои, writable-layer, тома и bind mounts (простым языком)

## 0) Картина мира за 60 секунд

- **Image (образ)** = слоёная пицца (read-only слои).
- **Container** = образ **+** тонкий **writable-layer** (то, что ты меняешь).
- Всё, что ты записываешь **внутрь контейнера без томов**, живёт в этом writable-layer и **исчезнет при удалении контейнера**.
- Постоянные данные → **Volume** или **Bind-mount**.

---

## 1) Образы и слои (как это устроено)

- Каждый шаг Dockerfile (`RUN/COPY/ADD`) делает **слой** (read-only).
- Поверх всех слоёв у запущенного контейнера есть **тонкий записываемый слой**.
- Хранилищем слоёв управляет **storage driver** (обычно **overlay2** на Linux).

Полезное:

```bash
docker images                      # список образов
docker history <image>             # из каких слоёв собран образ
docker image inspect <image>       # детали, размеры слоёв
docker system df                   # сколько диска жрут образы/контейнеры/тома
```

**Вывод:** чем меньше слоёв/меньше контента — тем быстрее пуш/пул и меньше диска.

---

## 2) Writable-layer контейнера (временный)

- Все изменения «без томов» → сюда. Удалил контейнер — **прощай данные**.
- Частые записи (логи/кеши/БД) **не клади** сюда — ест диск и пропадёт.
- Это слой медленнее, чем том/хостовая ФС (за счёт copy-on-write).

Посмотреть размер:

```bash
docker ps -s    # покажет SIZE (размер writable-layer)
```

---

## 3) Способы хранить данные

### A) **Volume** (рекомендуется для данных приложения)

- Создаётся/жрётся Docker’ом в `/var/lib/docker/volumes/...`.
- **Переживает пересоздание контейнеров**, легко переносится между ними.
- Можно бэкапить, отдавать права, подключать в несколько контейнеров.

```bash
# анонимный том (Docker сам создаст)
docker run -d -v /var/lib/postgresql/data postgres:16

# именованный том (лучше, можно переиспользовать)
docker volume create pgdata
docker run -d -v pgdata:/var/lib/postgresql/data postgres:16

docker volume ls
docker volume inspect pgdata
docker volume rm pgdata   # удалит ТОЛЬКО том
```

**Когда использовать:** БД, uploads, persistent-данные сервисов.

---

### B) **Bind mount** (подключить папку хоста)

- Монтирует **конкретный путь** хоста внутрь контейнера.
- Удобно в разработке: шаришь исходники, конфиги.
- Ответственность за права/SELinux/безопасность — на тебе.

```bash
# короткая форма
docker run -v $(pwd)/app:/app -w /app node:20 node server.js

# современная (--mount)
docker run --mount type=bind,src=$PWD/app,dst=/app node:20 node server.js
```

**Когда использовать:** Dev, конфиги, чтение секретов с хоста.  
**Осторожно в проде:** случайно можно дать контейнеру доступ к лишнему.

---

### C) **tmpfs** (только в памяти)

- Временный RAM-диск внутри контейнера (не пишется на диск).
- Быстро и «без следов» (и при остановке всё исчезает).

```bash
docker run --tmpfs /run:rw,noexec,nosuid,nodev nginx:alpine
# или
docker run --mount type=tmpfs,dst=/tmp,tmpfs-size=64m busybox
```

**Когда использовать:** временные файлы, секреты во время запуска, кэши.

---

## 4) Куда писать логи?

- По умолчанию `stdout/stderr` → собирает Docker (доступно через `docker logs`).
- Внутри контейнера в файлы писать стоит **на том** (если надо хранить).
- На проде часто: app → stdout, а агрегация делается лог-драйвером/агентом.

---

## 5) Выбор: Volume vs Bind-mount (шпаргалка)

|Что нужно|Выбор|
|---|---|
|Постоянные данные сервиса (БД, uploads)|**Volume**|
|Шарить исходники/конфиги с хоста|**Bind-mount**|
|Очень быстрые временные файлы|**tmpfs**|
|«Всё внутри и не сохранять»|Writable-layer (но понимая, что пропадёт)|

---

## 6) Правильная сборка образа (чтобы слои были «дешёвыми»)

- Используй `.dockerignore` (не тащи в build контекст node_modules/.git/tmp).
- Сначала копируй **lock-файлы** (`package-lock.json`, `poetry.lock`) → `RUN install` → потом `COPY . .` (так кэш слоёв живёт дольше).
- **Не клади секреты** в `ENV/ARG` и в образ (останутся в слоях навсегда).
- Разделяй «runtime» и «build» (multi-stage): меньше итоговый размер.

---

## 7) Частые грабли и решения

**«Permission denied» на томе/бинде**

- Проверь UID/GID процесса в контейнере и владельца каталога на хосте.
- В Kubernetes/SELinux-системах используй `:z`/`:Z` для SELinux-меток (в Docker на RHEL-семействе):
```bash
    docker run -v /host/data:/data:Z ...
    ```
- Либо создай том и настрой владельца:
```bash
    docker run --rm -v pgdata:/var/lib/postgresql/data busybox sh -lc 'chown -R 999:999 /var/lib/postgresql/data'
    ```
    

**Логи/БД пропадают после пересоздания**
- Ты писал в **writable-layer**. Решение — **том**: `-v pgdata:/var/lib/postgresql/data`.

**Образ разросся**
- Слишком много слоёв/пакетов. Применяй multi-stage, `apt-get clean`, `--no-install-recommends`, distroless/slim.

**«text file busy» при разработке на bind-mount**
- Это норм при заменах бинарей «на горячую». Решение — перезапуск процесса/контейнера.

---

## 8) Compose примеры

**App + DB с томом**

```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

  web:
    build: .
    ports: ["8080:80"]
    depends_on: [db]
    # для девелопмента: шарим исходники
    volumes:
      - ./:/app

volumes:
  pgdata:
```

**tmpfs в Compose**

```yaml
services:
  api:
    image: myapi:latest
    tmpfs:
      - /tmp:size=64m
```

**Bind-mount только для чтения**

```yaml
services:
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
```

---

## 9) Чистка мусора (аккуратно!)

```bash
# удалить остановленные контейнеры, висячие сети, dangling-образы/кеши
docker system prune

# + удалить неиспользуемые тома (внимание: удалит ДАННЫЕ в них!)
docker system prune --volumes

# точечные команды
docker image prune
docker volume prune
docker network prune
```

---

## 10) Мини-FAQ

**Где тома на диске?**  
— `/var/lib/docker/volumes/<name>/_data` (не копайся там руками, лучше `docker cp`/бэкап через контейнер).

**Можно подключить один том к нескольким контейнерам?**  
— Да. Для БД — осторожно (одновременная запись = риск).

**Как сделать бэкап тома?**

```bash
docker run --rm -v pgdata:/data -v $PWD:/backup busybox \
  sh -lc 'tar czf /backup/pgdata.tgz -C /data .'
```

**Чем `-v` отличается от `--mount`?**  
— `--mount` более явный и одинаков для всех типов. Рекомендуется в новых манифестах.

---

## TL;DR

- **Образ** — read-only слои; **контейнер** — образ + временный writable-layer.
- Долгоживущие данные → **Volume**. Разработка/конфиги → **Bind-mount**. Быстрая «оперативка» → **tmpfs**.
- Следи за правами (UID/GID), не храни секреты в слоях, чисти мусор осознанно.