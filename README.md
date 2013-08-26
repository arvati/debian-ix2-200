Debian Linux on ix2-200
=======

Configs and tools for installing vanilla Debian Linux on iomega ix2-200 NAS

Setup cross compiler
------

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
Patch the sources
------
```bash
cd /usr/src/
tar xjvf linux-sources-3.2.tar
wget config-3.2.46-ix2_200
wget 0001-Modified-rd88f6281-setup-file-to-support-ix2-200.patch
```
```bash
cd linux-sources-3.2
patch -p1 < 0001-Modified-rd88f6281-setup-file-to-support-ix2-200.patch
```
Configure the kernel
------
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
Build the kernel
------
```bash
fakeroot make-kpkg -j2 --arch arm --cross-compile arm-linux-gnueabi- --append-to-version=-ix2-200 --initrd kernel_image kernel_headers
```

Install the new kernel on the device
------
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

### Links and sources
+ http://blog.nobiscuit.com/2011/08/06/installing-debian-to-disk-on-an-ix2-200/
+ http://blog.nobiscuit.com/2012/02/15/kernel-patch-to-support-leds-buttons-and-sensors/
+ http://copyninja.info/2013/01/multi-arch_based_cross-compilation.html
