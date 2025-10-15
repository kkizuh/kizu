# Сети в Docker: bridge / host / кастомный DNS / порты (по-человечески)

## База: как контейнеры вообще «видят сеть»

- **Контейнер** — обычный процесс Linux в _своём_ сетевом **namespace**.
- Внутри у него свой `lo`, свой `eth0`, свой маршрутизатор (`ip r`), свой `/etc/resolv.conf`.  
- Docker создаёт виртуальные интерфейсы и мосты, настраивает NAT/форвардинг и **встроенный DNS**.
- Управляет всем **dockerd** через netfilter (iptables/nftables) + Linux routing.

Полезные команды:

```bash
docker network ls                         # список сетей
docker network inspect <net>              # детали сети (подсети, DNS, контейнеры)
docker exec -it <ctr> sh -lc 'ip a; ip r; cat /etc/resolv.conf'
```

---

## Важные термины (на пальцах)

- **Драйвер сети** — тип сети Docker’а: `bridge`, `host`, `none`, `macvlan`, `overlay` (в Swarm/K8s — свои).
- **bridge** — программный L2-мост на хосте. Контейнерам выдаётся IP из подсети (обычно 172.x.x.x), наружу — через NAT.
- **host** — без сетевой изоляции: контейнер использует _сетевой стек хоста_. Портов указывать не нужно и нельзя.
- **none** — полностью без сети (только `lo`), для джоб/билдов/air-gapped.
- **macvlan** — контейнер получает «собственный MAC и IP» в L2-сегменте как будто отдельная машина.
- **Publish портов (`-p`)** — проброс порта **хоста** → **контейнер** (DNAT). Нужен только на `bridge`/`none`.
- **EXPOSE** — _декларация_ в Dockerfile, не открывает порт сам по себе.
- **Встроенный DNS** — адрес `127.0.0.11` внутри container’а. Резолвит имена **других контейнеров** в той же сети.

---

## Драйверы и когда их использовать

### 1) `bridge` (дефолт)

- Контейнеры получают IP из подсети сети (по умолчанию `bridge`, можно создавать свои).
- Между собой общаются по именам сервиса (в кастомных сетях).
- Для доступа снаружи — **публикуем порты**: `-p 8080:80` (host:container).
- NAT настроит Docker автоматически.

Создать свою сеть (лучше, чем дефолтная `bridge`):

```bash
docker network create --driver bridge --subnet 172.18.0.0/16 mynet
docker run -d --name web --network mynet -p 8080:80 nginx:alpine
```

Плюсы: изоляция, встроенный DNS, гибкость. Минусы: NAT (доп. хоп), забота о портах.

### 2) `host`

- Контейнер «живёт» в сетевом стеке хоста: слушает `:80` — значит хост слушает `:80`.
- **Нельзя использовать `-p`**: он не нужен.
- Плюсы: без NAT (ниже задержки), простой доступ к локальным сервисам.
- Минусы: нет изоляции по портам, конфликт имён портов, сложнее безопасность.
Пример:
```bash
docker run --rm --network host nginx:alpine
# теперь nginx слушает прямо на 0.0.0.0:80 хоста
```

### 3) `none`

- Только `lo`. Полная изоляция.  
    Полезно для оффлайн-тасков/сборок или когда сетью управляешь сам (`nsenter` / `ip netns`).

```bash
docker run --rm --network none busybox:latest ip a
```
### 4) `macvlan`

- Контейнер выглядит как «физическая машина» в вашей L2 сети (собственный MAC).
- Не идёт через NAT; нужен прямой доступ в VLAN/подсеть.
- Подводный камень: **хост обычно не видит macvlan-контейнер** без доп. бриджа/route.
- Точно согласуй с сетевиками (VLAN, DHCP/статические IP, фильтры по MAC).

Пример (статический IP):

```bash
# parent — физ. интерфейс хоста в нужной L2-сети
docker network create -d macvlan \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  -o parent=eth0 pubnet

docker run -d --name app --network pubnet --ip 192.168.10.50 nginx:alpine
```

> Нужна связность «хост ↔ macvlan»? Гугли «macvlan bridge mode workaround» (veth + локальный bridge), или используй `ipvlan`.

---

## Порты: publish vs expose

