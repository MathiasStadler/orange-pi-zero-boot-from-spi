# orange-pi-zero-boot-from-spi

## sources

```txt
https://forum.armbian.com/topic/3252-opi-zero-boot-with-spi/
```

## hardware

- [orange pi zero](http://www.orangepi.org/orangepizero/)
  - 512 MByte version
  - ** with SPI memory **
- usb pen drive or disk with usb adapter

## os

- [ARMBIAN strech](https://www.armbian.com/orange-pi-zero/)

```bash
> uname -a
Linux radio 4.14.12-sunxi #1 SMP Tue Jan 9 13:53:33 UTC 2018 armv7l GNU/Linux
```


## install software

- **first** freeze the kernel packages
  the package not anytime updated

  -via armbian-config

  -or via cli

```bash
sudo apt-mark hold linux-dtb-next-sunxi
sudo apt-mark hold linux-image-next-sunxi
sudo apt-mark hold linux-stretch-root-next-orangepizero
sudo apt-mark hold linux-u-boot-orangepizero-next
```

    - un hold

```bash
sudo apt-mark unhold linux-dtb-next-sunxi
sudo apt-mark unhold linux-image-next-sunxi
sudo apt-mark unhold linux-stretch-root-next-orangepizero
sudo apt-mark unhold linux-u-boot-orangepizero-next
```


- display packages on hold

```bash
dpkg --get-selections |awk '$2 == "hold" { print $1 }'
```

```bash
sudo apt update && sudo apt upgrade && sudo apt autoremove
sudo apt-get install flashrom
```

## activate overlay for spi bus (rom)

**PLEASE BE SURE** you have a OPI zero with a spi ic :-) Some version came w/o spi

- edit in file /boot/armbianEnv.txt and add spi-spidev AND param_spidev_spi_bus=0

```bash
overlays=analog-codec usbhost2 usbhost3 spi-spidev
param_spidev_spi_bus=0
```

- with bash

```bash
ARMBIAN_ENV_FILE="/boot/armbianEnv.txt"
if sudo grep spi-spidev $ARMBIAN_ENV_FILE; then
echo "[ok] found spi-spidev"
else
echo "[missing] NOT found spi-spidev"
echo "[todo] add spi-spidev"
sudo sed  -i '/overlays=/s/$/ spi-spidev/' $ARMBIAN_ENV_FILE
echo "param_spidev_spi_bus=0"|sudo tee -a $ARMBIAN_ENV_FILE
fi
```

- sample /boot/armbianEnv.txt file from working device

```bash
verbosity=1
logo=disabled
console=both
disp_mode=1920x1080p60
overlay_prefix=sun8i-h3
overlays=usbhost2 usbhost3
rootdev=UUID=84168f41-20b8-4905-9021-54488d09dc33
rootfstype=ext4
overlays=analog-codec usbhost2 usbhost3 spi-spidev
param_spidev_spi_bus=0
usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u
```

## cold reboot the OPI zero

```bash
sudo shutdown -Fh now
# unplug/plug the power adapter
```

**AFTER THE COLD REBOOT** should you see the spi device under /dev

```bash
>ls -l /dev/spidev0.0
crw------- 1 root root 153, 0 Nov 10 21:17 /dev/spidev0.0
```


## prepare usb or disk

- install Armbian on pendisk or disk

```bash
sudo nand-sata-install
# follow instruction inside the menu

# mount the pendrive/stick
sudo mount /dev/sda1 /mnt

# copy the /boot directory
sudo cp -a /boot /mnt

# edit  the file /mnt/boot/boot.cmd and fix the line setenv rootdev
# old entry
# setenv rootdev "/dev/mmcblk0p1"
# it is the same device did you mounted in last step
setenv rootdev "/dev/sda1"

# create a new boot.scr on the boot device
sudo mkimage -C none -A arm -T script -d /mnt/boot/boot.cmd /mnt/boot/boot.scr

# edited /etc/fstab
# uncomment the line with /media/mmcboot
# AND
# uncomment the line with /media/mmcboot/boot
sudo sed -i '/mmcboot/s/^/#/' /mnt/etc/fstab


# and ADD the the boot media
#
# **ATTENTION** you must used the id from YOUR device
# you get the id of YOUR device with the command
>sudo blkid /dev/sda1
/dev/sda1: UUID="84168f41-20b8-4905-9021-54488d09dc33" TYPE="ext4" PARTUUID="9810d5f7-01"
#
# save /mnt/etc/fstab
sudo cp /mnt/etc/fstab /mnt/etc/fstab_save
# edit the /etc/fstab with YOUR data to
UUID=84168f41-20b8-4905-9021-54488d09dc33   /    ext4    defaults,noatime,nodiratime,commit=600,errors=remount-ro,x-gvfs-hide    0   1

printf "UUID=%s\t/\t%s\tdefaults,noatime,nodiratime,commit=600,errors=remount-ro,x-gvfs-hide\t0\t1" $(sudo blkid /dev/sda1 -o value -s UUID) $(sudo blkid /dev/sda1 -o value -s TYPE)|sudo tee -a /mnt/etc/fstab

```

## prepare spi for boot

```bash
# create empty image
sudo dd if=/dev/zero count=2048 bs=1K | tr '\000' '\377' > spi.img
# ATTENTION in some tutorial used the size 4096k BUT in the some OPI zero board are only a 2048K
# We need only 537k space for the u-boot image in this case

# write/copy u-boot in spi image
# this file is already of the images/sd card
# only the version number is depend from the armbian version
sudo dd if=/usr/lib/linux-u-boot-next-orangepizero_5.65_armhf/u-boot-sunxi-with-spl.bin of=spi.img bs=1k conv=notrunc

ARMBIAN_VERSION=$(grep VERSION /etc/armbian-release|sed 's/.*=//g')
sudo dd if=/usr/lib/linux-u-boot-next-orangepizero_${ARMBIAN_VERSION}_armhf/u-boot-sunxi-with-spl.bin of=spi.img bs=1k conv=notrunc


# flash the image to spi rom
sudo flashrom -p linux_spi:dev=/dev/spidev0.0 -w spi.img -c MX25L1605A/MX25L1606E/MX25L1608E
```

## troubleshooting flashrom

- the first 100 byte of the spi.img must the same as the u-boot-sunxi-with-spl.bin

```bash
>hexdump -C -n100 spi.img
00000000  16 00 00 ea 65 47 4f 4e  2e 42 54 30 d5 5b 4c 75  |....eGON.BT0.[Lu|
00000010  00 60 00 00 53 50 4c 02  00 00 00 00 00 00 00 00  |.`..SPL.........|
00000020  2c 00 00 00 00 00 00 00  00 00 00 00 73 75 6e 38  |,...........sun8|
00000030  69 2d 68 32 2d 70 6c 75  73 2d 6f 72 61 6e 67 65  |i-h2-plus-orange|
00000040  70 69 2d 7a 65 72 6f 00  00 00 00 00 00 00 00 00  |pi-zero.........|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000060  0f 00 00 ea                                       |....|

> hexdump -C -n100 /usr/lib/linux-u-boot-next-orangepizero_5.65_armhf/u-boot-sunxi-with-spl.bin
00000000  16 00 00 ea 65 47 4f 4e  2e 42 54 30 d5 5b 4c 75  |....eGON.BT0.[Lu|
00000010  00 60 00 00 53 50 4c 02  00 00 00 00 00 00 00 00  |.`..SPL.........|
00000020  2c 00 00 00 00 00 00 00  00 00 00 00 73 75 6e 38  |,...........sun8|
00000030  69 2d 68 32 2d 70 6c 75  73 2d 6f 72 61 6e 67 65  |i-h2-plus-orange|
00000040  70 69 2d 7a 65 72 6f 00  00 00 00 00 00 00 00 00  |pi-zero.........|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000060  0f 00 00 ea                                       |....|
```

- after flashrom write can you reread the image back and compare

```bash
# read spi rom to img file
>sudo flashrom -p linux_spi:dev=/dev/spidev0.0 -r spi_read.img -c MX25L1605A/MX25L1606E/MX25L1608E
```

- the hexdump of the spi_read.img must the same as the spi.img
- check e,g, with md5sum and other checksum methods

```bash
>md5sum spi.img 
c61f5c3823f1752d59505148ce90ae89  spi.img
>md5sum spi_read.img 
c61f5c3823f1752d59505148ce90ae89  spi_read.img
```

- if the reread images is only collection of null repeat all step in prepare spi for boot
- WRONG WRITING OR WRITING WITHOUT COPY THE UBOOT.BIN

```bash
>hexdump -C -n100 spi_read.img
00000000  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
```

- Warning: Chip content is identical to the requested image. give you a hint of any error

## cold reboot the device

```bash
shutdown -Fh now
# unplug the power adapter
# remove the sd card
# start the device hopeful boot from spi/usb pendrive
```

## delete spi device / enable boot from sdcard

- delete the u-boot image in the spi memory

```bash
# erase
sudo flashrom -p linux_spi:dev=/dev/spidev0.0 -E -c MX25L1605A/MX25L1606E/MX25L1608E
# check it empty
sudo flashrom -p linux_spi:dev=/dev/spidev0.0 -r spi_read.img -c MX25L1605A/MX25L1606E/MX25L1608E
hexdump -C -n100 spi_read.img
00000000  ff ff ff ff ff ff ff ff  ff ff ff ff ff ff ff ff  |................|
# you should see a collection of null
```

- now should we boot from sd card again
