---
layout: post
title: "Compile aarch64 Linux kernel Upstream for Raspberry Pi 3"
date: Sun May 8 17:31:51 EEST 2018
---
This is a guide how to compile and install 64 bit (aarch64) kernel for the Raspberry pi 3 b or 3 b+.

Install build tools

	sudo apt-get install build-essential libgmp-dev libmpfr-dev libmpc-dev bc git-core binutils-aarch64-linux-gnu gcc-aarch64-linux-gnu cpp-aarch64-linux-gnu g++-aarch64-linux-gnu libncurses-dev

Clone Linux kernel sources

	git clone https://github.com/ChAndreas/rpi3-kernel.git
	cd rpi3-kernel

Create output folder

	mkdir ../out

Compile kernel

	make O=../out/ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- rpi3_defconfig
	make -j4 O=../out/ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
	mkdir ../out/out-modules/
	make O=../out/ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=../out/out-modules/ modules_install

Install (plug the sdcard and replace sdX with your device)

	mount /dev/sdX1 /media/
	cd ../out/
	rm -rf /media/overlays
	cp -R arch/arm64/boot/dts/overlays /media/
	cp arch/arm64/boot/Image /media/kernel8.img
	cp arch/arm64/boot/dts/broadcom/bcm2710-rpi-3-b.dtb /media/bcm2710-rpi-3-b.dtb
	echo "kernel=kernel8.img" >> /media/config.txt
	umount /media
	mount /dev/sdX2 /media/
	cd out-modules/lib/modules/
	cp -R 4.14.* /media/lib/modules/
	umount /media
