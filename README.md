# Linux kernel cross compile

Docker container to cross-compile linux kernel for let say raspi, or x86 system from Mac M1. But this README will only provide to compile the kernel for raspi zero (arm 32bit), as for raspi 3 and 4 (with arm 64bit), you can check directly on [geerlingguy/extras/cros-compile](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile), or [from raspi documentation](https://www.raspberrypi.com/documentation/computers/linux_kernel.html#cross-compiling-the-kernel).


## Compile kernel for Raspi Zero W
```shell
# here i clone the branch rpi-5.15.y with depth 1 since i just need the latest commit of this repo, so doesn't need to get all history
git clone --depth 1 https://github.com/raspberrypi/linux --branch rpi-5.15.y
cd linux

# create config for raspi 32bit
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcmrpi_defconfig

# compile the kernel modules from configuration above
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs

# create new directory to hold raspi partition
mkdir -p /mnt/pi-fat32
mkdir -p /mnt/pi-ext4

# and mount them with sshfs
sshfs root@homebridge.local:/ /mnt/pi-ext4
sshfs root@homebridge.local:/boot /mnt/pi-fat32

# copy new kernel to the boot partition
cp arch/arm/boot/zImage /mnt/pi-fat32/kernel7l.img
cp arch/arm/boot/dts/*.dtb /mnt/pi-fat32/
cp arch/arm/boot/dts/overlays/*.dtb* /mnt/pi-fat32/overlays/
cp arch/arm/boot/dts/overlays/README /mnt/pi-fat32/overlays/

# and install the new modules from this kernel version
env PATH=$PATH make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/mnt/pi-ext4 modules_install
```

If you want to change something in the kernel, maybe add, remove, or change modules used, you can configure it through GUI with menuconfig. Make  sure run this after `bcmrpi_defconfig` command
```shell
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
```

# Credits
Thanks to
- [geerlingguy](https://github.com/geerlingguy) for providing [this tutorial and docker files](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile)
