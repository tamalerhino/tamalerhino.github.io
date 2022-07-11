---
title: Containers From Scratch
date: 2022-02-11 12:00:00 -500
categories: [Containerization]
tags: [containerization,docker,docker desktop]
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

For a deeper explanation of containers,namespaces,cgroups etc. click here todo-add-link

## Prereqs
You will need some linux folders and structures, there are two ways to go about this. 
- Using a Docker image, or rather exporting a Docker image(we will be using this method since youre trying to learn about isolation ie: containers, not building a linux distros)
- downloading a basic filesystem or creating it from scratch.

## Creating The Container Image
On the machine a you have Docker installed.

Pull the alpine image
`docker pull alpine`

Run the container
`docker run -d --name alpine alpine`

Finally export the image to a tar file.
`docker export --output=alpine.tar alpine`

You will have tar that you can put anywhere to save for later

# Creating Container
## With CHROOT
All commands are for a Debian or ubuntu distro, for RedHat change it to `yum`
Get your tar image
`wget alpine.tar`

Untar the image and take a look around, its just like any other linux distro.
Be adviced if you mess up the commands you will need to `su` to `root` because of permissions to delete stuff.
```bash
tamalerhino@localhost:~$ sudo mkdir alpine
tamalerhino@localhost:~$ sudo tar -xf alpine.tar -C alpine
tamalerhino@localhost:~$ ls -la alpine/
tamalerhino@localhost:~$ ls -al alpine/etc/
```

We will use `chroot` (old school!) to restrict the view of the process. This is what docker does when it does `docker run`
```bash
tamalerhino@localhost:~$ sudo chroot alpine /bin/sh
/ # ls
bin    etc    lib    mnt    proc   run    srv    tmp    var
dev    home   media  opt    root   sbin   sys    usr
/ # echo "Meow"
Meow
/ #
```
Note: Instead of a shell, we can run a command inside our `chroot`. Similar to dockers `docker exec -d <container> echo "Hello World"`
```bash
tamalerhino@localhost:~$ sudo chroot alpine echo "meow"
meow
tamalerhino@localhost:~$
```

Now of course if you try to install something uing `apk add` you will get an error since it does not have connectivity to the outside. Everything were running is from within our container.

However it's not completely isolated yet!
You can check this by running `top` from outside the chroot container then run the following commands to see that you can see the top command inside the chroot container.
```bash
tamalerhino@localhost:~$ top &
[2] 1619
tamalerhino@localhost:~$ sudo chroot alpine /bin/sh
/ # mount -t proc proc /proc
/ # ps aux | grep top
 1606 1000      0:00 top
 1619 1000      0:00 top
 1626 root      0:00 grep top
/ #
```
You should not be able to see that root process inside the container.
Whats worse is you can actually kill that service from inside the container by running
```bash
/ # pkill top
```
That defeats the purpouse of containes right?!

