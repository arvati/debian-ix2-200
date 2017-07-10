Debian Linux on ix2-200
=======

Install Debian 'Jessie' Linux on a iomega ix2-200 NAS

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
+ Debian or Ubuntu Machine

### Create an LXC container ###
Create a dev environment using an LXC container with Debian Jessie i686:

```bash
sudo lxc-create -t debian -n crossjessie -- -r jessie -a i686
sudo lxc-create -t debian -n ix2-200 -- -r stretch -a i386

sudo lxc-stop -n crossjessie --kill
sudo lxc-stop -n ix2-200 --kill

sudo chroot /var/lib/lxc/crossjessie/rootfs
sudo chroot /var/lib/lxc/ix2-200/rootfs

apt-get install sysvinit-core curl nano
exit
echo "lxc.aa_profile = unconfined" | sudo tee -a /var/lib/lxc/crossjessie/config
echo "lxc.aa_profile = unconfined" | sudo tee -a /var/lib/lxc/ix2-200/config

```

To run some command you can do:
```bash
sudo lxc-start -d -n crossjessie
sudo lxc-start -d -n ix2-200

sudo lxc-attach -n crossjessie -- apt-get update
sudo lxc-attach -n ix2-200 -- apt-get update
```

To have a full termnal you can do (user root password root):
```bash
sudo lxc-start -d -n crossjessie
sudo lxc-start -d -n ix2-200

sudo lxc-console -n crossjessie -t 1
sudo lxc-console -n ix2-200 -t 1
```
### Prepare the LXC container ###
Make some adjustments inside LXC container
```bash
nano /etc/apt/sources.list
deb http://http.debian.net/debian          stretch         main contrib non-free
#deb http://http.debian.net/debian          jessie         main contrib non-free

deb http://http.debian.net/debian          unstable         main contrib non-free
deb http://security.debian.org/ stretch/updates main contrib non-free
#deb http://security.debian.org/ jessie/updates main contrib non-free

deb http://emdebian.org/tools/debian/ stretch  main
#deb http://emdebian.org/tools/debian/ jessie  main

deb http://emdebian.org/tools/debian/ unstable  main
deb-src http://ftp.br.debian.org/debian/ stretch main contrib non-free
#deb-src http://ftp.br.debian.org/debian/ jessie main contrib non-free

deb-src http://ftp.br.debian.org/debian/ unstable main contrib non-free
```

```bash
nano /etc/apt/preferences.d/main
Package: * 
Pin: release a=stable 
Pin-Priority: 900

Package: *
Pin: release a=stretch
#Pin: release a=jessie
Pin-Priority: 600

Package: * 
Pin: release a=unstable 
Pin-Priority: 100
```

```bash
nano /etc/apt/apt.conf.d/70debconf
// Pre-configure all packages with debconf before they are installed.
// If you don't like it, comment it out.
DPkg::Pre-Install-Pkgs {"/usr/sbin/dpkg-preconfigure --apt || true";};
APT::Default-Release "stretch";
//APT::Default-Release "jessie";
```

```bash
curl http://emdebian.org/tools/debian/emdebian-toolchain-archive.key | sudo apt-key add -
dpkg --add-architecture armel 
apt-get update 
apt-get install wget devscripts crossbuild-essential-armel kernel-package gcc-arm-linux-gnueabi ncurses-dev u-boot-tools sshpass xz-utils fakeroot
```

Cross compiling the kernel
------


### Get the sources ###

```bash
apt-get  install -t unstable linux-source-4.8
mkdir ~/src; cd ~/src
tar xf /usr/src/linux-source-4.8.tar.xz
cd linux-source-4.8
unxz /usr/src/linux-config-4.8/config.armel_none_marvell.xz
cp -vi /usr/src/linux-config-4.8/config.armel_none_marvell .config

# IF want to compile rt kernel THEN
unxz /usr/src/linux-patch-4.8-rt.patch.xz
cp /usr/src/linux-patch-4.8-rt.patch ~/src/linux-source-4.8
patch -p1 --dry-run -i ../linux-patch-4.8-rt.patch

# IF THERE IS NO ERRORS THEN
# patch -p1 -i ../linux-patch-4.8-rt.patch

echo "-kirkwood-iomega-ix2-200" > localversion-rt
echo "-kirkwood-iomega-ix2-200" > localversion
```

### Modify your name ###
```bash
nano /etc/kernel-pkg.conf
# The maintainer information.
maintainer := Ademar Arvati Filho
email := vanaware@vanaware.com
# Priority of this version (or urgency, as dchanges would call it)
priority := Low
# This is the Debian revision number (defaulted to $(version)-10.00.Custom in debian/rules) 
 debian = 1
```

### Get some more sources ###
download overlay dir (full sources/overlay/) to ~/src/overlay/

download config file (at sources/configs/) to ~/src/linux-source-4.8/.config

download dtb and dts files (full /sources/dtb/) to ~/src/linux-source-4.8/arch/arm/boot/dts

