## Как думать

1. **Посмотреть, что есть** → 2) **Разметить (если нужно)** → 3) **Сделать ФС** → 4) **Смонтировать** → 5) **Добавить в автозагрузку (fstab)**.  
    LVM — это надстройка для гибкого увеличения/разбиения: **можно позже**, не обязательно сразу.
    

---

## 0) Что у меня за диски? (только читаем)

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID,MODEL
# показать диски/разделы/куда примонтированы

blkid
# показать UUID и метки (LABEL) разделов

sudo fdisk -l
# подробный список дисков и их таблица разделов (MBR/GPT)

sudo smartctl -H /dev/sda
# краткий SMART-статус: PASSED/FAILED (пакет smartmontools)
```

**Запомни:** /dev/sda — первый диск, /dev/sdb — второй и т.д. Разделы — /dev/sdb1, /dev/sdb2…

---

## 1) Разметка нового диска (если пустой)

**Цель:** на диске `/dev/sdb` сделать **один** раздел `/dev/sdb1` на весь объём.

Вариант 1 (проще — `fdisk`, интерактивно):

```bash
sudo fdisk /dev/sdb
# n (new partition) → ENTER (по умолчанию primary) → ENTER (начало)
# ENTER (до конца диска) → w (write)
```

Вариант 2 (автоматом — `parted`, GPT):

```bash
sudo parted /dev/sdb --script \
  mklabel gpt \
  mkpart primary ext4 1MiB 100%
# будет /dev/sdb1, тип под ext4 (подойдёт и для xfs)
```

Проверяем:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE
```

---

## 2) Создать файловую систему (ФС)

**Что выбрать?**

- **ext4** — универсально (уменьшать можно, увеличивать тоже).
- **xfs** — быстро, хорошо растёт онлайн (уменьшать НЕЛЬЗЯ).
- **btrfs** — есть сжатие из коробки (тонкости не трогаем сейчас).

Команда (на **разделе** `/dev/sdb1`, не на диске!):

```bash
# ext4
sudo mkfs.ext4 -L data /dev/sdb1

# xfs
sudo mkfs.xfs -L data /dev/sdb1
```

`-L data` — читаемая метка (не обязательно).

---

## 3) Временно смонтировать и проверить

```bash
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
mount | grep /mnt/data   # видим, что примонтировано
sudo umount /mnt/data    # отмонтировать (для проверки, по желанию)
```

---

## 4) Автомонт при старте (fstab)

Почему **UUID**? Не сломается, если имена `/dev/sdX` сменятся.

1. Узнать UUID:
    

```bash
blkid /dev/sdb1
# пример: UUID="1111-2222-3333-4444"
```

2. Записать в `/etc/fstab` (пример для ext4):
    

```
# <что>               <куда>  <тип>  <опции>                 <dump> <pass>
UUID=1111-2222...     /data   ext4   noatime,errors=remount-ro  0     2
```

для XFS:

```
UUID=1111-2222...     /data   xfs    noatime                    0     0
```

3. Применить и проверить:
    

```bash
sudo mkdir -p /data
sudo mount -a                 # прочитать fstab и примонтировать
systemd-analyze verify /etc/fstab   # проверить на ошибки синтаксиса
```

Если `mount -a` молча отработал → всё ок.

---

## 5) LVM — по-простому (если нужен гибкий размер)

**Кейс:** хочешь уметь **увеличивать** раздел без плясок.

Создаём VG и LV поверх **раздела**:

```bash
# инициализируем раздел как PV (physical volume)
sudo pvcreate /dev/sdb1

# создаём VG (volume group)
sudo vgcreate vgdata /dev/sdb1

# создаём логический том (LV) размером 100G
sudo lvcreate -n lvdata -L 100G vgdata
```

Файловая система на LV:

```bash
sudo mkfs.xfs /dev/vgdata/lvdata     # xfs удобно для роста
sudo mkdir /data
sudo mount /dev/vgdata/lvdata /data
```

fstab:

```
/dev/vgdata/lvdata   /data   xfs   noatime   0 0
```

### Увеличить без даунтайма

Добавился ещё диск `/dev/sdc` → делаем ещё раздел `/dev/sdc1` и:

```bash
sudo pvcreate /dev/sdc1
sudo vgextend vgdata /dev/sdc1
sudo lvextend -L +50G /dev/vgdata/lvdata
sudo xfs_growfs /data                # xfs растёт по точке монтирования
# (ext4: sudo resize2fs /dev/vgdata/lvdata)
```

### Уменьшать

- **ext4**: только оффлайн (umount → e2fsck → resize2fs SIZE → lvreduce).
    
- **xfs**: **уменьшать нельзя** (только перенос данных и создать заново).
    

---

## 6) Проверка/ремонт ФС (когда плохо)

> Делай на **размонтированной** ФС (кроме некоторых специальных случаев).

```bash
# общая проверка (не для xfs)
sudo fsck -f /dev/sdb1

# для ext4
sudo e2fsck -f /dev/sdb1

# для XFS
sudo xfs_repair /dev/sdb1
```

---

## 7) SWAP — два варианта

**Раздел:**

```bash
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
echo 'UUID=<uuid> none swap sw 0 0' | sudo tee -a /etc/fstab
```

**Файл (быстро и удобно):**

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 8) Полезные мелочи (только когда надо)

```bash
sudo tune2fs -l /dev/sdb1    # инфо про ext4-параметры
sudo xfs_info /dev/sdb1      # инфо про xfs
sudo wipefs -a /dev/sdb      # стереть сигнатуры (ОСТОРОЖНО! УНИЧТОЖИТ ДАННЫЕ)
sudo fstrim -av              # TRIM на SSD вручную (лучше по таймеру)
```

---

## 9) Быстрый “хелсчек” диска

```bash
smartctl -H /dev/sda                           # краткий статус
smartctl -a /dev/sda | egrep 'Reallocated|Pending|Uncorrect'
dmesg | egrep -i 'blk|disk|I/O error'
```

---

## 10) Анти-грабли

- Перед ребутом всегда: `sudo mount -a` — чтобы не словить чёрный экран.
    
- Для SSD: периодический `fstrim -av` (timer), а не только опция `discard` в fstab.
    
- XFS нельзя уменьшать. Если нужен “плюс-минус туда-сюда” — бери ext4 или планируй перенос.
    
- Перед уменьшением ext4 — **бэкап обязателен**.
    
- В LVM следи за использованием **snapshot** — переполнение снапа убивает снапшот.
    

---

### Мини-рецепты (копипаст)

**Новый диск → раздел → ext4 → автомонт в /data**

```bash
sudo parted /dev/sdb --script mklabel gpt mkpart primary ext4 1MiB 100%
sudo mkfs.ext4 -L data /dev/sdb1
sudo mkdir -p /data
sudo blkid /dev/sdb1
# вставь UUID в /etc/fstab:
# UUID=...  /data  ext4  noatime,errors=remount-ro  0 2
sudo mount -a
```

**Увеличить XFS на LVM на 50ГБ онлайн**

```bash
sudo lvextend -L +50G /dev/vgdata/lvdata
sudo xfs_growfs /data
```

Если что-то из этого всё ещё непонятно — скажи, на каком шаге стопор, разберём **именно его** с примерами (“вот у меня sdb, хочу /data, но боюсь сломать”).