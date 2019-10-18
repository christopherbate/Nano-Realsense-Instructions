# Working with the Intel Realsense 435i on the Nvidia Jetson Nano

This document briefly outlines how to patch the Nvidia Tegra Kernel in order to make the Intel Realsense 435i camera work. The method used 
is to cross-compile the linux kernel on an x86 (Linux) host and then copy the kernel to device.

For L4T Development please reference's NVidia's official guide (here)[https://docs.nvidia.com/jetson/l4t/index.html#page/Tegra%2520Linux%2520Driver%2520Package%2520Development%2520Guide%2Fquick_start.html%23].

Prereq (might not be comprehensive, if you get an error indicating you don't have a required package, please submit an issue) listed below. 
Note that you need your Linux computer setup for cross compiling for ARM64. Either download Nvidia's official toolchain (listed above), or 
use the toolchain provided by the Ubuntu aarch64 package. (I did the latter.)

```
buildessential
git
gcc-aarch64-linux-gnu-gcc
libncurses5 [for menuconfig]
```

0. Backup your work.


## Install L4T Development Kit

1. Go to https://developer.nvidia.com/embedded/linux-tegra
2. Download the files under Jetson Nano called "Sample Root Filesystem" and "BSP"

(The following instructions are now reproduced from the L4T Dev Guide )

```
sudo tar xpf Jetson-Nano-Tegra210_Linux_R32.1.0_aarch64.tbz2
cd Linux_for_Tegra/rootfs/
sudo tar xpf ../../Jetson-Nano-Tegra_Linux_Sample-Root-Filesystem_R32.1.0_aarch64.tbz2
cd ..
sudo ./apply_binaries.sh
./source_sync.sh
```

At this point the sources will sync for the kernel, bootloader, etc. When the prompt asks you for a "tag" enter the following: `tegra-l4t-r32.1`.

Note that after you apply "source_sync" above, the kernel source will now be in `Linux_for_Tegra/sources/kernel/kernel/kernel-4.9`. I'll just
refer to this as the "kernel source directory" or `$KERNEL_SRC_DIR`. 

## Build Kernel


1. Make a new directory on your hdd where you want all the build files to be stored. I used `/mnt/tegra` for simplicity in typing. 
2. `cd $KERNEL_SRC_DIR`
3.`sudo make ARCH=arm64 O=/mnt/tegra tegra_defconfig` produces an initial config in `/mnt/tegra`. You can also do the following on the Nano device: `zcat /proc/config.gz > .config`. Then SCP  that .config from the nano to your host and place in `/mnt/tegra`
4. Now either manually go into the `.config` and enable:

```
CONFIG_HID_SENSOR_GYRO_3D=m
CONFIG_HID_SENSOR_ACCEL_3D=m
CONFIG_IIO=y
CONFIG_IIO_BUFFER=y
CONFIG_IIO_BUFFER_CB=m
CONFIG_IIO_TRIGGERED_BUFFER=y
CONFIG_HID_SENSOR_ACCEL_3D=m
CONFIG_USB_VIDEO_CLASS=m
CONFIG_VIDEO_DEV=y
CONFIG_VIDEO_V4L2_SUBDEV_API=y
CONFIG_VIDEO_V4L2=y
CONFIG_EXTRA_FIRMWARE="tegra21x_xusb_firmware"
CONFIG_EXTRA_FIRMWARE_DIR="firmware"
CONFIG_MODULE_SIG=n
```

Or just copy the `.config` file from this repo into `O=/mnt/tegra`, overwriting the previous.

5. Note that the last two are for enabling USB RootFS if wanted later. Also need to copy "tegra21x_xusb_firmware" from `Linux_for_Tegra/rootfs/lib/firmware` to `/mnt/tegra/firmware`.

5. Apply patches from this repo: `patch -p1 < intel_suggested_patches.patch`

6. (still in the kernel source directory) `sudo make ARCH=arm64 O=/mnt/tegra silentoldconfig`

7. `sudo make ARCH=arm64 O=/mnt/tegra CROSS_COMPILE=aarch64-linux-gnu- -j11` Note to change the "11" after the j to the number of cores you have on your system minus one.

8. Let the compile go, this will take a while.

9. Get your L4T Nano SD card that you created before and mount it onto your system somehwere. You want the rootfs  to be accessible. Let's assume the mount point is `/mnt/SDCARD/`.

10. Backup the original kernel Image: `sudo mv /mnt/SDCARD/boot/Image /mnt/SDCARD/boot/Image.bk`

11. Copy kernel modules. From the kernel source directory: `sudo make ARCH=arm64 O=/mnt/tegra modules_install INSTALL_MOD_PATH=/mnt/SDCARD`

11. Copy new kernel `sudo cp /mnt/tegra/arch/arm64/boot/Image /mnt/SDCARD/boot/Image`

12. Eject and use SD card to boot new system.

Note if any issues, you might want to try flashing the system directly rather than copying to SD card. See L4T guide for more details.
