
# Kali NetHunter Porting Guide for Samsung Galaxy S4 (jflte) on LineageOS 16

## Overview

Kali NetHunter isn’t a full Android ROM—it’s an overlay that adds penetration testing tools to an existing system. It relies on a custom kernel (with specific patches) to support advanced features like packet injection and HID (Human Interface Device) functionalities. This guide explains how to build a custom kernel for the Samsung Galaxy S4 (jflte) running LineageOS 16 and then integrate it into the Kali NetHunter installer.

## Prerequisites

- **Host Machine:** Ubuntu 18.04 LTS (or similar) running on VirtualBox with a decent setup (e.g., 24GB RAM, 6 vCPUs, SSD storage, network connection)
- **Android Device:** Samsung Galaxy S4 (GT-I9505 / jflte)
- **Basic Knowledge:** Linux command line and cross-compiling
- **Tools:** Git, a cross-compiler for ARM, and patched source files

---

## 1. Building the Custom Kernel

### 1.1 Understand the Role of Kali NetHunter
- **Not a ROM:** Kali NetHunter adds pentesting features on top of an existing Android system.
- **Kernel Modifications:** It requires a custom kernel with patches for packet injection and HID support.

### 1.2 Obtain the Kernel Source
Clone the LineageOS 16 kernel repository for the Samsung Galaxy S4 (jflte):

```bash
git clone -b lineage-16.0 https://github.com/LineageOS/android_kernel_samsung_jf.git

```

### 1.3 Set Up the Compiler/Toolchain

-   **ARM Architecture:** The Galaxy S4 uses a 32-bit ARM CPU, so use a cross-compiler.
    
-   **Download the Google ARM Toolchain:**
    
    ```bash
    git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.8 toolchain
    
    ```
    
-   **Set up Environment Variables:**
    
    ```bash
    export ARCH=arm
    export SUBARCH=arm
    export CROSS_COMPILE=$(pwd)/toolchain/bin/arm-eabi-
    
    ```
    

----------

## 2. Patching the Kernel Source

### 2.1 Apply the Injection Patch

Enable mac80211 injection for packet injection (based on an aircrack-ng patch):

```bash
cd android_kernel_samsung_jf
patch -p1 < ../patch/injection/mac80211.compat13102019.wl_frag+ack_v1.patch

```

### 2.2 Apply the HID Patch

Add HID functionality (using code from the Android Keyboard Gadget):

```bash
cd android_kernel_samsung_jf
patch -p1 < ../patch/hid/hid.patch

```

----------

## 3. Configuring the Kernel

Before compiling, create a configuration file (.config) tailored for your device.

### 3.1 Manual Configuration (Hard Way)

```bash
cd android_kernel_samsung_jf
mkdir -p ../kernel
make O=../kernel clean

```

Start the menu configuration with:

```bash
make O=../kernel ARCH=arm VARIANT_DEFCONFIG=jf_eur_defconfig lineage_jf_defconfig menuconfig

```

After making your changes in the menuconfig interface, save your configuration as **.config**.

### 3.2 Pre-Configured Option (Easy Way)

Copy the provided configuration file and build:

```bash
cd android_kernel_samsung_jf
cp ../patch/defconfig/nethunter_jf_defconfig ./arch/arm/configs/
mkdir -p ../kernel
make O=../kernel clean
make O=../kernel ARCH=arm VARIANT_DEFCONFIG=jf_eur_defconfig nethunter_jf_defconfig

```

----------

## 4. Building the Kernel

### 4.1 Apply Additional Patches

#### Bluetooth Patch

Fix USB Bluetooth module compile errors:

```bash
cd android_kernel_samsung_jf
patch -p1 < ../patch/bluetooth/bluetooth_compile_fix.patch

```

#### Warning Suppression

Suppress warnings for implicit function declarations by editing the Makefile:

```bash
cd android_kernel_samsung_jf
nano Makefile

```

Comment out or remove the line:

```
-Werror-implicit-function-declaration \

```

Alternatively, apply the provided patch:

```bash
patch -p1 < ../patch/makefile/makefile_warning.patch

```

#### Kernel Name Adjustment

Commit all changes to remove the “-dirty” tag from the kernel name:

```bash
git add -A
git commit -m "NetHunter Kernel"

```

### 4.2 Compile the Kernel and Modules

Compile the kernel using all available threads:

```bash
cd android_kernel_samsung_jf
make O=../kernel ARCH=arm -j$(nproc --all) LOCALVERSION="-NetHunter"

```

