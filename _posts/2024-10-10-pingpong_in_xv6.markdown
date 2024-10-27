---
layout: post
title:  "pingpong in xv6"
date:   2024-10-10 19:39:55 +0100
categories: xv6
---

Low level stuff is what usually interests me the most. So things like operating systems, assembly etc etc. Lately I've been reading on xv6 (which is a simple os) as well as doing some exercises to get better understanding how it works. So here's a write up on one of the exercises about pipes.

First a short intro to pipes. Pipes are typically used in a shell to connect an output of one program to an input of another program. Let's see how to use a pipe to check if there's a file fox.txt in the current directory

{% highlight bash %}
ls -l | grep fox.txt
{% endhighlight %}

In the above line an output of 'ls -l' (which is a list of files in the current directory) is redirected [^1] to the input of 'grep fox.txt'. what happens behind the scenes (xv6 code for this is in sh.c) is a system call function pipe gets called (which connects input and output of the processes that run the commands). Now let's look at an example use of it in C/C++ code

{% highlight c %}
#include <stdio.h>
#include <unistd.h>
#define MSGSIZE 4
char* msg1 = "DOG";
char* msg2 = "FOX";

int main()
{
    char buf[MSGSIZE];
    int fd[2], i;

    if (pipe(fd) < 0) return 1;

    write(fd[1], msg1, MSGSIZE);
    write(fd[1], msg2, MSGSIZE);

    read(fd[0], buf, MSGSIZE); printf("%s\n", buf);
    read(fd[0], buf, MSGSIZE); printf("%s\n", buf);

    return 0;
}
{% endhighlight %}

A word of explanation. 'pipe(int fd[2])' system call creates a pipe (which is in fact a pair of file descriptors held in fd). fd[1] is a write-end of the pipe. So we can write to it. fd[0] is a read-end of the pipe. So we can read from it (provided sth was written to the write-end).

Now on to the task. A pinpong task is about writing a program that forks a child process. Then the parent process sends 'ping' to the child process. The child reads it, then sends 'pong' to the parent. The parent reads that. The end. Below is the code.

{% highlight c %}
//#include <stdio.h>     // needed on linux
//#include <unistd.h>    // needed on linux
//#include <sys/types.h> // needed on linux
//#include <sys/wait.h>  // needed on linux

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main() {
    int p[2];  // file descriptors for pipe
    char buf[5];

    pipe(p); // i assume there's no error here or in fork()...

    if (fork() == 0) {  // child
        read(p[0], buf, 5);
        printf("%d: received %s\n", getpid(), buf);
        close(p[0]);

        write(p[1], "pong", 5);
        close(p[1]);

    } else {  // parent
        write(p[1], "ping", 5);
        close(p[1]);

        wait(0); // wait for child to terminate. Without this, 'pong' might not be printed.
        read(p[0], buf, 5);
        printf("%d: received %s\n", getpid(), buf);
        close(p[0]);
    }
    exit(0);
}
{% endhighlight %}

The code was put in a pingpong.c and a corresponding entry was added in Makefile. Then after running make, I was able to call a pingpong utility from the shell. Also on ubuntu the above worked OK after compiling to an executable file. It seems pretty simple but actually when calling pipe() there's quite a bit happening behind the scenes. I hope to be able to explore that in the next posts.



[^1]: here i was scratching my head a bit but then did some reading (sh.c in xv6 codebase) and it became clear. Normally we call grep like this 'grep pattern file1 file2 ...'. Utility opens each file separately and does separate searches. In 'ls \| grep pattern', grep is called with just a one param (pattern), a special logic is performed wherein 0 fd (standard input) is opened and read. Then search is done in the content.

