---
layout: post
title:  "Android Chromium Build"
date:	Tue Jan 29 17:32:51 EEST 2019
---

How to build chromium for Android with Hardening Patches.

**Download Tools**

  sudo apt install python git
  
	git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
	export PATH="$PATH:/path/to/depot_tools"

**Cet Chromium Project**

	mkdir chromium
	cd chromium
	fetch --nohooks android
	echo "target_os = [ 'android' ]" >> ./gclient
	gclient sync --with_branch_heads -r 72.0.3626.105 --jobs 32

**Hardening Patches(Credits to Daniel Micay)**

	git clone https://github.com/AndroidHardening/chromium_patches.git
	cd src
	git am ../chromium_patches/*.patch

**Install dependencies**

	echo ttf-mscorefonts-installer msttcorefonts/accepted-mscorefonts-eula select true | sudo debconf-set-selections
	./build/install-build-deps-android.sh
	gclient runhooks
	build/linux/sysroot_scripts/install-sysroot.py --arch=i386
	build/linux/sysroot_scripts/install-sysroot.py --arch=amd64

**Setting up the build**

	mkdir -p out/Default
	
	cat <<EOF > out/Default/args.gn
	target_os = "android"
	target_cpu = "arm64"

	android_channel = "stable"
	android_default_version_name = "72.0.3626.105"
	android_default_version_code = "362607652"
	
	is_component_build = false
	is_debug = false
	is_official_build = true
	symbol_level = 1
	fieldtrial_testing_like_official_build = true
	
	ffmpeg_branding = "Chrome"
	proprietary_codecs = true
	EOF
  
If you want to build the chromium for debugging or to fuzz with libfuzzer change or add the following

	is_asan=true
	is_msan=true
	is_ubsan_security=true
	is_debug = true
	dcheck_always_on = true
	is_java_debug = true
	is_component_build = true
  

GN build configuration

	gn gen out/Default


**Build**

The following command is used to build Monochrome, which provides both Chromium and the WebView.

	autoninja -C out/Default monochrome_public_apk

The apk that we have compiled is supported on Devices with minSdkVersion=24 (Nougat), if you have older device use one of the following command.

for devices with minSdkVersion=19 (KitKat) and above use

	autoninja -C out/Default chrome_public_apk

or for devices with minSdkVersion=21 (Lollipop) and above use

	autoninja -C out/Default chrome_modern_public_apk

**Install to Device**

	adb install out/Default/apks/MonochromePublic.apk

**Running to test**

	out/Default/bin/content_shell_apk launch [--args='--foo --bar'] http://example.com
	out/Default/bin/monochrome_public_apk launch [--args='--foo --bar'] http://example.com
	
**Logging and debugging**

You can see the logs with adb logcat when you will install the chrome apk or with

	out/Default/bin/monochrome_public_apk logcat [-v]

Debugging Java and c/c++ code

C/C++ Debugger

	out/Default/bin/content_shell_apk gdb
	out/Default/bin/monochrome_public_apk gdb

Java Debugger

	out/Default/bin/monochrome_public_apk run --wait-for-java-debugger
	
Symbolizing Crash Stacks and Tombstones (C++)

	build/android/tombstones.py --output-directory out/Default
	adb logcat -d | third_party/android_platform/development/scripts/stack --output-directory out/Default
	
