---
layout: post
title:  "Building Chromium for Android"
date:	Tue Jan 29 17:32:51 EEST 2019
---

How to build chromium for Android.

**Download Tools**
	
	sudo apt install python git
	git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
	export PATH="$PATH:/path/to/depot_tools"

**Get Chromium Project**

	mkdir chromium
	cd chromium
	fetch --nohooks android
	cd src
	git fetch --tags
	git checkout 99.0.4844.73
	gclient sync -D --with_branch_heads --with_tags --jobs 32
	third_party/android_deps/fetch_all.py

**Install dependencies**

	./build/install-build-deps-android.sh

**Setting up the build**

Create args.gn

	mkdir -p out/Default
	
	cat <<EOF > out/Default/args.gn
	target_os = "android"
	target_cpu = "arm64"
	android_channel = "stable"
	android_default_version_name = "99.0.4844.73"
	android_default_version_code = "484407300"
	is_component_build = false
	is_debug = false
	is_official_build = true
	symbol_level = 1
	disable_fieldtrial_testing_config = true
	dfmify_dev_ui = false
	disable_autofill_assistant_dfm = true
	disable_tab_ui_dfm = true
	ffmpeg_branding = "Chrome"
	proprietary_codecs = true
	is_cfi = true
	use_cfi_cast = true
	enable_gvr_services = false
	enable_remoting = false
	enable_reporting = true
	EOF
  
If you want to build the chromium for debugging or to fuzz with libfuzzer change or add the following

	is_asan=true
	is_msan=true
	is_ubsan_security=true
	is_debug=true
	dcheck_always_on=true
	is_java_debug=true
	is_component_build=true
	use_libfuzzer=true
	
  

GN build configuration

	gn gen out/Default

**Build**

The following command is used to build Trichrome for Android Q.

	autoninja -C out/Default trichrome_chrome_bundle

The following command is used to build Monochrome, which provides both Chromium and the WebView for Android N - P

	autoninja -C out/Default monochrome_public_bundle

The apk that we have compiled is supported on Devices with minSdkVersion=24 (Nougat), if you have older device use one of the following command.

for devices with minSdkVersion=19 (KitKat) and above use

	autoninja -C out/Default chrome_public_apk

or for devices with minSdkVersion=21 (Lollipop) and above use

	autoninja -C out/Default chrome_modern_public_bundle

**Install to Device**

	adb install out/Default/apks/MonochromePublic.apk

**Running a test**

	out/Default/bin/monochrome_public_apk launch http://google.com
	
**Logging and debugging**

You can see the logs with adb logcat when you will install the chrome apk or with

	out/Default/bin/monochrome_public_apk logcat [-v]

Debugging Java and c/c++ code

C/C++ Debugger

	out/Default/bin/monochrome_public_apk gdb

Java Debugger

	out/Default/bin/monochrome_public_apk run --wait-for-java-debugger
	
Symbolizing Crash Stacks and Tombstones (C++)

	build/android/tombstones.py --output-directory out/Default
	adb logcat -d | third_party/android_platform/development/scripts/stack --output-directory out/Default
	
