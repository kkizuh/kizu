# Docker Compose — профили и шаблоны (extends)

## Зачем это вообще

- **Профили** (`profiles`) — включать/выключать сервисы одним флагом, без 100 разных compose-файлов.
- **Шаблоны** (`extends`) — не дублировать одно и то же; вынеси базовую конфигурацию и наследуйся.

---

## Профили: включай ровно то, что нужно

```yaml
version: "3.9"
services:
  db:
    image: postgres:16
    profiles: ["db-only", "local", "e2e-local"]
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports: ["5432:5432"]

  api:
    build: ./api
    profiles: ["local", "e2e-local"]
    ports: ["5000:5000"]
    depends_on: [db]

  tests:
    build:
      context: .
      dockerfile: KarateDockerfile
    profiles: ["e2e-local"]
    depends_on: [api]
    command: ["/karate"]
    volumes:
      - .:/karate
    environment:
      API_URL: "http://api:5000"
```

### Как запускать

```bash
# только БД (разраб крутит API у себя в IDE)
docker compose --profile db-only up -d

# локальная среда: API + DB
docker compose --profile local up -d

# E2E локально: API + DB + тесты
docker compose --profile e2e-local up --abort-on-container-exit
```

> Плюсы: один `docker-compose.yml`, никакой путаницы с `--file a.yml --file b.yml`.

---

## Шаблоны (extends): DRY для сервисов

### В одном файле

```yaml
services:
  base_tests:
    build:
      context: .
      dockerfile: KarateDockerfile
    command: ["/karate"]
    volumes:
      - .:/karate

  tests_local:
    extends:
      service: base_tests
    profiles: ["e2e-local"]
    environment:
      API_URL: "http://api:5000"

  tests_prod:
    extends:
      service: base_tests
    profiles: ["e2e-prod"]
    environment:
      API_URL: "https://api.example.com"
```

### Шаблон в отдельном файле

**templates.yml**

```yaml
version: "3.9"
services:
  base_tests:
    build:
      context: .
      dockerfile: KarateDockerfile
    command: ["/karate"]
    volumes:
      - .:/karate
```

**docker-compose.yml**

```yaml
services:
  tests_local:
    extends:
      file: templates.yml
      service: base_tests
    profiles: ["e2e-local"]
    environment:
      API_URL: "http://api:5000"
```

---

## Комбо: профили + depends_on + healthcheck

```yaml
services:
  api:
    build: ./api
    profiles: ["local", "e2e-local"]
    expose: ["5000"]
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:5000/health"]
      interval: 10s
      timeout: 3s
      retries: 8

  tests:
    extends:
      service: base_tests
    profiles: ["e2e-local"]
    depends_on:
      api:
        condition: service_healthy   # дождись готовности API
    environment:
      API_URL: "http://api:5000"
```

---

## В CI/CD (GitHub Actions пример)

```yaml
- name: Run E2E
  run: |
    set -e
    # поднимаем стек и падаем, если тесты упали
    docker compose --profile e2e-local up --abort-on-container-exit
    # чистим после
    docker compose --profile e2e-local down
```

> Если парсишь логи руками — это костыль. Лучше пусть **контейнер тестов завершает процесс кодом <> 0**, тогда `--abort-on-container-exit` и общий exit code пайплайна всё сделают сами.

---

## Паттерны использования (что реально помогает)

- **db-only профиль** для разработчиков IDE-first.
- **local профиль** — быстрый запуск с минимальным набором (API + DB).
- **e2e-local** — гоняем тесты по локальному API в докере.
- **e2e-prod** — тот же контейнер тестов, но `API_URL` на прод/стейдж (через `extends`).
- __base__ шаблоны_* для повторяющихся контейнеров: тесты, миграции, линтеры.

---

## Частые грабли → как не словить

- **Сервис не стартует, но зависит**: используй `healthcheck` + `depends_on.condition: service_healthy`, а не просто порядок.
- **Клон-файлы compose расползаются**: профили решают это — держи всё в одном `compose`
- **Дублирование конфигов**: выноси в `extends` и, при росте, — в `templates.yml`.
- **Порты конфликтуют**: для локалки — `127.0.0.1:PORT:PORT`; в e2e обычно `expose` достаточно.
- **Секреты**: не пихай пароли в `environment`. Для MySQL/MariaDB есть `*_FILE`; для прод — Docker/Swarm/K8s secrets.

---

## Быстрые шпаргалки

```bash
# выбрать профиль
docker compose --profile local up -d

# несколько профилей
docker compose --profile local --profile obs up -d

# смотреть, что поднято
docker compose ps

# логи одного сервиса
docker compose logs -f tests

# аварийно упасть, если тесты упали
docker compose --profile e2e-local up --abort-on-container-exit --exit-code-from tests
```

---

## TL;DR

- **Профили** = «какие сервисы включать сейчас».
- **Extends** = «не копипастим конфиг, наследуем общий шаблон».
- С этим у тебя **один** compose-файл, чисто, читаемо, CI-дружелюбно.

