# namespaces + cgroups — что это, зачем и как применить

## Идея за 30 секунд

- **namespaces** — «шоры» для процесса: он видит **свой** список процессов, **свою** сеть, **свои** монтирования, hostname и т.д. → изоляция **видимости**.
- **cgroups** — «ограничители и счётчики»: лимитируют **CPU/RAM/PIDs/I/O** и ведут учёт. → изоляция **ресурсов**.
- Контейнер = обычный процесс Linux **в своих namespaces** + **с лимитами cgroups**.

---

## Зачем это тебе

- **Изоляция**: сервисы не лезут друг к другу (процессы, сеть, FS).
- **Стабильность**: один контейнер не «съедает» весь сервер.
- **Безопасность**: меньше поверхность атаки (особенно с non-root + LSM).
- **Плотность**: десятки сервисов на одном хосте без VM-оверхеда.

---

## Какие бывают namespaces (коротко)

|NS|Что изолирует|На практике|
|---|---|---|
|`pid`|дерево процессов|внутри свой PID 1; `ps` не видит хост|
|`net`|интерфейсы/маршруты/порт|свой `eth0`, свои iptables|
|`mnt`|точки монтирования|свой `/`, свои bind-mounts|
|`uts`|`hostname`/`domainname`|свой hostname|
|`ipc`|shared memory/sem/msgq|отдельные shm сегменты|
|`user`|UID/GID mapping|root внутри ≠ root снаружи|
|`cgroup`|видимость иерархий cgroup|видит только свои контроллеры|

---

## Быстрые проверки (реально полезно)

```bash
# PID главного процесса контейнера
docker inspect -f '{{.State.Pid}}' myapp

# Посмотреть его namespaces (нужен sudo)
sudo ls -l /proc/$(docker inspect -f '{{.State.Pid}}' myapp)/ns

# Зайти в сетевой namespace и посмотреть интерфейсы/маршруты
sudo nsenter --net=/proc/$(docker inspect -f '{{.State.Pid}}' myapp)/ns/net ip a
sudo nsenter --net=/proc/$(docker inspect -f '{{.State.Pid}}' myapp)/ns/net ip r

# Посмотреть к каким cgroups привязан процесс
cat /proc/$(docker inspect -f '{{.State.Pid}}' myapp)/cgroup

# Обзор по ресурсам (systemd-хост)
systemd-cgtop
systemd-cgls
```

---

## cgroups: как ограничить ресурсы

### Docker

```bash
docker run -d --name app \
  --cpus=1.5 \          # максимум 1.5 vCPU
  --memory=512m \       # лимит RAM
  --pids-limit=256 \    # ограничение количества процессов/потоков
  --restart=always \
  myimg:tag
```

### systemd (для обычных сервисов)

```ini
# /etc/systemd/system/my.service
[Service]
ExecStart=/usr/local/bin/myapp
MemoryMax=512M
CPUQuota=150%
TasksMax=256
```

```
sudo systemctl daemon-reload && sudo systemctl restart my.service
```

---

## Не root внутри контейнера — must-have

```dockerfile
# создаём юзера с фиксированным uid (не root)
RUN adduser -D -u 10001 appuser
USER 10001
```

Плюсы:
- меньше прав → меньше урона при компромете;
- в связке с **user-ns** хоста «root внутри» мапится в нелегитимный UID снаружи

Порт <1024 без root:

```bash
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/myapp
```

---

## Траблшут (типовые кейсы)

- **Контейнер «не видит» сеть хоста** → это норма (`net` ns). Для диагностики: `nsenter --net … ip a`, `tcpdump -i any`.
- **OOM/убивает память** → проверь `--memory`, логи ядра: `dmesg | grep -i oom`, `journalctl -k`.
- **Процессов слишком много** → `--pids-limit`, `TasksMax` в systemd.
- **Не видишь `/proc/<pid>/ns`** → нужен `sudo` (доступ к чужому `/proc`).

---

## Мини-шпаргалка (на каждый день)

```bash
# Список ns процесса
sudo ls -l /proc/<PID>/ns

# Войти в net-ns процесса и выполнить команду
sudo nsenter --net=/proc/<PID>/ns/net sh -c 'ip a; ip r'

# Проверка лимитов cgroup для сервиса systemd
systemctl show my.service | egrep 'MemoryMax|CPUQuota|TasksMax'

# Ограниченный запуск Docker
docker run --cpus=1 --memory=256m --pids-limit=128 myimg
```

---

## TL;DR

- **namespaces** — изолируют **что процесс видит**.
- **cgroups** — ограничивают **что процесс может съесть**.
- Контейнер = процесс Linux с обоими механизмами.
- Всегда **non-root + лимиты**. Это база безопасности и стабильности.