- `EXPOSE 80` в Dockerfile — **подсказка**, реального эффекта нет.
- `-p 8080:80` (или `--publish`) — DNAT: хост:8080 → контейнер:80.
- `-P` (заглавная) — публикует все `EXPOSE` на случайные порты хоста.

Проверка:

```bash
docker run -d --name web -p 8080:80 nginx
ss -tlpn | grep 8080                 # слушает ли хост
curl -I http://127.0.0.1:8080
```

---

## DNS по-умолчанию и кастомный DNS

### Встроенный Docker DNS

- Внутри контейнера `/etc/resolv.conf` указывает на `127.0.0.11`.
- Имена `service`/`container` резолвятся в **той же пользовательской bridge-сети**.
- Напр.: `app` может достучаться до `db:5432` просто по имени `db`.

### Перезаписать DNS/hosts

```bash
# Своё DNS (например, корпоративный резолвер)
docker run --dns 10.0.0.53 --dns-search corp.local alpine getent hosts intranet

# Добавить записи hosts
docker run --add-host internal.api:10.10.10.10 alpine getent hosts internal.api
```

> В Compose используем `dns:`, `dns_search:`, `extra_hosts:`.

---

## Типовые топологии

### 1) Web + DB в одной пользовательской сети

```bash
docker network create appnet

docker run -d --name db --network appnet \
  -e POSTGRES_PASSWORD=secret postgres:16

docker run -d --name web --network appnet -p 8080:80 \
  -e DATABASE_URL=postgres://postgres:secret@db:5432/postgres \
  nginx:alpine
```

- `web` обращается к `db:5432` по имени.
- Снаружи доступ к web → `http://<host>:8080`.

### 2) Несколько сетей для разных зон

```bash
docker network create frontnet
docker network create backnet

docker run -d --name api --network frontnet --network backnet myapi
# теперь в /etc/hosts два IP (по одному в каждой сети), доступ к DB — только через backnet
```

### 3) Статический IP в пользовательской сети

```bash
docker network create --subnet 172.22.0.0/16 mynet
docker run -d --network mynet --ip 172.22.0.10 --name cache redis:7
```

---

## Эквиваленты в Docker Compose

```yaml
version: "3.9"
services:
  web:
    image: nginx:alpine
    ports: ["8080:80"]
    networks: [appnet]
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    networks: [appnet]

networks:
  appnet:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16
```

Кастом DNS/hosts:

```yaml
services:
  web:
    image: nginx:alpine
    dns: [10.0.0.53]
    extra_hosts:
      - "internal.api:10.10.10.10"
```

`host`-сеть:

```yaml
services:
  web:
    image: nginx:alpine
    network_mode: host
```

---

## IPv6 в Docker (кратко)

1. Включи IPv6 в демоне Docker (`/etc/docker/daemon.json`):

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/64"
}
```

2. Перезапусти Docker: `sudo systemctl restart docker`.
3. Создай сеть с IPv6:

```bash
docker network create --ipv6 --subnet fd00:dead:beef::/64 v6net
```

---

## Взаимодействие с firewall (UFW/iptables)

- Docker по умолчанию сам пишет правила в `iptables`/`nftables`.
- Если UFW стоит «поперёк» — включи форвардинг и разреши Docker NAT:

UFW (пример):

```bash
# /etc/default/ufw
DEFAULT_FORWARD_POLICY="ACCEPT"

# /etc/ufw/before.rules (до *filter секции)
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
COMMIT
```

И перезапусти UFW: `sudo ufw reload`.

---

## Траблшут сети (шпаргалка)

```bash
docker network ls
docker network inspect <net>
docker ps --format 'table {{.Names}}\t{{.Networks}}\t{{.Ports}}'

# проверить маршруты/интерфейсы внутри контейнера
docker exec -it <ctr> sh -lc 'ip a; ip r; ss -lnt; nc -vz db 5432 || true'

# нет сети? зайдём в net-namespace снаружи (нужен sudo)
pid=$(docker inspect -f '{{.State.Pid}}' <ctr>)
sudo nsenter --net=/proc/$pid/ns/net sh -lc 'ip a; ip r; ping -c1 8.8.8.8 || true'

# сниффинг трафика контейнера (по veth-интерфейсу)
sudo tcpdump -ni any host 172.18.0.5

