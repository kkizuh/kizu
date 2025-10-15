# Docker: архитектура по-простому (dockerd → containerd → runc → overlayfs)

## Картина мира

![[Pasted image 20251014191921.png]]
![[Pasted image 20251014191946.png]]
![[Pasted image 20251014192129.png]]
![[Pasted image 20251014192141.png]]
- **docker CLI** — твои команды (`docker run/build/ps/...`).
- **dockerd** — демон «мозг»: API, сборка, сети/тома, кэш слоёв, pull/push.
- **containerd** — runtime-надсмотрщик: lifecycle контейнеров и снапшотов (FS-слои).
- **containerd-shim** — маленький процесс на контейнер: держит stdio, reparent, не убивает контейнер при рестарте `dockerd`.
- **runc** — низкоуровневый **OCI runtime**: создаёт контейнер через **namespaces/cgroups**.
- **overlayfs** (или др. storage driver) — как слои образов становятся ФС контейнера.

> OCI = стандарт. Образы (OCI Image), runtime (OCI Runtime). Благодаря этому Docker/Podman/CRI-O совместимы на уровне образов/рантайма.

---

## Что происходит при `docker run nginx`

1. **CLI** шлёт запрос в **dockerd** (по сокету `/var/run/docker.sock`).

2. **dockerd** проверяет образ, при нужде тянет из **registry** (Docker Hub/priv).    
3. Передаёт задачу в **containerd** (gRPC).
4. **containerd** создаёт **snapshot** ФС из слоёв (через драйвер, обычно overlay2).
5. Стартует **containerd-shim**, который вызывает **runc**.
6. **runc** делает `clone()` с нужными **namespaces** + вешает **cgroups** лимиты, монтирует слои → запускает `nginx` как **PID 1** в контейнере.
7. Порты/сети/объёмы уже подготовлены `dockerd` (bridge, iptables/nft, bind/volume).

---

## Где что живёт (Linux)

- Сокеты:
    - Docker API: `/var/run/docker.sock`
    - containerd: `/run/containerd/containerd.sock`
- Данные Docker: `/var/lib/docker/`
    - `overlay2/` — слои образов и «верхний» слой контейнеров
    - `containers/` — метаданные/логи контейнеров
    - `volumes/` — Docker-тома
- Данные containerd (часто): `/var/lib/containerd/`

---

## overlayfs/overlay2 — 60 секунд

Слои образа **read-only** складываются пирамидой. Контейнер получает **rw-слой** (upperdir), а все нижние слои «видны» через **merge**:

```
lowerdir: слой-3:слой-2:слой-1   (ro)
upperdir: контейнерный слой      (rw)
workdir:  служебный
mount -> merged/
```

- Копирование-при-записи: меняешь файл → он попадает в upperdir.
- Экономит место и ускоряет доставку.

Проверить драйвер:

```bash
docker info | grep -i 'Storage Driver'
```

---

## Полезные команды админа

### Версии/здоровье

```bash
docker version
docker info
containerd --version
runc --version
```

### Дебаг демонов

```bash
# логи Docker
sudo journalctl -u docker -n 200 -f
# логи containerd
sudo journalctl -u containerd -n 200 -f
```

### Взаимодействие с containerd напрямую (на всякий)

```bash
sudo ctr -n moby containers list
sudo ctr -n moby tasks list
```

> namespace `moby` — дефолт у Docker.

### Что использует контейнер

```bash
docker inspect <name> --format '{{.State.Pid}} {{.GraphDriver.Name}}'
sudo ls -l /proc/$(docker inspect -f '{{.State.Pid}}' <name>)/ns
cat /proc/$(docker inspect -f '{{.State.Pid}}' <name>)/cgroup
```

### Образы/слои

```bash
docker image ls
docker image history <image>:tag
```

---

## Сети (очень коротко)

- **bridge** (по умолчанию): контейнеры на одном хосте, NAT наружу.
- **host**: без сетевой изоляции.
- **macvlan**: контейнер с «реальным» MAC/IP в L2 сети.    
- **overlay**: межхостовая (в Swarm; в Kubernetes — свои плагины CNI).

Проверить:

```bash
docker network ls
docker network inspect bridge
```

---

## Registry и образы

- **pull/push**: `docker pull nginx:alpine`, `docker push myreg:5000/app:1.2.3`
- Свой регистр: `registry:2` (как у тебя `TrexDockerRegistry`).
- Закрепляй по **digest** для повторяемости:

```bash
docker build -t myapp:1.0 .
docker inspect --format='{{index .RepoDigests 0}}' myapp:1.0
# используешь myreg/app@sha256:...
```

---

## Безопасность/практики

- **non-root** в контейнере (`USER 10001`) + **capabilities** по минимуму.
- Не пихай секреты в `ENV/ARG`; используй **docker secrets** (в Swarm) или внешние хранилища.
- Лимиты cgroups всегда: `--memory`, `--cpus`, `--pids-limit`.
- rootless Docker — когда нужно без root на хосте.

---

## Частые поломки — как смотреть

- «Не тянется образ» → сеть/прокси/DNS: `docker info`, `docker login`, `curl` к реестру.    
- «containerd errors» → смотри `journalctl -u containerd`, `ctr -n moby tasks list`.
- «диск забит» → `docker system df`, чистка: `docker system prune -a` (осторожно!).
- «порт занят» → `ss -tlpn | grep :80`, смени publish `-p`.
- «overlay2 сломался» (редко) → проверяй файловую систему, inodes, dmesg.

---

## TL;DR

- **dockerd** — API и менеджмент; **containerd** — lifecycle контейнеров; **runc** — создаёт процессы в ns/cgroups; **overlayfs** — слои ФС.    
- `docker run` = цепочка CLI → dockerd → containerd → shim → runc → ядро.
- Смотри логи юнитов, проверяй storage driver, держи лимиты и non-root.