---
layout: post
title: "Introduction to Linux Kernel Debugging"
date: Mon Jan 28 17:31:51 EEST 2019
---

**Kernel Debugging**

Kernel debugging is used to identify kernel bugs.

**Debugging by Printing**

The printk is similar with printf on C standard library and can be called from anywhere in the kernel at any time, from interrupt or process context.
The printk is not used on boot process, if you want to use similar debug on boot mode you can use ``early_printk()`` unless you are writing to console you can always use ``printk()``.

The printk is having the following capabilities to specify the loglevel

**Loglevels that can be used on printk are below**

	KERN_EMERG	0	emergency condition.System maybe hang.
	KERN_ALERT	1	problem that requires immediate attention
	KERN_CRIT	2	critical condition
	KERN_ERR	3	error
	KERN_WARNING	4	warning
	KERN_NOTICE	5	normal
	KERN_INFO	6	informational message
	KERN_DEBUG	7	debugging message
	KERN_DEFAULT	d	the default kernel loglevel
	KERN_CONT		continued line of printout

Example
	
	printk(KERN_ERR "Error, return code: %d\n",ret);

**pr_* macros**

The ``pr_* macros`` (with exception of ``pr_debug``) are simple shorthand definitions in ``include/linux/printk.h`` for their respective printk call and should probably be used in newer drivers.
``pr_devel`` and ``pr_debug`` are replaced with ``printk(KERN_DEBUG ...`` if the kernel was compiled with DEBUG, otherwise replaced with an empty statement. 
For drivers the ``pr_debug`` should not be used anymore (use ``dev_dbg`` instead). 

	pr_emerg
	pr_alert
	pr_crit
	pr_err
	pr_warning
	pr_notice
	pr_info
	pr_debug,pr_devel if DEBUG is defined.
	pr_cont

Example

	pr_err("Error,return code: %d\n", ret);

Note that if you don't specifying the loglevel the default is KERN_WARNING.

To determine your current console_loglevel

	cat /proc/sys/kernel/printk
	4	4	1	7
	current	default minimum boot-time-default

To change console_loglevel

	echo 8 > /proc/sys/kernel/printk

if is set to 8, all messages, including debugging ones, are displayed

Another way to change the console log level is to use dmesg with the -n parameter 

	 dmesg -n 7

To debug an oops and you don't know where this happened you can use the following line

	printk(KERN_ALERT "DEBUG: Passed %s %d \n",__FUNCTION__,__LINE__);

**Log Buffer**

Kernel messages are stored in a circular buffer of size LOG_BUF_LEN which is configurable on compile time. The default size is 16KB. 

**Enable Kernel debugging Option**

The following options should be enabled on kernel config

    CONFIG_DEBUG_KERNEL=y
    CONFIG_DEBUG_INFO=y

**Bugs on code**

There are two functions that you can use on the kernel to show an oops message.

    BUG();
    BUG_ON();

Further a more critical error is signaled via panic() which is printing the error and halt the system.

Example:

    panic("%s: failed to map registers\n", __func__);


Sometimes we need stack trace and dump_stack() can be used which is dumps the contents of the registers and function back trace.

Example:

    dump_stack();


**Decoding an oops message**

addr2line translates addresses into file names and line numbers. Given an address in an executable it uses the debugging information to figure out which file name and line number are associated with it.

Examples:

    addr2line -f -e vmlinux 0xffffffff85037434

with gdb 

    gdb -q vmlinux
    list *(0xffffffff85037434)

or with objdump to determine the offending line

    cat /proc/modules

get the value of module and use it on objdump

    objdump -dS --adjust-vma=0xffffffff85037434 vmlinux

**Kernel Debugging with gdb**

GDB can be used to debug the kernel. As you can see on the below command we are running gdb on uncompressed kernel image vmlinux. The kcore parameter is acting as a core file to let gdb peek into the memory 

    gdb -q vmlinux /proc/kcore

KGDB patch enables gdb to debug kernel remotely with serial cable.

How to enable KGDB on your kernel 

    CONFIG_FRAME_POINTER=y
    CONFIG_KGDB=y
    CONFIG_KGDB_SERIAL_CONSOLE=y
    CONFIG_KGDB_KDB=y 
    CONFIG_KDB_KEYBOARD=y

**Example kernel Debugging with GDB and QEMU**

First step is to download and compile the buildroot with the following config file.

Download buildroot

    cd /root/
    wget https://buildroot.uclibc.org/downloads/buildroot-2018.02.10.tar.gz
    tar xvf buildroot-2018.02.10.tar.gz
    cd buildroot-2018.02.10
    make menuconfig

   
Follow the below configuration

    Target options
      Target Architecture - Aarch64 (little endian)
      Target Architecture Variant (cortex-A57)  --->
    Toolchain  --->
      Toolchain type (Buildroot toolchain)  --->
      Toolchain type (External toolchain)  --->
      Toolchain (Linaro AArch64 2017.11)  ---> 
        *** Host GDB Options ***
      [*] Build cross gdb for the host                                     
      [*]   TUI support
      [*]   Python support
      [*]   Simulator support
      GDB debugger Version (gdb 8.0.x)  --->  
    System configuration  ---> 
    [*] Enable root login with password
        ( ) Root password ⇐= set your password using this option
    [*] Run a getty (login prompt) after boot --->
        TTY port - ttyAMA0
    Target packages  ---> 
      [*]   Show packages that are also provided by busybox
    Networking applications
        [*] dhcpcd
        [*] iproute2
        [*] openssh
    Filesystem images
        [*] ext2/3/4 root filesystem
             ext2/3/4 variant (ext4)  --->
        (120M) exact size
        [*] tar the root filesystem

Build it

    make -j4

Then we will need to install the aarch64 toolchain to compile the kernel

    apt install gcc-aarch64-linux-gnu

Compile the Kernel with specific config
    
    cd /root/
    git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
    cd linux/
    wget https://raw.githubusercontent.com/ChAndreas/linux-config/master/config-aarch64
    mv config-aarch64 .config
    ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make oldconfig
    ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j40

Next step is to install QEMU for the architecture that we have compiled our kernel and buildroot.

    sudo apt install qemu-system-arm

Boot the QEMU with following configuration

    qemu-system-aarch64 \
          -machine virt \
          -cpu cortex-a57 \
          -nographic -smp 2 \
          -hda ~/buildroot-2018.02.10/output/images/rootfs.ext4 \
          -kernel ~/linux/arch/arm64/boot/Image \
          -append "console=ttyAMA0 root=/dev/vda oops=panic panic_on_warn=1 panic=-1 ftrace_dump_on_oops=orig_cpu debug earlyprintk=serial slub_debug=UZ" \
          -m 2G \
          -net user,hostfwd=tcp::10023-:22 -net nic \
          -pidfile vm.pid -s \
           2>&1 | tee vm.log

Run the gdb and verify that everything is configured correctly by checking symbols

    gdb-multiarch vmlinux
    (gdb) add-auto-load-safe-path ~/linux/scripts/gdb/vmlinux-gdb.py
    (gdb) info address vmalloc_user

Now on QEMU type the follow to compare with the address from the gdb 

    grep 'T vmalloc_user' /proc/kallsyms

The addresses should match exactly

Attach to the booted guest:

    (gdb) target remote :1234

To see the source context 

    (gdb) info source
    (gdb) list

We can see the backtrace with the following command	
      
    (gdb) bt full

Use c to continue execution, and Control-C to halt execution. Use n to step over, s to single-step, and finish to finish the current stack frame. Remember that we’re running a build with optimizations, so sometimes variables will be optimized out or the source line may change unexpectedly.

    (gdb) b printk
    (gdb) c
