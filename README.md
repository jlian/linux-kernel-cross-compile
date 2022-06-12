# Linux kernel cross compile
Basically, this one is from [geerlingguy/cros-compile](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile), but i just take the docker files from there and create my script to compile for my raspi zero.


## Compile kernel for Raspi Zero W
```shell
# here i clone the branch rpi-5.15.y with depth 1 since i just need the latest commit of this repo, so doesn't need to get all history
git clone --depth 1 https://github.com/raspberrypi/linux --branch rpi-5.15.y
cd linux

# create config for raspi 32bit
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcmrpi_defconfig

# if you want to change something in the kernel, or maybe want to add, you can configure with GUI
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig

# compile the kernel modules from configuration above
make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs

# create new directory to hold raspi partition
mkdir -p /mnt/pi-fat32
mkdir -p /mnt/pi-ext4

# and mount them with sshfs
sshfs root@raspberrypi.local:/ /mnt/pi-ext4
sshfs root@raspberrypi.local:/boot /mnt/pi-fat32

# copy new kernel to the boot partition
cp arch/arm/boot/zImage /mnt/pi-fat32/kernel7l.img
cp arch/arm/boot/dts/*.dtb /mnt/pi-fat32/
cp arch/arm/boot/dts/overlays/*.dtb* /mnt/pi-fat32/overlays/
cp arch/arm/boot/dts/overlays/README /mnt/pi-fat32/overlays/

# and install the new modules from this kernel version
env PATH=$PATH make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/mnt/pi-ext4 modules_install
```
