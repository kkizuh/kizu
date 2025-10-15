# sysctl / ulimit — человеческим языком

## Картинка мира

- **sysctl** — это _тумблеры ядра_. Включают/подкручивают поведение сети и системы **для всей машины**.
- **ulimit** — это _крышка на банке_ для **одного процесса/сервиса**: сколько файлов он может открыть, сколько потоков и т.п.

Дальше — 5 вещей, которые пригодятся в реальной работе. Всё остальное забудь.

---

## 1) “Много соединений — сервер задыхается”

**Зачем:** веб/прокси/БД иногда не успевают принимать новые коннекты.

Сделай один файл:
```bash
sudoedit /etc/sysctl.d/99-basics.conf
```
Вставь:
```conf
# Больше очередей для входящих соединений
net.core.somaxconn = 1024
net.ipv4.tcp_max_syn_backlog = 4096
```
Применить:
```bash
sudo sysctl --system
```
**Как проверить, что стало лучше:**
- `ss -s` — смотреть сводку TCP (меньше SYN-RECV на пике = лучше).
- Логи веб-сервера: меньше “connection refused”/“too many open files”.

---

## 2) “Клиент отвалился, а соединение висит”

**Зачем:** быстрее чистить зомби-соединения.
Добавь (в тот же `99-basics.conf`):
```conf
# Быстрее пингуем и закрываем мёртвые TCP-сессии
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5

# Состояние FIN долго не держать
net.ipv4.tcp_fin_timeout = 15
```
Применить: `sudo sysctl --system`.

---

## 3) “Не хватает исходящих портов для кучи исходящих запросов”

(актуально для сервисов, которые делают много внешних HTTP-запросов)
```conf
# Больше эфемерных (временных) портов
net.ipv4.ip_local_port_range = 10240 65000
```

---

## 4) “Процесс упирается в ‘Too many open files’”

Это **ulimit**, не sysctl. Для **systemd-сервиса** так правильно:
```bash
sudo systemctl edit myapp.service
```
Откроется редактор — вставь:
```ini
[Service]
LimitNOFILE=65535   # максимум открытых файлов (сокетов) для процесса
```
Заставь systemd перечитать и перезапусти:
```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
systemctl show myapp -p LimitNOFILE
```
Проверить у живого процесса:
```bash
cat /proc/$(pidof myapp)/limits | grep "open files"
```
> Если это **не сервис**, а твоя интерактивная сессия: `ulimit -n 65535` (только до выхода из шелла).

---

## 5) “Много мелких файлов, watch-процессы ругаются”

(например, node/npm, inotify watchers)
```bash
sudoedit /etc/sysctl.d/50-inotify.conf
```
Вставь:
```conf
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024
```
Применить: `sudo sysctl --system`.

---

# Чеклист “делай/проверяй”

## Делай (копипастой, без думать сильно)

1. Создай `99-basics.conf`, вставь блоки из пунктов 1–3, и:
```bash
    sudo sysctl --system
    ```
2. Для сервиса с ошибкой `Too many open files` — добавь `LimitNOFILE=65535` (п.4).
3. Если IDE/инструменты ноют про watchers — добавь inotify (п.5).
    

## Проверяй (прям команды)

```bash
# Текущие ключевые параметры
sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog net.ipv4.ip_local_port_range

# Сводка TCP (станы соединений)
ss -s

# Лимиты процесса (open files, процессы, memlock)
cat /proc/$(pidof myapp)/limits | egrep "open files|processes|max locked memory"

# Inotify текущие лимиты
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances
```

---

# Важно: что **НЕ** трогать, пока не поймёшь

- `vm.*` (своп/кэш) — можно ухудшить работу БД.
- Любые “магические” тюнинги из блогов типа `tcp_tw_recycle` (его больше нет) и сомнительных твиков TIME_WAIT.
- Большие буферы `rmem/wmem` без нужды — память улетит

---

## Если всё равно страшно — минималка

Сделай только это (и хватит на 90% кейсов веб/апп):
```bash
sudoedit /etc/sysctl.d/99-basics.conf
```
Вставь:

```conf
net.core.somaxconn = 1024
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.ip_local_port_range = 10240 65000
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_fin_timeout = 15
```
Применить:
```bash
sudo sysctl --system
```
Если “мало файлов” — добавь `LimitNOFILE=65535` в юнит сервиса как выше.

---

хочешь — я упакую это в твою базу `[[sysctl-ulimits]]`: сверху “минималка” (6 строк), ниже — “когда это надо” и “как проверить”, без лишних слов.