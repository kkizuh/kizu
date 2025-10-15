всё норм — это классика. Разберём твой `ulimit -a` “на человеческом”, что это значит и когда трогать. Буду очень прямолинейным.

---

# Что такое `ulimit`

Это лимиты для процессов текущего шелла (и того, что он запускает): сколько файлов можно открыть, какой размер стека, можно ли писать core-дампы и т.д.  
Есть **soft** (текущее) и **hard** (максимум). `ulimit -a` показывает soft.

---

## Разбор твоего вывода по пунктам (простыми словами)

|Поле (`ulimit` ключ)|Что это по-жизни|Когда трогать|Типичное значение|
|---|---|---|---|
|**real-time non-blocking time (-R)**|Время для RT-потоков (редкая экзотика).|Никогда, если не RT-софтина.|`unlimited` ок.|
|**core file size (-c)**|Размер core-дампов (файл с памятью процесса при крэше).|Включают, чтобы отладить крэш.|`0` = выкл. Включить: `ulimit -c unlimited`.|
|**data seg size (-d)**|Размер сегмента данных процесса (исторический).|Не трогай.|`unlimited` норм.|
|**scheduling priority (-e)**|Стартовый приоритет планировщика (nice/rt).|Меняют через `nice`/`chrt`.|`0` ок.|
|**file size (-f)**|Максимальный размер создаваемого файла.|Для build/логов может упереться.|`unlimited` ок.|
|**pending signals (-i)**|Сколько сигналов в очереди у юзера.|Не трогай.||
|**max locked memory (-l)**|Сколько памяти можно **залочить** (mlock). Нужно БД/eBPF.|Если софт ругается на `mlock`/`MEMLOCK`.|У тебя `251380 KB` ≈ 245 MB. Для БД: поднять (напр. `LimitMEMLOCK=infinity`).|
|**max memory size (-m)**|Устаревшее (RSS limit).|Игнор.|`unlimited`.|
|**open files (-n)**|Сколько **одновременно открытых файлов/сокетов** на процесс.|Важнейший! Веб/БД/прокси — поднимать.|1024 у тебя — **мало** для серверов. Делай 32k–64k.|
|**pipe size (-p)**|Размер pipe в блоках по 512 B.|Не трогай.||
|**POSIX message queues (-q)**|Очереди сообщений POSIX.|Редко трогают.||
|**real-time priority (-r)**|RT-приоритеты.|Только для RT-софта.||
|**stack size (-s)**|Размер стека потока.|Для Java/Go/много рекурсии иногда меняют.|8192 KB (8 MB) норм.|
|**cpu time (-t)**|Лимит CPU-времени процесса (сек).|Обычно `unlimited`.||
|**max user processes (-u)**|Сколько процессов/тредов у юзера.|Для app-серверов иногда увеличить.|7422 у тебя — ок.|
|**virtual memory (-v)**|Лимит виртуальной памяти.|Обычно `unlimited`.||
|**file locks (-x)**|Сколько блокировок файлов.|Обычно `unlimited`.||

---

## Окей, что мне реально важно?

Три вещи, которые чаще всего спасают прод:

1. **`open files (-n)`** — подними до **65535** (или выше) для веба/БД/прокси/воркеров.
2. **`max locked memory (-l)`** — если БД/eBPF жалуются, ставь больше (или `infinity`).
3. **`max user processes (-u)`** — если создаёшь кучу потоков/воркеров.
    

---

## Как поднять лимиты — правильно

### Временно (в текущем шелле)

```bash
ulimit -n 65535            # файлов/сокетов
ulimit -l unlimited        # memlock
ulimit -u 32768            # процессов/тредов
ulimit -c unlimited        # core dumps
```

Это пропадёт при выходе из шелла.

### Постоянно для **systemd-сервиса** (лучший способ)

```bash
sudo systemctl edit myapp.service
```

В открывшемся файле:

```ini
[Service]
LimitNOFILE=65535
LimitNPROC=32768
LimitMEMLOCK=infinity
LimitCORE=infinity
```

Применяем:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
systemctl show myapp -p LimitNOFILE -p LimitNPROC -p LimitMEMLOCK -p LimitCORE
```

Проверить у конкретного процесса:

```bash
pidof myapp | xargs -I{} sh -c 'echo PID {}; cat /proc/{}/limits | egrep "open files|processes|max locked memory|Max core file size"'
```

### Для интерактивных SSH-сессий (shell), через PAM

```bash
sudoedit /etc/security/limits.conf
```

Добавь (пример):

```
*      soft  nofile  65535
*      hard  nofile  65535
*      soft  nproc   32768
*      hard  nproc   32768
```

Перелогинься по SSH.  
Учти: на демоны через systemd это может **не подействовать** — для демонов лучше предыдущий способ.

### В Docker (если у тебя контейнеры)

```bash
docker run --ulimit nofile=65535:65535 --ulimit nproc=32768 ...
```

---

## Короткие “комбухи” по ситуациям

### Веб/прокси (nginx/haproxy/node)

- Поставь: `LimitNOFILE=65535` в юните.
    
- В `sysctl` подними сетевые очереди (мы уже делали в шпоре по sysctl).
    
- Чек:
    
```bash
    lsof -p $(pidof nginx) | wc -l
    ss -s
    ```
    

### Postgres/MySQL

- `LimitNOFILE=65535`
- (если просит) `LimitMEMLOCK=infinity`
- Core dumps при редких крэшах: `LimitCORE=infinity` + `ulimit -c unlimited` (для интерактива) + `kernel.core_pattern` (по желанию).
    

### Build-сервер/CI

- `LimitNOFILE=65535`, `LimitNPROC=32768`
- Core dumps по необходимости.

---

## Частые вопросы

- **Почему `ulimit -n` важен?** Каждый TCP-коннект = файл-дескриптор. Тысячи коннектов → упрёшься в 1024 и пойдут ошибки `EMFILE`/“Too many open files”.
- **Где вижу реальные лимиты процесса?** `/proc/<PID>/limits`.
- **Почему я поднял `limits.conf`, а сервис всё равно 1024?** Потому что демоны запускает **systemd**, и он не читает `limits.conf`. Делай через `systemctl edit`.
- **Поднял лимит, а всё равно плохо?** Возможно, упрёшься в сетевые буферы/conntrack/`somaxconn`. Смотри шпору “sysctl”.
    

---

## Быстрый чек-лист (сделай и забудь)

1. Для своих сервисов (systemd):
```ini
LimitNOFILE=65535
LimitNPROC=32768
```
2. Для БД/eBPF, если просит:
```ini
LimitMEMLOCK=infinity
```

3. Для отладки крэшей (по ситуации):
```ini
LimitCORE=infinity
```

4. Проверяй:
```bash
systemctl show myapp -p LimitNOFILE -p LimitNPROC -p LimitMEMLOCK -p LimitCORE
```

Если хочешь, скажи какие сервисы у тебя крутятся (nginx? postgres? node?), я дам точные значения и покажу, чем проверить, что именно **они** увидели новые лимиты.