# Embedded Linux From Scratch for Raspberry Pi 2

This repository documents my journey of building a custom Linux distribution for the Raspberry Pi 2 Model B from the ground up. The project starts from a pre-built OS, moving to building a custom OS with the Yocto Project, and culminating in manually compiling and porting a newer Linux Kernel.

This project serves as a hands-on learning experience and a portfolio piece demonstrating core skills in Embedded Linux engineering.

## Key Skills Demonstrated
*   **Custom Linux Build System:** Using the **Yocto Project (Kirkstone)** to build a minimal, functional Linux image.
*   **Cross-Compilation:** Setting up and using a cross-compiler toolchain (`arm-linux-gnueabihf-`) to build software for the ARM target on an x86_64 host.
*   **Low-Level Storage Management:** Manually partitioning and formatting an SD card using `fdisk` and `mkfs` instead of relying on automated tools.
*   **Kernel Development:** Manually compiling the Linux Kernel, installing modules, and deploying it to the target hardware.
*   **Kernel Porting:** Successfully porting the system from Linux Kernel `5.15` to `6.1`.
*   **Device Tree:** Understanding, reading, and modifying Device Tree Source (`.dts`) files to control hardware peripherals (e.g., disabling the ACT LED).
*   **Embedded Debugging:** Proficient use of the **Serial Console (UART)** with Tera Term/PuTTY as the primary channel for debugging the boot process.
*   **System Analysis:** Analyzing kernel drivers (`printk`, `dmesg`) and understanding the link between hardware description (Device Tree) and driver code.

---

## Final Result
The project successfully produced a bootable, custom Linux system on the Raspberry Pi 2 with the following specifications:
*   **Custom Yocto Image:** `core-image-base` with SSH server and `opkg` package manager.
*   **Manually Ported Kernel:** Successfully running **Linux Kernel 6.1.93-v7+**.
*   **Minimal Footprint:** The initial minimal rootfs occupied only **~11MB**, demonstrating the power of custom builds.
*   **Full Manual Deployment:** The final system was deployed by manually partitioning the SD card and deploying the rootfs, kernel, and bootloader files.


---

## Hardware Used
*   **Board:** Raspberry Pi 2 Model B
*   **Debugging:** USB to TTL Serial Adapter (CP2102)
*   **Storage:** 32GB MicroSD Card
*   **Host Machine:** PC running Ubuntu 24.04.03 LTS.

---

## How to Reproduce This Project

This guide outlines the steps to build and deploy the `core-image-base` with a manually compiled kernel, just as accomplished in this project.

### Part 1: Setting Up the Yocto Build Environment

**1. Install Dependencies on Ubuntu:**
```bash
sudo apt update
sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libsdl1.2-dev python3-subunit mesa-common-dev zstd liblz4-tool file locales
sudo locale-gen en_US.UTF-8
```

**2. Clone Yocto and Layers (Kirkstone release):**
```bash
mkdir ~/yocto && cd ~/yocto
git clone -b kirkstone git://git.yoctoproject.org/poky.git
git clone -b kirkstone git://git.openembedded.org/meta-openembedded
git clone -b kirkstone https://github.com/agherzan/meta-raspberrypi.git
```

**3. Configure the Build:**
*   Initialize the build environment:
    ```bash
    cd poky
    source oe-init-build-env
    ```
*   Add the necessary layers to `conf/bblayers.conf`. Ensure it looks like this (replace `your_user`):
    ```makefile
    BBLAYERS ?= " \
      /home/your_user/yocto/poky/meta \
      /home/your_user/yocto/poky/meta-poky \
      /home/your_user/yocto/poky/meta-yocto-bsp \
      /home/your_user/yocto/meta-openembedded/meta-oe \
      /home/your_user/yocto/meta-openembedded/meta-python \
      /home/your_user/yocto/meta-raspberrypi \
      "
    ```
*   Add machine and feature configurations to the end of `conf/local.conf`:
    ```makefile
    # Target Machine
    MACHINE ?= "raspberrypi2"

    # Enable UART for serial debugging
    ENABLE_UART = "1"

    # Add SSH server and package management features
    EXTRA_IMAGE_FEATURES += "ssh-server-openssh package-management"
    
    # Specify the .ipk package format for opkg
    PACKAGE_CLASSES = "package_ipk"
    ```