# конфликт портов на хосте
ss -ltnp | grep ':8080'

# DNS внутри контейнера реально 127.0.0.11?
docker exec <ctr> cat /etc/resolv.conf
```

Частые причины:

- Занят порт на хосте → поменяй `-p 8080:80`.
- Host firewall режет форвардинг/NAT.
- Неверная сеть (контейнер не в той user-defined сети).
- DNS: сервисы в разных сетях → имя не резолвится (решение: подключить в одну сеть или прокинуть `extra_hosts`/внешний DNS).
- macvlan: хост не видит контейнер — это ожидаемо.

---

## Безопасность и best practices

- Для внешнего трафика — заводи **user-defined bridge** (не дефолтный `bridge`), он даёт сервис-DNS и изоляцию.
- Публикуй **только нужные порты**. Часто достаточно `expose` + общая сеть внутри.
- Нужен низкий latency и нет конфликтов портов? `network_mode: host` (осознанно).
- В продах — управляемый reverse-proxy/ingress (Traefik/NGINX) + TLS, а приложения — внутри приватных сетей без публикации портов наружу.
- Сервис-дискавери: полагайся на встроенный DNS user-defined сетей, а не на «магические IP».

---

## TL;DR

- **bridge**: дефолт. Контейнеры с собственными IP, наружу через `-p`. Внутри — DNS по именам.
- **host**: без изоляции, зато без NAT (быстрее). Порты не публикуем.
- **none**: полностью оффлайн.
- **macvlan**: как «железный» хост в L2; нужна сеть/правила, хост его не видит по умолчанию.
- DNS: встроенный `127.0.0.11` (+ `--dns`, `--add-host`).
- Тесты/диагностика: `inspect`, `ip a/r`, `ss`, `tcpdump`, `nsenter`.

## L2 = **канальный уровень** (Data Link) в модели OSI.

## По-человечески

- **L2 работает с кадрами и MAC-адресами.**  
    Коммутаторы (switch) принимают **Ethernet-кадры**, смотрят **MAC dst**, и пересылают внутри **одного широковещательного домена** (broadcast domain).
- **Без IP/маршрутов.** На L2 нет IP, только MAC + сервисные протоколы типа **ARP** (спросить “какой MAC у IP X?”).

## Что туда входит

- **MAC-адреса**, **Ethernet-кадры**.
- **Switch/Bridge**, **VLAN’ы (802.1Q)**, **STP**, **ARP/ND**, **LLDP**.
- Порты **access** (один VLAN) / **trunk** (несколько VLAN с тегами).

## Чем L2 отличается от L3

|Уровень|Что двигает|Адреса|Устройства|Зона действия|
|---|---|---|---|---|
|**L2**|Кадры|MAC|**Коммутатор**, мост|Один VLAN/бродкаст-домен|
|**L3**|Пакеты|IP|**Маршрутизатор**, L3-свитч|Между сетями/VLAN|
|**L4**|Сегменты|Порты TCP/UDP|Фаерволы, балансеры|Сессии/приложения|

## Почему я говорил “macvlan = L2”

`macvlan` даёт контейнеру **свой MAC и IP** в _той же L2-сети_, что интерфейс хоста. Контейнер выглядит как отдельный хост, видимый **свитчу** напрямую:

- Получает адрес от **DHCP** сети или статический.
- Отвечает на **ARP** как обычная машина.
- **Хост обычно не видит macvlan-контейнер напрямую**, потому что это всё L2 на одном интерфейсе — нужен обход (доп. bridge/veth) или `ipvlan`/роут.

## Когда это важно

- **Нужен “реальный” адрес в вашей LAN/VLAN** (принцы/принтеры, старые сетевые ACL, мультикаст).
- **Никакого NAT** (в отличии от bridge), всё как у живого сервера.

## Быстрые примеры

- **L2 (VLAN):** два сервера в VLAN 10 видят друг друга без маршрутизатора.
- **L3:** серверы в VLAN 10 и VLAN 20 общаются через маршрутизатор (gateway).
- **Docker macvlan:** контейнер получает IP из VLAN 10 как будто это ещё один сервер.

Если коротко: **L2 — это “коммутатор и MAC-адреса” внутри одного сегмента; L3 — “маршрутизатор и IP” между сегментами.**