---
layout: post
title:  "Understanding Zombie Process"
date:   Fri Nov 17 15:15:51 EEST 2017
---
Zombie Process is a state of process which has completed execution but still has an entry in the process table.
These process have irresponsible parent process. The responsible 
Process table is a data structure in Linux kernel, that stores information about all current process.

The process table contain the PID, User, Scheduling value, Virtual memory, State, CPU, Memory etc.

Location of process is /proc and you can use ps, top, htop, pstree and many other tools to see the process or you can get the process table with a kernel module.

Example

{% highlight c %}
#include <linux/module.h>
#include <linux/printk.h>
#include <linux/sched.h>

static int __init proc_list(void)
{
    struct task_struct *task;

    for_each_process(task)
        pr_info("%s [%d]\n", task->comm, task->pid);

    return 0;
}

static void __exit proc_exit(void)
{
	pr_info("Cleaning up");
}

module_init(proc_list);
module_exit(proc_exit);
{% endhighlight %}


An easy way to create a zombie process is to run fork and exit the child immediately and the parent will sleep and will not be informed about the exit.
When a process exit(), all of the memory and resources associated with it are deallocated.  Further note that if the parent process dies before the child, the kernel will reparent the child process to init.


{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main()
{
    pid_t child;

    if ((child = fork()) < 0)
        exit(1);

    if (child == 0)
        exit(0);

    sleep(120);

    return 0;
}
{% endhighlight %}

The parent process use wait() to get info about the child state. 

{% highlight c %}
if ( waitpid(child, &status, 0) == -1 ) {
    return EXIT_FAILURE;
    }
{% endhighlight %}

To get information if the process exit() correctly

{% highlight c %}
if (WIFEXITED (status))
    return WEXITSTATUS(status);
{% endhighlight %}

To fix and remove zombie process entry from process table we need to inform parent process that child called exit. 

We need to make the parent to wait() the status of child so it can report the exit of the child. 

First we need to find the process parent id and zombie id, then we need to use gdb to attach on parent and call waitpid within the zombie id. 

Example

{% highlight c %}
ps axo stat,ppid,pid,comm | grep -w defunct
gdb  
attach <process id of parent process>
call waitpid (<zombie id>,0,0)
detach
{% endhighlight %}

sources
    
    https://linux.die.net/man/2/waitpid:
    https://en.wikipedia.org/wiki/Zombie_process
