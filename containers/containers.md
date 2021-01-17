# Stuff about Containers

So I've been working with containers are their associated management ecosystems(docker, kubernetes..) for a while now. And well, they can still seem a bit magical sometimes. How are we running a full operating system within this..container process thing.
 What even are containers?

I've noticed that I usually understand stuff better by building concepts up from the base. So through this, I plan to explain (and discover more about) containers, from their very basics. So while at first I thought there weren't too many resources on this topic, turns out there are a lot of well written blogs on it. I will link to other sites for their examples wherever relavant. 


## The very basics
So processes exist. They can talk to each other, do some work, get compute resources etc etc. All processes running on your computer can be listed by ps or its flagged variants. But can processes run which you don't know of? 
The question of isolating processes is essentially where the concept of containers stems from. In fact, containers are nothing but a process or group of processes, running in a kind of isolated manner from the rest of the computer.


## Beyond the /

The revered '/' directory. The top of the food chain. Is there anything beyond it? Running stuff like `cd ..` at the top level just returns you there. But imagine...is it actually special?

Actually, any directory could be a root directory, as long as you couldn't get out of it..right? Like as long as you're not allowed to do a `cd ..`, how could you ever tell the difference?

In comes `chroot`. Standing for change-root (I think? [1](https://en.wikipedia.org/wiki/Chroot)), it's a wrapper around the syscall which basically lets you create a view of the filesystem for the current process + all other child processes of this. This concept forms the basis of containers....so let's play around with it a bit right?


To not mess up my own system with weird syscalls and commands, I use a vagrant vm to try everything out, so all commands given here will be based on that.

Quickstart on vagrant setup for this:
```
vagrant init ubuntu/xenial64
vagrant up
vagrant ssh
```


Ok so now we want to try out chroot to see what levels of isolation we can achieve with this. The claim was that we can essentially make any directory look like the root directory. So in our vm (or normal directory structure if you're feeling risky :) )

```
mkdir test_dir
sudo chroot test_dir
```

This should yield an error. First off, chroot needs a command to run (usually a shell) once executed. For those familiar with docker, this is very similar to the concept of an entrypoint.
We retry with:

```
sudo chroot test_dir /bin/bash
```

This should in fact yield the same error as before (may differ slightly based on chroot/linux versions?). Essentially the issue right now is that /bin/bash isn't present in the chrooted directory structure. So ok, lets go get it.

```
mkdir -p test_dir/bin
cp /bin/bash test_dir/bin/    # Path may be slightly different
sudo chroot test_dir /bin/bash
> chroot: cannot change root directory to '/bin/bash': Not a directory
```

Why does this error occur? Under the hood, bash needs glibc which isn't present in our chrooted directory structure. Also we'll need some of the other commands like `ls` and stuff so lets grab those. Instead of copying them over (which would also work perfectly fine), we'll bind mount some of the directories. By opening up another connection to the vm, we can play around with the files in the mount path and see what it looks like in our chrooted instance as well.

```
> mkdir -p test_dir/lib
> mkdir -p test_dir/lib64
> mkdir -p test_dir/usr
> sudo mount -o bind /lib test_dir/lib/
> sudo mount -o bind /lib64 test_dir/lib64/
> sudo mount -o bind /usr test_dir/usr/
> cp /bin/ls test_dir/bin/
> sudo chroot test_dir /bin/bash
```

Playing around with it a bit, you'll see that you can't really exit the chrooted instance unless you kill the process (ctrl d). Navigating around to the other parts of the directory structure looks impossible though right. `cd ..` at `/` yields the same directory.

```
> ls
bin  lib  lib64  proc  usr
> cd ..
> ls
bin  lib  lib64  proc  usr
```

This structure we've created is called a chroot jail. However if a program within it is run with root privileges, it is possible to "break out" [2](http://www.unixwiz.net/techtips/mirror/chroot-break.html). I've not really played with trying to break out, but given that processes with root privileges can access and modify pretty much any part of a system, it seems reasonable that this should be possible.


## No Sharing!

Refer [3](https://ericchiang.github.io/post/containers-from-scratch/) for examples as well.

Anyway, we've digressed. The point of talking about chroot was to show isolation of information. At this point we've sort of isolated the filesystem right? But a filesystem isn't the only part of a computer right? What about other stuff? Instead of copying just ls, lets mount bin within our chrooted instance as well (mainly for convenience. We get all of our executables then). We didn't do this before to prove that the executables aren't actually necessary.

```
> sudo mount -o bind /bin test_dir/bin/
> sudo chroot test_dir /bin/bash
```

With our full set of executables, we can now pretty much explore all the stuff we'd normally do within the machine. Let's look at the processes

```
> ps
Error, do this: mount -t proc proc /proc
> mount -t proc proc /proc
> ps
  PID TTY          TIME CMD
12055 ?        00:00:00 sudo
12056 ?        00:00:00 bash
12063 ?        00:00:00 ps
```

Executing `ps -aux` shows us pretty much all the processes running on the vm. That's very un-container-like. We want isolated systems. It's not very isolated if I can kill other processes(`pkill <pid>`). You can try running a command like `top` and viewing it from the vm itself via `ps -aux | grep top` (do another `vagrant ssh` in a separate shell), and then kill the process as well. Vice versa as well (run `top` on main machine and view/kill it from chroot)


So how do we fix this issue? Linux natively provides us the ability to create restricted namespaces. With these namespaces, you can create a restricted view of shared resources for a new process. Complicated words, but lets take a look at what it would look like, using the `unshare` [link](https://linux.die.net/man/2/unshare) command.

(Use the previous steps to setup the folders and all)
```
> sudo unshare -p -f
> sudo chroot test_dir /bin/bash
> mount -t proc proc /proc
> ps -A
  PID TTY          TIME CMD
    1 ?        00:00:00 bash
   17 ?        00:00:00 bash
   20 ?        00:00:00 ps
```

We notice here that our PID is 1. Try running top on the main machine and searching for it here now, and we'll see that it is no longer visible. What happened? Let's quickly examine the commands that we ran. Quick look at the man page of `unshare` yields:
```
-p, --pid[=file]
        Unshare the pid namespace. If file is specified then persistent namespace is created by bind mount. See also the --fork and --mount-proc options.

-f, --fork
        Fork the specified program as a child process of unshare rather than running it directly.  This is useful when creating a new pid namespace.
```

sing thin wrappers on native syscalls, we were able to easily create a new process without access to the original (host) pid namespace. Quite container-like. Naturally we can look a bit further and see all the options which unshare provides us with. More advanced configuration if you will. 
Anyways, from this we kind of understand that you can restrict views of quite a few parts of the system from a process.


Only one part left to complete this part of the story. How do I enter my container?
Lets see after the previous step, this is our output:
```
> ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
0            1  0.0  0.5  21376  5140 ?        S    05:31   0:00 -bash
0           31  0.0  0.3  19840  3648 ?        S    06:24   0:00 /bin/bash
0           35  0.0  0.3  36080  3120 ?        R+   06:25   0:00 ps aux
bash-4.3#
```
Now go look for this `/bin/bash` process on the host. `ps aux | grep /bin/bash | grep root`. We can view all the process related information by examining the directory `/proc/<pid>` and it's namespace related information at `/proc/<pid>/ns`. Entering a container is essentially just entering the namespace with all the same initial conditions right? This can be achieved via `nsenter`


So just nsenter. We don't need to unshare the pid namespace this time (thus no -p flag), as our goal is to join the same namespace that we enter in the first step.
```
> sudo nsenter --pid=/proc/<pid>/ns/pid
> unshare -f
> chroot test_dir /bin/bash
```

`ps aux` and other commands between your two sessions will show you that you're in the same 'container'. The previous steps wrapped together can easily be imagined to be quite similar to `docker exec` or `kubectl exec` in a way right?


## Sometimes Sharing is Good?

### Storage
What if we want to add files or storage to a container? Just a process isn't very useful. To achieve this, lets revist the mounts we used in the starting. In our current model, we chroot a directory and then use this instance normally. But this directory exists on the root filesystem of the main machine right? Which basically means that if we were to download stuff within that directory, our root filesystem space would get filled up right? Since this is essentially just a path within the filesystem.


Mounts clearly work inside the container (from above examples). What all can we do with them?

For starters, files can easily be injected into the running container using mounts...even from the host

On the host do:
```
> mkdir rofiles
> mkdir -p test_dir/var/rofiles
> echo "hello world" > rofiles/sample.txt
> sudo mount --bind -o ro rofiles/ test_dir/var/rofiles
```

In chroot do:
```
> echo "hello world" > /var/rofiles/sample.txt
bash: /var/rofiles/sample.txt: Read-only file system
```

Very simply put, using mounts we can control the filesystem. And the chrooted instance is just a view of this filesystem.

But taking this a step further...imagine that the directory were actually a mount of an external block device. By this method, we could actually "attach" a device to a container.


### Networking
I don't think I can provide any better story or explanations regarding this topic than what is already present at [5](https://blog.mbrt.dev/2017-10-01-demystifying-container-networking/#fnr.15). Highly recommend going through this to understand the basics of how containers can communicate with one another. Using the same chrooted instance they setup basic networking and discovery. At some point in the future I may again replicate some of the examples here, once I have a better handle on networks :)


### Resource Restrictions

Present in `/sys/fs/cgroup` is the holy grail of control of containers. All the fancy restrictions on size, usage...etc all are controlled by cgroups aka control groups. Honestly, cgroups are a topic of their own right. But for the purpose of demonstrating restrictions on a container, all we must to is show that this feature can easily be used to restrict the amount of X a process can use.
For now I shall leave it to you for a simple google search on the topic :). [3](https://ericchiang.github.io/post/containers-from-scratch/) also has a simple example restricting memory.

## Interlude
Up to this point, we've essentially seen that the high level features of a container, are all basically linux features. And yeah, that's pretty much the goal of me writing this. But there's one part of the story that's still missing. In the beginning I said that containers seem like full blown operating systems. So far all we've seen is just some shell / process running. Not very operating-system-like. Sure the top level directory structure LOOKS like the files you see normally, but that's only because we literally mounted our system files. Right? 

Given a process, we know how to make it act like a container. Or at the very least, we've now understood some of the high level constructs of a container. And their simple base with linux. Now let's make it run an operating system..


## Build that weird tar
Inspired by the video (procedure will also be very similar and the video itself is brilliant, so PLEASE give it a watch) [4](https://www.youtube.com/watch?v=gMpldbcMHuI), I definitely believe that a crucial part to understanding containers is understanding that, well, their filesystems are literally tarballs. Restricting processes and all seems like something linux would be capable of. But until you actually build your own container from scratch, you just won't _feel_ it. So let's do exactly that.


Though as we've already seen, docker is essentially a wrapper around the core constructs of containers....lets use it to make our life a bit easier. Can we build our own docker image? Sure! It's just a tar.

So I created a small directory structure:
```
> mkdir hello_world
> mkdir -p hello_world/bin
> mkdir -p hello_world/something
> touch bin/abcd
> echo "hello world" > hello_world/something/a.txt
> tree hello_world/
hello_world/
├── bin
│   └── abcd
└── something
    └── a.txt

> tar czvf hello.tar hello_world
```

Let's make this into a docker image shall we?
```
> cat hello.tar | sudo docker import - hello    # sudo may or may not be required depending on how you installed docker
```

Only issue with this, as we saw with our original chrooted instances, is what do we run inside it? Let's bundle this with `bash` perhaps? Add in `ls` as well maybe. You know what? There's an easy way to get this stuff inside it. Add mounts (kind of like what we did before). We'll mount bin, lib, and lib64 (dependencies for some of the executables essentially). This is a temporary hack...we'll make our container end-to-end in a bit.

```
> # Change the next command with the apprpriate hash from (sudo?) docker images
> sudo docker run -it -v /bin:/bin -v /lib:/lib -v /lib64:/lib64 784f52c46a8a /bin/bash
```

Voila! Look at that. We have a fully functional container. Some additional directories were created by docker as well in this process. `docker ps` and all will also show us that we have a fully functional container running.

```
bash-4.3# ls
bin  dev  etc  hello_world  lib  lib64	proc  sys
bash-4.3# ls hello_world
bin  something
bash-4.3# cat hello_world/something/a.txt
hello world
bash-4.3#
```

So without a dockerfile...or anything of the sort actually, we have a running container? Interesting...

## MyoL
Make your own linux. Everything that a container does essentially boils down to what is in that tar ball. So we want to customize that to different distributions. Based on the video [4](https://www.youtube.com/watch?v=gMpldbcMHuI)

Now, download `buildroot` from [7](https://buildroot.org/download.html). Extract and run `make menuconfig`. (may have to install some dependencies like ncurses and gcc). From this menu, you can configure various parts of the to-be linux container. You can, for example, remove the init system (under System Configuration), as containers do not require one. Then run `make` and wait.

You can modify some parts of the files created to suit your own custom requirements. Then just tar it up and use it as a docker image :)

## References:
1. [https://en.wikipedia.org/wiki/Chroot](https://en.wikipedia.org/wiki/Chroot)
2. [http://www.unixwiz.net/techtips/mirror/chroot-break.html](http://www.unixwiz.net/techtips/mirror/chroot-break.html)
3. [https://ericchiang.github.io/post/containers-from-scratch/](https://ericchiang.github.io/post/containers-from-scratch/)
4. [https://www.youtube.com/watch?v=gMpldbcMHuI](https://www.youtube.com/watch?v=gMpldbcMHuI)
5. [https://blog.mbrt.dev/2017-10-01-demystifying-container-networking/](https://blog.mbrt.dev/2017-10-01-demystifying-container-networking/)
6. [https://blog.lizzie.io/linux-containers-in-500-loc.html](https://blog.lizzie.io/linux-containers-in-500-loc.html)
7. [https://buildroot.org/download.html](https://buildroot.org/download.html)
