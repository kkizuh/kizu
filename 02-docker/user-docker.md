## Как работает `USER` в Dockerfile

`USER` задаёт **под кем** будут выполняться:
- все следующие `RUN` в этом **stage**,
- и потом `CMD`/`ENTRYPOINT` в рантайме.

Форматы:
- `USER <name>` → по имени из `/etc/passwd` внутри **образа**
- `USER <uid>` → по числовому UID (даже если нет в `/etc/passwd`)
- `USER <name>:<group>` или `USER <uid>:<gid>`

Если ты пишешь `USER kizu`, то **пользователь `kizu` должен существовать в образе**. Иначе — ошибка “user kizu not found”.

---

## Надёжный паттерн (Debian/Ubuntu slim)

```Dockerfile
# создать юзера и группу с фиксированными id (удобно для volume)
RUN addgroup --system --gid 10001 kizu \
 && adduser  --system --uid 10001 --ingroup kizu --home /app --shell /usr/sbin/nologin kizu

WORKDIR /app
# сразу кладём файлы с нужным владельцем (BuildKit)
COPY --chown=10001:10001 . .

USER kizu:kizu  # теперь можно по имени
```

- `--system` — без пароля/логина, “сервисный” аккаунт
- фиксированные `uid/gid` = меньше боли с правами на тома
- `--shell /usr/sbin/nologin` — чтобы нельзя было интерактивно зайти

### Alpine-вариант

```Dockerfile
RUN addgroup -S -g 10001 kizu && adduser -S -u 10001 -G kizu -h /app -s /sbin/nologin kizu
WORKDIR /app
COPY --chown=10001:10001 . .
USER kizu:kizu
```

### Distroless

Пакетного менеджера нет → юзера не добавить.

- Используй готового **`65532`** (часто `nonroot`): `USER 65532:65532`
- Или перенеси на distroless уже собранный артефакт с правильными владельцами.

---

## Частые вопросы

**Q: Можно просто `USER 10001` без создания юзера?**  
A: Да, процесс запустится под UID=10001 (без имени). Это ОК для безопасности, но:

- `~`/home и некоторые тулзы ожидают запись/права → лучше создать юзера и `chown` нужные пути.

**Q: А если нужен порт <1024 (80/443), а я не root?**  
A: Дай бинарю capability или слушай высокий порт и ставь `-p 80:8080`.

```Dockerfile
# во время build установить утилиту и повесить cap
RUN apt-get update && apt-get install -y --no-install-recommends libcap2-bin \
 && setcap 'cap_net_bind_service=+ep' /app/mybin \
 && apt-get purge -y libcap2-bin && rm -rf /var/lib/apt/lists/*
```

**Q: Что с томами (`-v`)?**  
A: Файлы на томе имеют права хоста. Если внутри нет прав — согласуй `uid/gid` (10001), делай `COPY --chown`/`chown` в build, либо на запуске добавь группы:

```bash
docker run --user 10001:10001 --group-add 0 ...
```

(трюк с `:0` иногда спасает при смешанных правах)

**Q: После `USER kizu` команды `RUN` тоже под `kizu`?**  
A: Да. Если нужно снова root — явно: `USER root` (и потом вернуть `USER kizu`).

---

## Мини-чеклист “`USER kizu` работает”

-  Пользователь/группа созданы с фиксированными `uid/gid` (напр. 10001).
-  Рабочая директория и файлы принадлежат `10001:10001` (`COPY --chown` топ).
-  Нет необходимости в root на рантайме (apt/gcc уже не нужны → multi-stage!).
-  Для портов <1024 либо capability, либо маппим высокий порт.
-  В distroless — используем `USER 65532` или готовим права заранее.

