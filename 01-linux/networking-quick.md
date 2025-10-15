## ping — «жив ли хост»

```bash
ping -c 5 ya.ru           # 5 пакетов
ping -4 ya.ru             # IPv4 (или -6)
ping -n ya.ru             # не резолвить имена
ping -i 0.5 ya.ru         # интервал 0.5с (>=0.2 обычно требует sudo)
ping -s 1400 ya.ru        # размер payload
ping -W 2 ya.ru           # timeout ответа, сек
```

- `rtt min/avg/max/mdev` — задержки.
- если молчит — может блокироваться ICMP фаерволом.


## traceroute / tracert / tracepath — путь до хоста

```bash
traceroute ya.ru          # UDP-пробы (Linux)
sudo traceroute -I ya.ru  # ICMP-пробы (часто проходит через firewall)
traceroute -T ya.ru       # TCP SYN (к полезному порту)
traceroute -n ya.ru       # без DNS
traceroute -m 20 ya.ru    # max hops
traceroute -w 1 ya.ru     # timeout узла, сек
```

- **Windows:** `tracert ya.ru` (ICMP по умолчанию).
- **tracepath** (без root, показывает MTU):

```bash
tracepath ya.ru
```

## mtr — ping+traceroute в реальном времени

```bash
mtr ya.ru                 # интерактивно (обновляется)
mtr -n ya.ru              # IP без DNS
mtr -rwc 100 ya.ru        # report-режим, 100 пакетов и выйти
mtr -i 0.5 ya.ru          # интервал 0.5с
mtr -4 / -6 ya.ru         # стек
```

- Смотри `Loss%` (потери), `Avg/Best/Wrst`, `StDev`.
- Потери **на промежуточном хопе** при нормальном конце — часто просто rate-limit на ICMP.

## ss — активные соединения/порты (замена netstat)

```bash
ss -tulpen                # TCP/UDP listen + процессы + порты
ss -s                     # сводка (ESTAB/SYN/тем проч)
ss -tan state established dst 1.2.3.4   # фильтр по dst
ss -ltn 'sport = :443'    # кто слушает 443/tcp
ss -tp | grep nginx       # TCP + процессы, отфильтровать
```

## tcpdump — сниффер (быстрое разминирование)

**База**

```bash
sudo tcpdump -i any                   # смотреть всё
sudo tcpdump -i eth0 -nn              # без DNS/сервис-имен
sudo tcpdump -i eth0 port 53          # только DNS
sudo tcpdump -i eth0 host 8.8.8.8     # трафик к/от хоста
sudo tcpdump -i eth0 net 10.0.0.0/24  # подсеть
```

**Полезные флаги**

```bash
sudo tcpdump -i eth0 -s 0 -vvv -X 'tcp port 443'    # полный срез, болтливый, hex+ASCII
sudo tcpdump -i any icmp                             # ICMP (ping/traceroute)
sudo tcpdump -i any 'tcp[tcpflags] & (syn|ack) != 0' # TCP флаги
sudo tcpdump -G 60 -W 10 -w cap-%Y%m%d-%H%M%S.pcap  # ротация: файл/мин, 10 файлов
sudo tcpdump -r file.pcap                           # прочитать pcap
```

**Фильтры BPF (логика):** `and / or / not`  
Примеры:

- только SYN без ACK (новые коннекты):  
    `tcpdump 'tcp[13] & 2 != 0 and tcp[13] & 16 = 0'`
- только исходящие к порту 25:  
    `tcpdump 'tcp dst port 25 and src net 10.0.0.0/8'`
    

---

## Быстрые паттерны диагностики

- **“Пингуется, но веб не работает”**
    
    1. `ss -ltn | grep :80\|:443` — слушает ли?
    2. `sudo tcpdump -i eth0 'tcp port 443'` — доходят ли пакеты?
    3. `journalctl -u nginx -p err..crit -b` — ошибки сервиса.
        
- **“Потери на маршруте”**
    
    1. `mtr -rwzc 100 ya.ru` — отчёт по потере/джиттеру.
    2. Если последний хост ок, a промежуточные теряют — вероятно, ICMP rate-limit (не паниковать).
        
- **“Иногда рвётся SSH”**
    
    1. `mtr -c 200 -i 0.2 <ip>` — ищем интермиттент.
    2. `sudo tcpdump -i eth0 'tcp port 22 and (tcp[tcpflags] & 4 != 0)'` — RST.
    3. Проверь MTU: `tracepath <ip>` (флаг `pmtu`).
        

---

## Памятка по стеку/правам

- ICMP-traceroute (`traceroute -I`) часто требует `sudo`.
- `tracepath` не требует root, хорош для PMTU.
- `tcpdump` почти всегда `sudo`.
- `ss` быстрый и безопасный, не блокирует.

