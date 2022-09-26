---
title: Systemd Namespace Containers
date: 2022-03-05 12:00:00 -500
categories: [Containerization]
tags: [containerization,docker,systemd]
img_path: /assets/img/posts/
image:
  path: pexels-pixabay-262353.jpg
  width: 1000
  height: 800
  alt: Containers
---

In my journey to demystify containers and get away from a product and run containers with nothing but my Linux laptop i decided that Containers From Scratch were great, but what if it was easier to manage your container once its created? It is! And you can do it all with nothing but systemd!

# Prerequisites
Have `machinectl` installed. This package provides `systemd` tools for `nspawn` and container/VM management: `* systemd-nspawn` `* systemd-machined` and `machinectl * systemd-importd`

If its not installed install it the following ways:

## Ubuntu/Debian/Kali/Raspbian
```bash
apt install systemd-container
```
## RHEL/CentOS
```bash
yum install systemd
```
## Fedora
```bash
dnf install systemd-container
```
## Arch
```bash
pacman -S systemd
```

# Container Images
There are multiple ways to create/get a container image.
- From scratch ie `bootstrap`
- From a already made tar or img file such as a Ubuntu's cloudimage or from [snpawn.org](https://nspawn.org/)
- From a docker image/container

## From scratch
See my blog "Bootstrapping Debian As a Container Image".

## From a ready to go image
There are several cloud images out there here are the ones for ubuntu:
-  https://cloud-images.ubuntu.com/kinetic/current/kinetic-server-cloudimg-arm64.tar.gz
Theres also this great nspawn image hub (similar to docker hub) for namespace container images!
- https://hub.nspawn.org/images/

>This is what we will use, since its the fastest and simplest method
{: .prompt-info }

## From a Docker container/image
This not work in most cases since you will need to create an init script on the container to start your services. But it is a container it just wont run anything automatically until you login to the container and run it manually.

First find the docker image you want from docker hub - in my example i have chosen dokuwiki.
Second pull the image down.
```bash
docker pull dokuwiki
```
Third run the container how you would like it to run.
```bash
docker run -d \
  --name=dokuwiki \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=America/Chicago \
  -p 80:80 \
  --restart unless-stopped \
  lscr.io/linuxserver/dokuwiki:latest
```
Finally export the image to a tar file that can be used with systemd-namespace containers.
```bash
docker export --output=dokuwiki.tar dokuwiki
```
![exporting a docker image](exporting-docker-image.png)
# Downloading or Oulling the Image

## wget/curl
If you got your image hosted somewhere on the internet just use `wget` or `curl` to download it.

## Pulling an "image" using machinectl

### tarball image
The below example uses the "ready to go image" example
```bash
machinectl pull-tar https://hub.nspawn.org/storage/centos/8/tar/image.tar.xz
```
### img/raw image
The below example uses the "ready to go image" example
```bash
machinectl pull-raw https://hub.nspawn.org/storage/centos/8/raw/image.raw.xz
```
Example:
```bash
tamalerhino@localhost:~$ sudo machinectl pull-raw https://hub.nspawn.org/storage/centos/8/raw/image.raw.xz --verify=no
Enqueued transfer job 1. Press C-c to continue download in background.
Pulling 'https://hub.nspawn.org/storage/centos/8/raw/image.raw.xz', saving as 'image'.
HTTP request to https://hub.nspawn.org/storage/centos/8/raw/image.roothash failed with code 404.
Root hash file could not be retrieved, proceeding without.
HTTP request to https://hub.nspawn.org/storage/centos/8/raw/image.roothash.p7s failed with code 404.
Root hash signature file could not be retrieved, proceeding without.
Downloading 381.8M for https://hub.nspawn.org/storage/centos/8/raw/image.raw.xz.
HTTP request to https://hub.nspawn.org/storage/centos/8/raw/image.verity failed with code 404.
Verity integrity file could not be retrieved, proceeding without. https://hub.nspawn.org/storage/centos/8/raw/image.verity
Downloading 32B for https://hub.nspawn.org/storage/centos/8/raw/image.nspawn.
Download of https://hub.nspawn.org/storage/centos/8/raw/image.nspawn complete.
Got 1% of https://hub.nspawn.org/storage/centos/8/raw/image.raw.xz. 1min 49s left at 3.4M/s.
........
Got 99% of https://hub.nspawn.org/storage/centos/8/raw/image.raw.xz. 284ms left at 13.1M/s.
Download of https://hub.nspawn.org/storage/centos/8/raw/image.raw.xz complete.
Created new local image 'image'.
Created new file /var/lib/machines/image.nspawn.
Operation completed successfully.
Exiting.
tamalerhino@localhost:~$
```
### Troubleshooting
You might run into an issue where it wants to be verified, in that case run:
```bash
--verify=no
```

