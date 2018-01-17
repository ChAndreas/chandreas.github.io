---
layout: post
title:  "Linux Namespaces"
date:   Sun Jan 14 17:31:51 EEST 2018
---

Namespaces are a feature on the Linux kernel that isolate and virtualize system resources of a collection of processes.
Examples of softwares using namespaces are Docker, LXC and Mesos.

There are two essential set of namespaces, label based namespaces and mapping based namespaces.

Label based namespace are used to attach resources to a namespace, for example when you want to add a network card on a namespace you will not see it on the host and it will reappear when your remove it from the namespace. The first label based namespace was network namespace. 

Mapping namespaces are used to map resources from parent process to namespace itself, for example pid namespace takes the process pid and projects it into the namespace with different pid. Very useful when you want to run a system container and you want to run a service that required init. 

To enable namespaces support on your kernel you must enable if not already the below configs on your kernel .config file:
   
    CONFIG_NAMESPACES
    CONFIG_UTS_NS
    CONFIG_IPC_NS
    CONFIG_USER_NS
    CONFIG_PID_NS
    CONFIG_NET_NS

**Linux supports 7 Namespaces:**

    1. Mount - isolate filesystem mount points
    2. UTS - isolate hostname and domain name
    3. IPC - isolate interprocess communication (IPC) resources
    4. PID - isolate the PID number space
    5. Network - isolate network interfaces
    6. User - isolate UID/GID number spaces
    7. Cgroup - isolate cgroup root directory

**The namespaces API includes the following system calls:**

    1. clone - creates a new process similar to fork() but allows child process to share parts of its execution.
    2. setns - allows the calling process to join an existing namespace and allow any type of namespace to be joined.
    3. unshare - moves the calling process to a new namespace and allows a process to disassociate parts of its execution that currently being shared with other process.
   
**Mount:**

Used to control mount points. Process can have their own rootfs, private or share mount.

  Mount namespaces serve a variety of purposes. For example, they can be used to provide per-user views of the filesystem.   Other uses include mounting a /proc filesystem for a new PID namespace without causing side effects for other process and chroot()-style isolation of a process to a portion of the single directory hierarchy. In some use cases, mount namespaces are combined with bind mounts.

Example
```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mount.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE]; /* Stack size for cloned child */

int child_main(void* arg)
{
    if (mount("proc", "/proc", "proc", MS_BIND|MS_REC|MS_SHARED, NULL) == -1)
        perror("mount");
    char *child[] = {
                    "/bin/bash",
                    "-c",
                    "cat /proc/$$/mountinfo | sed 's/ - .*//'",
                    NULL
                    };
    execvp(child[0], child);
    return 1;
}

int main(int argc, char *argv[])
{
        int child_pid = clone(child_main, child_stack+STACK_SIZE,
            CLONE_NEWNS | SIGCHLD, NULL);

  waitpid(child_pid, NULL, 0);
  return 0;
}
```

**UTS:**

Used to set a hostname on the container.The main job of this namespace was to make nfs run in container.

Example
```c	
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE]; /* Stack size for cloned child */

int child()
{
    sethostname("Container", 10);
    char *shell[] = {
                    "/bin/bash",
                    "-c",
                    "echo $HOSTNAME",
                    NULL
                    };
    execvp(shell[0], shell);
    return 0;
}

int main(int argc, char *argv[])
{
    int child_pid = clone(child, child_stack+STACK_SIZE,
            CLONE_NEWUTS | SIGCHLD, NULL);
    waitpid(child_pid, NULL, 0);
    return 0;
}
```
**IPC:**

This namespace allows to unshare IPCs to have an isolated set of System V IPC objects (sem, shm, msg) and POSIX message queues inside namespace.

```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE]; /* Stack size for cloned child */

int child_main(void* arg)
{
    char *child[] = {
                    "/bin/bash",
                    "-c",
                    "ipcs -m",
                    NULL
                    };
    execvp(child[0], child);
    return 1;
}

int main(int argc, char *argv[])
{
	int child_pid = clone(child_main, child_stack+STACK_SIZE,
            CLONE_NEWIPC | SIGCHLD, NULL);

  waitpid(child_pid, NULL, 0);
  return 0;
}
```

**PID:**

Process within the PID namespace can see see process only in the same PID namespace and the PID will start with number 1.

Example  

```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE]; /* Stack size for cloned child */

int child_main(void* arg)
{
    char *child[] = {
                    "/bin/bash",
                    "-c",
                    "echo $$",
                    NULL
                    };
    execvp(child[0], child);
    return 0;
}

int main(int argc, char *argv[])
{
    int child_pid = clone(child_main, child_stack+STACK_SIZE,
			CLONE_NEWPID | SIGCHLD, NULL);
	waitpid(child_pid, NULL, 0);
	return 0;
}
```

**Network:**

Example if you run 
    
        ip addr  
    
you will see all the network interface of your system, but lets create a network namespace named test

        ip netns add test
    
and run again 

        ip netns exec test ip addr
    
you will see only loopback interface. If you want you can add interfaces.
        
        ip link set enp1s0 netns test.

to remove it 

        ip netns exec test ip link set enp1s0 netns 1

**User:**
	
User namespace gives enhanced privileges to a user and can emulate root user. Also used to map users and groups.If you run namespaces without the CLONE_NEWUSER flag is placed in the same user namespace as its parent process.

Example
```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)
 
static char child_stack[STACK_SIZE]; /* Stack size for cloned child */
 
int child_main(void* arg)
{ 
    char *child[] = {
                    "/bin/bash",
	            "-c",
		    "id",
		    NULL,
                    };
    execvp(child[0], child);
    return 0;
}
 
int main(int argc, char *argv[])
{
    int child_pid = clone(child_main, child_stack+STACK_SIZE,
             CLONE_NEWUSER | SIGCHLD, NULL);
    waitpid(child_pid, NULL, 0);
    return 0;
}
```
**Cgroups:**

A process can call unshare() using the CLONE_NEWCGROUP flag to enter a new cgroup namespace. Once it does that, it will no longer see the global cgroup hierarchy.

```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#ifndef CLONE_NEWCGROUP
#define CLONE_NEWCGROUP         0x02000000
#endif

#define STACK_SIZE (1024 * 1024)

static char child_stack[STACK_SIZE]; /* Stack size for cloned child */

int child_main(void* arg)
{
    char *child[] = {
                    "/bin/bash",
	            "-c",
	            "cat /proc/self/cgroup",
	            NULL,
                    };
    execvp(child[0], child);
    return 0;
}

int main(int argc, char *argv[])
{
  int child_pid = clone(child_main, child_stack+STACK_SIZE,
            CLONE_NEWCGROUP | SIGCHLD, NULL);
  waitpid(child_pid, NULL, 0);
  return 0;
}
```

**unshare**

The unshare command allows you to run a program with some namespaces ‘unshared’ from its parent. 

Example
```bash
sudo unshare --map-root-user --user bash -c id
```

**setns**

joining an existing namespace

Example
```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

int main(int argc, char *argv[])
{
    int fd;
    if (argc < 3) {
        fprintf(stderr, "%s /proc/PID/ns/FILE cmd arguments\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    fd = open(argv[1], O_RDONLY);
    if (fd == -1)
        perror("open");

    if (setns(fd, 0) == -1)
        perror("setns");

    execvp(argv[2], &argv[2]);
         perror("execvp");
}
```

Sources

    https://linux.die.net/man/2/setns
    https://en.wikipedia.org/wiki/Linux_namespaces
    http://man7.org/linux/man-pages/man7/namespaces.7.html
    https://lwn.net/Articles/531114/
