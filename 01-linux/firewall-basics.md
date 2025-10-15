## UFW (Ubuntu) — простой фронтенд над netfilter

```bash
# === БАЗА / ЖИЗНЕННЫЙ ЦИКЛ ===
sudo ufw status verbose      # текущие правила + политики
sudo ufw reload              # перечитать правила (без сброса соединений)
sudo ufw reset               # полный сброс к дефолту (внимание!)

# === БЕЗОПАСНЫЙ СТАРТ С СЕРВЕРА ===
sudo ufw allow OpenSSH       # обязательно: не отрежь себе SSH-доступ
sudo ufw default deny incoming   # входящие по умолчанию закрыты
sudo ufw default allow outgoing  # исходящие по умолчанию разрешены
sudo ufw enable              # включить UFW (подтвердить Y)

# === ТИПОВЫЕ ПРАВИЛА ===
sudo ufw allow 22/tcp        # явно открыть SSH по порту
sudo ufw limit 22/tcp        # rate-limit SSH (ограничение брутфорса)
sudo ufw allow 80,443/tcp    # открыть HTTP/HTTPS одним вызовом
sudo ufw allow 3000:3010/tcp # диапазон портов для dev/прокси

# === ФИЛЬТРЫ ПО ИСТОЧНИКУ/ИНТЕРФЕЙСУ ===
sudo ufw allow from 10.0.0.0/24 to any port 5432 proto tcp
# только подсеть /24 может подключаться к PostgreSQL

sudo ufw deny from 203.0.113.0/24
# жёстко рубим шумную/подозрительную сеть

sudo ufw allow in  on eth0 to any port 80
sudo ufw allow out on eth0 to any port 53 proto udp
# можно задавать направление (in/out) и интерфейс

# === УДАЛЕНИЕ/АУДИТ ===
sudo ufw status numbered     # показать правила с номерами
sudo ufw delete <N>          # удалить правило по номеру
sudo ufw delete allow 80/tcp # удалить по шаблону правила
```

> Примечание: UFW и ручной `iptables` одновременно — плохая идея. Держи единый способ управления.

---

## firewalld (RHEL/Fedora) — зоны, сервисы, динамичность

```bash
# === ОБЗОР СОСТОЯНИЯ ===
sudo firewall-cmd --state                # running / not running
sudo firewall-cmd --get-active-zones     # какие зоны активны и на каких интерфейсах
sudo firewall-cmd --list-all             # подробно по дефолтной зоне (обычно public)
sudo firewall-cmd --list-all --zone=public

# === ОТКРЫТЬ ПОРТЫ/СЕРВИСЫ ===
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload               # применяем постоянные изменения

# === ПРИВЯЗКА ИНТЕРФЕЙСА К ЗОНЕ ===
sudo firewall-cmd --permanent --zone=public --add-interface=eth0
sudo firewall-cmd --reload

# === NAT / МАСКАРАД / ПОРТ-ФОРВАРД ===
sudo firewall-cmd --permanent --add-masquerade
# включаем маскарадинг (исходящий NAT), нужен для LAN->WAN

sudo firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toaddr=10.0.0.10:toport=80
# пробрасываем 80/tcp на внутренний хост 10.0.0.10:80

sudo firewall-cmd --reload

# === RICH-RULES (ТОЧЕЧНЫЕ ПРАВИЛА) ===
sudo firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="203.0.113.5" port port="22" protocol="tcp" accept'
sudo firewall-cmd --reload
# удобно, когда надо условие по IP + порту/протоколу без кастомных зон
```

> Под капотом firewalld работает поверх nftables/iptables. Не мешай ему ручными правками без необходимости.

---

## nftables (современно, единый стек IPv4/IPv6)

```bash
# === ЧИСТЫЙ СТАРТ И БАЗОВЫЙ ПОЛИТИКИ ===
sudo nft flush ruleset

sudo nft add table inet filter
sudo nft add chain inet filter input   '{ type filter hook input   priority 0; policy drop; }'
sudo nft add chain inet filter forward '{ type filter hook forward priority 0; policy drop; }'
sudo nft add chain inet filter output  '{ type filter hook output  priority 0; policy accept; }'
# policy drop на input/forward: по умолчанию всё закрыто

# === РАЗРЕШАЕМ БАЗОВОЕ: LOOPBACK, ESTABLISHED, ICMP, СЕРВИСЫ ===
sudo nft add rule inet filter input iif lo accept
# loopback всегда разрешаем

sudo nft add rule inet filter input ct state established,related accept
# пропускаем ответы на исходящие коннекты

sudo nft add rule inet filter input ip protocol icmp accept
sudo nft add rule inet filter input ip6 nexthdr icmpv6 accept
# ICMP/ICMPv6 нужны для PMTU, пинга, корректной работы сети

sudo nft add rule inet filter input tcp dport {22,80,443} accept
# открываем SSH/HTTP/HTTPS на самом хосте

# === NAT: МАСКАРАДИНГ (WAN=eth0) ===
sudo nft add table ip nat
sudo nft add chain ip nat postrouting '{ type nat hook postrouting priority 100; }'
sudo nft add rule  ip nat postrouting oif "eth0" masquerade
# чтобы машины из LAN уходили в интернет через этот хост

# === DNAT: ПРОБРОС ВНУТРЬ (пример 80 -> 10.0.0.10:80) ===
sudo nft add chain ip nat prerouting '{ type nat hook prerouting priority -100; }'
sudo nft add rule  ip nat prerouting iif "eth0" tcp dport 80 dnat to 10.0.0.10:80
# перенаправляем внешние входящие 80/tcp на внутренний бэкенд

# === РАЗРЕШАЕМ ФОРВАРД ДЛЯ ПРОБРОСА ===
sudo nft add rule inet filter forward ct state established,related accept
sudo nft add rule inet filter forward iif "eth0" oif "lan0" tcp dport 80 ct state new,established accept
sudo nft add rule inet filter forward iif "lan0" oif "eth0" ct state established,related accept

# === ПЕРСИСТЕНТНОСТЬ ===
sudo sh -c 'nft list ruleset > /etc/nftables.conf'
sudo systemctl enable --now nftables
```

