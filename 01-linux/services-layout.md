ок, по-простому: **“где что лежит” в Debian/Ubuntu vs RHEL/CentOS/Fedora**. Держи одну таблицу и пару правил памяти.

# Где что лежит (Debian/Ubuntu ↔ RHEL/CentOS/Fedora)

|Тема|Debian/Ubuntu|RHEL/CentOS/Fedora|Комментарий/запомнить|
|---|---|---|---|
|Пакетный менеджер|`apt`, `dpkg`|`dnf` (или `yum` на старых), `rpm`|apt/dnf — высокоур.; dpkg/rpm — низкоур.|
|Файлы репозиториев|`/etc/apt/sources.list` + `/etc/apt/sources.list.d/*.list`|`/etc/yum.repos.d/*.repo`|Ключи: `/etc/apt/keyrings/*.gpg`; в RHEL в `.repo` есть `gpgkey=`|
|Кэш пакетов|`/var/cache/apt/archives/`|`/var/cache/dnf/`|Чистка: `apt clean`, `dnf clean all`|
|Логи пакетных операций|`/var/log/apt/history.log`, `/var/log/dpkg.log`|`dnf history` (внутри БД dnf)|В Debian — обычные логи, в RHEL — команда|
|Службы (systemd юниты)|`/lib/systemd/system` (пакетные), overrides в `/etc/systemd/system`|`/usr/lib/systemd/system` (пакетные), overrides в `/etc/systemd/system`|Всегда правь **/etc/systemd/system** (override)|
|Логи (journald)|`journalctl` (хранение по умолчанию в RAM)|то же самое|Persistent: `/var/log/journal` создать и перезапустить journald|
|syslog (rsyslog)|`/etc/rsyslog.d/` → файлы в `/var/log/*`|то же самое|Не везде включён по умолчанию|
|Сетевые настройки (legacy)|`/etc/network/interfaces` (старый Debian)|`/etc/sysconfig/network-scripts/ifcfg-*` (старый RHEL)|Сегодня чаще **NetworkManager** или **systemd-networkd**|
|NetworkManager keyfiles|`/etc/NetworkManager/system-connections/*`|то же|Формат одинаков|
|systemd-networkd|`/etc/systemd/network/*.network`|то же|Если используешь именно его|
|hostname / hosts|`/etc/hostname`, `/etc/hosts`|то же|Универсально|
|DNS (systemd-resolved)|`resolvectl status`, stubs `/etc/resolv.conf` → symlink|аналогично|Иногда без resolved — тогда файл редактируют вручную|
|Файрвол|`ufw` (упрощалка над `nft/iptables`)|`firewalld` (зоны)|В обоих низкоуровневое — **nftables**|
|SELinux/AppArmor|**AppArmor** по умолчанию (Ubuntu): `/etc/apparmor.d/`|**SELinux** по умолчанию: `/etc/selinux/config`, политики в `/etc/selinux/…`|Проверка: `aa-status` vs `getenforce`|
|crontab|`/etc/crontab`, `/etc/cron.*`, `crontab -e`|то же|Таймеры systemd: `/etc/systemd/system/*.timer`|
|sudo|`/etc/sudoers`, `/etc/sudoers.d/*`|то же|Редактировать `visudo`|
|Пользователи/группы|`/etc/passwd`, `/etc/group`, `/etc/shadow`|то же|Универсально|
|Конфиг SSH|`/etc/ssh/sshd_config`|то же|Сервис: `ssh.service`/`sshd.service` (в RHEL — `sshd`)|
|NTP по умолчанию|`systemd-timesyncd` (Ubuntu) или `chrony`|`chrony`|Конфиг chrony: `/etc/chrony/chrony.conf` (Deb), `/etc/chrony.conf` (RHEL)|
|Web-сервер имена|`apache2`, `nginx`|`httpd` (Apache), `nginx`|В RHEL Apache — **httpd**, не apache2|
|Пути Apache|`/etc/apache2/`, сайты в `sites-available/enabled`|`/etc/httpd/`, конфиги в `conf.d/*.conf`|Разные layout’ы каталогов|
|Пути Nginx|`/etc/nginx/`, сайты: `sites-available/enabled` (Ubuntu)|`/etc/nginx/nginx.conf`, обычно `conf.d/*.conf`|На RHEL нет sites-* из коробки|
|MariaDB/MySQL|сервис `mysql`/`mariadb`, конфиг `/etc/mysql/`|сервис `mariadb`, конфиг `/etc/my.cnf`, `conf.d/`|Проверяй имя юнита|
|PostgreSQL|`/etc/postgresql/<ver>/main/`|`/var/lib/pgsql/<ver>/data/`, конфиг `/var/lib/pgsql/.../postgresql.conf`|В RHEL конфиг в data-dir|
|Docker|`/etc/docker/daemon.json`|то же|Юнит `docker.service` в обеих|
|Виртуализация KVM|libvirt: `/etc/libvirt/`, образы `/var/lib/libvirt/images/`|то же|Универсально|

---

## Две простые мнемоники

1. **Systemd-юниты:**
    - **Debian/Ubuntu:** пакетные в `/lib/systemd/system`, твои оверрайды — **всегда** `/etc/systemd/system/…`
    - **RHEL/Fedora:** пакетные в `/usr/lib/systemd/system`, оверрайды — тоже `/etc/systemd/system`
2. **Имена сервисов:**
    - Apache: **`apache2`** (Deb) vs **`httpd`** (RHEL).
    - SSH: **`ssh.service`** (Deb) vs **`sshd.service`** (RHEL).
    - MariaDB: чаще **`mariadb`** (RHEL), на Ubuntu может быть `mysql`/`mariadb`.

---

