---
title: "Build a 64bit bootable image for a Raspberry Pi 4"
date: 2019-07-14T00:09:27+02:00
lastmod: 2019-10-17T19:38:21+0200
---

Given the [traction][0] our [previous post][1] got, we thought we should jot
down the steps to build a 64-bit bootable image for a RPi4. The distro we're
most familiar with is Debian, so we'll go with a debian-like distro like
Ubuntu. If you don't feel like playing with kernel compilation and FS images,
just [grab][6] the binary and dd it to an SD card!

First step, download the 64-bit ubuntu server distro for the RPi3:

[http://cdimage.ubuntu.com/ubuntu/releases/bionic/release/ubuntu-18.04.2-preinstalled-server-arm64+raspi3.img.xz][2]

Then make sure you follow the instructions from [these][3] [posts][4] which
help us build the kernel and update the boot firmware. The steps from these
posts are summarized below:

#### Build the toolchain ####

```
mkdir -p toolchains/aarch64
cd toolchains/aarch64
export TOOLCHAIN=`pwd` # Used later to reference the toolchain location

cd "$TOOLCHAIN"
wget https://ftp.gnu.org/gnu/binutils/binutils-2.32.tar.bz2
tar -xf binutils-2.32.tar.bz2
mkdir binutils-2.32-build
cd binutils-2.32-build
../binutils-2.32/configure --prefix="$TOOLCHAIN" --target=aarch64-linux-gnu --disable-nls
make -j4
make install
```

```
cd "$TOOLCHAIN"
wget https://ftp.gnu.org/gnu/gcc/gcc-9.1.0/gcc-9.1.0.tar.gz
tar -xf gcc-9.1.0.tar.gz
mkdir gcc-9.1.0-build
cd gcc-9.1.0-build
```

```
../gcc-9.1.0/configure --prefix="$TOOLCHAIN" --target=aarch64-linux-gnu --with-newlib --without-headers --disable-nls --disable-shared --disable-threads --disable-libssp --disable-decimal-float --disable-libquadmath --disable-libvtv --disable-libgomp --disable-libatomic --enable-languages=c
make all-gcc -j4
make install-gcc
```

#### Build the Raspberry Pi kernel ####

```
apt-get install bison flex
git clone https://github.com/raspberrypi/linux.git rpi-linux
cd rpi-linux
git checkout origin/rpi-4.19.y # change the branch name for newer versions
mkdir kernel-build
PATH=$PATH:$TOOLCHAIN/bin make O=./kernel-build/ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-  bcm2711_defconfig
PATH=$PATH:$TOOLCHAIN/bin make -j4 O=./kernel-build/ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
export KERNEL_VERSION=`cat ./kernel-build/include/generated/utsrelease.h | sed -e 's/.*"\(.*\)".*/\1/'` 
make -j4 O=./kernel-build/ DEPMOD=echo MODLIB=./kernel-install/lib/modules/${KERNEL_VERSION} INSTALL_FW_PATH=./kernel-install/lib/firmware modules_install
```

```
git clone https://github.com/raspberrypi/tools.git rpi-tools
cd rpi-tools/armstubs
git checkout 7f4a937e1bacbc111a22552169bc890b4bb26a94
PATH=$PATH:$TOOLCHAIN/bin make armstub8-gic.bin
```

```
echo "armstub=armstub8-gic.bin" >> config-extra.txt 
echo "enable_gic=1" >> config-extra.txt 
echo "arm_64bit=1" >> config-extra.txt
echo "total_mem=1024" >> config-extra.txt
```

We now have all the necessary files to create the boot partition and boot into
the ubuntu-preinstalled image. Specifically:

Kernel: `rpi-linux/kernel-build/arch/arm64/boot/Image`

Bootstub: `rpi-tools/armstubs/armstub8-gic.bin`

Modules: `rpi-linux/kernel-build/kernel-install/lib/modules/${KERNEL_VERSION}`

Firmware: `https://github.com/RPi-Distro/firmware-nonfree`


So, first thing to do after we have finished building the above is to
de-compress and loop mount the ubuntu-preinstalled image we downloaded:

```
xzcat ubuntu-18.04.2-preinstalled-server-arm64+raspi3.img.xz > ubuntu-18.04.2-preinstalled-server-arm64+raspi4.img
kpartx -av ubuntu-18.04.2-preinstalled-server-arm64+raspi4.img
```

You should end up with 2 device files:

```
/dev/mapper/loop0p1
/dev/mapper/loop0p2
```

Mount them under /mnt like this:

```
mount /dev/mapper/loop0p2 /mnt
mount /dev/mapper/loop0p1 /mnt/boot/firmware
```

Then copy in the kernel/stub, modules and firmware:

```
cp rpi-linux/kernel-build/arch/arm64/boot/Image /mnt/boot/firmware/kernel8.img
cp rpi-tools/armstubs/armstub8-gic.bin /mnt/boot/firmware/armstub8-gic.bin
cp -avf rpi-linux/kernel-build/kernel-install/lib/modules/${KERNEL_VERSION} /mnt/lib/modules/
git clone https://github.com/RPi-Distro/firmware-nonfree firmware-nonfree
cp -avf firmware-nonfree/* /mnt/lib/firmware
```

Append config-extra.txt to config.txt:

```
cat config-extra.txt >> /mnt/boot/firmware/config.txt
```

and we're done! Unmount / detach the loop device, dd it to an sdcard, plug it into a RPi4 and party!

```
umount /mnt/boot/firmware
umount /mnt
kpartx -dv ubuntu-18.04.2-preinstalled-server-arm64+raspi4.img
losetup -d /dev/loop0
dd if=ubuntu-18.04.2-preinstalled-server-arm64+raspi4.img of=/dev/sdXX
```

If you're too lazy to do the above, feel free to grab our image built using the above steps: 

[https://cloudkernels.net/ubuntu-18.04.2-preinstalled-server-arm64+raspi4+kvm.img.xz][6]

(sha1sum: 0b1d8b72ea5410fb7928925fd76dd0218b4f7a94)

**UPDATE**

Links to the previous images were removed, so here's the new ones:
[http://cdimage.ubuntu.com/ubuntu/releases/bionic/release/ubuntu-18.04.3-preinstalled-server-arm64+raspi3.img.xz][7]
[https://cloudkernels.net/ubuntu-18.04.3-preinstalled-server-arm64+raspi4+kvm.img.xz][8]



[0]: https://news.ycombinator.com/item?id=20410169
[1]: https://blog.cloudkernels.net/posts/rpi4-64bit-virt/
[2]: http://cdimage.ubuntu.com/ubuntu/releases/bionic/release/ubuntu-18.04.2-preinstalled-server-arm64+raspi3.img.xz
[3]: https://andrei.gherzan.ro/linux/raspbian-rpi-64/
[4]: https://andrei.gherzan.ro/linux/raspbian-rpi4-64/
[6]: https://cloudkernels.net/ubuntu-18.04.2-preinstalled-server-arm64+raspi4+kvm.img.xz
[7]: http://cdimage.ubuntu.com/ubuntu/releases/bionic/release/ubuntu-18.04.3-preinstalled-server-arm64+raspi3.img.xz
[8]: https://cloudkernels.net/ubuntu-18.04.3-preinstalled-server-arm64+raspi4+kvm.img.xz


