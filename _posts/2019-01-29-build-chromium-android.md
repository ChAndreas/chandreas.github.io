---
layout: post
title:  "Building Chromium for Android"
date:	Tue Jan 29 17:32:51 EEST 2020
---

How to build chromium for Android.

**Download Tools**
```bash
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PATH:/path/to/depot_tools"
```
**Get Chromium Project**
```bash
mkdir ~/chromium && cd ~/chromium
fetch --nohooks android
 
cd src
echo "target_os = [ 'linux', 'android' ]" >> ../.gclient
gclient sync
build/install-build-deps.sh
gclient runhooks
```
 Fetch the latest tags for Android and replace VERSION with the correct value.
```bash
git fetch --tags
git checkout VERSION
gclient sync -D --with_branch_heads --with_tags --jobs 32
```
**Install dependencies**
```bash
build/install-build-deps.sh
```
**Setting up the build**

Create args.gn
```bash
mkdir -p out/Default
	
cat <<EOF > out/Default/args.gn
target_os = "android"
target_cpu = "arm64"
android_channel = "stable"
android_default_version_name = "131.0.6778.39"
android_default_version_code = "677803900"
ext_version_enabled = true
ext_version_increment = "0"
is_component_build = false
is_debug = false
is_official_build = true
symbol_level = 1
disable_fieldtrial_testing_config = true
dfmify_dev_ui = false
ffmpeg_branding = "Chrome"
proprietary_codecs = true
is_cfi = true
use_cfi_cast = true
use_relative_vtables_abi = false
include_both_v8_snapshots = false
enable_vr = false
enable_arcore = false
enable_openxr = false
enable_cardboard = false
enable_remoting = false
enable_reporting = false
EOF
```
If you want to build the chromium for debugging or to fuzz with libfuzzer change or add the following
```bash
is_asan = true
is_hwasan = true
is_msan = true
is_lsan = true
is_tsan = true
msan_track_origins = 2
is_ubsan_no_recover = false
is_ubsan_security = true
is_debug = true
dcheck_always_on = true
is_java_debug = true
is_component_build = true
use_libfuzzer = true
use_fuzzilli = true
```

GN build configuration
```bash
gn gen out/Default
```
**Build**

The following command is used to build Trichrome for Android minSdkVersion=29.
```bash
autoninja -C out/Default trichrome_chrome_bundle
```
The following command is used to build Monochrome, which provides both Chromium and the WebView for Android minSdkVersion=26
```bash
autoninja -C out/Default monochrome_public_bundle
```
for devices with minSdkVersion=26 and for local development (to avoid building WebView).
```bash
autoninja -C out/Default chrome_public_apk
```

**Install to Device**
```bash
out/Default/bin/trichrome_chrome_bundle install
```
	
**Running a test**
```bash
out/Default/bin/trichrome_chrome_bundle launch https://google.com
```
**Logging and debugging**

You can see the logs with adb logcat when you will install the chrome apk or with
```bash
out/Default/bin/trichrome_chrome_bundle logcat [-v]
```
Debugging Java and c/c++ code

C/C++ Debugger
```bash
out/Default/bin/trichrome_chrome_bundle gdb

out/Default/bin/trichrome_chrome_bundle lldb
```
Java Debugger
```bash
out/Default/bin/trichrome_chrome_bundle run --wait-for-java-debugger
```
Symbolizing Crash Stacks and Tombstones (C++)
```bash
build/android/tombstones.py --output-directory out/Default
adb logcat -d | third_party/android_platform/development/scripts/stack --output-directory out/Default
```
