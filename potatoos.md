# Как создать свою ОС: Пошаговый гайд для начинающих

Мы соберем настоящую, работающую операционную систему на базе Linux. Она будет весить всего пару десятков мегабайт, загружаться за секунды и работать целиком из оперативной памяти.

## Что внутри нашей ОС?
* **Ядро (Kernel)** — «директор» системы, который управляет железом.
* **BusyBox** — «набор инструментов». Один файл, который заменяет собой сотни программ.
* **Initramfs** — «рюкзак» с файлами, который ядро берет с собой при загрузке.
* **Syslinux** — «зажигание», которое заводит двигатель (ядро).
---

## Шаг 1: Подготовка (Установка и Загрузка)

Для начала подготовим рабочее место. Создадим папку проекта и скачаем туда **абсолютно всё**, что нам понадобится.

### 1.1 Установка инструментов сборки
Открой терминал и установи программы-помощники:
```bash
sudo apt update
sudo apt install build-essential libncurses-dev flex bison bc libelf-dev libssl-dev qemu-system-x86 xorriso wget cpio
```

### 1.2 Загрузка всех исходников
Создаем папку и скачиваем ядро, BusyBox и загрузчик:
```bash
mkdir ~/myos_project && cd ~/myos_project

# Скачиваем Ядро Linux
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.0.tar.xz

# Скачиваем BusyBox
wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2

# Скачиваем Загрузчик Syslinux
wget https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/syslinux-6.03.tar.xz

# Распаковываем всё сразу
tar -xf linux-7.0.tar.xz
tar -xf busybox-1.37.0.tar.bz2
tar -xf syslinux-6.03.tar.xz
```
---

## Шаг 2: Сборка Ядра (Linux Kernel)

1.  **Заходим в папку ядра:** `cd linux-7.0`
2.  **Настраиваем:** `make defconfig` (стандартные настройки).
3.  **Строим:** `make -j$(nproc)`
    *Это может занять 10–20 минут.*
4.  **Забираем результат:**
    `cp ./linux-7.0/arch/x86/boot/bzImage ~/myos_project/vmlinuz`
    `cd ~/myos_project`

---

## Шаг 3: Сборка BusyBox (Инструменты)

1.  **Заходим в папку:** `cd busybox-1.37.0`
2.  **Настраиваем:** Введите `make menuconfig`. Зайдите в **Settings** и отметьте пробелом **Build static binary (no shared libs)** `[*]`.
3.  **Собираем:**
    ```bash
    make -j$(nproc)
    make install
    ```
    Результат будет в папке `_install`.

---

## Шаг 4: Создаем файловую систему (RootFS)

Теперь соберем все файлы нашей ОС в одну кучу.

```bash
cd ~/myos_project
mkdir -p ./rootfs/{dev,proc,sys,etc}
cp -a busybox-1.37.0/_install/* ./rootfs/
```

Создадим сценарий запуска — файл `init`:
`nano ./rootfs/init`

Вставь в него текст:
```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs devtmpfs /dev
dmesg -n 1
echo "------------------------------"
echo "  Welcome to Mini Linux!   "
echo "------------------------------"
exec setsid cttyhack sh
```
ОБЯЗАТЕЛЬНО сделай файл исполняемым: `chmod +x ./rootfs/init`

---

## Шаг 5: Настройка загрузчика Syslinux

Мы используем скачанный ранее Syslinux. Будьте очень внимательны с путями!

1.  **Готовим папку для диска:**
    ```bash
    cd ~/myos_project
    mkdir -p ./iso_root/boot/isolinux
    cp vmlinuz ./iso_root/boot/
    ```
2.  **Копируем файлы загрузчика:**
    ```bash
    cp ~/syslinux-6.03/bios/core/isolinux.bin ./iso_root/boot/isolinux/
    # Используем правильный путь для версии 6.03:
    cp ~/syslinux-6.03/bios/com32/elflink/ldlinux/ldlinux.c32 ./iso_root/boot/isolinux/
    ```
3.  **Создаем конфиг:**
    `nano ./iso_root/boot/isolinux/isolinux.cfg`
    ```text
    default myos
    label myos
        kernel /boot/vmlinuz
        append initrd=/boot/initrd.img
    ```

---

## Шаг 6: Финальная упаковка и Запуск

Запаковываем нашу систему в архив (Initramfs):
```bash
cd ~/myos_project/rootfs
find | cpio -oH newc | gzip > ../iso_root/boot/initrd.img
```

Создаем сам образ диска:
```bash
cd ~/myos_project
genisoimage -o my_os.iso \
    -b boot/isolinux/isolinux.bin \
    -c boot/isolinux/boot.cat \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    ./iso_root
```

---

## Шаг 7: Момент истины!

Запускаем твою ОС в эмуляторе:
```bash
qemu-system-x86_64 -cdrom my_os.iso
```

Если ты видишь приглашение `/#` — ты только что стал создателем собственной операционной системы. Теперь ты можешь смело хвастаться этим перед друзьями (или в резюме)!
