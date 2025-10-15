# Дюжина инструкций Dockerfile — кратко и по делу

# **Кеш Docker:**  
Dockerfile идёт **сверху вниз**. Любое изменение в строке **№N ломает кеш с неё и ниже**; всё, что **выше**, берётся из кеша. Поэтому **стабильное — вверх** (база, apt, deps по lock-файлам), **частое — вниз** (код, сборка).

```dockerfile
# стабильные слои ↑
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
# меняется часто ↓
COPY . .
RUN npm run build
```

| Инстр.         | Зачем                                                                    | Мини-пример                                                                      | Подводные камни / best-practices                                                                                                                                         |
| -------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **FROM**       | Базовый образ (стартовый слой)                                           | `FROM python:3.12-slim`                                                          | Ставь конкретные теги (`:3.12-slim`), а не `latest`. Для size/security — `*-slim`/distroless.                                                                            |
| **LABEL**      | Метаданные образа (OCI)                                                  | `LABEL org.opencontainers.image.source="https://..."`                            | Впиши `source, version, revision, created`. Удобно для трейсинга в CI.                                                                                                   |
| **ENV**        | Постоянные переменные среды в образе                                     | `ENV PIP_NO_CACHE_DIR=1 TZ=UTC`                                                  | Не пихай секреты. Для временных значений на рантайме — `docker run -e`.                                                                                                  |
| **RUN**        | Выполнить команду при **сборке** (новый слой)                            | `RUN apt-get update && apt-get install -y curl \ && rm -rf /var/lib/apt/lists/*` | Комбайни команды (меньше слоёв), чисти кэш пакетов. Кладёшь только то, что нужно **в runtime** (или используй multi-stage).                                              |
| **COPY**       | Копировать файлы в образ                                                 | `COPY --chown=10001:10001 . /app`                                                | Предпочитай **COPY** вместо ADD. Используй `--chown` сразу, чтобы не делать `chown` отдельным слоем. Уважай `.dockerignore`.                                             |
| **ADD**        | COPY + фишки: локальный `.tar` авто-распаковка, **может** тянуть **URL** | `ADD app.tar.gz /opt/`                                                           | Best practice: почти всегда **COPY**. `ADD <url>` не даёт контроля/хэшей — лучше `curl -fSL ...                                                                          |
| **CMD**        | Команда/арги **по умолчанию** при запуске                                | `CMD ["python","app.py"]`                                                        | В Dockerfile **только одна CMD**: последняя побеждает. **Переопределяется** любыми аргументами `docker run image ...`. Часто используется как **арги** к ENTRYPOINT.     |
| **WORKDIR**    | Рабочая директория для следующих RUN/CMD/COPY                            | `WORKDIR /app`                                                                   | Не городи `cd` в `RUN`. `WORKDIR` можно использовать много раз — создаёт директорию, если нет.                                                                           |
| **ARG**        | Переменные **на этапе build**                                            | `ARG VERSION` → `RUN curl -LO app-$VERSION.tgz`                                  | Доступны **только во время сборки**. Чтобы протащить в ENV: `ARG V=1.2 && ENV APP_VER=$V`. Значения можно задать `--build-arg`.                                          |
| **ENTRYPOINT** | «Что запускать всегда»                                                   | `ENTRYPOINT ["python","app.py"]`                                                 | Не путай: **ENTRYPOINT не переопределяется аргументами** (они **добавляются** к нему). Чтобы заменить — `--entrypoint`. Часто: `ENTRYPOINT ["app"]` + `CMD ["--serve"]`. |
| **EXPOSE**     | Документирует порт контейнера                                            | `EXPOSE 8080`                                                                    | Это **подсказка**, порт **не публикует**. Публикуешь хост-порт флагом `-p 80:8080` или в compose.                                                                        |
| **VOLUME**     | Точка монтирования для данных                                            | `VOLUME ["/data"]`                                                               | Создаёт **анонимный** volume, если не смонтирован свой. Подумай, надо ли — иначе потом «не могу найти файлы». В compose лучше явно `volumes:`.                           |

---

## Мини-паттерны (правильные связки)

### ENTRYPOINT + CMD (дефолтные аргументы)

```Dockerfile
ENTRYPOINT ["myapp"]        # фиксированная программа
CMD ["--serve","--port=8080"]  # можно заменить: docker run img --port=9090
```

### Кэш зависимостей (Node/Python/Go)

```Dockerfile
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
# аналогично: requirements.txt → pip install; go.mod → go mod download
```

### Non-root сразу владельцем

```Dockerfile
RUN addgroup --system --gid 10001 app \
 && adduser --system --uid 10001 --ingroup app app