## Isolation (namespaces)
ok now to isolate the whole thing using a syscall called [unshare](https://linux.die.net/man/2/unshare) by using the CLI tool by the same name. This will create a namespace for our container to run in. This is similar to dockers `docker run ` command.

```bash
tamalerhino@localhost:~$ cd alpine
tamalerhino@localhost:~/alpine$ sudo unshare --mount --uts --ipc --net --pid --fork bash
root@localhost:/home/tamalerhino/alpine#
```

Just to test that youre in a container lets try changing the hostname
```bash
root@localhost:/home/tamalerhino/alpine# hostname container
root@localhost:/home/tamalerhino/alpine# exec bash
root@container:/home/tamalerhino/alpine#
```
Open up a second terminal so you can check to see that your host system hostname has not changed!


Now to fix the issue with the fact that we can kill processes, if you were to run `ps` from inside the container right now this is what you would see:
```bash
root@container:/home/tamalerhino/alpine# ps
    PID TTY          TIME CMD
   2065 pts/1    00:00:00 sudo
   2066 pts/1    00:00:00 unshare
   2067 pts/1    00:00:00 bash
   2109 pts/1    00:00:00 ps
root@container:/home/tamalerhino/alpine#
```
We dont want that! The reason we are seeing that is because when we mounted everything we also added the global /proc directory to the container. Lets only add our alpine /proc directory to our namespace
```bash
root@container:/home/tamalerhino/alpine# mount -t proc proc /proc
root@container:/home/tamalerhino/alpine# ps
    PID TTY          TIME CMD
      1 pts/1    00:00:00 bash
     16 pts/1    00:00:00 ps
root@container:/home/tamalerhino/alpine#
```
Now we can only see our processes.

Lets go ahead and take it one step further by telling the system that our root `/` directory is at the alpine directory, we can do this by using `pivot_root`. 
From the man page: "pivot_root() changes the root mount in the mount namespace of the calling process. More precisely, it moves the root mount to the directory put_old and makes new_root the new root mount."



## Volumes?
For any good use case directories or volumes are needed, thats how you can run immutable containers with your own files. Lets create a few files and show you how to mount them.

First, let’s make a new directory and `touch` a new file inside.

```bash
tamalerhino@localhost:~$ mkdir mahfiles
tamalerhino@localhost:~$ echo "meow" > mahfiles/meow.txt
```

Next, we’ll create a  directory in our container and bind mount the directory providing the -o ro argument to make it read-only. Similar to dockers `docker -v /<host directory>:/<container directory>`command.

```bash
tamalerhino@localhost:~$ sudo mkdir -p alpine/tmp/mahfiles
tamalerhino@localhost:~$ sudo mount --bind -o ro $PWD/mahfiles $PWD/alpine/tmp/mahfiles
```

The chrooted process can now see the mounted files.

 ```bash
tamalerhino@localhost:~$ sudo chroot alpine /bin/sh
/# cat /tmp/mahfiles/hi.txt
hello
```

Because it's read-only it cannot write things back from the container back to the host. But again you wouldn't want it to since its supposed to be immutable.

```bash
tamalerhino@localhost:~$ sudo chroot alpine /bin/sh
/ # echo "woof" > /tmp/mahfiles/woof.txt
/bin/sh: can't create /tmp/mahfiles/woof.txt: Read-only file system
/ #
```

Use umount to remove the bind mount (rm won’t work).
```bash
tamalerhino@localhost:~$ sudo umount $PWD/alpine/tmp/mahfiles/
```

## Further Isolation or limits (Cgroups)
cgroups, short for control groups, allow kernel-imposed isolation on resources like memory and CPU. This is so one container cant kill things in other containers by using up all the ram etc.

The kernel exposes cgroups through the `/sys/fs/cgroup` directory. If your machine doesn’t have one you may have to [mount the memory cgroup](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-memory.html#memory_example-usage) to follow along.

```bash
tamalerhino@localhost:~$ ls /sys/fs/cgroup/
cgroup.controllers      cpu.stat               io.pressure                    sys-kernel-config.mount
cgroup.max.depth        cpuset.cpus.effective  io.prio.class                  sys-kernel-debug.mount
cgroup.max.descendants  cpuset.mems.effective  io.stat                        sys-kernel-tracing.mount
cgroup.procs            dev-hugepages.mount    memory.numa_stat               system.slice
cgroup.stat             dev-mqueue.mount       memory.pressure                user.slice
cgroup.subtree_control  init.scope             memory.stat
cgroup.threads          io.cost.model          misc.capacity
cpu.pressure            io.cost.qos            sys-fs-fuse-connections.mount
```

For this example, we’ll create a cgroup to restrict the memory of a process. 

In oder to creat a cgroup all youy need to do is create a directory. we will create one to limit the memory and we will call the folder "meow. Once the directory has been created, the kernel automatically creates the nesseasary files to configure the cgroup itself.

```bash
tamalerhino@localhost:~$ sudo su
root@localhost:/home/tamalerhino# mkdir /sys/fs/cgroup/meow
root@localhost:/home/tamalerhino# ls /sys/fs/cgroup/meow
cgroup.controllers      cpu.max                cpuset.mems.effective  memory.min
cgroup.events           cpu.max.burst          io.max                 memory.numa_stat
cgroup.freeze           cpu.pressure           io.pressure            memory.oom.group
cgroup.kill             cpu.stat               io.prio.class          memory.pressure
cgroup.max.depth        cpu.uclamp.max         io.stat                memory.stat
cgroup.max.descendants  cpu.uclamp.min         io.weight              memory.swap.current
cgroup.procs            cpu.weight             memory.current         memory.swap.events
cgroup.stat             cpu.weight.nice        memory.events          memory.swap.high
cgroup.subtree_control  cpuset.cpus            memory.events.local    memory.swap.max
cgroup.threads          cpuset.cpus.effective  memory.high            pids.current
cgroup.type             cpuset.cpus.partition  memory.low             pids.events
cpu.idle                cpuset.mems            memory.max             pids.max
root@localhost:/home/tamalerhino#
```
To configure the memory value we just need to write to the correspoding file. We will limit it to 100MB of memory.
```bash
root@localhost:/home/tamalerhino# echo "100000000" > /sys/fs/cgroup/meow/memory.max
```
In order to assign our process to that cgroup we will need to edit the `cgroup.procs` file. By adding our PID into this file.

```bash
root@localhost:/home/tamalerhino# echo $$ > /sys/fs/cgroup/meow/cgroup.procs
```
Lets test cgroup by running a quick python script.
```python
f = open("/dev/urandom", "r")
data = ""
 
i=0
while True:
    data += f.read(10000000) # 10mb
    i += 1
    print "%dmb" % (i*10,)
```

If you’ve set up the cgroup correctly, this program won’t crash your computer.

```bash
root@localhost:/home/tamalerhino# python2.7 testing.py
10mb
20mb
30mb
40mb
50mb
60mb
70mb
80mb
Killed
```

cgroups can’t be removed until every process in the procs has exited or been reassigned to another group. 

Exit out of the current shell and remove the directory with rmdir (don’t use rm - or youull get the `operation not permited` errors).

DO NOT DO THIS UNLESS YOU'RE DONE WITH YOUR CONTAINER

```bash
root@localhost:/home/tamalerhino# exit
tamalerhino@localhost:~$ sudo rmdir /sys/fs/cgroup/meow
```


## Networking
Now to get some networking in the container so that we can do stuff you can do this by pairing the interfaces from the host to the container.

Start by creating the interface on the host

$ sudo ip netns add netns0
$ ip netns
netns0
With this single command, we just create a pair of interconnected virtual Ethernet devices. The names veth0 and ceth0 have been chosen arbitrarily:

$ sudo ip link add veth0 type veth peer name ceth0
Both veth0 and ceth0 after creation resides on the host's network stack (also called root network namespace). To connect the root namespace with the netns0 namespace, we need to keep one of the devices in the root namespace and move another one into the netns0:

$ sudo ip link set ceth0 netns netns0
Once we turn the devices on and assign proper IP addresses, any packet occurring on one of the devices will immediately pop up on its peer device connecting two namespaces. Let's start from the root namespace:

$ sudo ip link set veth0 up
$ sudo ip addr add 172.18.0.11/16 dev veth0
And continue with the netns0:

$ sudo nsenter --net=/var/run/netns/netns0
$ ip link set lo up  # whoops
$ ip link set ceth0 up
$ ip addr add 172.18.0.10/16 dev ceth0
$ ip link
Check connectivity from `netns0`, ping root's veth0 and exit

$ ping -c 2 172.18.0.11
$ exit
Check connectivity from root namespace, ping ceth0

$ ping -c 2 172.18.0.10
However, at this time you cant access anything to the outside so let's add some routes.

For more info click here: https://iximiuz.com/en/posts/container-networking-is-simple/ 

# Container Security

Capabilities
Containers are extremely effective ways of running arbitrary code from the internet as root, and this is where the low overhead of containers hurts us. Containers are significantly easier to break out of than a VM. As a result, many technologies used to improve the security of containers, such as SELinux, seccomp, and capabilities involve limiting the power of processes already running as root.

In this section, we’ll be exploring Linux capabilities.

Giving Capabilities
Create the following  Go program which attempts to listen on port 80.

listen.go
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
package main
 
import (
    "fmt"
    "net"
    "os"
)
 
func main() {
    if _, err := net.Listen("tcp", ":80"); err != nil {
        fmt.Fprintln(os.Stdout, err)
        os.Exit(2)
    }
    fmt.Println("success")
}


What happens when we compile and run this?

$ go build -o listen listen.go
$ ./listen
listen tcp :80: bind: permission denied
Predictably this program fails; listing on port 80 requires permissions we don’t have. Of course, we can just use sudo, but we’d like to give the binary just the one permission to listen on lower ports.

Capabilities are a set of discrete powers that together make up everything root can do. This ranges from setting the system clock to killing arbitrary processes. In this case, CAP_NET_BIND_SERVICE allows executables to listen on lower ports.

We can grant the executable CAP_NET_BIND_SERVICE using the setcap command.

$ sudo setcap cap_net_bind_service=+ep listen
$ getcap listen
listen = cap_net_bind_service+ep
$ ./listen
success
Taking away Capabilities
For things already running as root, like most containerized apps, we’re more interested in taking capabilities away than granting them. First, let’s see all powers our root shell has:

$ sudo su
# capsh --print
Current: = cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,37+ep
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,37
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=0(root)

Yeah, that’s a lot of capabilities.

As an example, we’ll use capsh to drop a few capabilities including CAP_CHOWN. If things work as expected, our shell shouldn’t be able to modify file ownership despite being root.

$ sudo capsh --drop=cap_chown,cap_setpcap,cap_setfcap,cap_sys_admin --chroot=$PWD/rootfs --
root@localhost:/# whoami
root
root@localhost:/# chown nobody /bin/ls
chown: changing ownership of '/bin/ls': Operation not permitted
Conventional wisdom still states that VMs isolation is mandatory when running untrusted code. But security features like capabilities are important to protect against hacked applications running in containers.

Beyond more elaborate tools like seccomp, SELinux, and capabilities, applications running in containers generally benefit from the same kind of best practices as applications running outside of one. Know what your linking against, don’t run as root in your container, update for known security issues in a timely fashion.


# Resources
- https://ericchiang.github.io/post/containers-from-scratch/
- https://lwn.net/Articles/531114/
- https://blog.nicolasmesa.co/posts/2018/08/container-creation-using-namespaces-and-bash/
- https://www.redhat.com/sysadmin/net-namespaces
- https://en.wikipedia.org/wiki/Linux_namespaces
- https://wiki.archlinux.org/title/Linux_Containers
