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

In comes `chroot`. Standing for change-root (I think? [1]), it's a wrapper around the syscall which basically lets you create a view of the filesystem for the current process + all other child processes of this. This concept forms the basis of containers....so let's play around with it a bit right?


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
mkdir -p test_dir/lib
mkdir -p test_dir/lib64
mkdir -p test_dir/usr
sudo mount -o bind /lib test_dir/lib/
sudo mount -o bind /lib64 test_dir/lib64/
sudo mount -o bind /usr test_dir/usr/
cp /bin/ls test_dir/bin/
sudo chroot test_dir /bin/bash
```

Playing around with it a bit, you'll see that you can't really exit the chrooted instance unless you kill the process (ctrl d). Navigating around to the other parts of the directory structure looks impossible though right. `cd ..` at `/` yields the same directory.

```
> ls
bin  lib  lib64  proc  usr
> cd ..
> ls
bin  lib  lib64  proc  usr
```

This structure we've created is called a chroot jail. However if a program within it is run with root privileges, it is possible to "break out" [2]. I've not really played with trying to break out, but given that processes with root privileges can access and modify pretty much any part of a system, it seems reasonable that this should be possible.


## No Sharing!

Refer [3] for examples as well
Anyway, we've digressed. The point of talking about chroot was to show isolation of information. At this point we've sort of isolated the filesystem right? But a filesystem isn't the only part of a computer right? What about other stuff? Instead of copying just ls, lets mount bin within our chrooted instance as well (mainly for convenience. We get all of our executables then). We didn't do this before to prove that the executables aren't actually necessary.

```
sudo mount -o bind /bin test_dir/bin/
sudo chroot test_dir /bin/bash
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

Executing `ps -aux` shows us pretty much all the processes running on the vm. That's very un-container-like. We want isolated systems. It's not very isolated if I can kill other processes(`pkill <pid>`). You can try running a command like `top` and viewing it from the vm itself via `ps -aux | grep top` (do another `vagrant ssh` in a separate shell), and then kill the process as well.





## References:
- [1] https://en.wikipedia.org/wiki/Chroot
- [2] http://www.unixwiz.net/techtips/mirror/chroot-break.html
- [3] https://ericchiang.github.io/post/containers-from-scratch/
- [4] https://www.youtube.com/watch?v=gMpldbcMHuI
- [5] https://blog.mbrt.dev/2017-10-01-demystifying-container-networking/
- [6] https://blog.lizzie.io/linux-containers-in-500-loc.html