WORKDIR /app
COPY --chown=10001:10001 . .
USER 10001:10001
```

### ADD для локального tar (редкий, но законный кейс)

```Dockerfile
ADD app.tar.gz /opt/app/   # распакует содержимое в /opt/app
```

---

## Частые заблуждения (разрушаем мифы)

- «`EXPOSE` откроет порт» — нет, это только документация. Публикует **`-p`**.
- «`ENTRYPOINT` нельзя менять» — можно флагом **`--entrypoint`**, аргументами ты меняешь **CMD**, а не ENTRYPOINT.
- «`ADD` лучше `COPY`» — почти всегда **наоборот**. `ADD` береги для **локальных tar** или, в крайняке, URL (но лучше `curl` + хэши).
- «`RUN apt-get install` в runtime нормально» — нет. Всё, что нужно только для сборки — убирай в build-stage (multi-stage).
- «Оставлю root, так проще» — потом боль с безопасностью и томами. Делай **`USER`**.

---

## Быстрый чек-лист «Dockerfile OK»

-  База `*-slim`/distroless; пин тугoв.
-  `.dockerignore` настроен.
-  Кэш: lock-файлы копируются первыми.
-  Никаких секретов в `ENV/ARG`.
-  **Non-root** (`USER 10001`), права через `--chown`.
-  `ENTRYPOINT` + `CMD` по месту.
-  Нет `ADD <url>` (кроме крайней нужды).
-  `EXPOSE` корректен; публикуем порты флагами/compose.
-  `VOLUME` — осознанно (или выносим в compose).

## Разница

- **ENTRYPOINT** — что **всегда** запускать (основной бинарь контейнера).
- **CMD** — **аргументы по умолчанию** (или «что запускать», если ENTRYPOINT нет).

## Как они работают вместе

```Dockerfile
ENTRYPOINT ["myapp"]            # фиксированно
CMD ["--serve","--port=8080"]   # можно заменить при запуске
```

Запуски:

```bash
docker run img                       # myapp --serve --port=8080
docker run img --port=9000           # myapp --port=9000        (CMD заменили)
docker run --entrypoint bash img -lc 'echo hi'  # ENTRYPOINT заменили целиком
```

## Правила запоминалки

- **Последний CMD** в Dockerfile побеждает (может быть только один активный)
- **ENTRYPOINT** не меняется обычными аргументами `docker run` — только `--entrypoint`.
- Если **ENTRYPOINT отсутствует**, тогда **CMD** — это команда целиком.

## Exec vs Shell форма

- **Exec (рекомендуется):** JSON-массив, без шелла.
```Dockerfile
	ENTRYPOINT ["python","app.py"]
    ```
- **Shell:** запускает через `/bin/sh -c` (сигналы, цитирование — аккуратнее).
```Dockerfile
    CMD python app.py --port=8080
    ```

## Когда что использовать

- Хочешь, чтобы образ был «как утилита» (всегда один бинарь) → **ENTRYPOINT** = бинарь, **CMD** = дефолтные опции.
- Хочешь просто задать одну команду, которую легко заменить → только **CMD**.

## Быстрые примеры

**1) Утилита `curl` с дефолтным URL**

```Dockerfile
FROM alpine
RUN apk add --no-cache curl
ENTRYPOINT ["curl","-sS"]   # утилита фиксирована
CMD ["https://example.com"] # можно заменить на любой URL
```

**2) Node-сервис (команду можно заменить)**

```Dockerfile
FROM node:20-slim
WORKDIR /app
COPY . .
USER node
CMD ["node","server.js"]    # нет ENTRYPOINT — всю команду легко заменить
```

## Подводные

- Не пиши и `ENTRYPOINT`, и «полную» команду в `CMD` — дублирование.
- Для корректной обработки сигналов используйте **exec-форму** (`["prog","arg"]`).
- Переопределить только аргументы → меняй **CMD** (просто добавь их в `docker run`).
- Полностью заменить программу → `--entrypoint ...`.

## 1) «База `*-slim`/distroless; пин тэгов»

- **`*-slim`** — тот же Debian/Ubuntu образ, но без лишнего хлама → меньше размер, меньше уязвимостей.  
    Примеры: `python:3.12-slim`, `node:20-bookworm-slim`, `eclipse-temurin:21-jre`.
- **distroless** — вообще без shell/apt, только рантайм → ещё меньше и безопаснее. Хорош для Go/Java.
- **«Пин тэгов»** = фиксируй версию, а не `latest`, чтобы сборка **не менялась сама по себе**.  
    Идеально — фиксируй **по digest** (неизменяемо):
```Dockerfile
    FROM python:3.12-slim@sha256:<длинный_хэш>
