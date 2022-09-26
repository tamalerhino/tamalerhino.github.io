---
title: Containers From Scratch Part 1
date: 2022-02-11 12:00:00 -500
categories: [Containerization]
tags: [containerization,docker]
img_path: /assets/img/posts/
image:
  path: pexels-pixabay-73833.jpg
  width: 1000
  height: 800
  alt: Containers
---

At the end of the day, all a container is an isolated service with its dependencies, this document will go over how to create a container from scratch, using nothing but the built-in Linux kernel modules.

## How?
As mentioned before, a container is nothing more than an isolated service running on the linux Kernel. How it does this is by using kernel level modules known as `namespaces`,`cgroups` and something called `capabilities`.

For a deeper explanation of containers,namespaces,cgroups etc see my blog on Namespaces

## Prereqs
You will need some linux folders and structures, there are two ways to go about this. 
- Using a Docker image, or rather exporting a Docker image.
- Downloading a basic filesystem or creating it from scratch.(checkout my other blog on creating debian distro from scratch "Bootstrapping Debian As a Container Image")

### Option 1 Creating The Container Image
On the machine a you have Docker installed.

1. Pull the alpine image
`docker pull alpine`

2. Run the container
`docker run -d --name alpine alpine`

3. Finally export the image to a tar file.
`docker export --output=alpine.tar alpine`

You will have a tar file that you can put anywhere to save for later.
### Option 2 From Scratch 
Follow my other blog "Bootstrapping Debian As a Container Image" to get a tar.gz of a basic debian filesystem.

# Creating Container
## With CHROOT
>If you went the docker route just replace `debian` to `alpine` or whatever you named your tar file.
{: .prompt-warning }

Get your tar image using `wget` or `curl`

Untar the image and take a look around, its just like any other linux distro.

>Be adviced if you mess up the commands you will need to `su` to `root` because of permissions to delete stuff.
{: .prompt-tip}
```bash
tamalerhino@localhost:~$ sudo mkdir debian
tamalerhino@localhost:~$ sudo tar -xzf debian.tar.gz -C debian
tamalerhino@localhost:~$ ls -la debian/
tamalerhino@localhost:~$ ls -al debian/etc/
```

We will use `chroot` (old school!) to restrict the view of the process. 
>This is what docker does when it does `docker run`
{: .prompt-info }

```bash
tamalerhino@localhost:~$ sudo chroot debian /bin/bash
root@localhost:/#  ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@localhost:/#  echo "Meow"
Meow
root@localhost:/# exit
```

>Instead of a shell, we can run a command inside our `chroot`.
{: .prompt-tip }

>Similar to dockers `docker exec -d <container> echo "Hello World"`
{: .prompt-info }
```bash
tamalerhino@localhost:~$ sudo chroot debian echo "meow"
meow
tamalerhino@localhost:~$ 
```

>Now of course if you try to install something using `apt` you will get an error since it does not have connectivity to the outside. Everything were running is from within our container.
{: .prompt-warning }
>However it's not completely isolated yet!
{: .prompt-danger }

You can check this by running `top` from outside the chroot container.
```bash
tamalerhino@localhost:~$ top &
[2] 1619
```
Then run the following commands to see that you can see the top command inside the chroot container.
```bash
tamalerhino@localhost:~$ sudo chroot debian /bin/bash
root@localhost:/# mount -t proc proc /proc
```
>The `mount -t proc..` commands are needed because when you run `ps` it is checking the `/proc` filesystem for the file representation of the running processes, and so we need to mount the `/proc` global FS.
{: .prompt-info }
```bash
root@localhost:/# ps aux | grep top
1000      7725  0.0  0.0  10396  3240 ?        T    22:18   0:00 top
root      7761  0.0  0.0   8172   668 ?        S+   22:19   0:00 grep top
root@localhost:/#
```
You should not be able to see that root process inside the container.
Whats worse is you can actually kill that service from inside the container by running
```bash
root@localhost:/# pkill top
```
That defeats the purpouse of containers right?!