**4. Build the `core-image-base` Rootfs:**
```bash
# On newer Ubuntu versions, you may need to enable user namespaces
# sudo sysctl -w kernel.unprivileged_userns_clone=1

# Start the build
bitbake core-image-base
```
The resulting rootfs tarball will be located at `tmp/deploy/images/raspberrypi2/core-image-base-raspberrypi2.tar.bz2`.

### Part 2: Manually Compiling the Linux Kernel (v6.1)

**1. Install Dependencies for Kernel Compilation:**
```bash
sudo apt update
sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev libgmp-dev libmpc-dev libmpfr-dev
```

**2. Install the Cross-Compiler Toolchain:**
```bash
sudo apt install crossbuild-essential-armhf
```

**3. Clone and Compile the Kernel:**
```bash
cd ~
git clone https://github.com/raspberrypi/linux.git linux-6.1
cd linux-6.1
git checkout rpi-6.1.y

# Set up the environment for cross-compilation
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-

# Load the default configuration for RPi 2
make bcm2709_defconfig

# Compile the Kernel, modules, and device trees
make -j$(nproc) zImage modules dtbs

#The compiled kernel (`zImage`) and device tree (`.dtb`) will be in `arch/arm/boot/` and `arch/arm/boot/dts/` respectively.
```
### Part 3: Manually Partitioning and Deploying to SD Card

**WARNING: This process can wipe your drives. Be absolutely sure you are targeting the correct device (e.g., `/dev/sdb`).**

**1. Partition the SD Card:**
*   Identify your SD card with `lsblk`.
*   Use `sudo fdisk /dev/sdX` to create two primary partitions:
    1.  **Partition 1 (BOOT):** `100M`, Type `c` (W95 FAT32 LBA).
    2.  **Partition 2 (ROOTFS):** Use the remaining space, Type `83` (Linux).

**2. Format the Partitions:**
```bash
sudo mkfs.vfat -n BOOT /dev/sdX1
sudo mkfs.ext4 -L rootfs /dev/sdX2
```

**3. Deploy the Files:**
*   Mount the partitions:
    ```bash
    mkdir -p ~/boot_mount ~/rootfs_mount
    sudo mount /dev/sdX1 ~/boot_mount
    sudo mount /dev/sdX2 ~/rootfs_mount
    ```
*   **Deploy Rootfs:** Extract the Yocto-built rootfs tarball to the `rootfs` partition.
    ```bash
    sudo tar -xvpf ~/yocto/poky/build/tmp/deploy/images/raspberrypi2/core-image-base-raspberrypi2.tar.bz2 -C ~/rootfs_mount/
    ```
*   **Deploy Kernel Modules:** From within your `~/linux-6.1` source directory:
    ```bash
    # Make sure ARCH and CROSS_COMPILE are set
    sudo make modules_install INSTALL_MOD_PATH=~/rootfs_mount
    ```
*   **Deploy Boot Files:** Copy the bootloader/firmware files (found in Yocto's deploy directory), the kernel, and the device tree.
    ```bash
    # From Yocto deploy dir
    sudo cp bootcode.bin start.elf fixup.dat ~/boot_mount/

    # From your linux-6.1 source dir
    sudo cp arch/arm/boot/zImage ~/boot_mount/kernel7.img
    sudo cp arch/arm/boot/dts/bcm2709-rpi-2-b.dtb ~/boot_mount/
    ```
*   **Create `cmdline.txt`:** This file tells the kernel where to find the root filesystem.
    ```bash
    echo "dwc_otg.lpm_enable=0 console=ttyAMA0,115200 root=/dev/mmcblk0p2 rootwait" | sudo tee ~/boot_mount/cmdline.txt
    ```
*   Unmount the partitions:
    ```bash
    sudo umount ~/boot_mount ~/rootfs_mount
    ```

**4. Boot and Connect:**
Insert the SD card into the Raspberry Pi 2, connect your USB-to-Serial adapter, and power it on. Use Tera Term or PuTTY to connect to the serial console (115200 baud). Login as `root` with no password.

Enjoy your custom-built Linux system!