```
    Минимум — фиксируй мажор/минор:
```Dockerfile
    FROM python:3.12-slim  # ок; лучше, чем latest
```

## 2) «Кэш: lock-файлы копируются первыми»

Сборка идёт слоями. Если слой **не изменился**, Docker берёт его из **кэша**.  
Главная идея: **сначала копируй файлы с зависимостями**, ставь зависимости, а код — потом. Тогда «дорогая» установка кэшируется.

Примеры:
- **Node**
```Dockerfile
    COPY package.json package-lock.json ./
    RUN npm ci --omit=dev
    COPY . .
    ```
- **Python (pip)**
```Dockerfile
    COPY requirements.txt .
    RUN pip install -r requirements.txt
    COPY . .
    ```
- **Poetry**
```Dockerfile
    COPY pyproject.toml poetry.lock ./
    RUN poetry install --no-root --only main
    COPY . .
    ```
- **Go**
```Dockerfile
    COPY go.mod go.sum ./
    RUN go mod download
    COPY . .
    ```

Если ты сначала сделаешь `COPY . .`, любое изменение в коде **сломает кэш** на установке зависимостей → долгий билд каждый раз.

## 3) «Никаких секретов в `ENV/ARG`»

- `ENV` записывается в **образ навсегда** (видно в `docker history`, любой с образом увидит).
- `ARG` тоже **виден в истории сборки** и слоях.
- Значит: **НЕ** зашивай в Dockerfile токены, пароли, приватные ключи.

Как правильно:

- **Build-time секреты (BuildKit)**:
```Dockerfile
    # syntax=docker/dockerfile:1.6
    RUN --mount=type=secret,id=NPM_TOKEN \
        sh -c 'echo "//registry.npmjs.org/:_authToken=$(cat /run/secrets/NPM_TOKEN)" > ~/.npmrc && npm ci'
    ```
    
```bash
    DOCKER_BUILDKIT=1 docker build \
      --secret id=NPM_TOKEN,env=NPM_TOKEN \
      -t myimg .
```
- **Runtime секреты** (заводи вне образа):
    - Docker Swarm/Compose: `secrets:` → монтируются файлом в `/run/secrets/...`.
    - Kubernetes: `Secret` → env или файлы.
    - Локально: `.env` в compose (но **не коммить**), или `-e KEY=...` при `docker run`.
- Конфиги: храни шаблон `config.tpl`, подставляй переменные **на старте**, а не на билде (entrypoint-скриптом).

### Мини-табличка «плохо/хорошо»

|Задача|Плохо|Хорошо|
|---|---|---|
|База образа|`python:latest`|`python:3.12-slim@sha256:...`|
|Зависимости|`COPY . .` → `npm ci`|`COPY package*.json .` → `npm ci` → `COPY . .`|
|Секреты|`ENV TOKEN=abcd`|BuildKit `--secret`, или runtime secrets|
|Размер|`apt-get install ...` в runtime|multi-stage: сборка → тонкий рантайм|
|Безопасность|root в контейнере|`USER 10001`, `COPY --chown=10001:10001`|

