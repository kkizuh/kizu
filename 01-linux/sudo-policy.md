# Политики sudo и best practices (с подробными комментами)

## Зачем sudo

- `sudo` запускает **конкретные команды** с правами другого юзера (обычно root).
- Цель: **минимум прав**, всё **логируется**, меньше шансов убить прод.

---

## Базовые правила (золотой набор)

1. **Минимум прав.** Никаких `NOPASSWD: ALL` для людей.
2. **Через группы.** Выдаём права группам (`%admins`, `%devops`, `%deploy`), а людей включаем в группы.
3. **Редактируем только через `visudo`.** Проверка синтаксиса → не положишь sudo.
4. **Логируем.** Отдельный лог `sudo.log` + системный `journal/syslog`.
5. **Окружение чистим.** `env_reset`, whitelist через `env_keep` только при необходимости.
6. **Конфиги через `sudoedit`.** Безопаснее, чем `vim` с root.

---

## Где лежит и как править

```
/etc/sudoers            # трогай минимально
/etc/sudoers.d/*.conf   # твои роли/правила (рекомендуется)
```

Открыть безопасно:

```bash
sudo visudo                          # /etc/sudoers
sudo visudo -f /etc/sudoers.d/devops.conf
visudo -cf /etc/sudoers              # проверка синтаксиса
visudo -cf /etc/sudoers.d/*.conf
```

---

## Стартовый «скелет» (вставь и живи)

```sudoers
# === Глобальные дефолты ===
Defaults use_pty                                  # лучшее логирование
Defaults logfile="/var/log/sudo.log"              # отдельный лог sudo
Defaults timestamp_timeout=5                      # кэш пароля 5 минут
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Defaults env_reset                                # очистка окружения
Defaults env_keep += "LANG LC_*"                  # оставить локали
# Пример точечного whitelist окружения для роли deploy:
# Defaults:%deploy env_keep += "http_proxy https_proxy no_proxy"

# === Роли по группам ===
# Полные админы, ВСЁ, с паролем (ок для людей)
%admins   ALL=(ALL) ALL

# DevOps: смотреть логи/статусы юнитов, НО не управлять ими
%devops   ALL=(root) /usr/bin/journalctl, \
                       /usr/bin/systemctl status *, \
                       /usr/bin/systemctl list-units, \
                       /usr/bin/systemctl list-timers

# Deploy/CI: управлять ТОЛЬКО своим сервисом (без пароля)
%deploy   ALL=(root) NOPASSWD: /usr/bin/systemctl start myapp.service, \
                               /usr/bin/systemctl stop myapp.service, \
                               /usr/bin/systemctl restart myapp.service, \
                               /usr/bin/systemctl reload myapp.service

# Редакторы: править конфиги ТОЛЬКО через sudoedit
%editors  ALL=(root) sudoedit /etc/myapp/app.conf
```

**Почему `sudoedit`:** файл редактируется во временном месте и аккуратно кладётся на место → меньше рисков, чем интерактивный root-редактор.

---

## Алиасы (читабельность и ревью)

```sudoers
Host_Alias   WEBS = web01, web02, web03           # где
User_Alias   APPTEAM = alice, bob, carol          # кто
Runas_Alias  RWWW = www-data, nginx               # от чьего имени
Cmnd_Alias   APPCTL = /usr/bin/systemctl restart myapp.service, \
                      /usr/bin/systemctl status myapp.service
Cmnd_Alias   LOGS   = /usr/bin/journalctl

APPTEAM WEBS = (RWWW) NOPASSWD: APPCTL, LOGS
# Читается как текст: команда, кто, где, от чьего имени и что именно можно.
```

---

## Типовые роли (готовые блоки)

### 1) Деплой/CI только перезапускает свой юнит

```sudoers
%deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp.service
# Ровно одна команда, ровно один юнит. Никаких звёздочек.
```

### 2) Просмотр логов и статусов (без управления)

```sudoers
%devops ALL=(root) /usr/bin/journalctl, /usr/bin/systemctl status *
# Диагностика без права «ломать».
```

### 3) Редактирование конфигов безопасно

```sudoers
%editors ALL=(root) sudoedit /etc/nginx/nginx.conf, sudoedit /etc/myapp/app.conf
# Только sudoedit — без шанса выполнить что-то «случайно».
```

### 4) Разрешить запуск одного root-скрипта из cron

```sudoers
deploy ALL=(root) NOPASSWD: /usr/local/bin/release.sh
# Юзер deploy может запускать только этот скрипт. Не давай /bin/bash!
```

### 5) Запуск от имени www-data (минимум прав)

```sudoers
%appteam ALL=(www-data) NOPASSWD: /usr/bin/php /var/www/app/bin/task.php *
# Не root, а www-data — безопаснее.
```

---

## Проверка и аудит

```bash
sudo -l -U someuser               # что разрешено юзеру
journalctl _COMM=sudo -n 200      # события sudo в журнале
sudo tail -f /var/log/sudo.log    # отдельный лог (если включён)
```

---

## Частые ошибки → правильные альтернативы