> Почему nftables: одна грамматика для IPv4/IPv6, удобные множества (`{22,80,443}`), читаемый экспорт `nft list ruleset`.

---

## iptables (легаси, но часто в проде)

```bash
# === СБРОС И ПОЛИТИКИ ===
sudo iptables -F                  # очистить цепочки фильтра
sudo iptables -t nat -F           # очистить NAT
sudo iptables -P INPUT DROP       # всё входящее закрыто
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# === БАЗОВОЕ РАЗРЕШЕНИЕ ===
sudo iptables -A INPUT -i lo -j ACCEPT
# loopback

sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# ответы на исходящие

sudo iptables -A INPUT -p icmp -j ACCEPT
# ICMP (ping/PMTU). Для IPv6 аналог в ip6tables.

sudo iptables -A INPUT -p tcp -m multiport --dports 22,80,443 -j ACCEPT
# SSH/HTTP/HTTPS на самом хосте

# === NAT: МАСКАРАДИНГ (WAN=eth0) ===
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
# исходящий NAT для клиентов за этим хостом

# === DNAT: ПРОБРОС ПОРТА (80 -> 10.0.0.10:80) ===
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 \
  -j DNAT --to-destination 10.0.0.10:80
# перенаправляем трафик с внешки на бэкенд

# Разрешим форвард туда-обратно:
sudo iptables -A FORWARD -i eth0 -o lan0 -p tcp --dport 80 \
  -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i lan0 -o eth0 \
  -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# === ПЕРСИСТЕНТНОСТЬ (Debian/Ubuntu) ===
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
# иначе правила пропадут после перезагрузки
```

> Совет: если у тебя ядро и iptables уже `nf_tables`-бэкенд, лучше мигрировать на nftables для единообразия.

---

## Диагностика / «почему не работает»

```bash
# 1) Включён ли форвардинг? (нужен для NAT/DNAT)
sysctl net.ipv4.ip_forward                 # должен быть = 1
sudo sysctl -w net.ipv4.ip_forward=1
# навсегда: /etc/sysctl.d/99-forward.conf -> net.ipv4.ip_forward=1

# 2) Слушает ли нужный порт локально?
ss -tulpen | grep -E ':22|:80|:443'        # LISTEN сокеты + PID/приложение

# 3) Доходит ли пакет до интерфейса? (снифф снаружи/внутри)
sudo tcpdump -i eth0 -nn 'tcp port 80'     # внешний интерфейс (WAN)
sudo tcpdump -i lan0 -nn 'tcp port 80'     # внутренний (к бэкенду)

# 4) Состояние фаерволов
sudo ufw status verbose
sudo firewall-cmd --list-all
sudo nft list ruleset
sudo iptables -S
```

---

## Быстрые «шаблоны запуска»

**UFW (сервер Ubuntu):**

```bash
sudo ufw allow OpenSSH
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80,443/tcp
sudo ufw enable
sudo ufw status verbose
```

**firewalld (RHEL/Fedora):**

```bash
sudo firewall-cmd --permanent --add-service=ssh --add-service=http --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

**nftables (универсальная база):**

```bash
sudo nft flush ruleset
sudo nft add table inet filter
sudo nft add chain inet filter input   '{ type filter hook input   priority 0; policy drop; }'
sudo nft add rule  inet filter input iif lo accept
sudo nft add rule  inet filter input ct state established,related accept
sudo nft add rule  inet filter input ip protocol icmp accept
sudo nft add rule  inet filter input ip6 nexthdr icmpv6 accept
sudo nft add rule  inet filter input tcp dport {22,80,443} accept
sudo sh -c 'nft list ruleset > /etc/nftables.conf'
sudo systemctl enable --now nftables
```

---

## Анти-грабли (частые ошибки)

- Не смешивай **UFW/firewalld** с ручным **iptables/nft** на одном хосте — правила конфликтуют.
- Забыли `net.ipv4.ip_forward=1` → NAT/DNAT/форвард не поедут.
- Потери ICMP на промежуточных хопах при ок последнем — это **rate-limit**, не обязательно проблема маршрута.
- В IPv6 не забывай про **ICMPv6**, без него ломается PMTU/ND (не «закрывай всё»).
- Сначала проверь **слушает ли сервис**, потом фаервол, и только потом сеть провайдера.

