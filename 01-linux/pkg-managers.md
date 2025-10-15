ок, вот та же шпора по пакетным менеджерам, но в **таблицах** — быстро смотреть и копипастить.

# Пакетные менеджеры: apt vs dnf/yum (таблицы)

## База: установка/обновление/удаление

| Задача                        | Debian/Ubuntu (apt)      | RHEL/CentOS/Fedora (dnf/yum) | Комментарий                            |
| ----------------------------- | ------------------------ | ---------------------------- | -------------------------------------- |
| Обновить индекс               | `sudo apt update`        | `sudo dnf update`            | apt тянет список пакетов               |
| Установить пакет              | `sudo apt install <pkg>` | `sudo dnf install <pkg>`     | на CentOS7 — `yum`                     |
| Удалить (оставив конфиги)     | `sudo apt remove <pkg>`  | `sudo dnf remove <pkg>`      |                                        |
| Удалить с конфигами           | `sudo apt purge <pkg>`   | —                            | в dnf конфиги обычно в `/etc` остаются |
| Обновить установленные        | `sudo apt upgrade`       | `sudo dnf upgrade`           |                                        |
| “Умный” апгрейд (меняет deps) | `sudo apt full-upgrade`  | `sudo dnf upgrade`           | dnf сам умеет конфликты решать         |
| Чистка сирот                  | `sudo apt autoremove`    | `sudo dnf autoremove`        |                                        |
| Поиск пакета                  | `apt search <name>`      | `dnf search <name>`          |                                        |
| Инфо о пакете                 | `apt show <pkg>`         | `dnf info <pkg>`             |                                        |

## Работа с файлами/владельцами

|Задача|Debian/Ubuntu|RHEL/CentOS/Fedora|Комментарий|
|---|---|---|---|
|Какой пакет даёт файл (по пути)|`apt-file search <path>`|`dnf provides <path>`|`apt-file` нужно установить и `apt-file update`|
|Кто “владеет” установленным файлом|`dpkg -S /usr/bin/ssh`|`rpm -qf /usr/bin/ssh`||
|Список файлов пакета (установленного)|`dpkg -L <pkg>`|`rpm -ql <pkg>`||

## Установка из локального файла

|Формат|Debian/Ubuntu|RHEL/CentOS/Fedora|Комментарий|
|---|---|---|---|
|`.deb` / `.rpm`|`sudo apt install ./file.deb`|`sudo dnf install ./file.rpm`|Так менеджер подтянет зависимости|
|“низкоуровневый” способ|`sudo dpkg -i file.deb` → `sudo apt -f install`|`sudo rpm -ivh file.rpm`|Лучше избегать без нужды|

## Репозитории

|Задача|Debian/Ubuntu|RHEL/CentOS/Fedora|Комментарий|
|---|---|---|---|
|Список реп|`grep -r . /etc/apt/sources.list*`|`dnf repolist all`||
|Добавить репо|файл в `/etc/apt/sources.list.d/*.list`|`dnf config-manager --add-repo URL`||
|Добавить ключ|`/etc/apt/keyrings/*.gpg` + `signed-by=`|(обычно в `.repo` указан gpgkey=)|Не юзай устаревший `apt-key`|
|Включить системный репо|—|`dnf config-manager --set-enabled crb`|пример для RHEL8/9|
|EPEL|—|`sudo dnf install -y epel-release`|много полезных пакетов|

## Версии/фиксация

|Задача|Debian/Ubuntu|RHEL/CentOS/Fedora|Комментарий|
|---|---|---|---|
|Поставить конкретную версию|`apt list -a nginx` → `sudo apt install nginx=1.24.0-1`|`dnf list --showduplicates nginx` → `sudo dnf install nginx-1.24*`|точное имя версии/релиза|
|Заморозить (hold/lock)|`sudo apt-mark hold <pkg>` / `apt-mark unhold`|`sudo dnf versionlock add <pkg>-<ver>` / `… delete`|нужно `dnf-plugins-core`|
|Проверить “hold”|`apt-mark showhold`|`dnf versionlock list`||

# Debian/Ubuntu (apt): истории есть, “отката кнопкой” — нет

## Где смотреть историю

```bash
# кратко — что ставилось/обновлялось/удалялось
less /var/log/apt/history.log

# подробно по каждому файлу (версии, пакеты)
less /var/log/dpkg.log

# старые логи (сжаты)
zgrep -E "install|upgrade|remove" /var/log/apt/history.log.*.gz
zgrep -E "install|upgrade|remove" /var/log/dpkg.log.*.gz
```

## Как “откатить” пакет (ручной способ)

1. Узнай доступные версии:
```bash
apt list -a nginx
```

2. Поставь нужную (предыдущую) версию:
```bash
sudo apt install nginx=1.24.0-1ubuntu1
```

3. Зафиксируй, чтобы не обновился снова:
```bash
sudo apt-mark hold nginx
# снять фиксацию:
sudo apt-mark unhold nginx
```

> Важно: откат сработает **только если** нужная старая версия ещё есть в репозиториях (или у тебя есть .deb в кэше/локально).

