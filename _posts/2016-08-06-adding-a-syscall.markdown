---
layout: post
title:  "Adding a syscall to Linux 4.7"
date:   2016-08-06
categories: linux programing
---

I've been spending more time learning about how Linux works and thought it was best to modify it myself. Syscalls weren't too complex of an idea to wrap my head around. I also haven't seen any guides on to do this on the Internet with a newer kernel. This is all using version 4.7. There doesn't seem to be too much of a diffrence between this version and older ones. The basic idea is the same.

There are a few files to modify: 

- The syscall table at `arch/x86/entry/syscalls/syscall_64.tbl`
- Headers at `include/linux/syscalls.h`
- The makefile where I added the implementation for the syscall

In the syscall table I scrolled down to the end, but before where the 32bit syscalls start and found a number that wasn't used.

~~~
329     common  myhello                 sys_myhello
~~~

In syscalls.h, I found a space that wasn't inside a ifdef block and added.

~~~
asmlinkage long sys_myhello(char *s)
~~~

This is defining the syscall, which will be named `sys_myhello` and will take one parameter, a string that's going to be printed in a kernel message.

The new syscall implenamtion is in a new file, `kernel/myhello.c`. It's very basic.

~~~ c
#include <linux/kernel.h>
#include <linux/syscalls.h>
#include <linux/linkage.h>

SYSCALL_DEFINE1(myhello,char*, s)
{
        printk("%s\n",s);
        return 0;
}
~~~

It consists of a couple includes and the syscall macro. There is only one parameter for the syscall so the macro to use is SYSCALL_DEFINE1. I put the name of the syscall, the type it takes and the parameter's name. Then it just prints the string.

Since I placed this in kernel the makefile, kernel/Makefile, need to the be modified to compile it the new code.
The end of the file seemed like a good place.

~~~
obj-y += myhello.o
~~~

Then I compiled the kernel, I also installed the headers to /usr/local `sudo make headers_install INSTALL_HDR_PATH=/usr/local` so I could use the header file that defines the numbers.

The next part is using the syscall.
Another really basic c program can do this. 

~~~ c
#include <stdio.h>
#include <sys/syscall.h>
#include <unistd.h>

int main()
{
        long int r = syscall(__NR_myhello,"test a message! hello");
        printf("syscall test\n%li\n",r);
        return 0;
}
~~~

including sys/syscall.h defines __NR_myhello. Compile, set dmesg to show the logs by setting the level and run the program and the message will show in the log.
