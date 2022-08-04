# Linux kernel cross compile on Apple Silicon

Docker container to cross-compile linux kernel from Apple Silicon. This README has instructions to compile the kernel for Raspberry Pi 4 arm 32bit. Adapted from https://akhmad.id/compile-linux-kernel-on-mac-m1/. For 64bit, you can check directly on https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile, or [from raspi documentation](https://www.raspberrypi.com/documentation/computers/linux_kernel.html#cross-compiling-the-kernel).

## Get into the container

1. Install Docker desktop (which includes Docker compose)
2. Clone the repo and run the container
   
   ```bash
   git clone https://github.com/jlian/linux-kernel-cross-compile.git
   cd linux-kernel-cross-compile
   
   docker-compose up -d
   ```
   
3. Log into the container 
   
   ```bash
   docker attach cross-compile
   ```

## Compile kernel for Raspberry Pi 4

Run these inside the container unless otherwise noted.

1. Clone the branch rpi-5.15.y with depth 1 since i just need the latest commit of this repo, so doesn't need to get all history

   ```bash
   git clone --depth 1 https://github.com/raspberrypi/linux --branch rpi-5.15.y
   cd linux
   ```

1. (Optional) apply a [patch](https://github.com/raspberrypi/linux/issues/3977#issuecomment-1200368214) to do remote wake up over USB-OTG

   ```bash
   wget https://raw.githubusercontent.com/pikvm/packages/master/packages/linux-rpi-pikvm/1003-remote-wakeup.patch
   # patch -p1 --dry-run -i 1003-remote-wakeup.patch
   patch -p1 -i 1003-remote-wakeup.patch
   ```

1. Create config for Raspberry Pi 4 32bit

   ```bash
   KERNEL=kernel7l
   make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
   ```

1. Compile the kernel modules from configuration above

   ```bash
   make -j8 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs
   ```

1. Create key inside container because mounting Pi volumes locally requires key authentication

   ```bash
   ssh-keygen -t ed25519 -C "build-container"
   ```
1. Permit root login over ssh for the Pi by following either the Ansible or manual steps [here](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile#copying-built-kernel-via-remote-sshfs-filesystem).

1. Add key to Pi (run as root on Pi). Replace `homebridge.local` with your Pi's local hostname or IP.

   ```bash
   ssh pi@homebridge.local
   sudo su
   mkdir -p /root/.ssh && echo 'KEY_HERE' >> /root/.ssh/authorized_keys
   exit
   ```

1. Create new directory to hold raspi partition and mount them with `sshfs`. Alternatively use the [SD card method](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile#copying-built-kernel-via-mounted-usb-drive-or-microsd) but I couldn't get that to work.

   ```bash
   mkdir -p /mnt/pi-fat32
   mkdir -p /mnt/pi-ext4
   sshfs root@homebridge.local:/ /mnt/pi-ext4
   sshfs root@homebridge.local:/boot /mnt/pi-fat32
   ```

1. Install the new modules from this kernel version

   ```bash
   env PATH=$PATH make -j4 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=/mnt/pi-ext4 modules_install
   ```

1. Copy new kernel to the boot partition

   ```bash
   cp arch/arm/boot/zImage /mnt/pi-fat32/kernel7l-remote-wake-up.img
   cp arch/arm/boot/dts/*.dtb /mnt/pi-fat32/
   cp arch/arm/boot/dts/overlays/*.dtb* /mnt/pi-fat32/overlays/
   cp arch/arm/boot/dts/overlays/README /mnt/pi-fat32/overlays/
   ```

1. Edit `config.txt` from Raspberry Pi's `/boot/` directory

   ```bash
   nano /mnt/pi-fat32/config.txt
   ```

1. Add pointer to the custom kernel we just created

   ```bash
   kernel=kernel7l-remote-wake-up.img
   ```

1. Lastly, unmount

   ```bash
   umount /mnt/pi-ext4
   umount /mnt/pi-fat32
   ```
   
## Compilation speed

The command `make -j8 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs` took 10 minutes to complete on my MacBook Air M2 at 50% CPU usage according to Activity Monitor. [Not too bad](https://www.jeffgeerling.com/blog/2021/apple-m1-compiles-linux-30-faster-my-intel-i9) considering it runs inside a container inside a VM. Theoretically, since the M2 MBA has 8 cores, we can set to `-j12` (8 cores * 1.5) to make it (33%?) faster, but I haven't tried that.

## Credits

Thanks to
- [geerlingguy](https://github.com/geerlingguy) for providing [this tutorial and docker files](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile)
- [rockavoldy](https://github.com/rockavoldy) for the guide for the 32 bit version https://akhmad.id/compile-linux-kernel-on-mac-m1/
