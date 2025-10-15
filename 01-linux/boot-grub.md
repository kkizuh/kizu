# Загрузка и GRUB: как чинить бут

## 0) Быстрый план (что делать по порядку)

1. **Понять тип системы:** BIOS/UEFI, разметка MBR/GPT.
2. **Смонтировать систему (часто из Live-ISO) и зайти в chroot.**
3. **Починить загрузчик (grub-install + обновить конфиги).**
4. **Починить initramfs (драйверы/UUID/crypto/LVM).**
5. **Проверить fstab/UUID/подсистемы (LVM/RAID/LUKS/BTRFS).**
6. **Перезагрузка и тест.**
    

---

## 1) Быстрая диагностика на месте (если грузится хоть как-то)

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT,UUID   # что за диски/ФС/UUID
blkid                                            # ещё раз UUID/тип ФС
cat /etc/fstab                                  # правильные ли UUID и точки монтирования
cat /proc/cmdline                               # что ядро думает о root= ...
```

### Понятные симптомы

- **grub rescue>** → повреждён GRUB/ESP/пути к модулям.
- **Kernel panic “VFS: unable to mount root fs”** → неверный `root=UUID` или нет драйвера в initramfs.
- **initramfs/busybox** → не видит корень: проверь UUID/LVM/RAID/crypto модули.
- **Черный экран/мигание курсора** → часто видеодрайвер → `nomodeset` временно.

---

## 2) Временная загрузка через меню GRUB (без Live)

На экране GRUB → выдели запись → **e** (edit):

- Добавь к строке ядра (linux …) временные параметры:
    - `systemd.unit=rescue.target` — однопользовательский режим
    - или `init=/bin/bash` — совсем минимум (без пароля)
    - `nomodeset` — если GUI падает
    - `rootdelay=5` — если диски “просыпаются” долго
- Сделай `/` rw после входа:
```bash
    mount -o remount,rw /
    ```
- Загрузиться: **Ctrl+x** или **F10**.

---

## 3) GRUB-подсказки из его консоли

### В `grub>` (полноценная консоль, не rescue):

```grub
ls                             # какие диски/разделы видит (hd0,gpt1)...
ls (hd0,gpt2)/                 # посмотреть содержимое раздела
set | grep -E 'root|prefix'    # где ищет модули
set root=(hd0,gpt2)            
set prefix=(hd0,gpt2)/boot/grub
insmod normal
normal                         # вернуться в обычное меню
```

### В `grub rescue>` (урезанный режим):

```grub
ls
set root=(hd0,gpt2)
set prefix=(hd0,gpt2)/boot/grub
insmod normal                  # если модуль найдётся
normal
```

> Подбирай раздел, где лежит `/boot`/`/boot/grub`.

---

## 4) Починка из Live-ISO (надёжный способ)

### 4.1. Смонтировать систему и войти в chroot

```bash
# Найди корневой раздел и EFI (если UEFI)
lsblk -f

# Примеры:
sudo mount /dev/sdXn /mnt                 # корень /
sudo mount /dev/sdXb /mnt/boot            # если есть отдельный /boot
sudo mount /dev/sdXa /mnt/boot/efi        # ESP (FAT32) для UEFI

# Если LVM:
sudo vgchange -ay
sudo mount /dev/mapper/vg0-root /mnt

# Если LUKS:
sudo cryptsetup luksOpen /dev/sdXY cryptroot
sudo mount /dev/mapper/cryptroot /mnt

# Примонтировать системные псевдоФС:
for i in /proc /sys /dev /run; do sudo mount --bind $i /mnt$i; done

