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

- **first** freeze the kernel via armbian-config

```bash
sudo apt update && sudo apt upgrade && sudo apt autoremove
apt-get install flashrom
```

## activate overlay for spi bus (rom)

**BE SURE** you have a OPI zero with a spi ic :-) Some version came w/o spi

- edit in file /boot/armbianEnv.txt and add spi-spidev AND param_spidev_spi_bus=0


```bash
overlays=analog-codec usbhost2 usbhost3 spi-spidev
param_spidev_spi_bus=0
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


- cold reboot the OPI zero

```bash
shutdown -Fh now
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
mount /dev/sda1 /mnt

# copy the /boot directory
cp -a /boot /mnt

# edit  the file /mnt/boot/boot.cmd and fix setenv rootdev
# it is the same device did you mounted in last step
setenv rootdev "/dev/sda1"

# create a new boot.scr on the boot device
mkimage -C none -A arm -T script -d /mnt/boot/boot.cmd /mnt/boot/boot.scr

# edited /etc/fstab
# uncomment the line with /media/mmcboot
# and
# uncomment the line with /media/mmcboot/boot
# and ADD the the boot media
#
# **ATTENTION** you must used the id from YOUR device
# you get the id of YOUR device with the command
>blkid
/dev/sda1: UUID="84168f41-20b8-4905-9021-54488d09dc33" TYPE="ext4" PARTUUID="9810d5f7-01"
#
# edit the /etc/fstab with YOUR data to

UUID=84168f41-20b8-4905-9021-54488d09dc33   /    ext4    defaults,noatime,nodiratime,commit=600,errors=remount-ro,x-gvfs-hide    0   1

```

## prepare spi for boot

```bash
# create empty image
dd if=/dev/zero count=2048 bs=1K | tr '\000' '\377' > spi.img
# ATTENTION in some tutorial used the size 4096k BUT in the some OPI zero board are only a 2048K
# We need only 537k space for the u-boot image in this case

# write/copy u-boot in spi image
# this file is already of the images/sd card
dd if=/usr/lib/linux-u-boot-next-orangepizero_5.37_armhf/u-boot-sunxi-with-spl.bin of=spi.img bs=1k conv=notrunc

# flash the image to spi rom
flashrom -p linux_spi:dev=/dev/spidev0.0 -w spi.img -c MX25L1605A/MX25L1606E/MX25L1608E
```

## cold reboot the device

```bash
shutdown -Fh now
# unplug the power adapter
# remove the sd card
# start the device hopeful boot from spi/usb pendrive
```
