## Быстрый минимум

```bash
# пользователи/группы
id user                  # UID/GIDs
getent passwd user       # строка из /etc/passwd (или LDAP)
getent group grp         # инфо о группе
sudo useradd -m -s /bin/bash user
sudo passwd user
sudo usermod -aG sudo user     # добавить в группу (не забыть -a!)
groups user
sudo userdel -r user

# смена владельца/прав
sudo chown -R user:grp /srv/app
chmod 640 file            # rw-r-----
chmod 750 dir             # rwx r-x ---
chmod 2775 /srv/share     # setgid на папке → новые файлы наследуют группу
```

---

## Пользователи/группы: практика

- **Создать**: `useradd -m -s /bin/bash user` (создаст `/home/user`), **пароль**: `passwd user`.
- **Добавить в группу**: `usermod -aG docker user` (без `-a` — выкинешь из остальных групп).
- **Истекающий доступ**: `chage -E 2025-12-31 user` (автодеактивация в дату).
- **Заблокировать**: `usermod -L user` или `passwd -l user`. Разблок: `-U`.
- **Сменить shell**: `chsh -s /usr/sbin/nologin user` (запрет интерактива).
- **Шаблон домашней**: `/etc/skel` (файлы попадут в новый `$HOME`).

### Где лежит что

- `/etc/passwd` (uid/gid/shell/home), `/etc/shadow` (хэши), `/etc/group`.
- Политика паролей: `/etc/login.defs`, PAM: `/etc/pam.d/common-password` (Debian) / `system-auth` (RHEL).

---

## sudo: как надо

> Редактируй **только** через `visudo` → безопасный синтаксис.

### Файл и include
- Основной: `/etc/sudoers`, фрагменты: `/etc/sudoers.d/<имя>`.    
- Всегда указывай **пользователей/группы** и **команды**, минимальные права.
### Паттерны
```sudoers
# Разрешить группе wheel всё с паролем
%wheel ALL=(ALL) ALL

# Точечное право: рестарт nginx без пароля
%webops ALL=(root) NOPASSWD: /bin/systemctl restart nginx

# Ограничение: только tail логов nginx
logreader ALL=(root) NOPASSWD: /usr/bin/tail -F /var/log/nginx/access.log

# Запуск от app-пользователя (не root)
deployer ALL=(app) NOPASSWD: /usr/bin/systemctl restart myapp

# Жёстче: запретить шелл-эскейп
Defaults!LESS noexec
Defaults!vim  noexec

# Логи по максимуму (по умолчанию в journal/secure)
Defaults log_output, logfile="/var/log/sudo-audit.log"
```

**Проверить**: `sudo -l -U user` (что доступно), `sudo -k` (сброс кэша пароля).

---

## Права/биты: что помнить

- `r=4 w=2 x=1` → `750` = `rwx r-x ---`.
- **setuid (4xxx)**: исполнять с UID владельца (используй с осторожностью).
- **setgid (2xxx)**:
    - на **файле** — GID владельца;
    - на **директории** — новые файлы/папки наследуют **группу** директории (очень полезно!).
- **sticky (1xxx)**: на общих каталогах (например, `/tmp`), удалять файл может только владелец. Пример: `chmod 1777 /share/tmp`.

### Шэринг каталога командой

```bash
# цель: общий каталог для группы proj, все новые файлы — группы proj, права rw для группы
sudo groupadd proj
sudo mkdir -p /srv/proj
sudo chown -R root:proj /srv/proj
sudo chmod 2775 /srv/proj                    # setgid
# (лучше ещё ACL, см. ниже)
```

---

## ACL: когда chmod не хватает

> ACL позволяет гранулировать доступ: нескольким пользователям/группам, дефолтные правила для нового контента.

### База

```bash
# дать user1 rwx на папку
setfacl -m u:user1:rwx /srv/proj

# дать группе qa только r-x
setfacl -m g:qa:rx /srv/proj

# дефолтные ACL (наследуются для новых файлов/папок!)
setfacl -d -m u:user1:rwx /srv/proj
setfacl -d -m g:proj:rwx /srv/proj

# удалить ACL
setfacl -b /srv/proj

# посмотреть
getfacl /srv/proj
```

### Шаблон “рабочий общий каталог”

```bash
sudo mkdir -p /srv/share
sudo chown root:proj /srv/share
sudo chmod 2770 /srv/share
setfacl -m g:proj:rwx /srv/share
setfacl -d -m g:proj:rwx /srv/share
setfacl -m o::--- /srv/share
setfacl -d -m o::--- /srv/share
```

