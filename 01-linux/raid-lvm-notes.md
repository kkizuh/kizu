![[Сравнение уровней RAID.png]]

## Что для чего

- **RAID** — склеить **несколько дисков** в один:
    - **RAID0** — скорость (без защиты).
    - **RAID1** — надёжность (зеркало, можно потерять 1 диск).
    - **RAID5/6** — компромисс (1–2 диска можно потерять).
    - **RAID10** — быстро и надёжно (но нужно ≥4 диска).
- **LVM** — сделать **гибкие тома** поверх дисков/RAID:
    - Легко **увеличивать** размер.
    - Можно **добавлять** новые диски в пул.
    - **Снапшоты** для бэкапов/тестов.

> Часто: **RAID (mdadm) → LVM → Файловая система (ext4/xfs)**

---

## Мини-словарь (1 строка — 1 мысль)

- **PV (Physical Volume)** — “сырой” диск/раздел, отданный LVM.
- **VG (Volume Group)** — общий **пул** из PV.
- **LV (Logical Volume)** — “виртуальный диск” из пула VG → на него ставим ФС и монтируем.

---

## Быстрый выбор (не думай долго)

- Хочешь **просто надёжно** на 2 дисках → **RAID1**.
- Хочешь **увеличивать тома онлайн** → **LVM**.
- Не знаешь, какую ФС → **ext4** (универсально).  
    Нужен быстрый рост без отмонтирования → **XFS** (но **нельзя уменьшать**).

---

## Рецепт 1: RAID1 из двух дисков и примонтировать

**Зачем:** зеркала ради отказоустойчивости.

```bash
# Установить инструмент
sudo apt/dnf install mdadm -y

# Создать RAID1 (/dev/sdb и /dev/sdc — целые диски)
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
# Зачем: /dev/md0 = надёжный массив из двух зеркал.

# Проверка
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# Файловая система и монтирование
sudo mkfs.ext4 /dev/md0
sudo mkdir -p /data
sudo mount /dev/md0 /data
# Зачем: получил каталог /data на отказоустойчивом массиве.

# Автосборка на бут
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u   # Debian/Ubuntu; на RHEL не нужно
```

---

## Рецепт 2: LVM поверх одного диска (без RAID)

**Зачем:** чтобы **легко расширять** раздел позже.

```bash
# Подготовка: /dev/sdb — новый диск или раздел
sudo pvcreate /dev/sdb
# Зачем: пометили диск как PV (для LVM).

sudo vgcreate vgdata /dev/sdb
# Зачем: создали пул VG из этого диска.

sudo lvcreate -n lvdata -L 50G vgdata
# Зачем: выдали 50ГБ как логический том (LV).

sudo mkfs.ext4 /dev/vgdata/lvdata
sudo mkdir /data && sudo mount /dev/vgdata/lvdata /data
# Зачем: теперь /data — рабочий том, как обычный диск.
```

**Увеличить позже (онлайн):**

```bash
sudo lvextend -L +20G /dev/vgdata/lvdata   # +20ГБ к LV
sudo resize2fs /dev/vgdata/lvdata          # расширить ext4
# Зачем: добавили место без сноса данных.
```

---

## Рецепт 3: RAID1 → LVM → XFS (частая прод-схема)

**Зачем:** и надёжно, и гибко увеличивать.

```bash
# RAID1
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# LVM на RAID
sudo pvcreate /dev/md0
sudo vgcreate vgdata /dev/md0
sudo lvcreate -n lvdata -L 100G vgdata

# ФС XFS и монтирование
sudo mkfs.xfs /dev/vgdata/lvdata
sudo mkdir /data && sudo mount /dev/vgdata/lvdata /data
# Зачем: /data на надёжном RAID, размером управляет LVM, XFS растёт онлайн.
```

**Увеличить (онлайн):**

```bash
sudo lvextend -L +50G /dev/vgdata/lvdata
sudo xfs_growfs /data
# Зачем: прирост без отмонтирования. (XFS уменьшать нельзя)
```

---

## Рецепт 4: добавить новый диск в VG и расширить том

**Зачем:** кончилось место в пуле, докинуть диск.

```bash
# Новый диск /dev/sdd
sudo pvcreate /dev/sdd
sudo vgextend vgdata /dev/sdd
# Зачем: добавили место в общий пул.

sudo lvextend -l +100%FREE /dev/vgdata/lvdata
# Зачем: отдали всему LV весь свободный пул.

# Рост ФС:
sudo xfs_growfs /data           # XFS
# или
sudo resize2fs /dev/vgdata/lvdata   # ext4
```

---

## Рецепт 5: fstab — автомонт после ребута

**Зачем:** чтобы /data само монтировалось.

```bash
blkid /dev/vgdata/lvdata
# Зачем: получить UUID.

# /etc/fstab (пример для ext4)
UUID=<тут-uuid>  /data  ext4  noatime  0  2
# Проверить без ребута:
sudo mount -a
```

---

## Диагностика — что где и зачем

```bash
lsblk -f            # дерево устройств, ФС, точки монтирования (понять что где)
cat /proc/mdstat    # статус RAID (идёт ли ресинк, всё ли ок)
sudo mdadm --detail /dev/md0       # детали массива
sudo pvs; sudo vgs; sudo lvs       # сводка по LVM (PV/VG/LV)
df -hT /data       # сколько места и какой тип ФС
```

---

## Три правила, чтобы не убиться

1. **Проверь устройство дважды**: `/dev/sdb` ≠ `/dev/sda`. Команда **пишущая**? Остановись и сверь `lsblk`.
2. Перед уменьшением **ext4** — **umount + e2fsck**; **XFS уменьшать нельзя**.
3. Правишь `/etc/fstab` — сразу `mount -a`. Если тишина — всё нормально; ошибка — чини до ребута.
