---
layout: post
title: "Building Android AOSP for board hikey960"
date:	Sat Apr 06 19:30:54 EEST 2019
---

**Building AOSP**

This instruction will show you how to build AOSP for the Hikey960.

**Steps to Fetch AOSP Source:**

```bash
mkdir /home/$USER/aosp
repo init -u https://android.googlesource.com/platform/manifest -b master
sudo cp /home/$USER/aosp/.repo/repo/repo /usr/bin/repo
repo sync -j`nproc`
```

**Set up your build environment**

```bash
source build/envsetup.sh
```

**Fetch hikey960 vendor package**

```bash
./device/linaro/hikey/fetch-vendor-package.sh
```

**Target Selection for Build: hikey960**

The following example uses the current AOSP from the master branch for device hikey960. You can select the build variant as follows:

1. user: for production builds
2. userdebug: for development builds
3. eng: for development builds with faster build times

```bash
lunch hikey960-aosp_current-userdebug
```

*You can enable hardware AddressSanitizer (ASAN) for memory error detection.*

```bash
export SANITIZE_TARGET=hwaddress
```

**Build AOSP: Start the build process:**

```bash
m -j`nproc`
```

**Flashing all images to the device:**

1. Ensure the Device is in Fastboot Mode: Make sure the board is powered on with Switch 1 and Switch 3 set to ON.

2. Connect the Device: Connect the device to your computer via USB.

3. Open a Terminal: On your host machine, open a terminal window.

4. Verify Fastboot Connection: Run the following command to check if the device is recognised:

```bash
fastboot devices
```

**Flash the images:**

```bash
./device/linaro/hikey/installer/hikey960/flash-all.sh
```
**Switching to Normal Mode and Connecting via ADB:**

1. Turn off the device.

2. Change Jumper Switch: Set Switch 1 to the ON position and ensure Switch 2 and Switch 3 are set to OFF. This will put the board into normal mode.

3. Wait for the Board to Start: Allow some time for the board to fully boot up.

4. Connect to the Device: Once the board has started, open a terminal on your computer.

5. Execute ADB Shell: Run the following command to connect to the device:

```bash
adb shell
```

**Installation is Ready: You can now proceed with your setup or configuration as needed.**