— новые файлы всегда **группы `proj`**, группа имеет `rwx`, посторонним **ноль**.

---

## SSH-ключи и доступ

```bash
# создать ключ
ssh-keygen -t ed25519 -C "user@host"

# выдать доступ (как root на сервере)
mkdir -p ~user/.ssh && chmod 700 ~user/.ssh
echo 'ssh-ed25519 AAAA... коммент' >> ~user/.ssh/authorized_keys
chown -R user:user ~user/.ssh
chmod 600 ~user/.ssh/authorized_keys
```

Быстро ограничить доступ: `sshd_config`: `AllowUsers user1 user2` или `AllowGroups sshusers` → `systemctl reload sshd`.

---

## Диагностика “permission denied”

- Кто владелец/права? → `ls -l`, `namei -l /path/to/file` (цепочка прав до файла).
- В какой группе процесс? → `ps -o user,group,comm,pid …`
- Блокируют ACL? → `getfacl`.
- SELinux/AppArmor? (RHEL/Debian) → временно проверить: `getenforce` / `aa-status`. **Не** выключай на проде без понимания — смотри audit логи.
- Кто держит файл/порт? → `lsof | grep <path>` / `ss -tulpn`.
    

---

## “Спасаем прод” — быстрые приёмы

- **Срочно запретить интерактив пользователя**:  
    `usermod -L user && chsh -s /usr/sbin/nologin user && pkill -KILL -u user`
- **Отрубить доступ по SSH**:  
    Удалить ключи в `~user/.ssh/authorized_keys` + `DenyUsers user` в `sshd_config` → `systemctl reload sshd`.
- **Дать безопасный доступ только к нужной команде**:  
    `echo 'user ALL=(root) NOPASSWD: /bin/systemctl restart mysvc' >/etc/sudoers.d/mysvc`
- **Общий каталог без бардака**:  
    `chmod 2770` + **default ACL** (см. шаблон выше).
- **Исправить “всё принадлежит root”**:  
    `chown -R app:app /srv/app && find /srv/app -type d -exec chmod 750 {} \; && find /srv/app -type f -exec chmod 640 {} \;`
    

---

## Что спрашивают на собесах (и как отвечать коротко)

- **Разница между uid/gid/aux группы?**  
    UID — идентификатор пользователя; GID — основная группа; вторичные группы расширяют доступ.
- **setuid/setgid/sticky?**  
    setuid — исполнять с UID владельца; setgid — с GID (на папке — наследование группы); sticky — удалять в каталоге может только владелец.
- **ACL vs chmod?**  
    chmod — одна маска owner/group/other; ACL — произвольные правила для множества users/groups + default для новых файлов.
- **Почему `chmod 2770` на каталоге для команды?**  
    setgid гарантирует, что **все новые файлы наследуют группу каталога** → доступ команде без ручного `chgrp`.
- **Безопасный sudo?**  
    Право **на конкретные команды**, по группам, `NOPASSWD` только там, где безопасно, логирование включено, noexec на опасных просмоторщиках.
- **umask?**  
    Базовая маска для новых файлов/директории. Например, `umask 027` → файлы 640, каталоги 750.

---

## Мини-таблицы (на память)

### chmod (окталь)

```
7=rwx  6=rw-  5=r-x  4=r--  3=-wx  2=-w-  1=--x  0=---
```

### полезные комбо

```
600  файл с секретами (rw-------)
640  конфиги (rw-r-----)
700  скрипты в $HOME (rwx------)
750  каталоги приложений (rwxr-x---)
755  public каталоги/бинарники (rwxr-xr-x)
2775 shared dir с наследованием группы (rwxrwxr-x + setgid)
1777 tmp-подобные каталоги (rwxrwxrwx + sticky)
```

---

## Миграции/онбординг людей (чеклист)

- Создать юзера, выдать SSH-ключ, `-aG` нужные группы.
- Сформировать sudo-правила **по ролям** в `/etc/sudoers.d/role-*`.
- Создать общие каталоги с `setgid` + default ACL
- Задать `umask` (например, 027) в `/etc/profile` или через PAM.
- Включить аудит sudo (`Defaults log_output, logfile=...`).
- Break-glass аккаунт с MFA/аппаратным токеном хранится отдельно.
