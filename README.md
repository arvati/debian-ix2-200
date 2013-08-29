Debian Linux on ix2-200
=======

Configs and tools for installing vanilla Debian Linux on iomega ix2-200 NAS

About ix2-200
------
The IOmega ix2-200 NAS is a dual SATA NAS powered by a Marvell Kirkwood SoC clocked at 1GHz. It has 256MB of RAM and 32MB of flash memory. 3x USB 2.0 and 1x 1Gbit/s NIC are available.

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
+ Debian GNU Linux 7 Wheezy
+ Arch i386 is suggested

I suggest to create a dev environment using an LXC container with Debian Wheezy i386. You can find an appropiate template here: https://github.com/daniviga/ix2-200/blob/master/lxc-template/lxc-debian-i386

The config file provided is higly customized and stripped down. Please revise your configuration using ```make menuconfig``` before compile the kernel (see below).


Cross compiling the kernel
------
### Setup cross compiler ###

```bash
dpkg --add-architecture armel
apt-get update
```
```bash
apt-get install build-essential fakeroot kernel-package linux-source u-boot-tools zlib1g-dev libncurses5-dev binutils-multiarch
```

Into your /etc/apt/sources.list add:
```bash
echo "deb http://emdebian.org/~thibg/repo/ sid main" >> /etc/apt/sources.list
apt-get update
```
```bash
apt-get install gcc-4.7-arm-linux-gnueabi binutils-arm-linux-gnueabi
```
```bash
update-alternatives --install /usr/bin/arm-linux-gnueabi-gcc arm-linux-gnueabi-gcc /usr/bin/arm-linux-gnueabi-gcc-4.7 46 \
      --slave /usr/bin/arm-linux-gnueabi-gcov arm-linux-gnueabi-gcov /usr/bin/arm-linux-gnueabi-gcov-4.7
```
### Patch the sources ###
```bash
cd /usr/src/
tar xjvf linux-sources-3.2.tar
wget https://raw.github.com/daniviga/ix2-200/master/config-3.2.46-ix2_200
wget https://raw.github.com/daniviga/ix2-200/master/0001-Modified-rd88f6281-setup-file-to-support-ix2-200.patch
```
```bash
cd linux-sources-3.2
patch -p1 < 0001-Modified-rd88f6281-setup-file-to-support-ix2-200.patch
```
### Configure the kernel ###
```bash
export CC=arm-linux-gnueabi-gcc
export CROSS_COMPILE=arm-linux-gnueabi-
export ARCH=arm
export DEB_HOST_ARCH=armel

make-kpkg clean

cp ../config-3.2.46-ix2_200 .config
make oldconfig
make menuconfig
```
### Build the kernel ###
```bash
fakeroot make-kpkg -j2 --arch arm --cross-compile arm-linux-gnueabi- --append-to-version=-ix2-200 --initrd kernel_image kernel_headers
```

### Install the new kernel on the device ###
```bash
mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n initramfs -d initrd.img-3.2.46 uInitrd-ix200
mkimage -A arm -O linux -T kernel  -C none -a 0x00008000 -e 0x00008000 -n Linux-3.2.46 -d vmlinuz-3.2.46 uImage-ix200
```
```bash
flash_eraseall /dev/mtd0
flash_eraseall /dev/mtd1

nandwrite -p /dev/mtd0 /boot/uImage-ix200 
nandwrite -p /dev/mtd1 /boot/uInitrd-ix200 
```

Boot examples
------
### Boot from USB ###
```bash
usb start
setenv bootargs_console 'console=ttyS0,115200 mtdparts=orion_nand:0x300000@0x100000(uImage),0x1000000@0x540000(uInitrd) root=/dev/mapper/sata-root'
setenv bootargs $(bootargs_console)
ext2load usb 0:1 0x00800000 /uImage-new
ext2load usb 0:1 0x01100000 /uInitrd-new
bootm 0x00800000 0x01100000
```

## Boot from internal flash ###
```bash
setenv mtdparts 'mtdparts=orion_nand:0x100000@0x000000(uboot)ro,0x20000@0xA0000(uboot_env),0x300000@0x100000(uImage),0x1000000@0x540000(uInitrd)'
setenv bootargs_console 'console=ttyS0,115200 mtdparts=orion_nand:0x300000@0x100000(uImage),0x1000000@0x540000(uInitrd) root=/dev/mapper/sata-root'
setenv bootcmd 'setenv bootargs $(bootargs_console); nand read 0x800000 uImage; nand read 0x1100000 uInitrd; bootm 0x00800000 0x01100000'
saveenv
reset
```

Links and sources
------
+ http://blog.nobiscuit.com/2011/08/06/installing-debian-to-disk-on-an-ix2-200/
+ http://blog.nobiscuit.com/2012/02/15/kernel-patch-to-support-leds-buttons-and-sensors/
+ http://iomega.nas-central.org/wiki/Category:Ix2-200
+ http://copyninja.info/2013/01/multi-arch_based_cross-compilation.html