The kernel image will be located at: `../kernel/arch/arm/boot/zImage`.

Compile and install modules:

```bash
cd android_kernel_samsung_jf
make O=../kernel INSTALL_MOD_PATH="." INSTALL_MOD_STRIP=1 modules_install

```

Clean up unnecessary symbolic links:

```bash
rm ../kernel/lib/modules/$(ls ../kernel/lib/modules)/build
rm ../kernel/lib/modules/$(ls ../kernel/lib/modules)/source

```

----------

## 5. Creating the Kali NetHunter Installer

Kali NetHunter is flashed as a zip file via custom recovery.

### 5.1 Download and Bootstrap the NetHunter Project

Clone the NetHunter project (using release 2019.4 as an example):

```bash
git clone -b 2019.4 https://gitlab.com/kalilinux/nethunter/build-scripts/kali-nethunter-project.git
cd kali-nethunter-project/nethunter-installer
./bootstrap.sh

```

During bootstrapping, answer “No” to all prompts.

### 5.2 Add the New/Unsupported Device

Edit the devices configuration in `devices.cfg` to add the Galaxy S4 for LineageOS 16:

```
# Galaxy S4 for LineageOS 16
[jfltexx-los]
author = "Fabio Cogno"
version = "1.0"
kernelstring = "NetHunter kernel for jflte"
devicenames = jflte GT-I9505 i9505 jfltexx

```

Apply the configuration patch:

```bash
cd kali-nethunter-project/nethunter-installer/devices
patch -p1 < ../../../patch/devices/devices.cfg.patch

```

### 5.3 Add the Custom Kernel

Copy the kernel image and modules to the correct directory (for Android 9 Pie):

```bash
cd kali-nethunter-project/nethunter-installer/devices
mkdir -p pie/jfltexx-los
cp ../../../kernel/arch/arm/boot/zImage pie/jfltexx-los/
cp -r ../../../kernel/lib/modules/ pie/jfltexx-los/

```

### 5.4 Build the Installer Package

Verify that your device is listed:

```bash
cd kali-nethunter-project/nethunter-installer/
python build.py -h

```

Create a test installer (kernel-only):

```bash
python build.py -d jfltexx-los --pie -k

```

This will generate a flashable zip (e.g., `kernel-nethunter-jfltexx-los-pie-AAAAMMDD_hhmmss.zip`).

(Optional) Create the full installer (includes the NetHunter overlay):

```bash
python build.py -d jfltexx-los --pie --rootfs full

```

----------

## 6. Installing Kali NetHunter on Your Device

### 6.1 Gather Required Files

-   **LineageOS 16 Installer:** Download from [lineageos.org](https://lineageos.org/)
-   **Magisk Installer:** Download from [magiskmanager.com](https://magiskmanager.com/)
-   **(Optional) Pico GApps for ARM Android 9:** Download from [opengapps.org](https://opengapps.org/)
-   **Your Custom Kali NetHunter Installer Zip**

### 6.2 Flashing Process

1.  **Prepare an External SD Card:**  
    Copy the LineageOS installer, Magisk installer, GApps, and your Kali NetHunter zip to the SD card.
2.  **Install a Custom Recovery:**  
    Flash the latest TWRP using Odin or Heimdall.
3.  **Boot into Recovery Mode:**  
    With the device off, press and hold **Home + Volume Up + Power** to enter TWRP.
4.  **Wipe Data:**  
    In TWRP, wipe the following:
    -   Dalvik/ART Cache
    -   System
    -   Data (all user data will be lost)
    -   Internal Storage (if needed, may require reinstallation from SD card)
    -   Cache
5.  **Flash in Order:**
    -   First, flash the LineageOS 16 installer.
    -   Then, flash GApps.
    -   Finally, flash the Magisk installer.
6.  **Finalize:**  
    Boot into the system (ensure Wi-Fi is connected), then reboot into recovery, wipe the cache, and flash the Kali NetHunter installer zip.

----------

## Conclusion

This guide covers the steps to port Kali NetHunter onto a Samsung Galaxy S4 running LineageOS 16. You learned how to:

-   Set up the build environment and download the correct kernel sources.
-   Configure cross-compilation with a Google ARM toolchain.
-   Apply patches for injection and HID functionalities.
-   Configure, compile, and install a custom kernel and its modules.
-   Integrate the kernel into the Kali NetHunter installer and flash it via custom recovery.

Follow these steps to successfully run Kali NetHunter with the required kernel modifications on your device
```
