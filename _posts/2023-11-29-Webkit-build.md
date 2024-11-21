---
layout: post
title: "Building WebKit"
date:	Wed Nov 29 17:05:54 EEST 2023
---

**Building WebKit**

WebKit is the web browser engine used by Safari, Mail, App Store, and many other apps on macOS and iOS.

Building WebKit requires that you have installed Xcode from App Store and running the following command in Terminal to install the Xcode Command Line Tools.
```zsh
xcode-select --install
```

Open the terminal and clone the Webkit project from Github

```zsh
git clone https://github.com/WebKit/WebKit.git WebKit
cd Webkit
```

Running the build command, it will take around 20 minutes to finish the compilation. 

```zsh
./Tools/Scripts/build-webkit
```

You can add the debug flag if you need it and also ccache to improve re-compile times.

```zsh
./Tools/Scripts/build-webkit --debug --use-ccache
```

Running WebKit in Safari

```zsh
./Tools/Scripts/run-safari --debug
```

Running “Mini browser”, the minibrowser is good enough that you can run your WebKit build and do testing or debugging.

```zsh
./Tools/Scripts/run-minibrowser --debug
```


References

https://webkit.org/building-webkit/