---

## Одной строкой для заметок

```bash
# ping
ping -c 5 -W 2 -n ya.ru
# traceroute (ICMP)
sudo traceroute -I -n ya.ru
# tracepath (PMTU)
tracepath -n ya.ru
# mtr отчёт на 100 пакетов
mtr -n -rwzc 100 ya.ru
# кто слушает
ss -tulpen
# снифф DNS
sudo tcpdump -i any -nn -s 0 -vvv 'port 53'
```

да, докинем самое полезное — коротко и по делу. добавь эти блоки к твоей `networking-quick` шпоре.

## DNS (dig/drill/nslookup/resolvectl)

```bash
dig +short ya.ru A               # быстрый ответ
dig +trace ya.ru                 # полный путь до NS
dig @1.1.1.1 ya.ru ANY          # спросить у конкретного DNS
drill ya.ru                      # аналог dig
nslookup ya.ru 8.8.8.8           # старичок, но работает
resolvectl query ya.ru           # через systemd-resolved
```

## HTTP/TLS проверка (curl/openssl)

```bash
curl -I https://site.com                 # только заголовки
curl -vk https://site.com                # болтливо + игнор TLS ошибок
curl -sS https://site.com/health || echo "down"   # быстрая проверка
curl --resolve site.com:443:1.2.3.4 https://site.com  # DNS override
openssl s_client -connect site.com:443 -servername site.com </dev/null | openssl x509 -noout -dates -issuer -subject
# Быстрый telnet-заменитель:
nc -vz site.com 443
```

## Пропускная способность / потери (iperf3)

```bash
iperf3 -s                               # сервер
iperf3 -c ip -t 10 -P 4                 # клиент: 10с, 4 потока
iperf3 -u -b 50M -c ip                  # UDP тест (джиттер/потери)
```

## Маршруты/таблицы/ARP/MTU

```bash
ip r                                   # маршруты (route -n устар)
ip r get 8.8.8.8                       # какой маршрут/интерфейс
ip neigh                               # ARP/ND-кэш
tracepath -n host                      # путь + PMTU
ip link set dev eth0 mtu 1400          # тест MTU
```

## Интерфейсы/линк/драйвер

```bash
ip -br a                               # кратко адреса
ethtool eth0                           # линк, скорость
ethtool -S eth0 | head                 # счётчики
```

## Сокеты/порты (кроме `ss`)

```bash
lsof -i -P -n | grep LISTEN            # кто слушает порты
```

## Снифф HTTP/DNS быстро (tcpdump)

```bash
sudo tcpdump -i any -nn -s 0 -A 'tcp port 80'
sudo tcpdump -i any -nn 'port 53'      # DNS
```

## nmap (быстрый скан)

```bash
nmap -Pn -p 22,80,443 host
nmap -sS -T4 host/24                   # SYN скан, быстрый
```

## Firewall (шорт-ссылки)

```bash
# UFW
sudo ufw status verbose
# firewalld (RHEL)
sudo firewall-cmd --list-ports
sudo firewall-cmd --add-port=443/tcp --permanent && sudo firewall-cmd --reload
```

## Коннекты/NAT/conntrack

```bash
ss -tan state established              # активные TCP
sudo conntrack -L | wc -l              # сколько состояний NAT/CT
sudo conntrack -L | grep 1.2.3.4       # фильтр по IP
```

## Топ сетевой активности

```bash
sudo iftop -i eth0                     # трафик в реальном времени
sudo nload eth0                        # вход/выход графиком
```

## Быстрые one-liners диагностики

```bash
# DNS у разных резолверов:
for d in 1.1.1.1 8.8.8.8 9.9.9.9; do echo @$d; dig +short ya.ru @$d; done

# Проверить, что порт слушает и доступен локально/снаружи:
ss -ltn 'sport = :443' && nc -vz 127.0.0.1 443

# Проверка HTTPS цепочки/даты сертификата:
openssl s_client -connect site.com:443 -servername site.com </dev/null 2>/dev/null | openssl x509 -noout -dates

# Трасса TCP к конкретному сервису (обходит ICMP-блок):
sudo traceroute -T -p 443 site.com
```

## Мини-flow при «не грузится сайт»

1. DNS: `dig +short site.com`
2. Порт: `ss -ltn | grep :443`
3. Reachability: `nc -vz site.com 443`
4. Сертификат: `openssl s_client -connect site.com:443 -servername site.com | openssl x509 -noout -dates`
5. Путь/MTU: `tracepath site.com`
6. Фаервол: `sudo ufw status` / `sudo firewall-cmd --list-all`
7. Логи сервиса: `journalctl -u nginx -b -p err..crit`

