---
title: Containers From Scratch
date: 2022-02-11 12:00:00 -500
categories: [Containerization]
tags: [containerization,docker,docker desktop]
---

At the end of the day, all a container is an isolated service with its dependencies, this document will go over how to create a container from scratch, using nothing but the built-in Linux kernel modules.

## How?
As mentioned before, a container is nothing more than an isolated service running on the linux Kernel. How it does htis is by using kernel level modules known as `namespaces`,`cgroups` and something called `capabilities`.

For a deeper explanation of containers,namespaces,cgroups etc. click here todo-add-link
## Prereqs
You will need some linux folders and structures,you could create this all totally from scratch i guess but its easier this way since youre learning about isolation not linux.

There are two ways to go about this one is using a Docker image, or rather exporting a Docker image, the second is just downloading a basic filesystem which is what we will be doing but just to be thorough i will document the docker way as well.

## Creating The Container Image
On the machine you have docker installed run

```bash
docker run -d alpine true && docker export alpine
```
You will have tar that you can put anywhere to save for later

# Creating Container
## With CHROOT
All commands are for a Debian or ubuntu distro, for RedHat change it to 'yum'

Get your tar image
```bash
wget alpine.tar.gz
```
Untar the image ( tar needs 'sudo' to create '/dev' files and setup file ownership) and check that the files are there including the binaries.

```bash
$ sudo tar -zxf alpine.tar.gz
$ ls alpine
$ ls -al alpine/bin/ls
```
We will use `chroot` (old school!) to restrict the view of the process. This is what docker does when it does 'docker run'
```bash
$ sudo chroot alpine /bin/bash
root@localhost:/# ls /
root@localhost:/# which echo
/usr/bin/echo
root@localhost:/# /usr/bin/echo "Hello world!"'
Hello world!
root@localhost:/#
```
Note: Instead of a shell we can run one in our `chroot`. Similar to dockers `docker exec -d <container> echo "Hello World"`
```bash
$ sudo chroot alpine echo "Hello World!"
```
## Isolation (namespaces)
ok now to isolate the whole thing using a syscall called [unshare](https://linux.die.net/man/2/unshare) by using the CLI tool by the same name. This will create a namespace for our container to run in. This is similar to dockers `docker run ` command.
```bash
$ sudo unshare -p -f --mount-proc=$PWD/alpine/proc \
    chroot alpine /bin/bash
 ```
 Check to make sure its PID is 1 
```bash
root@localhost:/# ps aux
```
## Using nsenter
Alternatively you can use a different utility to login to the running container, similar to what `docker exec -it`will do when you run it. 
From an alternative terminal let’s find the shell running in a chroot from our last example.
```bash
$ ps aux | grep /bin/bash | grep root
```
The kernel exposes namespaces under `/proc/(PID)/ns` as files which is the process namespace we’re hoping to join.
```bash
$ sudo ls -l /proc/29840/ns
```
The `nsenter` command provides a wrapper around `setns` to enter a namespace. We’ll provide the `namespace file`, then run the `unshare` to remount `/proc` and `chroot` to setup a `chroot`. This time, instead of creating a new namespace, our shell will join the existing one.
```bash
$ sudo nsenter --pid=/proc/<PID>/ns/pid \
    unshare -f --mount-proc=$PWD/rootfs/proc \
    chroot rootfs /bin/bash
```
## Volumes?
When deploying an “immutable” container it often becomes important to inject files or directories into the chroot, either for storage or configuration. For this example, we’ll create some files on the host, then expose them read-only to the chrooted shell using mount.

First, let’s make a new directory to mount into the chroot and create a file there.
```bash
$ sudo mkdir readonlyfiles
$ echo "hello" > readonlyfiles/hi.txt
```
Next, we’ll create a target directory in our container and bind mount the directory providing the -o ro argument to make it read-only. If you’ve never seen a bind mount before, think of this like a symlink on steroids. similar to dockers `docker -v /<host directory>:/<container directory>`command
```bash
  $ sudo mkdir -p rootfs/var/readonlyfiles
$ sudo mount --bind -o ro $PWD/readonlyfiles $PWD/rootfs/var/readonlyfiles
```
The chrooted process can now see the mounted files.
 ```bash
$ sudo chroot rootfs /bin/bash
root@localhost:/# cat /var/readonlyfiles/hi.txt
hello
```
Because it's read-only it cannot write things back from the container back to the host. But again you wouldn't want it to since its supposed to be immutable.
```bash
root@localhost:/# echo "bye" > /var/readonlyfiles/hi.txt
bash: /var/readonlyfiles/hi.txt: Read-only file system
```

Use umount to remove the bind mount (rm won’t work).
```bash
$ sudo umount $PWD/rootfs/var/readonlyfiles
```
## Further Isolation or limits (Cgroups)
cgroups, short for control groups, allow kernel-imposed isolation on resources like memory and CPU. This is so one container cant kill things in other containers by using up all the ram etc.

The kernel exposes cgroups through the `/sys/fs/cgroup` directory. If your machine doesn’t have one you may have to [mount the memory cgroup](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-memory.html#memory_example-usage) to follow along.
```bash
$ ls /sys/fs/cgroup/
```
For this example, we’ll create a cgroup to restrict the memory of a process. Creating a cgroup is easy, just create a directory. In this case, we’ll create a memory group called “demo”. Once created, the kernel fills the directory with files that can be used to configure the cgroup.

$ sudo su
# mkdir /sys/fs/cgroup/memory/demo
# ls /sys/fs/cgroup/memory/demo/
To adjust a value we just have to write to the corresponding file. Let’s limit the cgroup to 100MB of memory and turn off swap.

# echo "100000000" > /sys/fs/cgroup/memory/demo/memory.limit_in_bytes
# echo "0" > /sys/fs/cgroup/memory/demo/memory.swappiness
The tasks file is special, it contains the list of processes that are assigned to the cgroup. To join the cgroup we can write our own PID.

# echo $$ > /sys/fs/cgroup/memory/demo/tasks
Time to test:

Create a python script with this inside and run it.

f = open("/dev/urandom", "r")
data = ""
 
i=0
while True:
    data += f.read(10000000) # 10mb
    i += 1
    print "%dmb" % (i*10,)
If you’ve set up the cgroup correctly, this program won’t crash your computer.

# python script.py
10mb
20mb
30mb
40mb
50mb
60mb
70mb
80mb
Killed
cgroups can’t be removed until every process in the tasks file has exited or been reassigned to another group. Exit the shell and remove the directory with rmdir (don’t use rm -r).

DO NOT DO THIS UNLESS YOU'RE DONE WITH YOUR CONTAINER

# exit
exit
$ sudo rmdir /sys/fs/cgroup/memory/demo


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