### Configure the kernel ###
```bash
cd ~/src/linux-source-4.8
export TOOL_PREFIX="arm-linux-gnueabi" 
export CC="${TOOL_PREFIX}-gcc" 
export CXX="${TOOL_PREFIX}-g++"
export CROSS_COMPILE="${TOOL_PREFIX}-" 
export CFLAGS="-march=armv5te -mfloat-abi=soft -marm" 
export ARCH="arm"
export DEB_HOST_ARCH=armel
export LOCALVERSION="-arvati1" 
export CONCURRENCY_LEVEL=`grep -m1 cpu\ cores /proc/cpuinfo | cut -d : -f 2`
export INSTALL_MOD_PATH=/root/src/modules
export MODULE_LOC="${INSTALL_MOD_PATH}"
export UIMAGE_LOADADDR=0x8000
export KPKG_OVERLAY_DIR=/root/src/overlay
export DEBIAN_REVISION_MANDATORY=true
export INITRD=true

# make some adjustments as needed
make menuconfig

# Check version number desired
make kernelrelease

make-kpkg clean 
make dtbs
make uImage
make modules
make modules_install
make-kpkg --rootcmd fakeroot binary-arch kernel_source  > log_file 2>&1 & 
watch 'head -n 2 log_file; tail log_file'
```

### Check files included in package ###
```bash
dpkg-deb -c ../linux-image-4.8.7-kirkwood-iomega-ix2-200-arvati1_1_armel.deb | less
```

Installing the kernel
------

### Install new kernel package ###
transfer files to target:
```bash
scp  ../*.deb root@192.168.1.13:/root
```

Install deb packages:
```bash
ssh root@192.168.1.13
sudo dpkg -i linux-image-4.8.7-kirkwood-iomega-ix2-200-arvati1_1_armel.deb
sudo dpkg -i linux-headers-4.8.7-kirkwood-iomega-ix2-200-arvati1_1_armel.deb
```

Exit target and lxc container:
```bash
exit
exit
CTRL+A + q
sudo lxc-stop -n crossjessie
```

Copy result files to a tftp server:
Config tftp server dir /var/lib/tftpboot at cat /etc/default/tftpd-hpa

```bash
scp root@192.168.1.13:/boot/*-4.8.7-kirkwood-iomega-ix2-200-arvati1 /var/lib/tftpboot/
scp root@192.168.1.13:/usr/lib/linux-image-4.8.7-kirkwood-iomega-ix2-200-arvati1/* /var/lib/tftpboot/
```


### Test the kernel ###

Conect with a serial device to target
```bash
screen /dev/ttyUSB0 115200
```

Test it using a tFtp Server
```bash
setenv bootargs_console 'root=/dev/mapper/root-root'
setenv bootargs $(bootargs_console)
setenv serverip 192.168.1.31
setenv ipaddr 192.168.1.13
tftpboot 0x01100000 uInitrd-4.8.7-kirkwood-iomega-ix2-200-arvati1
tftpboot 0x00800000 uImage+dtb-4.8.7-kirkwood-iomega-ix2-200-arvati1
bootm 0x00800000 0x01100000
```
Modify root=/dev/mapper/root-root as yours needs !!!


### Install the new kernel on the device ###

```bash
cat /proc/mtd
dev:    size   erasesize  name
mtd0: 00100000 00004000 "uboot"
mtd1: 00020000 00004000 "env"
mtd2: 00300000 00004000 "uImage"
mtd3: 01c00000 00004000 "rootfs"

# Modify mtd2 or mtd3 with the corrected partition !!!! 

flash_eraseall /dev/mtd2
nandwrite -p /dev/mtd2 /boot/uImage+dtb-4.8.7-kirkwood-iomega-ix2-200-arvati1

flash_eraseall /dev/mtd3
nandwrite -p /dev/mtd3 /boot/uInitrd-4.8.7-kirkwood-iomega-ix2-200-arvati1

reboot
```

### Make changes in uboot ###
```bash
setenv mtdids 'nand0=orion_nand'
setenv mtdparts 'mtdparts=orion_nand:0x100000@0x000000(uboot)ro,0x20000@0xA0000(env),0x300000@0x100000(uImage),0x1C00000@0x400000(rootfs)'
setenv bootargs_console 'root=/dev/mapper/root-root'
setenv bootcmd 'setenv bootargs $(bootargs_console); nand read 0x800000 uImage; nand read 0x1100000 rootfs; bootm 0x00800000 0x01100000'
saveenv
reset
```


Links and sources
------
+ https://github.com/daniviga/ix2-200
+ http://blog.nobiscuit.com/2011/08/06/installing-debian-to-disk-on-an-ix2-200/
+ http://blog.nobiscuit.com/2012/02/15/kernel-patch-to-support-leds-buttons-and-sensors/
+ http://iomega.nas-central.org/wiki/Category:Ix2-200
+ https://wiki.debian.org/CrossToolchains
+ http://lacie-nas.org/doku.php?id=making_kernel_with_dtb
+ http://debian-br.alioth.debian.org/docs/nacionais/sgml/kdebian/kdebian.txt
+ http://forum.doozan.com/read.php?2,12096
+ https://wiki.debian.org/CrossToolchains
+ https://www.howtoforge.com/kernel_compilation_debian_etch#how-to-compile-a-kernel-debian-etch-
+ http://kernel-handbook.alioth.debian.org/ch-common-tasks.html#s-common-official
+ http://crunchbang.org/forums/viewtopic.php?id=37681
+ https://debian-handbook.info/browse/stable/sect.kernel-compilation.html
+ https://syslint.com/blog/tutorial/how-to-compile-linux-kernel-in-debian-ubuntu-the-easy-way/
+ http://www.arm.linux.org.uk/docs/kerncomp.php
+ https://help.ubuntu.com/community/Kernel/Compile#AltBuildMethod
+ http://forum.odroid.com/viewtopic.php?f=112&t=11241
+ https://github.com/umiddelb/armhf/wiki/How-To-compile-a-custom-Linux-kernel-for-your-ARM-device
+ https://lists.debian.org/debian-powerpc/2009/07/msg00042.html
