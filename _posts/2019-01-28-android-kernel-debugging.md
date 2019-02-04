---
layout: post
title: "Android Emulator Kernel Debugging"
date: Mon Jan 28 17:40:51 EEST 2019
---

**Android Emulator Kernel Debugging**

Download and compile kernel

	git clone https://android.googlesource.com/kernel/goldfish
	cd goldfish
	git checkout android-goldfish-4.4-dev
	wget https://raw.githubusercontent.com/ChAndreas/linux-config/master/config-goldfish-android
	mv config-goldfish-android .config
	make oldconfig
	make -j64


Create emulator image
	
	avdmanager create avd -n test-emulator -k "system-images;android-28;google_apis;x86_64"		

Start the emulator with our compiled kernel

	emulator \
		-verbose \
		-show-kernel \ 
		-debug init \ 
		-avd test-emulator \
		-kernel ~/goldfish/arch/x86_64/boot/bzImage \ 
		-qemu \
		-s 


Add the following line to .gdbinit

	vim ~/.gdbinit
	add-auto-load-safe-path <path>/goldfish/scripts/gdb/vmlinux-gdb.py

Run gdb and attach to emulator.

	gdb -q ./vmlinux
	(gdb) target remote :1234

**Examples of using the Linux-provided gdb helpers**

Load symbols for all modules and vmlinux.

	lx-symbols

Display kernel log.

	lx-dmesg

List of loaded Modules.

	lx-lsmod

Access current task.

	p $lx_current().pid
	p $lx_current().comm
	p $lx_current().cred

Current kernel processes.

	(gdb) lx-ps
	0xffffffff822179c0 <init_task> 0 swapper/0
	0xffff88003d5a0000 1 init
	0xffff88003d5a1400 2 kthreadd
	0xffff88003d5a2800 3 ksoftirqd/0
	0xffff88003d5a5000 5 kworker/0:0H
	0xffff88003d600000 7 rcu_preempt
	0xffff88003d601400 8 rcu_sched
	0xffff88003d602800 9 rcu_bh
	0xffff88003d603c00 10 migration/0
	0xffff88003d649400 11 migration/1
	0xffff88003d64a800 12 ksoftirqd/1

Container_of macro is used to obtain the container structure address of given member.

	(gdb) p *(struct task_struct *)0xffff880041198000
	(gdb) p $container_of(init_task.tasks.next, "struct task_struct", "tasks")
	
Print task for specific pid.

	p/x $lx_task_by_pid(1)

Print thread_info structure for the task.

	p/x $lx_thread_info($lx_task_by_pid(1))
