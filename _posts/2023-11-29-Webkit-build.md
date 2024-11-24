---
layout: post
title: "Building WebKit"
date:	Wed Nov 29 17:05:54 EEST 2023
---

**Building WebKit**

WebKit is the web browser engine used by Safari, Mail, the App Store, and various other applications on macOS and iOS.

To build WebKit, you must have Xcode installed from the App Store. Then, run the following command in Terminal to install the Xcode Command Line Tools:
```zsh
xcode-select --install
```

**Cloning the WebKit Project**

Open Terminal on your macOS.
Use the following command to clone the WebKit repository from GitHub:
```zsh
git clone https://github.com/WebKit/WebKit.git WebKit
cd Webkit
```

**Building WebKit**

To build WebKit, run the following command in Terminal:
```zsh
./Tools/Scripts/build-webkit
```

**Building WebKit with Debug Flag and ccache**

To build WebKit with debugging information and improve recompile times, use the following command in Terminal:
```zsh
./Tools/Scripts/build-webkit --debug --use-ccache
```
The compilation will take about 20 minutes to complete.


**Running WebKit in Safari**
```zsh
./Tools/Scripts/run-safari --debug
```

**Running the MiniBrowser**

The MiniBrowser is a lightweight tool for testing and debugging your WebKit build. Here’s how to run it:
```zsh
./Tools/Scripts/run-minibrowser --debug
```


References

https://webkit.org/building-webkit/
