# Raspberry Pi Linux Cross-compilation Environment

This environment can be used to [cross-compile the Raspberry Pi OS kernel](https://www.raspberrypi.org/documentation/linux/kernel/building.md) from a Linux, Windows, or Mac workstation using Docker.

This build configuration has only been tested with the Raspberry Pi 4, CM4, and Pi 400, and run on macOS.

Modified from [Jeff Geerling's cross-compile environment](https://github.com/geerlingguy/raspberry-pi-pcie-devices/tree/master/extras/cross-compile)

## Bringing up the build environment

  1. Install Podman (and podman-compose).
  2. Bring up the cross-compile environment:

     ```
     podman-compose up -d
     ```

  3. Log into the running container:

     ```
     podman attach cross-compile
     ```

You will be dropped into a shell inside the container's `/build` directory. From here you can work on compiling the kernel.

> [!NOTE]
> After you `exit` out of that shell, the container will stop, but will not be removed. If you want to jump back into it, you can run `podman start -a cross-compile`.

## Compiling the Kernel

  1. Clone the linux repo (or clone a fork or a different branch):

     ```
     git clone --depth=1 https://github.com/raspberrypi/linux
     ```

  2. Run the following commands to make the .config file:

     ```
     cd linux
     make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
     ```

  3. (Optionally) Either edit the .config file by hand or use menuconfig:

     ```
     make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
     ```

  4. Compile the Kernel:

     ```
     make -j8 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
     ```

> [!NOTE]
> For 32-bit Pi OS, use `ARCH=arm`, `CROSS_COMPILE=arm-linux-gnueabihf-`, and `zImage` instead of `Image`.

> [!TIP]
> I set the jobs argument (`-j8`) based on a bit of benchmarking. For different types of processors you may want to use more (or fewer) jobs depending on architecture and how many cores you have.

## Editing the kernel source inside /build/linux

  - mount container file system to host (prints mount location):

    ```
    podman mount cross-compile
    ```

> [!NOTE]
> if running podman in rootless mode, run `podman unshare` first. 

### Install kernel modules and DTBs via SSHFS

The best option is to use the Automated script. Within the container, run the command:

```
PI_ADDRESS=10.0.100.170 copykernel
```

> [!NOTE]
> Change the `PI_ADDRESS` here to the IP address of the Pi you're managing.

The script will reboot Pi, and once rebooted, your new kernel should be in place!

#### Manual kernel module install via SSHFS

Or, if you want to run the process manually, run:

```
mkdir -p /mnt/pi-ext4
mkdir -p /mnt/pi-fat32
sshfs root@10.0.100.119:/ /mnt/pi-ext4
sshfs root@10.0.100.119:/boot /mnt/pi-fat32
```

Install all the kernel modules:

```
env PATH=$PATH make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=/mnt/pi-ext4 modules_install
```

Copy the kernel and DTBs onto the drive:

```
cp arch/arm64/boot/Image /mnt/pi-fat32/kernel8.img
cp arch/arm64/boot/dts/broadcom/*.dtb /mnt/pi-fat32/
cp arch/arm64/boot/dts/overlays/*.dtb* /mnt/pi-fat32/overlays/
cp arch/arm64/boot/dts/overlays/README /mnt/pi-fat32/overlays/
```

Unmount the filesystems:

```
umount /mnt/pi-ext4
umount /mnt/pi-fat32
```

Reboot the Pi and _voila!_, you're done!

> [!NOTE]
> For 32-bit Pi OS, use `ARCH=arm`, `CROSS_COMPILE=arm-linux-gnueabihf-`, `zImage` instead of `Image`, `kernel7l` instead of `kernel8`, and `arm` instead of `arm64`.

### Hard Reset on the CM4 IO Board

If you get a fatal kernel panic that doesn't get caught cleanly (which I do, quite often, when debugging PCI Express devices), you can quickly reset the CM4 IO Board by jumping pins 12 and 14 on J2 (`GLOBAL_EN` to `GND`).

You can also 'boot' a shutdown CM4 by jumping pins 13 and 14 on J2 (`GLOBAL_EN` to `RUN_PG`).

### Kernel Headers

If you also need the kernel headers and source available, you can do the following (before unmounting the filesystems):

```
# Get the kernel version (usually the last in the listing):
sudo ls -lah /mnt/pi-ext4/lib/modules

# Create a directory and copy the sources into it:
sudo mkdir /mnt/pi-ext4/usr/src/linux-headers-5.10.14-v8+
sudo rsync -avz --exclude .git /home/vagrant/linux/ root@10.0.100.119:/usr/src/linux-headers-5.10.14-v8+

# Update the symlinks in the modules directory on the Pi:
sudo rm -rf /mnt/pi-ext4/lib/modules/5.10.14-v8+/build
sudo rm -rf /mnt/pi-ext4/lib/modules/5.10.14-v8+/source
sudo ln -s /usr/src/linux-headers-5.10.14-v8+ /mnt/pi-ext4/lib/modules/5.10.14-v8+/build
sudo ln -s /usr/src/linux-headers-5.10.14-v8+ /mnt/pi-ext4/lib/modules/5.10.14-v8+/source
```

This is sometimes necessary if you're compiling modules on the Pi that require kernel headers.

> [!NOTE]
> This... doesn't work. I opened this issue to see if I could figure out the way: [Getting headers for custom cross-compiled kernel?](https://www.raspberrypi.org/forums/viewtopic.php?f=66&t=303289).

## Copying built Kernel via mounted USB drive or microSD

The other option is to shut down the Pi, pull it's card or USB boot drive, and connect it to your computer so it can attach to the VM.

Mount the FAT and ext4 partitions of the USB card to the system. First, insert your microSD card into the reader you attached to the VM earlier, then run the following commands:

```
mkdir -p mnt/fat32
mkdir -p mnt/ext4
sudo mount /dev/sdb1 mnt/fat32
sudo mount /dev/sdb2 mnt/ext4
```

Copy the kernel and DTBs onto the drive:

```
sudo cp arch/arm64/boot/Image mnt/fat32/kernel8.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/fat32/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
sudo cp arch/arm64/boot/dts/overlays/README mnt/fat32/overlays/
```

### Installing modules and copying the built Kernel

Install the kernel modules onto the drive:

```
sudo env PATH=$PATH make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=mnt/ext4 modules_install
```

> [!NOTE]
> For 32-bit Pi OS, use `sudo env PATH=$PATH make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=mnt/ext4 modules_install`

Copy the kernel and DTBs onto the drive:

```
sudo cp arch/arm64/boot/Image mnt/fat32/kernel8.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/fat32/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
sudo cp arch/arm64/boot/dts/overlays/README mnt/fat32/overlays/
```

> [!NOTE]
> For 32-bit Pi OS, use `kernel7l` instead of `kernel8`.

### Unmounting the drive

Unmount the disk before you remove it from the card reader or unplug it.

```
sudo umount mnt/fat32
sudo umount mnt/ext4
```

## Resetting the build environment

If you want to reset things without wiping out the VM or re-cloning the source, run:

```
git reset --hard
git clean -f -d -X
```
