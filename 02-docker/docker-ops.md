# Docker — Ops: Logs / Health / Troubleshooting / Cleanup / Recipes

Коротко, по делу, без шелухи. Это страница «подглядел — сделал — починил».

---

## 1) Логи

### 1.1 Build-логи (во время сборки)

```bash
# обычная сборка с прогрессом по шагам
docker build -t myapp:dev .

# подробный вывод
docker build -t myapp:dev . --progress=plain
```

### 1.2 Runtime-логи (запущенный контейнер)

```bash
# последние строки + следить
docker logs -f myapp

# показать только новые за последние 10 минут
docker logs --since 10m myapp

# включить время
docker logs -t myapp

# если контейнер падает сходу — сразу смотри логи + код выхода
docker logs myapp
docker inspect -f '{{.State.ExitCode}}' myapp
```

### 1.3 Драйверы логов + ротация (чтобы диск не забить)

По умолчанию: `json-file`. Добавь ротацию:

```bash
docker run -d --name myapp \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myimg:latest
```

Альтернативы: `local`, `journald`, `gelf`, `syslog`, `fluentd`, `awslogs` — выбирай под свою платформу.

---

## 2) Health / Метрики

### 2.1 HEALTHCHECK в Dockerfile

```dockerfile
# пример: HTTP-зонд на /health
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -fsS http://127.0.0.1:8080/health || exit 1
```

Проверить:

```bash
docker inspect -f '{{.State.Health.Status}}' myapp   # healthy / unhealthy
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

### 2.2 Codes / статус процесса

- Контейнер «жив» пока жив PID 1. Выход с кодом `0` — нормально; `!=0` — падение (CrashLoop).
- Возьми за правило: приложение корректно обрабатывает `SIGTERM` и завершает работу (graceful shutdown).

---

## 3) Troubleshooting (быстрая диагностика)

### 3.1 Сеть

```bash
# открыт ли порт снаружи
ss -lntp | grep :8080

# контейнер реально слушает внутри?
docker exec -it myapp ss -lntp

# пинг/HTTP из контейнера
docker exec -it myapp ping -c2 1.1.1.1
docker exec -it myapp curl -I http://127.0.0.1:8080

# какие сети у контейнера
docker inspect -f '{{json .NetworkSettings.Networks}}' myapp | jq
```

**Типовые фейлы**

- Снаружи `-p 8080:8080`, а приложение слушает `127.0.0.1:8080` → слушай `0.0.0.0`.
- Порт занят на хосте → поменяй левую часть маппинга.

### 3.2 Файлы/права (volume/bind)

```bash
# кто UID/GID внутри?
docker exec -it myapp id
# права на хосте должны совпасть
sudo chown -R 10001:10001 /data/dir
```

**Нюанс:** если `USER 10001` в образе, а примонтированный каталог принадлежит root — приложение не сможет писать.

### 3.3 CrashLoop / Exit Code

```bash
docker logs myapp
docker inspect -f '{{.State.ExitCode}} {{.State.Error}}' myapp
# перезапустить в интерактиве для отладки:
docker run --rm -it --entrypoint sh myimg:latest
```

**Чеклист:** переменные окружения? секреты? миграции БД? пути? зависший DNS?

### 3.4 ImagePullBackOff

- Неверный адрес/тег/политика Pull → проверь `docker login`, права на репо, имя `registry/repo:image`.
- DNS: `docker run --rm busybox nslookup registry.domain`.

### 3.5 Пространство на диске

```bash
docker system df
docker image ls --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
# самые жирные слои
docker history myimg:latest --no-trunc
```

---

## 4) Cleanup (аккуратно чистим)

```bash
# удалить всё неиспользуемое (dangling) — спросит подтверждение
docker system prune

# агрессивно: всё неиспользуемое + volumes + остановленные контейнеры
docker system prune -af --volumes

# почистить только образы без тега
docker image prune -f

# builder кеш (buildx)
docker builder prune -af
```

**Совет:** сначала `docker system df`, потом решай, что удалять. На проде — осторожно.

---

## 5) Полезные «рецепты»

### 5.1 «Снять» контейнер для дебага

```bash
# открыть шелл в уже запущенном
docker exec -it myapp sh    # или bash

# запустить тот же образ, но с шеллом и без entrypoint
docker run --rm -it --entrypoint sh myimg:latest
```

### 5.2 Проверить env/FS внутри

```bash
docker exec -it myapp env | sort
docker exec -it myapp ls -lah /app
```

### 5.3 Мини-sidecar с curl/ssh/diag

```bash
docker run --rm -it --network container:myapp curlimages/curl sh
# сеть общая с myapp, можно дёргать localhost:порт
```

### 5.4 Перезапуск с жёсткими флагами безопасности

```bash
docker run -d --name myapp \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  -p 8080:8080 myimg:latest
```

### 5.5 Переименовать/пересоздать без простоя (простейший blue/green)

```bash
docker run -d --name myapp-v2 -p 8081:8080 myimg:v2
# проверяешь 8081, переключаешь прокси/балансировщик,
docker stop myapp && docker rm myapp
docker rename myapp-v2 myapp
```

### 5.6 Логи в journald (если используешь systemd)

```bash
docker run -d --log-driver=journald --name myapp myimg
journalctl -u docker -t myapp -f
```

---

## 6) Быстрый чек-лист запуска (копипастить в мозг)

-  `docker logs -f` — пусто? тогда `--progress=plain` на сборке.
-  Порты: `-p host:container`, сервис слушает `0.0.0.0`.
-  Сеть ок: `curl` из контейнера, DNS резолвит.
-  Volume-права = UID процесса.
-  Healthcheck есть, статус `healthy`.
-  Логи ротируются (`max-size`, `max-file`).
-  Размер образа не улетел в космос.
-  Секреты НЕ в `ENV`, runtime-подгрузка.

---

## 7) Микро-FAQ

**Контейнер «висит», но трафика нет** — проверь firewall хоста, SELinux/AppArmor, NAT/forwarding, правильный порт.  
**`permission denied` при записи** — UID контейнера ≠ владелец каталога; выставь chown на хосте или `userns-remap`.  
**Греется CPU** — включай профилировку/метрики в приложении; на уровне докера смотри `docker stats`.  
**Падает при выходе из `docker exec`** — это нормально, `exec` — просто дополнительный процесс; главный PID 1 живёт отдельно.
