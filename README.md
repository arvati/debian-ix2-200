Debian Linux on ix2-200
=======

Configs and tools for installing vanilla Debian 'Jessie' Linux on a iomega ix2-200 NAS

About ix2-200
------
The iomega ix2-200 NAS is a dual SATA NAS powered by a Marvell Kirkwood SoC clocked at 1GHz. It has 256MB of RAM and 32MB of flash memory. 3x USB 2.0 and 1x 1Gbit/s NIC are available.

```
         __  __                      _ _
        |  \/  | __ _ _ ____   _____| | |
        | |\/| |/ _` | '__\ \ / / _ \ | |
        | |  | | (_| | |   \ V /  __/ | |
        |_|  |_|\__,_|_|    \_/ \___|_|_|
 _   _     ____              _
| | | |   | __ )  ___   ___ | |_ 
| | | |___|  _ \ / _ \ / _ \| __| 
| |_| |___| |_) | (_) | (_) | |_ 
 \___/    |____/ \___/ \___/ \__| 
 ** MARVELL BOARD: RD-88F6281A LE  

U-Boot 1.1.4 (Sep  8 2009 - 09:31:54) Marvell version: 3.4.14

Mapower version: 2.0 (32MB) (2009/09/08)

U-Boot code: 00600000 -> 0067FFF0  BSS: -> 006CEE60

Soc: 88F6281 A0 (DDR2)
CPU running @ 1000Mhz L2 running @ 333Mhz
SysClock = 333Mhz , TClock = 200Mhz 

DRAM CAS Latency = 5 tRP = 5 tRAS = 18 tRCD=6
DRAM CS[0] base 0x00000000   size 256MB 
DRAM Total size 256MB  16bit width
Flash:  0 kB
Addresses 8M - 0M are saved for the U-Boot usage.
Mem malloc Initialization (8M - 7M): Done
NAND:32 MB

CPU : Marvell Feroceon (Rev 1)

Streaming disabled 
Write allocate disabled

Module 0 is RGMII
Module 1 is TDM

USB 0: host mode
PEX 0: interface detected no Link.
Net:   egiga0, egiga1 [PRIME]
Fan lookup table initialized.

Current remote temperature: 25
Current fan speed: 0

Hit any key to stop autoboot:  0 
Marvell>>
```

More info available at http://iomega.nas-central.org/wiki/Category:Ix2-200


Pre-requisites
------
+ Debian GNU Linux 8 Jessie
+ Arch i386 is reccomended 

I suggest to create a dev environment using an LXC container with Debian Jessie i686:

```bash
sudo lxc-create -t debian -n crossdebian -- -r jessie -a i686
```

Cross compiling the kernel
------
### Setup cross compiler ###

```bash
# Add the embedian repo:
echo "deb http://emdebian.org/tools/debian/ jessie main" >> /etc/apt/sources.list
curl http://emdebian.org/tools/debian/emdebian-toolchain-archive.key | apt-key add -

# Add the source repo:
echo "deb-src http://mi.mirror.garr.it/mirrors/debian/ jessie main contrib non-free" >> /etc/apt/sources.list

dpkg --add-architecture armel
apt-get update
apt-get install wget devscripts crossbuild-essential-armel kernel-package ncurses-dev
```

### Get the sources ###
```bash
cd /usr/src
apt-get source linux
```

### Configure the kernel ###
```bash
export CC=arm-linux-gnueabi-gcc
export CROSS_COMPILE=arm-linux-gnueabi-
export ARCH=arm
export DEB_HOST_ARCH=armel

make-kpkg clean

wget -O- https://raw.githubusercontent.com/daniviga/ix2-200/master/configs/config-3.16.0-4-kirkwood-ix2-200 > .config
make oldconfig
```

### Generate the dtb only ###
```bash
make  dtbs
```

### Build the full kernel ###
```bash
# Optional
make menuconfig

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- EXTRAVERSION=-ix2-200 KDEB_PKGVERSION=2 KBUILD_DEBARCH=armel deb-pkg
```

### Append the dtb to the kernel ###
```bash
cp arch/arm/boot/dts/kirkwood-iomega_ix2_200.dtb /boot
cd /boot
cat vmlinuz-3.16.0-4-kirkwood kirkwood-iomega_ix2_200.dtb > vmlinuz-3.16.0-4-kirkwood-dtb
```

### Install the new kernel on the device ###
```bash
mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n initramfs -d initrd.img-3.16.0-4-kirkwood uInitrd
mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n Linux+dtb -d vmlinuz-3.16.0-4-kirkwood-dtb uImage-dtb
```
```bash
flash_eraseall /dev/mtd0
flash_eraseall /dev/mtd1

nandwrite -p /dev/mtd0 /boot/uImage-dtb
nandwrite -p /dev/mtd1 /boot/uInitrd 
```

Boot examples
------
### Boot from USB ###
```bash
usb start
setenv bootargs_console 'console=ttyS0,115200 mtdparts=orion_nand:0x300000@0x100000(uImage),0x1000000@0x540000(uInitrd) root=/dev/sda1'
setenv bootargs $(bootargs_console)
ext2load usb 0:1 0x00800000 /uImage-new
ext2load usb 0:1 0x01100000 /uInitrd-new
bootm 0x00800000 0x01100000
```

### Boot from internal flash ###
```bash
setenv mtdparts 'mtdparts=orion_nand:0x100000@0x000000(uboot)ro,0x20000@0xA0000(uboot_env),0x300000@0x100000(uImage),0x1000000@0x540000(uInitrd)'
setenv bootargs_console 'console=ttyS0,115200 mtdparts=orion_nand:0x300000@0x100000(uImage),0x1000000@0x540000(uInitrd) root=/dev/sda1'
setenv bootcmd 'setenv bootargs $(bootargs_console); nand read 0x800000 uImage; nand read 0x1100000 uInitrd; bootm 0x00800000 0x01100000'
saveenv
reset
```

Links and sources
------
+ http://blog.nobiscuit.com/2011/08/06/installing-debian-to-disk-on-an-ix2-200/
+ http://blog.nobiscuit.com/2012/02/15/kernel-patch-to-support-leds-buttons-and-sensors/
+ http://iomega.nas-central.org/wiki/Category:Ix2-200
+ https://wiki.debian.org/CrossToolchains
+ http://lacie-nas.org/doku.php?id=making_kernel_with_dtb
