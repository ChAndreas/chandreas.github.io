---
layout: post
title:  "Linux Capabilities"
date:   Wed Sep 14 18:31:51 EEST 2018
---

**Linux Kernel distinguishes its processes with the following two categories:**

    1. Privileged Processes: These processes allow the user to bypass all Kernel permission checks.
    2. Unprivileged Processes: These processes are subject to full permission checks, such as the effective UID, GID, and supplementary group list.

The goal of capabilities is divide the power of superuser into pieces, such that if a program that has one or more capabilities is compromised, its power to do damage to the system would be less than the same program running with root privilege.

Linux’s thread/process privilege checking is based on capabilities. 

**List of some of the capabilities that are enabled on Linux:**

    CAP_AUDIT_CONTROL - Allows enable and disable kernel auditing.
    CAP_AUDIT_READ - Allows reading the audit log via a multicast netlink socket.
    CAP_AUDIT_WRITE - Allows to write records to kernel auditing log.
    CAP_BLOCK_SUSPEND - Allows the ability to block system suspend.
    CAP_CHOWN - Allows to make arbitrary changes to file UIDs and GIDs
    CAP_DAC_OVERRIDE - Allows to bypass file read, write, and execute permission checks.
    CAP_DAC_READ_SEARCH - Allows to bypass file read permission checks and directory read and execute permission checks.
    CAP_NET_ADMIN - Allows various  network-related  operations.
    CAP_NET_RAW - Allows to use of RAW and PACKET sockets.
    CAP_SETUID - Allows arbitrary manipulations of process UIDs.
    CAP_SETGID - Allows arbitrary manipulations of process groups id.
    CAP_SETPCAP - Allows to grant or remove any capability in the caller's permitted capability set to or from any other process.
    CAP_MKNOD - Allows to create filesystem node 
    CAP_SYS_MODULE - Allows to load and unload modules
    CAP_SYS_NICE - Allows change process nice value
    CAP_SYS_PTRACE - Allows to trace process
    CAP_SYS_TIME - Allows to set system clock
    CAP_SYS_CHROOT - Allows to run chroot.
    CAP_SYS_ADMIN - Allows a range of system administration operation (dangerous capability)
    CAP_MAC_OVERRIDE - Allows to change mac address.
    CAP_NET_BIND_SERVICE - Allows to bind privileged ports(less than 1024)
    CAP_NET_BROADCAST - Allows socket broadcast and listen to multicast.
    CAP_SETFCAP - Allows to set file capabilities.
    CAP_SYS_BOOT - Allows to run reboot.
    CAP_SYS_PACCT - Allows to enable or disable process accounting
    CAP_SYS_RAWIO - Allows I/O port operation.
    CAP_KILL - Allows to send kill signals.

**There are 3 modes for Capabilities:**

    1. e: Effective - This indicates that the capability is "activated."
    2. p: Permitted - This indicates that the capability can be used.
    3. i: Inherited - This indicates that the capability is inherited by child elements/subprocesses and defines which capabilities stay permitted across an exec().

**Getting/Setting capabilities from userland:**
    
    getcap - get capabilities
    setcap - set capabilities
    
**Example to allow user run wireshark dumpcap utility without root privileges:**

Run getcap you will see that no capabilities are enabled for dumpcap

        getcap /usr/bin/dumpcap

Run the below command to allow raw socket and network administrator privileges for dumpcap

        sudo setcap cap_net_raw,cap_net_admin+eip  /usr/bin/dumpcap

Run again the getcap commnad and you will get the output that dumpcap has the new capabilities.

        /usr/bin/dumpcap = cap_net_admin,cap_net_raw+eip


**Example check capabilities on c code.**

```c
#define  _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <linux/capability.h>
#include <sys/capability.h>
#include <errno.h>

int cap_check(cap_value_t cap_flag)
{
    cap_t cap;
    cap = cap_get_proc();
    
    if (cap == NULL)
        return -1; 
    
    if (cap_set_flag(cap, CAP_EFFECTIVE, 1, &cap_flag, CAP_SET) == -1 )
    {
        perror("not enabled");
        cap_free(cap);
        return -1;
    }
    if (cap_set_proc(cap) != 0 )
    {
        perror("cannot set");
        cap_free(cap);
        return -1;
    }
    return 0;
}

int main(int argc, char *argv[])
{
    FILE *fd;
    int c;

    if (cap_check(CAP_DAC_READ_SEARCH) == -1)
    {   perror("cap_set_flag CAP_DAC_READ_SEARCH");
        return -1;
    }
    
    if (cap_check(CAP_DAC_OVERRIDE) == -1)
    {
        perror("cap_set_flag CAP_DAC_OVERRIDE");
        return -1;
    }
    if (cap_check(CAP_SYS_ADMIN) == -1)
    {
        perror("cap_set_flag CAP_SYS_ADMIN");
        return -1;
    }

    fd = fopen ("/etc/shadow", "r");
    if (fd == NULL)
    {
        printf("Open /etc/shadow failed\n");
        return -1;
    } else {
        while(1) {
            c = fgetc(fd);
            if( feof(fd) ) { 
                break ;
            }
            printf("%c", c);
        }
        fclose(fd);
    }
    return 0;
}
```