## Полезно

- “Сухой прогон” (посмотреть, что **сделало бы** apt):
```bash
sudo apt -s install nginx=1.24.0-1ubuntu1
```

---

# RHEL/CentOS/Fedora (dnf): есть **транзакции** и “undo”

## Где смотреть историю

```bash
sudo dnf history             # список транзакций с ID
sudo dnf history info 42     # подробности транзакции #42
sudo dnf history userinstalled   # что ставили вручную
```

## Как откатить транзакцию

```bash
sudo dnf history undo 42
```

- dnf попробует **вернуть пакеты к состоянию до транзакции #42**.
- Используй `--assumeno` сначала, чтобы посмотреть план:
```bash
sudo dnf history undo 42 --assumeno
```

### Если нужно откатиться к конкретной “старой” системе

Иногда удобнее не “undo X”, а “вернуться к состоянию транзакции Y”:

```bash
sudo dnf history rollback 37
```

(Доступно не во всех версиях dnf. Смысл тот же — восстановить состояние.)

> Как и в apt: откат пройдёт, **если старые версии доступны** в репах/кэше.

## Полезно

- Только посмотреть обновления (без установки):
```bash
sudo dnf check-update
```

- “Сухой прогон” любой команды:
```bash
sudo dnf upgrade --assumeno
```

---

## Почему в таблице было странно

Там поломался markdown (пайпы `|` внутри кода). Смысл строк был такой:

- **История операций**
    - Debian/Ubuntu: смотри `/var/log/apt/history.log` и `/var/log/dpkg.log`
    - RHEL/Fedora: `dnf history`
- **Откат транзакции**
    - Debian/Ubuntu: **ручной откат** — ставим предыдущую версию (`apt install pkg=версия`)
    - RHEL/Fedora: `dnf history undo <ID>` (или `rollback`)

---

## Мини-FAQ

- **А если нужной старой версии нет в репо?**
    - apt: ищи .deb (в `/var/cache/apt/archives/` или скачай из snapshot/old-repo), ставь `dpkg -i ./pkg.deb` → `sudo apt -f install`.
    - dnf: включи нужные репы/варианты, или используй локальный кэш, если не очищали (`/var/cache/dnf`).
- **Как понять, что именно обновилось сейчас?**
    - apt: `grep -i "upgrade" /var/log/apt/history.log | tail -n 20`
    - dnf: `dnf history info last`
- **Можно ли авто-создавать снапшоты перед апдейтами?**
    - На btrfs/lvm — да, но это уже отдельная тема (snapper, lvm snapshots).

## Очистка кэша

|Действие|Debian/Ubuntu|RHEL/CentOS/Fedora|Комментарий|
|---|---|---|---|
|Чистка кэша пакетов|`sudo apt clean`|`sudo dnf clean all`||
|Сироты|`sudo apt autoremove`|`sudo dnf autoremove`||

## Быстрые “комбухи”

|Сценарий|Команда|
|---|---|
|Поставить пакет последней версии и сразу посмотреть инфо|`sudo apt install -y pkg && apt show pkg` / `sudo dnf install -y pkg && dnf info pkg`|
|Найти, какой пакет содержит бинарь `nc`|`apt-file search bin/nc` / `dnf provides /usr/bin/nc`|
|Установить конкретную версию nginx|`apt list -a nginx ; sudo apt install nginx=<ver>` / `dnf list --showduplicates nginx ; sudo dnf install nginx-<ver>`|
|Включить EPEL и поставить htop|`—` / `sudo dnf install -y epel-release && sudo dnf install -y htop`|
|Удалить пакет и его конфиги|`sudo apt purge pkg` / `(нет полного аналога, проверь /etc)`|

## Где лежит конфиг/репо

|Дистрибутив|Пути|
|---|---|
|Debian/Ubuntu|`/etc/apt/sources.list`, `/etc/apt/sources.list.d/*.list`, ключи: `/etc/apt/keyrings/*.gpg`|
|RHEL/CentOS/Fedora|`/etc/yum.repos.d/*.repo`|

## Микро-памятка по отличиям

- **apt**: сначала `update`, потом `install/upgrade`. `purge` реально удаляет конфиги пакета.
- **dnf**: сам подтянет зависимости/конфликты, есть удобный `history undo`. Для «lock» — `dnf-plugins-core`.
# DNF: удобные фишки

|Задача|Команда|Комментарий|
|---|---|---|
|Обновить всё (с метаданными)|`sudo dnf upgrade --refresh`|эквивалент `dnf update`; `--refresh` как `apt update`|
|Посмотреть, что обновится|`sudo dnf check-update`|“сухой” список|
|История транзакций|`sudo dnf history` / `history info <ID>`|кто/когда/что ставил|
|Откат транзакции|`sudo dnf history undo <ID>`|сначала `--assumeno` посмотреть план|
|Фиксация версий (lock)|`sudo dnf install -y dnf-plugins-core && sudo dnf versionlock add nginx-1.24*`|`versionlock list/delete`|

---

# Snap vs Flatpak (настольные/CLI приложения)

