# Время, часовые пояса, NTP (chrony), локали

## 0) Самое нужное (60 сек)

```bash
timedatectl                # сводка: TZ, NTP, RTC
timedatectl set-timezone Europe/Helsinki     # смена TZ (без ребута)
timedatectl set-ntp true   # включить синхронизацию (systemd-timesyncd/chrony)

date                       # локальное время
date -u                    # UTC
TZ=UTC date                # разово в UTC (без смены TZ)

hwclock --systohc          # записать системное время в RTC
hwclock --hctosys          # считать время из RTC в систему
```

---

## 1) Кто у меня синхронизирует время?

- **systemd-timesyncd** — легковесный клиент NTP (по умолчанию в Ubuntu/Debian).
- **chrony** — мощный NTP (клиент/сервер), лучше под виртуалки/нестабильные сети.
- **ntpd** — старичок, обычно не нужен.

Проверь:

```bash
systemctl status systemd-timesyncd  # если установлен
systemctl status chrony             # если установлен
timedatectl                         # поле "System clock synchronized"/"NTP service"
```

> Одновременно держи **что-то одно**: или chrony, или timesyncd. Не два сразу.

---

## 2) Быстрый старт с **chrony** (рекомендуется)

### Установка и включение

```bash
# Debian/Ubuntu
sudo apt-get install -y chrony

# RHEL/Fedora
sudo dnf install -y chrony

sudo systemctl enable --now chrony
```

### Мини-настройка клиента (`/etc/chrony/chrony.conf` или `/etc/chrony.conf`)

```conf
# Пулы публичных серверов (можно оставить дефолты)
pool pool.ntp.org iburst

# Сохранять дрейф (калибровка тактового генератора)
driftfile /var/lib/chrony/chrony.drift

# Разрешить "жёсткую" коррекцию при большом сдвиге при старте
makestep 1.0 3
```

Применить:

```bash
sudo systemctl restart chrony
```

### Диагностика chrony

```bash
chronyc tracking        # точность/смещение
chronyc sources -v      # источники и их состояние
chronyc sourcestats -v  # статистика источников
chronyc ntpdata         # детали NTP-пакетов
```

> Нормально видеть **смещение (offset)** в миллисекундах/сотых. Если секунды — что-то не так (см. “Troubleshoot”).

---

## 3) Хочешь, чтобы сервер **раздавал время** (NTP-сервер)

В `chrony.conf` добавь:

```conf
# Разрешить клиентам из своей сети
allow 10.0.0.0/8
# или узко:
# allow 10.23.42.0/24

# Пулы вверх по иерархии
pool pool.ntp.org iburst

# (опц.) локальный фоллбек, когда интернета нет
local stratum 10
```

Firewall:

```bash
# UFW
sudo ufw allow 123/udp

# nftables/iptables (пример)
sudo iptables -A INPUT -p udp --dport 123 -s 10.0.0.0/8 -j ACCEPT
```

Проверка со стороны клиента:

```bash
ntpdate -q <ip-ntp-сервера>   # только для теста (утилита устаревшая, но ок как "ping")
# или chrony-клиент:
chronyc -a 'server <ip> iburst' ; chronyc sources -v
```

---

## 4) **systemd-timesyncd** (минимализм)

Включить и посмотреть:

```bash
sudo timedatectl set-ntp true
systemctl status systemd-timesyncd
timedatectl timesync-status  # источники, последний sync, drift
```

Свой список серверов:

```ini
# /etc/systemd/timesyncd.conf
[Time]
NTP=time.google.com time.cloudflare.com time.facebook.com
FallbackNTP=pool.ntp.org
```

Применить:

```bash
sudo systemctl restart systemd-timesyncd
```

> Если ставишь **chrony**, обычно **выключают** timesyncd:

```bash
sudo systemctl disable --now systemd-timesyncd
```

---

## 5) RTC (аппаратные часы) и UTC

Лучше хранить RTC в **UTC**:

```bash
timedatectl set-local-rtc 0
```

Проверка/синхронизация:

```bash
hwclock --verbose
hwclock --systohc     # записать системное время в RTC
```

---

## 6) Локали (язык/форматы, LC_*)

Посмотреть:

```bash
locale            # текущие переменные
localectl status  # через systemd
```

### Debian/Ubuntu

```bash
sudo dpkg-reconfigure locales       # интерактивно выбрать en_US.UTF-8 ru_RU.UTF-8 и т.п.
sudo update-locale LANG=ru_RU.UTF-8 LC_TIME=en_DK.UTF-8
```

> Трюк: держать `LANG=ru_RU.UTF-8`, а **формат времени** ISO — `LC_TIME=en_DK.UTF-8` (YYYY-MM-DD, 24h).

### RHEL/Fedora

```bash
sudo localectl set-locale LANG=ru_RU.UTF-8
sudo localectl set-locale LC_TIME=en_DK.UTF-8
```

**Приоритет переменных:**

- `LC_ALL` — переопределяет всё (обычно **не ставим** постоянно).
- `LC_TIME`, `LC_NUMERIC`, `LC_COLLATE` и пр. — точечные.
- `LANG` — базовый язык по умолчанию.

Разово для команды:

