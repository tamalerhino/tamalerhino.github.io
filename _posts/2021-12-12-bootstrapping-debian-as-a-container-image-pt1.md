---
title: Bootstrapping Debian as a Container Image
date: 2021-12-12 12:00:00 -500
categories: [Containerization]
tags: [containerization,docker,linux]
img_path: /assets/img/posts/
image:
  path: pexels-pixabay-73833.jpg
  width: 1000
  height: 800
  alt: Containers
---

In my research to get away from docker and docker images i needed a way to learn to create a conatiner image. After all a container image is just a filesystem that has been neatly packaged. In the docker world its just packaged using the fslayer driver, meaning it takes the artifact and cuts it up into layers for easy packaging. And when docker runs it copies all the layers as a read only artifact then adds a top writeable layer which is why you can login and add users,install packages, etc.

# Prereqs
Since we will be creating a debian filesystem you will need a tool called `debootstrap`.
```bash
apt install debootstrap
```
Im also assuming you are creating this on a Debian or Ubuntu distro.

# Bootstrapping The Filesystem
Begin by creating your directory where to put your FS.
```bash
mkdir debian
```

Then run the following commands to download and initialize that folder(replace `bullseye` with whatever version of debian you want).
```bash
debootstrap bullseye debian/
```

Once that is done you will get the following message:
```bash
I: Base system installed successfully.
```

At this point you have a full blown debian filesystem with all the directories needed. But its not all the way configured lets add a few files to make it easier on ourselves.

# Additional Configs!

## Apt repos

First create the `sources.list` file then copy and paste the contents below into it. For a full list [click here](https://wiki.debian.org/SourcesList)
```bash
vim debian/etc/apt/sources.list
```
```bash
deb http://deb.debian.org/debian bullseye main
deb-src http://deb.debian.org/debian bullseye main

deb http://deb.debian.org/debian-security/ bullseye-security main
deb-src http://deb.debian.org/debian-security/ bullseye-security main

deb http://deb.debian.org/debian bullseye-updates main
deb-src http://deb.debian.org/debian bullseye-updates main
```

## Resolve.conf
You will need to make sure you have propper DNS servers in your resolv.conf file otherwise you wont be able to download anything right away.
If you dont know what to put just put copy and paste this into the contents of `debian/etc/resolv.conf`
```bash
nameserver 8.8.4.4
nameserver 8.8.8.8
```
>From here on out you will need to be in the FS itself so we will use `chroot` to do so, do not skip this step!
{: .prompt-danger }
```bash
tamalerhino@localhost:~$ sudo chroot debian/
root@localhost:/#
```

## Install additional packages
If you want install some additional packages now.
```bash
apt install vim sudo curl wget ntp network-manager -y
```

## Set Locales and timezone
Install the locales package
```bash
apt install locales -y
```
Configure your language and locale
```bash
dpkg-reconfigure locales
```
Choose your region from the list, for US it should be `en_US.UTF-8 UTF-8.`

Set the timzone by running the following command and choosing your Country and timezone location.
```
dpkg-reconfigure tzdata
```

## Change the root password
Its best if you know the root password incase you need it later, here im going to change it to `debian`
```bash
passwd
```

## Create a User
You might want to go ahead and create a non root user as well.
```bash
useradd -mG sudo tamalerhino
passwd tamalerhino
```

## Hostname and hosts file
Finally if you want you can change the hostname.
```bash
echo debian-bullseye > /etc/hostname
```
And if you want to edit your hosts file
```bash
echo '127.0.1.1  debian-bullseye.localdomain  debian-bullseye' >> /etc/hosts
```

Then exit.
```bash
exit
```

# Package it up!
No that you have your filesystem its time to package it up to be used later.
```bash
tar -czf debian.tar.gz debian/
```
You should end up with a 171M file which you can use to run with systemd-namespace containers or from scratch calling on chroot and namespace.