|Критерий|**Snap**|**Flatpak**|Комментарий / когда что брать|
|---|---|---|---|
|База|Canonical, snapd|community, Flatpak + portals|Оба кросс-дистро|
|Команда|`snap find/install/list/remove`|`flatpak search/install/list/remove`||
|Источник|Snap Store|Flathub (обычно)||
|Обновления|авто фоном (`snap refresh`)|авто по расписанию/ручные (`flatpak update`)||
|Изоляция|confinement: `strict`/`classic`|sandbox + порталы|`classic` у snap ≈ без изоляции|
|Где хранит|`/var/lib/snapd/snaps`|`~/.local/share/flatpak` и `/var/lib/flatpak`|место быстро растёт|
|Права/пермишены|`snap connections`, `snap connect`|`flatpak permission-show`, `--filesystem=`|у flatpak гибче через порталы|
|CLI-утилиты|норм, но бывают “толстые”|реже для CLI|для чистых CLI лучше системный пакет|
|Когда брать|надо **самую свежую** версию и ок с sandbox|десктоп-аппы, “молодые” GUI|на серверах — обычно **не нужно**|

### Быстрые команды

```bash
# Snap
snap find app
sudo snap install app               # или --classic
snap list
sudo snap remove app
sudo snap refresh

# Flatpak
flatpak search app
sudo flatpak install flathub org.foo.Bar
flatpak list
sudo flatpak uninstall org.foo.Bar
flatpak update
```

---

# Python: pip / venv / pipx (best practices)

|Сценарий|Рекомендуется|Команды|Почему|
|---|---|---|---|
|Системные утилиты|**Пакет менеджера ОС**|`apt/dnf install …`|интеграция с deps/SELinux/paths|
|Проект (библиотеки)|**venv + pip**|`python3 -m venv .venv && . .venv/bin/activate && pip install -U pip`|изоляция зависимостей проекта|
|Утилиты для пользователя (CLI, глобально)|**pipx**|`pipx install black`|ставит в изолированный venv, безопасно|
|Никогда не делай|`sudo pip install …`|—|ломает системный Python/файлы пакетов|

### Быстрые команды

```bash
# venv для проекта
python3 -m venv .venv
. .venv/bin/activate
pip install -U pip wheel
pip install -r requirements.txt

# pipx для CLI-утилит
sudo apt/dnf install pipx   # или pip install --user pipx
pipx ensurepath
pipx install httpie
pipx list
pipx upgrade --all
```

---

# Node.js: npm / nvm / corepack (best practices)

|Сценарий|Рекомендуется|Команды|Почему|
|---|---|---|---|
|Версии Node|**nvm** или пакет дистрибутива|`nvm install --lts`|легко переключать версии|
|Зависимости проекта|**локально в проект**|`npm ci` / `npm install`|воспроизводимость, lockfile|
|Глобальные CLI|**corepack (pnpm/yarn)** или `npm -g`|`corepack enable`|аккуратнее с путями, меньше конфликтов|
|Никогда не делай|`sudo npm -g …`|—|портит права/систему|

### Быстрые команды

```bash
# nvm
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
. ~/.nvm/nvm.sh
nvm install --lts
nvm use --lts
node -v && npm -v

# проект
npm ci            # по lockfile
npm run build
```

---

# Когда что выбирать (в одном взгляде)

|Задача|Лучший инструмент|Почему|
|---|---|---|
|Серверный софт, демоны|`apt` / `dnf`|интеграция с systemd/SELinux, обновления безопаснее|
|Самая свежая десктоп-аппа|**Flatpak** (GUI) / **Snap**|удобные автообновления, sandbox|
|Python-библиотеки в проект|`venv + pip`|изоляция, контроль версий|
|Python-CLI для пользователя|**pipx**|изоляция, не ломает системный Python|
|Node-проект|`nvm` + `npm ci`|фиксированные версии Node и deps|
|Глобальные Node-CLI|`corepack enable` (+ pnpm/yarn)|чище управление менеджерами|

---

# Миксовать без боли

|Комбинация|Ок?|Комментарий|
|---|---|---|
|`dnf/apt` + `pipx`|✅|стандартный сетап|
|`dnf/apt` + `venv`|✅|библиотечные зависимости — в venv|
|`dnf/apt` + `snap/flatpak`|✅ (десктоп)|на серверах обычно не надо|
|`sudo pip` поверх системных пакетов|❌|сломаешь `apt/dnf` Python-модули|
|`sudo npm -g` системно|⚠️|легко поломать права PATH; лучше `corepack`/nvm|

---

# Чистка мусора / где место уходит

|Что|Где смотреть/чистить|
|---|---|
|DNF кэш|`sudo dnf clean all`|
|APT кэш|`sudo apt clean && sudo apt autoremove`|
|Snap образы|`snap list --all`, `sudo snap remove <pkg> --revision <old>`|
|Flatpak кэш|`flatpak uninstall --unused`|
|Пайтон колёса|`pip cache dir` / `pip cache purge`|
|npm кеш|`npm cache verify` / `npm cache clean --force`|