# chroot внутрь установленной системы
sudo chroot /mnt
```

### 4.2. Переустановка GRUB

#### UEFI (x86_64):

```bash
# Убедись, что ESP смонтирован на /boot/efi (FAT32)
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=linux
grub-mkconfig -o /boot/grub/grub.cfg    # Debian/Ubuntu
update-grub                              # алиас на grub-mkconfig (Deb/Ub)
```

#### BIOS/Legacy:

```bash
grub-install /dev/sdX                    # ВАЖНО: на диск, не на раздел (без цифры)
grub-mkconfig -o /boot/grub/grub.cfg
```

### 4.3. Обновить initramfs (ядро видит корень? LVM? mdadm? crypto?)

- Debian/Ubuntu:
```bash
    update-initramfs -u -k all
    ```
- RHEL/CentOS/Fedora:
```bash
    dracut -f
    ```

### 4.4. Выйти и перезагрузить

```bash
exit
for i in /run /dev /sys /proc; do umount /mnt$i; done
umount /mnt/boot/efi /mnt/boot 2>/dev/null || true
umount /mnt
reboot
```

---

## 5) fstab и UUID (очень частая причина)

Проверь, что **fstab** ссылается на **реальные** UUID:
```bash
blkid
cat /etc/fstab
```

- Совпадает ли `UUID=` для `/`, `/boot`, `/boot/efi`?
- Для **BTRFS** проверь subvol: `rootflags=subvol=@` (часто в cmdline/GRUB).
- Для **LVM** корня: `root=/dev/mapper/vg0-root` или `root=UUID=` корректный.
- Для **mdadm RAID**: убедись, что массив собирается (пакет `mdadm`, `update-initramfs/dracut`).

---

## 6) Особые подсистемы

### LUKS (шифрование)

- В `initramfs` должен быть модуль `cryptsetup`.
- Проверь `/etc/crypttab` и `update-initramfs -u` / `dracut -f`.

### LVM

- `vgchange -ay` в initramfs/chroot.
- Включи lvm в initramfs (обычно автоматически, но проверь).

### mdadm (RAID)

- Пакет `mdadm` установлен, массив описан, initramfs обновлён.

### Secure Boot (UEFI)

- При самосборе модулей/драйверов — подпись/MOK может блокировать загрузку.
- Временно отключи Secure Boot в UEFI для диагностики.

---

## 7) efibootmgr (для UEFI-записи)

Проверить и править записи прошивки:

```bash
efibootmgr                 # список BootXXXX
efibootmgr -v
efibootmgr -o 0003,0001    # порядок загрузки
# создать новую запись (обычно grub-install сам делает):
efibootmgr -c -d /dev/nvme0n1 -p 1 -L "linux" -l '\EFI\linux\grubx64.efi'
```

---

## 8) Частые “быстрые фиксы”

### Черный экран после обновы видеодрайвера

- Временно добавить в GRUB параметры: `nomodeset`
- Потом переустановить/почистить драйвер (NVIDIA/AMD).

### Panic “VFS … cannot mount root”

- Проверка: `blkid`, `cat /proc/cmdline` в initramfs-shell.
- Исправить `root=UUID=...`/драйверы → обновить initramfs → перезагрузка.

### Вылет в `initramfs` (BusyBox)

- `blkid`, `ls /dev`, `cat /proc/cmdline`, `dmesg | tail`.
- Если корень на LVM/RAID/LUKS — не хватает модулей в initramfs.

---

## 9) Мини-шпарка команд (копипаст)

```bash
# UEFI: полностью переустановить GRUB
mount /dev/sdXn /mnt
mount /dev/sdXa /mnt/boot/efi
for i in /proc /sys /dev /run; do mount --bind $i /mnt$i; done
chroot /mnt
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=linux
update-grub && update-initramfs -u
exit; for i in /run /dev /sys /proc; do umount /mnt$i; done
umount /mnt/boot/efi /mnt; reboot

# BIOS: установить GRUB на диск
chroot /mnt
grub-install /dev/sdX
grub-mkconfig -o /boot/grub/grub.cfg
update-initramfs -u
reboot

# Проверить совпадение UUID / fstab
blkid
grep -v '^#' /etc/fstab

# Редактирование параметров ядра “на раз”
# (в меню GRUB на записи ядра: 'e', дописать к строке linux)
systemd.unit=rescue.target nomodeset init=/bin/bash rootdelay=5
```

---

## 10) Что НЕ делать

- Не ставь GRUB на **раздел** (`/dev/sda1`) в режиме BIOS — нужно `/dev/sda`.    
- Не трогай `grub.cfg` руками — генерируй `update-grub/grub-mkconfig`.
- Не забывай монтировать **/boot/efi** перед `grub-install` на UEFI.
- Не игнорируй `initramfs` — без нужных модулей ядро не увидит корень.

---

Надо — упакую это в твою базу как `[[boot-grub]]` и добавлю мини-диаграмму “что сломалось → что проверять → какую команду жать”.