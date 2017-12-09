---
layout: post
title:  "Understanding Daemon Process"
date:   Fri Nov 17 15:20:23 EEST 2017
---
A daemon process is a process which runs in background and has no controlling terminal.
Normally daemon process are running on boot, running as root or other system user and handle system tasks.

The requirement to run a daemon it must run as a child of init process and not connected to a terminal.


How to create a daemon

    Call fork() off the parent process.
    Call exit() on parent process
    Call setsid() to give new process group and session ID (SID) to become child process as a session leader.
    Call signal() to ignore any signal sent from child to parent
    Call again fork() to check that the daemon is not a session leader, which prevents it from acquiring a controlling terminal.
    Call umask(0); Set new file permissions
    Call chdir() to change the current working directory
    Close file descriptors.
    Reopen all file descriptor and redirect them to /dev/null, call open() /dev/null  for stdin to return a file descriptor and the dublicate with dup() for stdout and stderr.
    Call daemon code.

Example code

{% highlight c %}
#include <sys/types.h>
#include <sys/stat.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <errno.h>
#include <unistd.h>
#include <signal.h>
#include <linux/fs.h>

int main(void) {

    pid_t pid;
    int i;

    pid = fork();

    if (pid == -1) 
    {
        return -1;
    }
    
    if (pid != 0) 
    {
        exit(EXIT_SUCCESS);
    }
    
    if (setsid() == -1)
    {
        return -1;
    }
    
    signal(SIGCHLD, SIG_IGN);

    pid = fork();

    if (pid < 0) 
    {
        exit(EXIT_FAILURE);
    }

    if (pid > 0) 
    {
        exit(EXIT_SUCCESS);
    }

    umask(0);

    if ((chdir("/")) == -1) 
    {
	    return -1;
    }

    for (i = sysconf(_SC_OPEN_MAX); i > 0; i--) 
    {
        close(i);
    }

    open("/dev/null", O_RDWR);
    dup(0);
    dup(0);

    while (1) 
    {
        /* run daemon code */
    }
    
    exit(EXIT_SUCCESS);
}
{% endhighlight %}

Other way on Linux systems you can call daemon() to create it. 
{% highlight c %}
       int daemon(int nochdir, int noclose);
{% endhighlight %}


    https://stackoverflow.com/questions/17954432/creating-a-daemon-in-linux/17955149#17955149
    https://linux.die.net/man/2/setsid
    http://man7.org/linux/man-pages/man2/umask.2.html
    http://man7.org/linux/man-pages/man3/sysconf.3.html
    http://man7.org/linux/man-pages/man2/open.2.html
    http://man7.org/linux/man-pages/man2/dup.2.html


