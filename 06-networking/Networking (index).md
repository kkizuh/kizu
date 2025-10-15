# Networking (index)

Мини-оглавление: короткие страницы-шпаргалки и практические лабы.

## База сети

- [[net-osi|OSI модель на пальцах (L2–L7)]]
- [[net-tcp-udp|TCP vs UDP: 3-way handshake, порты, окна, MSS/MTU, ECN]]
- [[net-dns|DNS: A/AAAA, CNAME, NS, SOA, TTL, кэш, dig/drill]]
- [[net-mail|SMTP/POP3/IMAP: где TLS, STARTTLS, SPF/DKIM/DMARC]]
- [[net-ip-routing|IP, подсети, маршрутизация: таблицы, метрики, policy-routing]]

## Инструменты

- [[tcpdump-recipes|tcpdump: шаблоны на каждый день]]
- [[nmap-basics|nmap: host discovery, service/version, скан портов]]
- [[ss-iproute2|ss/ip/route: кто слушает, куда ходит, как роутится]]

## Балансировка / прокси / VIP

- [[lb-l4-vs-l7|L4 vs L7: чем отличаются и когда что]]
- [[haproxy-quick|HAProxy: базовая конфа, healthchecks, sticky, SSL-offload]]
- [[keepalived-vip|Keepalived (VRRP): виртуальный IP и флоатинг между нодами]]
- [[nginx-revproxy|Nginx как reverse-proxy: 80→8080, proxy_pass, X-Forwarded-*]]

## TLS / mTLS / серты

- [[tls-notes|TLS-памятка: ключ/сертификат/цепочка, SAN, OCSP, HSTS]]
- [[cert-manual|Как выпустить: self-signed root → intermediate → server]]
- [[mtls-mini|mTLS кратко: client cert, trust store, где включать в HAProxy/Nginx]]

## NAT / фаервол / ACL

- [[nat-snat-dnat|SNAT/DNAT в iptables/nftables + примеры]]
- [[fw-basics|Firewall: nftables/ufw — deny-all, allow-list, логирование]]
- [[acl-request|ACL: черновик политики доступа (таблица источник→назначение, владелец, причина)]]

## Практика (мини-лабы)

- [[lab-net-01|Сеть ВМ: адресация, маршрутизация между ВМ]]
- [[lab-fw-ssh|Закрыть 22 извне и зайти по SSH (jump/bastion, порт-нок)]]    
- [[lab-nginx-8080|Nginx слушает 8080, трафик с 80 → на Nginx (почему так работает)]]
- [[lab-nat|Настроить SNAT/DNAT для публикации сервиса]]    
- [[lab-haproxy-chain|Поднять HAProxy и проксировать сквозь него на тот Nginx]]
- [[lab-dns-names|Прикрутить DNS-имена: записи A/AAAA/CNAME, TTL]]    
- [[lab-tls|Выдать серты и собрать полную цепочку (root/intermediate/server)]]

## Диагностика TCP (быстро)

- [[tcp-troubleshoot|SYN/SYN-ACK, retransmits, RST/FIN, MSS/MTU-mismatch, PMTU, ECN/CWR]]
- [[latency-loss|Задержка vs потери: где смотреть и чем мерить (ping/mtr/tcpdump)]]

## Гайды «для самых маленьких»

- [[net-zero-to-one|Сети с нуля: IP, маски, VLAN, ARP, DHCP, DNS]]
    
- [[ccna-notes|CCNA-конспект по сути: адресация, маршрутизация, L2/L3, STP, OSPF, ACL]]
    

---

### Что делать руками (чек-лист на вечер)

1. Схема из 2–3 ВМ: настроить адресацию и маршрутизацию.
2. Закрыть всё фаерволом, открыть только нужное; завести ACL-таблицу.
3. Поднять Nginx на 8080 и публиковать его с 80 через reverse-proxy.
4. Сделать SNAT/DNAT для внешнего доступа.
5. Поставить HAProxy + Keepalived → VIP → backend’ы (Nginx).
6. Выдать TLS-цепочку (self-signed root) и включить HTTPS.
7. Добавить DNS-имена и проверить резолв/кэш.
8. Пройтись `tcpdump`/`nmap` по каждому шагу и сохранить артефакты.

> Все страницы — максимально короткие, с командами и мини-кейсами. Когда будешь заполнять — добавляй свои примеры вывода и «типовые поломки + как чинил».