## Что отличает семейства на практике

- **Пакеты и репозитории:** `apt` использует текстовые списки репов; `dnf` — `.repo` файлы (понятно через `dnf repolist`).
- **Безопасность по умолчанию:** Ubuntu — AppArmor; RHEL — SELinux (не отключай, чини контексты).
- **Файрволы:** Ubuntu любит `ufw`, RHEL — `firewalld`; низкоуровневый стандарт — **nftables** везде.
- **Сети (legacy):** старые Debian — `/etc/network/interfaces`; старые RHEL — `ifcfg-*`. Сейчас часто **NetworkManager** везде.

---

## Мини-практика (как быстро найти, “где лежит”)

```bash
# Где юнит и откуда он взят:
systemctl status nginx
systemctl cat nginx        # покажет полный итог конфиг + путь
systemctl show -p FragmentPath nginx

# Какой пакет владеет файлом:
# Debian/Ubuntu
dpkg -S /etc/nginx/nginx.conf
# RHEL/Fedora
rpm -qf /etc/nginx/nginx.conf

# Где лежат конфиги сервиса целиком:
# Обычно man-страница скажет:
man nginx | grep -i -E "conf|path"
```
## Системные юниты (system-wide)

- **Debian/Ubuntu (пакетные):** `/lib/systemd/system/`
- **RHEL/CentOS/Fedora (пакетные):** `/usr/lib/systemd/system/`
- **Твои (ручные/оверрайды):** `/etc/systemd/system/` ← вот тут создаём и правим
- **Рантайм-сгенерённые:** `/run/systemd/system/` (временка, перезагрузка — и всё)

## User-юниты (для `--user`)

- Пакетные: `/usr/lib/systemd/user/` (на Debian может быть `/lib/systemd/user/`)
- Твои: `~/.config/systemd/user/`
- Рантайм: `/run/user/<UID>/systemd/`

## Drop-in (оверрайды к юниту)

- `/etc/systemd/system/<unit>.d/*.conf`
- (user) `~/.config/systemd/user/<unit>.d/*.conf`

### Подсказки

- Что реально используется: `systemctl cat <unit>` (покажет путь и drop-ins).
- Приоритет: `/etc` → `/run` → `/usr/lib`/`/lib`.
- После правок: `systemctl daemon-reload`.

![[Pasted image 20250716191611.png]]
Картинка норм как базовая FHS-памятка, но есть важные апдейты 2025
- **usr-merge** почти везде:  
    `/bin` → `/usr/bin`, `/sbin` → `/usr/sbin`, `/lib` → `/usr/lib` (и `/lib64`). Часто это **symlink’и**. Идея: весь софт живёт под `/usr`, корень — тонкий.
- **/run** (tmpfs, рантайм-состояние) — must know.  
    PID-файлы, сокеты, юниты во время работы.  
    Старые пути: `/var/run` → symlink на `/run`, `/var/lock` → `/run/lock`.
- **/boot/efi** для UEFI: FAT32-раздел (ESP) монтируется сюда.  
    GRUB/bootloaders кладут свои `.efi` сюда.
- **systemd юниты**: пакетные в `/lib/systemd/system` (Debian/Ubuntu) или `/usr/lib/systemd/system` (RHEL/Fedora); свои и overrides — в **`/etc/systemd/system`**.
- **/srv** редко используется как “данные сервисов”; чаще смотри в `/var/lib/<service>` (docker, postgres и т.п.).    
- **/tmp** иногда — tmpfs и/или чистится при ребуте (зависит от дистро/настроек).
- **/home** — ок; **/root** — домик для root — тоже ок.
- **/opt** и **/usr/local** живы для “чужих”/локальных установок, но в проде всё чаще контейнеры → смотри ещё `/var/lib/docker`, `/var/lib/containers`, `/var/lib/flatpak`, `/var/lib/snapd`.
Если коротко: добавь в голову **usr-merge, /run, /boot/efi** и пути для **systemd** — и картинка будет актуальной.
![[Pasted image 20250716184524.png]]
Коротко: почти да, но с поправками на «сегодня».
- `/etc` — конфиги; `/var` — меняющееся (логи, спулы); `/home` — дома;  
    `/boot` — загрузчик; `/dev` — девайсы (udev); `/proc` — виртуалка ядра.
- Точки монтирования: `/media` (съёмные), `/mnt` (временные) — норм.
    

## Что изменилось/нужно знать в 2025

- **usr-merge**: на большинстве дистров `/bin`, `/sbin`, `/lib` — это **symlink’и** в `/usr/bin`, `/usr/sbin`, `/usr/lib` (а ещё есть `/lib64`). Так и должно быть.
- **/run**: важная runtime-директория (tmpfs) для PID-файлов, сокетов, state’а systemd. На схеме её нет — добавь в голову.
- **UEFI**: обычно есть `/boot/efi` (FAT32) как точка монтирования ESP.
- **Журналы systemd**: для постоянного хранение — `/var/log/journal/` (если создано).
- **/srv**: «данные сервисов» — используется не везде; многие кладут в `/var/lib/<service>`.
- **/opt**: сторонние пакеты — да, но всё чаще ставят в контейнеры или в `/usr/local`.
- **Контейнеры/пакетные форматы**: могут создавать свои каталоги (`/var/lib/docker`, `/var/lib/containers`, `/var/lib/flatpak`, `/var/lib/snapd` и т.п.) — это вне FHS, но норма.
- **/tmp**: иногда чистится при перезагрузке, на некоторых системах тоже tmpfs.
- **/root**: домашний для root — да, так и осталось.

## Итог

Схема актуальна как базовая FHS-памятка, но подумай о трёх апдейтах: **usr-merge, /run, /boot/efi**. 