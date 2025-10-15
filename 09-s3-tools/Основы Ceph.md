# Ceph (Reef): база для методички — RADOS, RGW (S3), RBD, iSCSI, CephFS

**Источники:**

- [https://habr.com/ru/articles/313644/](https://habr.com/ru/articles/313644/)
- [http://onreader.mdl.ru/LearningCeph2ed/content/index.html#Preface](http://onreader.mdl.ru/LearningCeph2ed/content/index.html#Preface)
- [https://docs.ceph.com/en/reef/](https://docs.ceph.com/en/reef/)

---

## 1) Что такое Ceph (в двух словах)

**Ceph** — программно‑определяемое распределённое хранилище без единой точки отказа. Хранит данные как **объекты RADOS**, масштабируется горизонтально, самовосстанавливается при сбоях, расчёт размещения выполняет алгоритм **CRUSH** (без централизованного индекса).

Ключевые свойства:

- **Высокая доступность:** отказ диска/узла → автоматический ребилд/ребаланс без простоя для клиентов.
- **Горизонтальный рост:** добавляете OSD → растут ёмкость и IOPS.
- **Универсальность интерфейсов:** блок (**RBD**), файл (**CephFS**), объект (**RGW/S3**, Swift), прямой доступ (**librados**).

---

## 2) Архитектура и роли

- **MON (ceph-mon):** кворум и карты кластера (Paxos).
- **OSD (ceph-osd):** хранение объектов, репликация/EC, фоновые процессы recovery/backfill/scrub.
- **MGR (ceph-mgr):** телеметрия, панель, модули; **Orchestrator** (cephadm/k8s) для управления сервисами.
- **RGW (ceph-rgw):** шлюз S3/Swift, поддержка multisite.
- **MDS (ceph-mds):** метаданные CephFS (только для файлового доступа).

> Практика: минимально жизнеспособный кластер под прод: **3× MON**, **(много) OSD**, **1–N RGW** по нагрузке; MDS/FS — при необходимости.

---

## 3) Сети (разделять обязательно)

- **Public (client/front):** клиенты и сервисы (RGW/MDS/RBD) ↔ MON/OSD.    
- **Cluster (back):** меж‑OSD репликация/бекфилл.

Разделение снижает латентность клиентских операций и защищает их от «штормов» ребаланса. MTU/Jumbo — по результатам тестов; стабильность важнее тюнинга «на глаз».

---

## 4) Как Ceph размещает данные

1. Клиент/сервис пишет **объект RADOS** в **пул**.    
2. Объекты группируются в **PG (Placement Groups)** — единицы балансировки.
3. **CRUSH(pool_id + object_name) → pgid → набор OSD**. Никаких глобальных таблиц местоположений.

### PG/OSD (эмпирика)

- HDD: ≈ **50–100 PG/OSD**
- SATA SSD: ≈ **100–500 PG/OSD**
- NVMe: допустимо выше

Финальные цифры зависят от устройств, числа пулов и профиля нагрузки — бенчите.

---

## 5) Интерфейсы хранения

### 5.1) RGW / S3 (объектное)

- **Как устроено:** объект S3 через RGW маппится в один/несколько **объектов RADOS**; метаданные бакета/объекта — это тоже объекты RADOS, живущие в спец‑пулаx RGW.
- **Multisite:** иерархия **Realm → ZoneGroup → Zone**; асинхронная репликация между зонами; DR/геораспределение.
- **Производительность:** масштабируйте количеством **OSD** (бэкенд) и **инстансов RGW** (API). Пулы метаданных RGW чувствительны к латентности — держите их на быстрых устройствах.

### 5.2) RBD (блочное)

- **Что это:** тома блок‑устройств «как SAN» для VM/БД/сервисов.
- **Фичи:** снапшоты, клонирование (layering), exclusive‑lock, object‑map, fast‑diff, deep‑flatten и пр. (включайте осознанно).
- **Типовые команды:**
   ```bash
    # Создать пул и том
    ceph osd pool create rbd 128
    rbd pool init rbd
    rbd create --size 100G rbd/vm01
    
    # Маппинг (Linux)
    rbd map rbd/vm01
    mkfs.ext4 /dev/rbd/rbd/vm01 && mount /dev/rbd/rbd/vm01 /mnt
    
    # Снапшоты и клоны
    rbd snap create rbd/vm01@pre
    rbd clone rbd/vm01@pre rbd/vm01-clone
    ```
- **Практика:**
    - Для VM на KVM — используйте natively RBD через librbd (qemu‑rbd/virtio), это быстрее и надёжнее iSCSI/NFS.
    - Для БД — тестируйте latency; object‑map/fast‑diff помогают бэкапам.

### 5.3) iSCSI (ceph‑iscsi)

- **Зачем:** когда хост/гипервизор не умеет RBD напрямую.
- **Как работает:** **ceph‑iscsi** (gateways) + LIO (kernel target) экспортируют RBD как LUN по iSCSI; для отказоустойчивости — 2+ GW и multipath на клиентах.
- **Утилиты:** `gwcli`/`ceph-iscsi` для настройки; на клиентах — `open-iscsi` + `multipathd`.

### 5.4) CephFS (файловое)

- **Что это:** POSIX‑совместимая распределённая ФС поверх RADOS.
- **Компоненты:** MDS управляет метаданными; для производительности — 2+ MDS (active/standby, возможно active/active).
- **Экспорт:** напрямую (ceph-fuse, kernel client) или через NFS‑Ganesha.

### 5.5) librados (прямой доступ)

- **Для приложений:** минимальный оверхед, максимальный контроль; когда RGW/RBD/CephFS не подходят.

---

## 6) Пулы, репликация и EC

- **Репликация:** проще, быстрее recovery; стандарт для «горячих» данных    
- **Erasure Coding (EC):** экономия места, выше требования к сети/CPU, лучше для «холодных» данных/архива.
- **Разнос по CRUSH‑правилам:** уровень отказоустойчивости (узел/стойка/ДЦ) задаётся crush‑map/правилами.    

---

## 7) Надёжность и фоновые процессы

- **Recovery/Backfill:** восстановление копий/перераскладка при добавлении/убытии OSD.    
- **Scrubbing:** проверка целостности (daily/weekly); deep‑scrub по расписанию, троттлите чтобы не убить латентность.    
- **Тюнинг при авариях:** приоритет recovery vs client I/O (osd_max_backfills, recovery_sleep и др.).

---

## 8) Развёртывание (Reef) — кратко через cephadm

```bash
# Bootstrap (на первом админ‑узле)
cephadm bootstrap --mon-ip <MON_IP> --initial-dashboard-user admin --initial-dashboard-password <PW>

# Добавить узлы и роли
ceph orch host add <host1> <ip1>
ceph orch host add <host2> <ip2>
ceph orch host add <host3> <ip3>

# Развёрнуть OSD (автопоиск дисков или явный список)
ceph orch apply osd --all-available-devices
# или
ceph orch daemon add osd <host>:<device>

# Развёрнуть RGW (S3)
ceph orch apply rgw <realm.zone> --placement="count=2" --port 80

# Развёрнуть iSCSI‑шлюзы (пример)
ceph orch apply iscsi <gwname> --placement="count=2"
```

---

## 9) Наблюдаемость и эксплуатация

- **Панель:** mgr‑dashboard (здоровье, пулы, OSD/PG, RGW, CephFS).    
- **Метрики:** Prometheus + Alertmanager; логи — в централизованный стек.
- **RGW‑админ:** `radosgw-admin` (бакеты, пользователи, квоты, многосайтовость).    
- **Частые команды:**    
    ```bash
    ceph -s                      # общее здоровье
    ceph osd tree               # топология CRUSH
    ceph df                     # использование пулов
    ceph health detail          # подробности статусов
    rados df                     # низкоуровневые цифры RADOS
    rbd ls -p <pool>            # тома в пуле
    radosgw-admin bucket stats  # статистика по бакетам
    ```    

---

## 10) Практические гайдлайны (коротко)

- Разносите **public/cluster** сети; проверяйте MTU end‑to‑end.    
- Пулы RGW‑метаданных/индексов и CephFS‑метаданных держите на быстрых носителях    
- Не экономьте на **MON**: минимум три, на разных узлах/стойках.    
- Планируйте **PG** исходя из числа OSD и пулов; избегайте экстремумов.
- Для VM — предпочитайте **RBD через librbd** (нативная интеграция в QEMU/KVM).
- Для «не умеющих RBD» хостов — **iSCSI** с multipath и 2+ gateways.
- Multisite RGW — только после пилота: фиксируйте RPO/RTO и политику конфликтов версий.

---

## 11) Глоссарий

- **RADOS** — низкоуровневое надёжное хранилище объектов в Ceph.    
- **PG (Placement Group)** — группа размещения, единица балансировки.
- **CRUSH** — алгоритм расчёта размещения объектов по OSD без централизованного каталога.
- **RBD** — блочные тома Ceph.
- **RGW** — S3/Swift‑шлюз к RADOS.
- **CephFS** — POSIX‑файловая система поверх RADOS.
- **EC (Erasure Coding)** — кодирование с избыточностью вместо простых копий.