```bash
LC_TIME=en_DK.UTF-8 date
LANG=C ls       # англ. сообщения/сортировка "C"
```

---

## 7) Траблшут (частые кейсы)

### Огромный сдвиг времени (минуты/часы)

1. Убедись, что **RTC=UTC**: `timedatectl | grep RTC`.
2. В chrony есть “жёсткая” коррекция:
```bash
    chronyc makestep
    ```
    или в конфиге уже стоит `makestep 1.0 3` (см. выше).
3. В VM **отключи** параллельные таймсервисы гипервизора/агента, если они конфликтуют, или наоборот включи рекомендуемый (у Azure/EC2/GCP есть свои гайды).

### Время «плавает» на VM
- chrony лучше справляется с неровным тикающим часовым источником.
- Проверь `chronyc tracking` → “Root distance”, “Skew”, “Last offset”.
- Убедись, что `TSC`/временной источник у гипера норм (KVM/VMware/Hyper-V — см. доки провайдера).

### Ничего не синхронится
- Firewall режет UDP/123 → открой.
- DNS не резолвит NTP-имена → проверь `resolvectl status`/`dig`.
- В логах:
```bash
    journalctl -u chrony --no-pager -n 200
    journalctl -u systemd-timesyncd --no-pager -n 200
    ```

### Контейнеры

- Обычно **не запускаем** NTP-клиента **внутри** контейнера, время берётся от хоста.
- Для приложений в контейнере — выставляй TZ переменной окружения:
```bash
    docker run -e TZ=Europe/Helsinki ...
    ```

---

## 8) Полезные команды “на каждый день”

```bash
# Смена таймзоны
timedatectl list-timezones | grep -i helsinki
sudo timedatectl set-timezone Europe/Helsinki

# Распечатать ISO-время (удобно в логах/скриптах)
date -Is               # 2025-10-12T20:41:05+03:00
date -u -Is            # в UTC

# Проверить NTP-состояние (systemd)
timedatectl timesync-status

# chrony: качества источников
chronyc sources -v
chronyc tracking

# Разово синхнуть и уйти (только для диагностики):
sudo chronyc burst 4/4
sudo chronyc makestep

# Проверить, кто слушает 123/udp (для NTP-сервера)
ss -uap | grep :123
```

---

## 9) Мини-чеклист для прод

-  Выбрал **один** тайм-клиент: `chrony` **или** `systemd-timesyncd`.
-  TZ выставлен (например, `Europe/Helsinki`), RTC хранится в **UTC**.
-  Для chrony прописан `makestep` и нормальные пулы.
-  Если сервер раздаёт NTP — `allow` в `chrony.conf` + открыт UDP/123.
-  Локали: `UTF-8` везде, `LC_TIME=en_DK.UTF-8` если хочешь ISO-формат.
-  В VM проверил рекомендации провайдера по времени (иногда лучше их timesync).

---
# chrony: клиент за 3 строки

```bash
sudo apt-get install -y chrony            # RHEL/Fedora: sudo dnf install -y chrony
echo 'pool pool.ntp.org iburst' | sudo tee /etc/chrony/chrony.conf >/dev/null
sudo systemctl enable --now chrony && chronyc tracking
```

> `iburst` = быстрое начальное выравнивание. Проверка: `chronyc sources -v`.

---

# chrony: сервер для 10.0.0.0/8

```bash
sudo tee /etc/chrony/chrony.conf >/dev/null <<'EOF'
pool pool.ntp.org iburst
allow 10.0.0.0/8               # разрешаем клиентам из 10/8
local stratum 10               # фоллбек, когда аплинка нет (опц.)
makestep 1.0 3                 # резко правим больш. сдвиг при старте
EOF
sudo systemctl enable --now chrony && sudo ufw allow 123/udp
```

> Клиент-проверка: `ntpdate -q <ip-сервера>` или `chronyc -a 'server <ip> iburst'; chronyc sources`.

---

# systemd-timesyncd: свой список NTP

```bash
sudo tee /etc/systemd/timesyncd.conf >/dev/null <<'EOF'
[Time]
NTP=time.google.com time.cloudflare.com time.facebook.com
FallbackNTP=pool.ntp.org
EOF
sudo systemctl restart systemd-timesyncd && timedatectl timesync-status
```

> Включить синхру (если была off): `sudo timedatectl set-ntp true`.  
> Если используешь chrony — отключи timesyncd: `sudo systemctl disable --now systemd-timesyncd`.

---

# Локали: быстро en/ru + ISO-дата

```bash
# Debian/Ubuntu:
sudo dpkg-reconfigure locales <<< $'ru_RU.UTF-8\nen_US.UTF-8\n'
sudo update-locale LANG=ru_RU.UTF-8 LC_TIME=en_DK.UTF-8
locale && date && LC_TIME=en_DK.UTF-8 date

# RHEL/Fedora (альтернатива):
# sudo localectl set-locale LANG=ru_RU.UTF-8
# sudo localectl set-locale LC_TIME=en_DK.UTF-8
```

> `LC_TIME=en_DK.UTF-8` даёт удобный ISO-формат (YYYY-MM-DD, 24h), оставляя интерфейс на русском.