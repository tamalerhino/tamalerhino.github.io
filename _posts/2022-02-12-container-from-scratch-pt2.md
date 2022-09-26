---
title: Containers From Scratch Part 2
date: 2022-02-12 12:00:00 -500
categories: [Containerization]
tags: [containerization,docker]
img_path: /assets/img/posts/
image:
  path: pexels-pixabay-73833.jpg
  width: 1000
  height: 800
  alt: Containers
---

In the first part we created a container in the simplest form, using `namespaces`, `chroot`, and a little `pivot_root` magic to isolate our service. But there is still much more to do...

## Further Isolation or limits (cgroups)
cgroups, short for control groups, allow kernel-imposed isolation on resources like memory and CPU. This is so one container cant kill things in other containers by using up all the ram etc.

The kernel exposes cgroups through the `/sys/fs/cgroup` directory. If your machine doesn’t have one you may have to [mount the memory cgroup](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-memory.html#memory_example-usage) to follow along.

>This is only to show how cgroups work, in order to add the container to a cgroup you just need to add the `PID` of `unshare` to the`cgroup.procs` file.
{: .prompt-info }


>Run the following commands in a secondary terminal outside of your container.
{: .prompt-danger }

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

In oder to create a cgroup all you need to do is create a directory. we will create one to limit the memory and we will call the folder "meow". Once the directory has been created, the kernel automatically creates the nesseasary files to configure the cgroup itself.

```bash
tamalerhino@localhost:~$ sudo mkdir /sys/fs/cgroup/meow
tamalerhino@localhost:~$ ls /sys/fs/cgroup/meow/
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
```
To configure the memory value we just need to write to the correspoding file. We will limit it to 100MB of memory.
```bash
tamalerhino@localhost:~$ sudo su #sudo wont work for some reason
root@localhost:/home/tamalerhino# echo "100000000" > /sys/fs/cgroup/meow/memory.max
root@localhost:/home/tamalerhino# echo "0" > /sys/fs/cgroup/meow/memory.swap.max
```
In order to assign our process to that cgroup we will need to edit the `cgroup.procs` file. By adding our PID into this file.

```bash
root@localhost:/home/tamalerhino# echo $$ > /sys/fs/cgroup/meow/cgroup.procs
```
Lets test cgroup by running this script stolen from [here](https://ericchiang.github.io/post/containers-from-scratch/#cgroups)
I just named the file crash.py
```python
f = open("/dev/urandom", "r")
data = ""

i=0
while True:
    data += f.read(10000000) # 10mb
    i += 1
    print "%dmb" % (i*10,)
```

If you’ve set up the cgroup correctly, this  won’t crash your computer.

```bash
root@localhost:/home/tamalerhino# python2 crash.py
10mb
20mb
30mb
40mb
50mb
60mb
70mb
80mb
Killed
root@localhost:/home/tamalerhino#
```
As you can see it reached that limit and exited my process ie the bash process.

cgroups can’t be removed until every process in the procs has exited or been reassigned to another group. 

Remove the directory with rmdir (don’t use rm - or you will get the `operation not permited` errors).

>DO NOT DO THIS UNLESS YOU'RE DONE
{: .prompt-danger }

```bash
root@localhost:/home/tamalerhino# exit
exit
tamalerhino@localhost:~$ sudo rmdir /sys/fs/cgroup/meow
```

## Networking
Now to get some networking in the container so that we can do stuff.

Start by creating a network namesapce
```bash
tamalerhino@localhost:~/debian$ sudo ip netns add vnet0
tamalerhino@localhost:~/debian$ sudo ip netns
vnet0
```

You can view the ip links in the namespace created as well by running
```bash
tamalerhino@localhost:~$ sudo ip netns exec vnet0 ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
Or you can use the shortcut for `netns exec` just as `-n` like so:
```bash
tamalerhino@localhost:~$ sudo ip -n vnet0 link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Now we will create a network bridge/switch
```bash
sudo
tamalerhino@localhost:~/debian$ sudo ip link add v-net-0 type bridge
tamalerhino@localhost:~$ sudo ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:49:53:03 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
8: v-net-0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether 12:d5:3c:e1:f8:e6 brd ff:ff:ff:ff:ff:ff
```
And we will bring the network bridge "UP"
```bash
tamalerhino@localhost:~$ sudo ip link set dev v-net-0 up
```

Now we will connect the namespace to this swich.
Begin by creating the link pipe or 'cable'
```bash
tamalerhino@localhost:~$ sudo ip link add veth0 type veth peer name veth0-br
```
Then attach it to the namespace
```bash
tamalerhino@localhost:~$ sudo ip link set veth0 netns vnet0
```
and then the virtual switch
```bash
tamalerhino@localhost:~$ sudo ip link set veth0-br master v-net-0
```

Now lets give the namespace an ip address
```bash
tamalerhino@localhost:~$ sudo ip -n vnet0 addr add 192.168.15.1/24 dev veth0
```
And turn all the devices to "UP"
```bash
tamalerhino@localhost:~$ sudo ip -n vnet0 link set veth0 up
tamalerhino@localhost:~$ sudo ip link set veth0-br up
```
Now to check we can run the following commands
```bash
tamalerhino@localhost:~$ sudo ip -n vnet0 link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
10: veth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 6a:d8:d4:88:d0:2c brd ff:ff:ff:ff:ff:ff link-netnsid 0

tamalerhino@localhost:~$ sudo ip -n vnet0 addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
10: veth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 6a:d8:d4:88:d0:2c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.15.1/32 scope global veth0
       valid_lft forever preferred_lft forever
    inet6 fe80::68d8:d4ff:fe88:d02c/64 scope link
       valid_lft forever preferred_lft forever
```
Now this is good we have a network stack in the cointainer itself but it cannot access anything outside of it, and the host also cannot access the container inside. 

So we will go ahead and connect our host to that switch for interconnectivity first. 
```bash
tamalerhino@localhost:~$ sudo ip addr add 192.168.15.5/24 dev v-net-0
```
You might also need to put in a policy to allow network forwarding
```bash
tamalerhino@localhost:~$ sudo iptables --policy FORWARD ACCEPT
```

Now to test!
```bash
tamalerhino@localhost:~$ ping 192.168.15.1
PING 192.168.15.1 (192.168.15.1) 56(84) bytes of data.
64 bytes from 192.168.15.1: icmp_seq=1 ttl=64 time=0.088 ms
.....

```

It works!!

However all you got rght now is connectivity from the host to the container, now to add connectivity from the container to the host.
We can do that by adding a route to tell your container to forward the traffic through the virtual switch.
Make sure to replace `192.168.22.0/34` with your hosts IP
```bash
tamalerhino@localhost:~$ sudo ip netns exec vnet0 ip route add 192.168.22.0/24 via 192.168.15.5
```
Now test it
```bash
tamalerhino@localhost:~$ sudo ip netns exec vnet0 ping 192.168.22.144
PING 192.168.22.144 (192.168.22.144) 56(84) bytes of data.
64 bytes from 192.168.22.144: icmp_seq=1 ttl=64 time=0.029 ms
64 bytes from 192.168.22.144: icmp_seq=2 ttl=64 time=0.064 ms
```
Great that works! But what if you wanted to allow other things from outside of the host to connect to your container? For example if you were hosting a web service.

We will essentially need to enable NATing on the host acting as the gateway to allow the container send and receive traffic with its own name and address.

Again first lets enable traffic from the container to the outside world by adding a nat rule in our IP tables and a new route.
```bash
tamalerhino@localhost:~$ sudo iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
tamalerhino@localhost:~$ sudo ip netns exec vnet0 ip route add default via 192.168.15.5
```

And now finally adding port forwarding! Below is just an example. Since we are not running anything on port 80 at the moment you wont be able to hit anything.
```bash
tamalerhino@localhost:~$ tamalerhino@localhost:~$ sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.15.1:80
```

Below you can see how you can enter the namespace with the networking we just setup.
```bash
sudo unshare --mount --uts --net=/var/run/netns/vnet0 --ipc --pid --mount-proc=/proc --fork bash
```

There is still so much to learn, in order to learn about container security I recommend you read into Linux Capabilites, Seccomp AppArmor, and of course good ol' SELinux.

# Resources
- https://ericchiang.github.io/post/containers-from-scratch/
- https://iximiuz.com/en/posts/container-networking-is-simple/
- https://gist.github.com/cfra/39f4110366fa1ae9b1bddd1b47f586a3
- https://man7.org/linux/man-pages/man8/ip-netns.8.html
- https://stackoverflow.com/questions/67971506/use-unshare-to-start-process-in-existing-net-namespace