## Isolation (namespaces)
OK now to isolate the whole thing using a [syscall](https://en.wikipedia.org/wiki/System_call) called [unshare](https://linux.die.net/man/2/unshare) by using the CLI tool by the same name. This will create a namespace for our container to run in.

Below what we are doing is going into the debian directory, just like we would with `chroot` but now were giving ourselves specific namespaces to isolate the environment from the host environment and running `bash` as our process(the `--fork` flag is just saying we want to run bash as a child service of the unshare command.)

```bash
tamalerhino@localhost:~$ cd debian
tamalerhino@localhost:~/debian$ sudo unshare --mount --uts --fork bash
root@localhost:/home/tamalerhino/debian#
```
>When we run the `unshare` command and the namespaces eg `--mount`,`--uts` what were saying is: create an envionrment and give us our own instances of these namespaces for example `--mount` were saying we want our own mounts ie the ability to mount things only to our environement and not the host environment.
{: .prompt-info }

To test that youre in a container lets try changing the hostname
```bash
root@localhost:/home/tamalerhino/debian# hostname container
root@localhost:/home/tamalerhino/debian# exec bash
root@container:/home/tamalerhino/debian#
```
>We have the ability to change the internal hostname of the container without changing the host's hostname because we gave ourselves the `--uts` namespace earlier.
{: .prompt-info }

Open up a second terminal so you can check to see that your host system hostname has not changed!
```bash
# Second Terminal
tamalerhino@localhost:~$ hostname
localhost
```

Now to fix the issue with the fact that we can kill processes, if you were to run `ps` from inside the container right now this is what you would see:
```bash
root@container:/home/tamalerhino/debian# ps
  PID TTY          TIME CMD
10976 pts/3    00:00:00 sudo
10977 pts/3    00:00:00 unshare
10978 pts/3    00:00:00 bash
10989 pts/3    00:00:00 ps
root@container:/home/tamalerhino/debian#
```
We dont want that! The reason we are seeing that is because when we mounted everything we also added the global `/proc` directory to the container. Lets only mount our debian `/proc` directory to our namespace as well as the `--pid`, and `--ipc` namespaces.
```bash
root@container:/home/tamalerhino/debian# mount -t proc proc /proc
root@container:/home/tamalerhino/debian# ps
  PID TTY          TIME CMD
    1 pts/3    00:00:00 bash
   11 pts/3    00:00:00 ps
root@container:/home/tamalerhino/debian#
```
Now we can only see our processes.
> Another way to do this is to add the `/proc` mountpoint to our original `unshare` command ` sudo unshare --mount --uts --ipc --pid --mount-proc=/proc --fork bash`
{: .prompt-tip }

Lets go ahead and take it one step further by telling the system that our root `/` directory is at the debian directory, we can do this by using `pivot_root`.

>From the man page: "pivot_root() changes the root mount in the mount namespace of the calling process. More precisely, it moves the root mount to the directory put_old and makes new_root the new root mount."
{: .prompt-info }
First we need to create a directory where our current root filesystem will be mounted.
>Before you being create a file in your debian directory so you know that you are actually in your deban root directory later on like so `touch /home/tamalerhino/debian/DEEEEEEB`
{: .prompt-tip }

First we need to make a new directory close to our `/` root FS now.
```bash
root@container:/home/tamalerhino/debian# mkdir /newroot
```
Then we will bind mount our debian root fs to this newroot directory.
```bash
root@container:/home/tamalerhino/debian# mount --bind /home/tamalerhino/debian /newroot
```

Now we can go to the root FS and show that the `newroot` directory is there and our debian fs is mounted there.
```bash
root@container:/home/tamalerhino/debian# cd /
root@container:/# ls
bin   dev  home  lib32  libx32      media  newroot  proc  run   snap  swap.img  tmp  var
boot  etc  lib   lib64  lost+found  mnt    opt      root  sbin  srv   sys       usr
root@container:/# cd newroot/
root@container:/newroot# ls
DEEEEEEB  boot  etc   lib    lib64   media  opt   root  sbin  sys  usr
bin       dev   home  lib32  libx32  mnt    proc  run   srv   tmp  var
```

Now comes the cool part we will use the command `pivot_root` to tell the system that our root FS is no longer the `/` but really now its `/newroot` and to treat it as such.
```bash
root@container:/newroot# pivot_root . .
root@container:/newroot# cd /
root@container:/# ls
DEEEEEEB  boot  etc   lib    lib64   media  opt   root  sbin  sys  usr
bin       dev   home  lib32  libx32  mnt    proc  run   srv   tmp  var
```
See! now everytime we use `/` it points to our debian FS.


### Now to do a little cleanup


This is a good stoping point, at this point we have a container that only sees its own processes, its own mounts and its own rootfs but we still need immutable volumes, networking and a little security. All this and much more to come in Part 2!

First run the following command since mount relies on this.
```bash
root@container:/# mount -t proc proc /proc
```

Now to see whats mounted. Only reason im running head is to get the first 10 lines otherwise it would be too much to render for the blog.
```bash
root@container:/# mount | head
/dev/mapper/ubuntu--vg-ubuntu--lv on / type ext4 (rw,relatime)
udev on /dev type devtmpfs (rw,nosuid,relatime,size=1936540k,nr_inodes=484135,mode=755,inode64)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,inode64)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,pagesize=2M)
tmpfs on /run type tmpfs (rw,nosuid,nodev,noexec,relatime,size=398444k,mode=755,inode64)
tmpfs on /run/lock type tmpfs (rw,nosuid,nodev,noexec,relatime,size=5120k,inode64)
none on /run/credentials/systemd-sysusers.service type ramfs (ro,nosuid,nodev,noexec,relatime,mode=700)
tmpfs on /run/snapd/ns type tmpfs (rw,nosuid,nodev,noexec,relatime,size=398444k,mode=755,inode64)
```
Go ahead and unmount everything
```bash
root@container:/# umount -a
umount: /: target is busy.
...
```
>If you get errors like the one above saying something is busy just run `umount -l <the mount thats busy>` in my case `umount -l /`
{: .prompt-tip }

Now when we run mount we see we get a nice little isolated container mounts.
```bash
root@container:/# mount
/dev/mapper/ubuntu--vg-ubuntu--lv on / type ext4 (rw,relatime)
proc on /proc type proc (rw,relatime)
proc on /proc type proc (rw,relatime)
```

# Resources
- https://ericchiang.github.io/post/containers-from-scratch/
- https://lwn.net/Articles/531114/
- https://blog.nicolasmesa.co/posts/2018/08/container-creation-using-namespaces-and-bash/
- https://www.redhat.com/sysadmin/net-namespaces
- https://en.wikipedia.org/wiki/Linux_namespaces
- https://wiki.archlinux.org/title/Linux_Containers
