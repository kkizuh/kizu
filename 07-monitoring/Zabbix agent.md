##### Install Zabbix agent https://www.zabbix.com/download
```bash
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu22.04_all.deb
dpkg -i zabbix-release_latest_7.4+ubuntu22.04_all.deb   
apt update
apt install zabbix-agent2  
apt install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql
   
systemctl restart zabbix-agent2
systemctl enable zabbix-agent2
```

**Сбор данных.**

Чтобы начать забирать информацию с агента, нужно выбрать один из параметров:
- Server
- ServerActive

**Server** - адреса, куда агент будет отправлять данные в пассивном режиме. Он определяет назначенное место приема по транспортному адресу агента.

**ServerActive** - адреса на который агент будет отправлять активные запросы в активном режиме. Он позволяет агенту установить соединение с сервером Zabbix для передачи данных.

**Server и ServerActive**

Использование обоих параметров (Server и ServerActive) дает возможность настроить агент для работы в пассивном и активном режимах одновременно. Это увеличивает гибкость и обеспечивает надежное соединение между агентом и сервером Zabbix.

Поэтому мы укажем оба параметра

```bash
vim /etc/zabbix/zabbix_agent2.conf

Server=ip_zabbix_server
ServerActive=ip_zabbix_server

systemctl restart zabbix-agent2
```

Также по умолчанию Zabbix Agent доступен по порту 10050, чтобы переопределить это, в zabbix_agent2.conf нужно поменять параметр ListenPort (рисунок 92).

![](https://ucarecdn.com/9a167ef4-7472-419b-afab-cf4ec1dcd031/)

Рисунок 92. Смена параметра ListenPort.