## Importing a local image

### tarball image
```bash
machinectl import-tar dokuwiki.tar
```
### img/raw image
```bash
machinectl import-raw dokuwiki.img
```
## View the installed images.
```bash
machinectl list-images --all
machinectl show-image <imagename>
```
>When i downloaded my centos image it named it `image` for some reason, so just replace that with whatever it named yours.
{: .prompt-info }
Example:
```bash
tamalerhino@localhost:~$ sudo machinectl list-images
NAME  TYPE RO USAGE CREATED                     MODIFIED
image raw  no  3.2G Sun 2022-08-14 02:41:35 UTC Sun 2022-08-14 02:41:35 UTC

1 images listed.
tamalerhino@localhost:~$ sudo machinectl show-images image
Unknown command verb show-images.
tamalerhino@localhost:~$ sudo machinectl show-image image
Name=image
Path=/var/lib/machines/image.raw
Type=raw
ReadOnly=no
CreationTimestamp=Sun 2022-08-14 02:41:35 UTC
ModificationTimestamp=Sun 2022-08-14 02:41:35 UTC
Usage=3489710080
Limit=3489701888
UsageExclusive=3489710080
LimitExclusive=3489701888
tamalerhino@localhost:~$
```
# Starting The Container
There are two main ways to start the container, using `machinectl` or `nspawn`,which is like `chroot` but on steroids, sometimes `machinectl` wont start it so use `nspawn`.

## machinectl
```bash
tamalerhino@localhost:~$ sudo nspawn -i centos/8/raw
```
>When i downloaded my centos image it named it `image` for some reason, so just replace that with whatever it named yours.
{: .prompt-info }
Example:
```bash
tamalerhino@localhost:~$ machinectl start image
tamalerhino@localhost:~$ sudo machinectl login image
Connected to machine image. Press ^] three times within 1s to exit session.

CentOS Linux 8
Kernel 5.15.0-46-generic on an x86_64

centos8 login: root
Password:
[root@centos8 ~]# cat /etc/redhat-release
CentOS Linux release 8.5.2111
[root@centos8 ~]#
```

## nspawn
```bash
sudo systemd-nspawn -M <image>
```
Example:
```bash
tamalerhino@localhost:~$ sudo systemd-nspawn -M image
Ignoring Capability= setting, file /var/lib/machines/image.nspawn is not trusted.
Spawning container image on /var/lib/machines/image.raw.
Press ^] three times within 1s to kill container.
[root@image ~]# cat /etc/redhat-release
CentOS Linux release 8.5.2111
[root@image ~]#
```

## Using a directory
Depending if you want a shell or not it might be best to untar it and run it with the `--boot` `-b` flag to let systemd run systemd inside of the container.

>This is assuming you used the docker tar image or some image that is using a directory
{: .prompt-info }

```bash
systemd-nspawn -bD dokuwiki/
```

# Clean Up
## Stopping an image
```bash
machinectl stop image
```
## Deleting the images
Just as an FYI the default directory where all of this gets downloaded to is `/var/lib/machines`
To clean most of this up you can run `machinectl clean --all`


# References
- https://en.wikipedia.org/wiki/POSIX
- https://www.freedesktop.org/software/systemd/man/machinectl.html
- https://wiki.archlinux.org/title/Systemd-nspawn
- https://docs.docker.com/engine/reference/commandline/export/
- https://cloud-images.ubuntu.com/
- https://www.nomadproject.io/plugins/drivers/community/nspawn
- https://wiki.debian.org/Debootstrap
- https://wiki.polaire.nl/doku.php?id=airspy_in_nspawn_chroot
- https://medium.com/@huljar/setting-up-containers-with-systemd-nspawn-b719cff0fb8d
