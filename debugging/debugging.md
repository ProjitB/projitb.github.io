# Debug stories

Debugging can be pretty painful, but sometimes it's quite fun to look back on. 

Contents:
- [Buildroot Stuff](#buildroot-for-custom-image)

## Buildroot for custom image
While playing around with buildroot to create a linux container...I ran into a couple issues.
Excited by the number of options I had, I didn't pay too close attention to what I was choosing...which was my downfall.
Choosing linux headers 5.x, I ran the `make`. Compiling the whole thing through a vm, took about an hour / hour and a half. Basically I didn't want to repeat the process. Adding the `rootfs.tar` to docker was smooth, but when I went to do a docker run of it, I got:

```
> docker run -it ....
FATAL: kernel too old
```

Thought I had the latest headers? A bunch of digging later I realized the error was due to my host kernel's version. I was running ubuntu/xenial64 in vagrant which had some 4.x kernel headers. Ok so upgrade them right? 
Turns during the upgrade process, I needed `libssl1.1` or greater. Tried upgrading libssl, but apparently installing other versions of libssl on ubuntu 16.04 can completely mess up the system. So I copied over my rootfs.tar file to my laptop, and created a new vm running bionic beaver (ubuntu 18.04, and is fine with libssl1.1). However, when I went to actually upgrade the headers, on restart, my /vagrant synced folder wasn't being recognized. 
```
Vagrant was unable to mount VirtualBox shared folders. This is usually
because the filesystem "vboxsf" is not available. This filesystem is
made available via the VirtualBox Guest Additions and kernel module.
Please verify that these guest additions are properly installed in the
guest. This is not a bug in Vagrant and is usually caused by a faulty
Vagrant box. For context, the command attempted was:

mount -t vboxsf -o uid=1000,gid=1000 vagrant /vagrant

The error output from the command was:

: No such device
```

To get the file onto the vagrant vm, I scp'd it from my local machine. After installing docker on it and adding the image...

Seeing this was a relief :)
```
vagrant@ubuntu-bionic:~$ sudo docker images | grep linux1
linux1       latest    383b59237066   32 minutes ago   58.2MB
vagrant@ubuntu-bionic:~$ sudo docker run -it 383b59237066 /bin/sh
/ # ls
bin      dev      etc      lib      lib64    linuxrc  media    mnt      opt      proc     root     run      sbin     sys      tmp      usr      var
/ # node
Welcome to Node.js v12.19.1.
Type ".help" for more information.
> console.log("Hello World")
Hello World
undefined
```
