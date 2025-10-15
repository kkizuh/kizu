# OpenStack CLI — базовые команды (шпаргалка)

> Перед началом: **подгрузи креды** проекта (source в свой `proj-...` файл).

```bash
source proj-<PROJECT>.rc   # автодополнение по Tab упростит жизнь
```

---

## 🔎 Поиск/инвентарь

```bash
# Порты проекта + поиск порта по fixed IP
openstack port list --project <PROJECT_ID> --fit | grep <IP>

# Инфо по порту
openstack port show <PORT_ID> --insecure --fit

# Список floating IP в проекте + поиск по адресу
openstack floating ip list --project <PROJECT_ID> --fit | grep <FIP>

# Инфо по floating IP
openstack floating ip show <FIP_ID> --fit

# Поиск ВМ по имени (во всех проектах)
openstack server list --all --name "<VM_NAME>"

# Инфо по конкретной ВМ
openstack server show <SERVER_ID>

# Все ВМ в проекте
openstack server list --project <PROJECT_ID>

# Инфо по хосту-гипервизору (ресурсы/проекты)
openstack host show <HYPERVISOR_NAME>

# Инфо по проекту
openstack project show <PROJECT_ID>

# История действий с ВМ (nova)
nova instance-action-list <SERVER_ID>

# События сервера (детально)
openstack server event list <SERVER_ID> --fit --long

# Найти порт по fixed-ip и (опц.) сети/подсети
openstack --insecure port list --fixed-ip subnet=<SUBNET_ID> --network <NETWORK_ID>
```

---

## ✍️ Изменение

```bash
# Сменить статус тома на available (когда застрял)
openstack volume set --state available <VOLUME_ID>
```

---

## 🗑 Удаление

```bash
# Удалить ВМ
openstack server delete <SERVER_ID>

# Удалить том
openstack volume delete <VOLUME_ID>

# Удалить порт
openstack port delete <PORT_ID>

# Удалить floating IP
openstack floating ip delete <FIP_ID>
```

---

## 🚛 Миграция (live-migration)

```bash
# Одна ВМ
openstack server migrate --live-migration --wait <SERVER_ID>

# Пакетно (список ID/имён в файле kubmig, последний столбец)
for vm in $(awk '{print $NF}' kubmig); do
  openstack server migrate --live-migration --wait "$vm"
done
```

---

## 🔧 Cinder: «отвязать и удалить» застрявший том

```bash
# Сбросить состояния и удалить том
cinder reset-state --state available <VOLUME_ID>
cinder reset-state --reset-migration-status <VOLUME_ID>
cinder reset-state --attach-status detached <VOLUME_ID>
cinder delete <VOLUME_ID>
```

---

## 🧭 Как найти ВМ по Floating IP (быстро)

1. Узнай объект FIP:
    

```bash
openstack floating ip show <FLOATING_IP> --fit
```

2. Из вывода возьми `project_id` и `Fixed IP Address`.
3. Посмотри проект и найди ВМ по **Fixed IP** через веб-консоль  
    _(или CLI внутри проекта):_

```bash
openstack project show <PROJECT_ID>
openstack server list --project <PROJECT_ID> --long | grep "<FIXED_IP>"
```

---

### Примечания

- `--fit` / `--insecure` используй только если так принято в твоей инсталляции.
- Для массовых операций всегда проверяй фильтры (`--project`, `--name`) — меньше шансов снести лишнее.