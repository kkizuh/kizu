## Что это?

“Бронежилеты” для Linux:
- **SELinux** (обычно RHEL/Rocky/Alma/Fedora): метки и строгие правила, кто к чему может.
- **AppArmor** (обычно Ubuntu/Debian): профили по путям типа “процесс X может читать Y”.

Если политика не разрешает — действие блокируется, даже если `chmod`/`chown` разрешают.

## Зачем тебе?

Чтобы сервисы не творили дичь (чтение чужих файлов, сетевые приколы). Но иногда это ломает деплой — тогда надо быстро диагностировать и выдать нужное разрешение, а не вырубать всё нафиг.

---

## Узнать, что у тебя

```bash
# SELinux?
getenforce        # Enforcing | Permissive | Disabled
sestatus

# AppArmor?
aa-status         # loaded profiles, complain/enforce
```

---

## Симптомы “это они”

- Сервис “permission denied”, хотя права ОК.
- Порт нестандартный — сервис не стартует/не слушает.
- Логи ядра/аудита ругаются на **AVC** (SELinux) или **DENIED** (AppArmor).

---

## Быстрый выживальник (не руша безопасность насмерть)

### Если SELinux (RHEL/Fedora)

1. Временно ослабь (логирует, но не блокирует):

```bash
sudo setenforce 0        # вернёшь потом: setenforce 1
```

2. Поймай, что блокировалось:

```bash
sudo ausearch -m avc -ts recent
sudo sealert -a /var/log/audit/audit.log | less
```

3. Почини по шаблону (самые частые кейсы ниже).
4. Верни строгость:

```bash
sudo setenforce 1
```

### Если AppArmor (Ubuntu/Debian)

1. Профиль в “жалобный” режим, чтобы не блокировал:

```bash
sudo aa-complain /etc/apparmor.d/*nginx*   # пример для nginx
```

2. Посмотри в `journalctl -k` что именно запретили.
3. Добавь путь/права в профиль → загрузить:

```bash
sudo aa-enforce /etc/apparmor.d/*nginx*
sudo systemctl reload apparmor
```

---

## 5 частых фиксов — копипаст и готово

### 1) Web-сервис не может читать/писать каталог (RHEL/SELinux)

```bash
# дать вебу читать контент
sudo semanage fcontext -a -t httpd_sys_content_t '/srv/www(/.*)?'
sudo restorecon -Rv /srv/www

# дать вебу ПИСАТЬ в uploads
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/srv/www/uploads(/.*)?'
sudo restorecon -Rv /srv/www/uploads
```

### 2) Web-сервис ходит в интернет (RHEL/SELinux)

```bash
sudo setsebool -P httpd_can_network_connect 1
```

### 3) Слушаем нестандартный порт (RHEL/SELinux)

```bash
sudo semanage port -a -t http_port_t -p tcp 8080
```

### 4) Docker volume монтируется — “permission denied” (RHEL/SELinux)

Добавь метку при монтировании:

```bash
# compose:
# volumes:
#   - ./data:/var/lib/app:z   # :z — шарим между контейнерами
#   - ./secret:/secret:Z      # :Z — приватно для одного
```

### 5) AppArmor: дать nginx читать доп. путь (Ubuntu)

Открой профиль, добавь строку и применяй:

```
/srv/www/**  r,
```

```bash
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
sudo systemctl reload apparmor
```

---

## Где смотреть логи

- **SELinux:**  
    `sudo ausearch -m avc -ts recent`  
    `sudo sealert -a /var/log/audit/audit.log`
- **AppArmor:**  
    `sudo dmesg | grep -i DENIED`  
    `sudo journalctl -k | grep -i apparmor`

---

## DO / DON’T

**DO:**

- В инциденте — Permissive/Complain **временно**, собрать логи, выдать точечное разрешение, вернуть Enforcing.
- Для файлов — `semanage fcontext` + `restorecon` (а не только `chcon`, он слетает).
- Регать порты через `semanage port`.

**DON’T:**

- Не ставь **Disabled** на SELinux постоянно — теряешь логи и защиту.
- Не оставляй AppArmor профили в `complain` навсегда.

---

## Мини-шпора команд (распечатай мысленно)

**SELinux:**

```bash
getenforce | setenforce 0|1
sudo sealert -a /var/log/audit/audit.log
sudo semanage fcontext -a -t TYPE 'PATH(/.*)?' && sudo restorecon -Rv PATH
sudo setsebool -P VAR 1
sudo semanage port -a -t TYPE -p tcp 8080
```

**AppArmor:**

```bash
aa-status
sudo aa-complain /etc/apparmor.d/PROFILE
sudo aa-enforce  /etc/apparmor.d/PROFILE
sudo systemctl reload apparmor
```

Если хочешь — сделаю это как компактную страницу `selinux-apparmor-basics` в твоей базе с ещё парой готовых рецептов (PostgreSQL, NFS, Samba).