- ❌ `NOPASSWD: ALL` для людей → ✅ конкретные команды, NOPASSWD только там, где автоматизация.
- ❌ Правка `/etc/sudoers` без `visudo` → ✅ только `visudo`, + `visudo -cf` в CI.
- ❌ `systemctl *` в списке команд → ✅ перечислять конкретные юниты.
- ❌ Интерактивный root-редактор → ✅ `sudoedit`.
- ❌ Свободный `PATH` → ✅ `Defaults secure_path=...`.
- ❌ `sudo -E` и протаскивание среды → ✅ `env_keep` точечно и только для роли.

---

## Безопасные паттерны (спасают прод)

- Доступ “наблюдать” (логи/статусы) даёшь многим, “управлять” — узкой роли.
- Конфиги — через `sudoedit` + `systemctl reload` (если разрешено роли).
- Скрипты, которым даёшь право через sudoers, кладёшь в `/usr/local/bin/` и делаешь их **иммутабельными** по смыслу (никаких «выполнить всё из аргумента»).
- Логи sudo отправляешь в централизованное хранилище (rsyslog/journal-remote).
- Раз в квартал — ревизия: чистим мёртвые правила и пользователей.

---

## Мини-FAQ (собес и бой)

- **sudo vs su?** `sudo` — минимальные права на конкретные команды с аудитом; `su` — полный root без треков.
- **зачем sudoedit?** Безопасное редактирование без запуска произвольного кода от root.
- **как ограничить PATH?** `Defaults secure_path=...` в sudoers.
- **как проверить права юзера?** `sudo -l -U username`.
- **как дать рестарт ровно одного сервиса?** Разрешить `/usr/bin/systemctl restart myapp.service` и всё.

---

## Чеклист перед выкладкой правил

-  Валидация `visudo -cf` пройдена.
-  Нет широких `*`/`ALL`, где можно перечислить точно.
-  Для редактирования — `sudoedit`.
-  `secure_path` задан.
-  Логи включены и просматриваются.
-  Все правила через группы, не на конкретных людей.

---

Если захочешь, разнесу это на роли (`admins`, `devops`, `deploy`, `editors`) отдельными файлами в `/etc/sudoers.d/` и дам Ansible-слепки.

**`sudo -e`** — это **sudoedit-режим**.

## Что делает `sudo -e`
- Открывает файл **твоим обычным редактором** (nano/vim), **без запуска самого редактора с root**.
- sudo **делает копию** файла во временном месте, даёт тебе её отредактировать от своего юзера, потом **аккуратно пишет изменения назад** под root.
- Это **безопаснее**, чем `sudo vim /etc/...`, потому что:
    - редактор и его плагины не получают root-права;
    - меньше шансов случайно выполнить что-то от root;
    - защита от трюков с `$PATH`, алиасами и т.п.

## Как пользоваться

```bash
export EDITOR=vim        # или nano, или другой редактор
sudo -e /etc/nginx/nginx.conf
sudo -e /etc/ssh/sshd_config /etc/ssh/ssh_config   # можно несколько файлов
```

> Если не задано `$EDITOR`/`$VISUAL`, sudo выберет дефолт (часто `vi`). Поставь своё.

## Чем отличается от `sudo vim /etc/file`

|Команда|Кто с root|Риски плагинов/макросов|Рекомендуется|
|---|---|---|---|
|`sudo vim /etc/file`|**vim**|высокие|нет|
|`sudo -e /etc/file`|**только запись**|низкие|**да**|

## В sudoers (ограниченный доступ)

Лучше давать право **только на sudoedit**, а не на запуск редактора с root:

```sudoers
%editors ALL=(root) sudoedit /etc/myapp/app.conf
```

Пользователи из `%editors` смогут править **только этот файл**, безопасно.

## Частые вопросы

- **“А как выбрать редактор?”**  
    `export EDITOR=nano` (или `vim`), можно в `~/.bashrc`. Для текущей сессии хватит одной команды.
    
- **“Почему ругается на права/симлинки?”**  
    sudoedit **следует симлинку к целевому файлу**, но не даёт править произвольные пути. Если симлинк ведёт в “неразрешённое” место при строгих sudoers — запретит.
    
- **“А если нужно создать новый файл?”**  
    Создай пустой: `sudo install -m 0644 /dev/null /etc/myapp/app.conf && sudo -e /etc/myapp/app.conf`
    
- **“Можно ли бинарники редактировать?”**  
    Нет, это для текстовых конфигов.
    
- **“Почему изменения не сохранились?”**  
    Редактор мог сохранить в другой файл/сделать swap. Сохраняй как обычно (`:wq` в vim, `Ctrl+O, Enter, Ctrl+X` в nano). Ошибки записи sudo покажет.
    

## Мини-шпарка

```bash
# выбери редактор один раз
echo 'export EDITOR=nano' >> ~/.bashrc && source ~/.bashrc

# правь конфиги безопасно
sudo -e /etc/ssh/sshd_config
sudo systemctl reload sshd

# правильная политика
sudo visudo -f /etc/sudoers.d/editors
# %editors ALL=(root) sudoedit /etc/ssh/sshd_config
```

Если коротко: **`sudo -e` = “редактировать как человек, безопасно для прод”**.
Короткая табличка:

|Как|Кто с root|Риски плагинов|Лучше|
|---|---|---|---|
|`sudo vim /etc/file`|vim|высокие|❌|
|`sudo -e /etc/file`|только запись|низкие|✅|
