# Linux kernel cross compile

Docker container to cross-compile linux kernel for let say raspi, or x86 system from Mac M1. But this README will only provide to compile the kernel for RPi 4 (arm 32bit). For 64bit, you can check directly on [geerlingguy/extras/cros-compile](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile), or [from raspi documentation](https://www.raspberrypi.com/documentation/computers/linux_kernel.html#cross-compiling-the-kernel).

## Get into the container

1. Install Docker desktop (whcih includes Docker compose)
2. Clone the repo and run the container
   
   ```
   git clone https://github.com/jlian/linux-kernel-cross-compile.git
   cd linux-kernel-cross-compile
   
   docker-compose up -d
   ```
   
3. Log into the container 
   
   ```
   docker attach cross-compile
   ```

## Compile kernel for Raspberry Pi 4

Basically run this script while inside the container.

```shell
# Clone the branch rpi-5.15.y with depth 1 since i just need the latest commit of this repo, so doesn't need to get all history
git clone --depth 1 https://github.com/raspberrypi/linux --branch rpi-5.15.y
cd linux

# (Optional) apply a patch to do remote wake up over USB-OTG
wget https://raw.githubusercontent.com/pikvm/packages/master/packages/linux-rpi-pikvm/1003-remote-wakeup.patch
# patch -p1 --dry-run -i 1003-remote-wakeup.patch
patch -p1 -i 1003-remote-wakeup.patch

# Create config for Raspberry Pi 4 32bit
KERNEL=kernel7l
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig

# Compile the kernel modules from configuration above
make -j8 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs

# Create new directory to hold raspi partition
# And mount them with sshfs
# Optionally use the SD card method at:
# https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile#copying-built-kernel-via-mounted-usb-drive-or-microsd
mkdir -p /mnt/pi-fat32
mkdir -p /mnt/pi-ext4
sshfs root@homebridge.local:/ /mnt/pi-ext4
sshfs root@homebridge.local:/boot /mnt/pi-fat32

# Install the new modules from this kernel version
env PATH=$PATH make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/mnt/pi-ext4 modules_install

# Copy new kernel to the boot partition
cp arch/arm/boot/zImage /mnt/pi-fat32/kernel7l-remote-wake-up.img
cp arch/arm/boot/dts/*.dtb /mnt/pi-fat32/
cp arch/arm/boot/dts/overlays/*.dtb* /mnt/pi-fat32/overlays/
cp arch/arm/boot/dts/overlays/README /mnt/pi-fat32/overlays/

```

Edit `config.txt` to select our new custom kernel instead of the stock one

```
nano /mnt/pi-fat32/config.txt
```

And add 

```
kernel=kernel7l-remote-wake-up.img
```

Lastly, unmount

```
umount /mnt/pi-ext4
umount /mnt/pi-fat32
```

## Credits
Thanks to
- [geerlingguy](https://github.com/geerlingguy) for providing [this tutorial and docker files](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile)
