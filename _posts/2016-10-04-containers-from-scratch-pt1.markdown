---
layout: post
title:  "Containers From Scratch pt1"
date:   2016-10-04
categories: c linux containers docker
---

I thought it would be an interesting project to implement containers. This would hopefully breakdown the black box that is containers for me. I figured doing this with C would let me interact directly with the OS and see what is happening.

The first thing for me is to figure out is what exactly a container is. After much reading the idea that seemed consistent was containers are something that provides process isolation. The end goal for this would be to write something that allows a process to run in isolation. 

My next step is how to get to an isolated process. Linux provides a few ways and after a bit of research I came across cgroups, namespaces and chroot. Namespaces in Linux looked like they offered everything chroot would offer and cgroups didn't offer much at this point as far as accomplishing this goal so I focused on taking advantage of namespaces.

The system I am using for this is Ubuntu 16.04.1 with 4.8-rc5 kernel. I ran into some odd problems with the distribution and packages while working on this. 

I found alot of great information on using namespces <a href="http://crosbymichael.com/creating-containers-part-1.html" >here</a>a> . There are some code similarities since I used this tutorial as a starting point for getting something running.

For part 1 I am focusing on hacking together a mount and process namespace.
The final output will be something recognizable as a container.

~~~ shell
matthewtejo@matthewtejo:~/c/containers$ sudo ./main bash
starting...
/ # ps aux
PID   USER     COMMAND
    1 root     bash
    2 root     ps aux
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/sda2                22.9G     19.0G      2.8G  87% /
tmpfs                   806.1M         0    806.1M   0% /dev
/ # 
~~~

The most important part to this is <a href="https://linux.die.net/man/2/clone">clone()</a>. Clone is similar to fork() but allows some extra flags. These extra flags will allow you to create the namespaces. The program will run the arguments to the program in the child process. 

~~~ c
int main(int argc, char *argv[])
{
        char c_stack[1024];    
        char **args = &argv[1];

        printf("starting...\n");
        pid_t pid = clone(child_exec,c_stack, SIGCHLD ,args);
        if(pid<0)
                fprintf(stderr, "clone failed %s\n", strerror(errno));
        waitpid(pid,NULL,0);
        return 0;
}
~~~

~~~ c
int child_exec(void *arg)
{
        char **commands = (char **)arg;

        execvp(commands[0],commands);   
        return 0;
}
~~~ 

The two parts are similar to doing something like fork and exec. The program passes the command line arguments to the child then uses that in execvp().

To create a filesystem namespace add the flag CLONE_NEWNS to clone `SIGCHLD | CLONE_NEWNS`.

This isn't very straight foward. Mounts can be shared between namesspaces. Clone with CONE_NEWNS will propagate the file system being shared. A situation I ran into is creating a new mount point on a child process namespace and having that exist in the parents. Check /proc/self/mountinfo for shared. If the directory is shared you need to make it private.

`sudo mount --make-rprivate  /`
Recursively make the mount private.

I ran into an issue where the distribution(possibly more specifically systemd but I need to read more into that) makes the "/" mount shared.

You can check the namesspaces in proc

``` bash
$ ls -l /proc/self/ns/
total 0
lrwxrwxrwx 1 matthewtejo matthewtejo 0 Oct  1 14:16 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 matthewtejo matthewtejo 0 Oct  1 14:16 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 matthewtejo matthewtejo 0 Oct  1 14:16 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 matthewtejo matthewtejo 0 Oct  1 14:16 net -> net:[4026531957]
lrwxrwxrwx 1 matthewtejo matthewtejo 0 Oct  1 14:16 pid -> pid:[4026531836]
lrwxrwxrwx 1 matthewtejo matthewtejo 0 Oct  1 14:16 user -> user:[4026531837]
lrwxrwxrwx 1 matthewtejo matthewtejo 0 Oct  1 14:16 uts -> uts:[4026531838]
```

or `ls -l /proc/1/ns/`

The next part is creating an isolated process list.
We can do this by adding another flag to clone. `SIGCHLD | CLONE_NEWNS | CLONE_NEWPID`
Checking the process with something like `ps aux` requires fixing /proc first. We unmount it then mount it.

``` c
umount2("/proc",MNT_DETACH);
mount("proc", "/proc", "proc",0, NULL);
```

This needs to be added to the child_exec function. 

Now with filesystem and process namespaces set up we can do some really simple containers. the syscall pivot_root comes up as a better alternative to chroot in everything I read. <a href="https://lk4d4.darth.io/posts/unpriv3/" > This article </a> had a nice suggestion of using busy box as the new root file system. Download it and extract it. 

Pivot root doesn't have a wrapper function so I needed to write my own. http://man7.org/linux/man-pages/man2/pivot_root.2.html

``` c
//wrapper for pivot root syscall
int pivot_root(char *a,char *b) 
{
        if (mount(a,a,"bind",MS_BIND | MS_REC,"")<0){
                printf("error mount: %s\n",strerror(errno));
        }   
        if (mkdir(b,0755) <0){
                printf("error mkdir %s\n",strerror(errno));
        }   
        printf("pivot setup ok\n");
        return syscall(SYS_pivot_root,a,b);
}
```

Then in child_exec we set up the new file system

``` c
pivot_root("./busy","./busy/.old");
mount("tmpfs","/dev","tmpfs",MS_NOSUID | MS_STRICTATIME,NULL);
mount("proc", "/proc", "proc",0, NULL);
umount2("/.old",MNT_DETACH);
```

We call pivot_root to place us into the new filesystem. The first argument is the location of the extracted rootfs. Then we mount dev and proc and unmount the old fs.

If we compile all of this and run exec "bash" we'll end up in the new container mostly isolated from the original environment.

Part two contains how I set up networking.
Check the final code here https://github.com/tejom/container


