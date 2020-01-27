---
layout: post
title:  "Compile Linux Kernel (Ubuntu-Debian)"
date:   Wed Sep 20 16:40:51 EEST 2018
---

**Download neccessary packages**

    sudo apt-get install build-essential libncurses-dev bison flex libssl-dev libelf-dev bc ccache gcc-8-plugin-dev

**Download Kernel Config hardened check, to verify that we will use the best practice kernel config options for security.**

    git clone https://github.com/a13xp0p0v/kconfig-hardened-check.git

**Download Kernel**

    wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.3.13.tar.xz
    wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.3.13.tar.sign

**Extract tar.xz kernel**

    unxz linux-5.3.13.tar.xz

**Download gpg key of Greg Kroah-Hartman and Linus Torvalds Linux Maintainers.**

    gpg --locate-keys torvalds@kernel.org gregkh@kernel.org

**Verify kernel tarball with gpg.**

    gpg --verify linux-5.3.13.tar.sign

**If everything is OK and you didn't get "BAD signature" output from the "gpg –verify" extract the tarball.**

    tar xvf linux-5.3.13.tar

**Enter the kernel folder.**

    cd linux-5.3.13 

**Copy running kernel config.**

    cp -v /boot/config-$(uname -r) .config
    make oldconfig

**Check kernel config with kconfig-hardened-check for security.**

    python3 ../kconfig-hardened-check/kconfig-hardened-check.py -c .config

**Make the nessesary changes that you found with kconfig-hardened-check.**

    make menuconfig

**Build the kernel**

    make -j$(nproc)

**Install Kernel Modules**

    make INSTALL_MOD_STRIP=1 modules_install -j$(nproc)

**Install Kernel**

    make install -j$(nproc)

**Update initramfs**

    update-initramfs -u -k all
    
**Reboot your system**

    reboot
