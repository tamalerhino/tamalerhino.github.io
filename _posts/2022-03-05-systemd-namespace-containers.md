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

# Creating a container image
There are multiple ways to create a container image.
- From scratch ie `bootstrap`
- From a already made tar or img file such as a Ubuntu's cloudimage
- From a docker image/container

## From Scratch
Todo: creating an image using `bootstrap`

## From a ready to go image
There are several cloud images out there here are the ones for ubuntu:
-  https://cloud-images.ubuntu.com/kinetic/current/kinetic-server-cloudimg-arm64.tar.gz

## From a Docker container/image
This is the fastest and easiest but will not work in most cases since you will need to create an init script on the container to start your services. But it is a container it just wont run anything automatically until you login to the container and run it manually.

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
# Downloading or Pulling the Image

## wget/curl
If you got your image hosted somewhere on the internet just use `wget` or `curl` to download it.

## Pulling an "image" using machinectl

### tarball image
The below example uses the "ready to go image" example
```bash
machinectl pull-tar https://cloud-images.ubuntu.com/kinetic/current/kinetic-server-cloudimg-arm64.tar.gz
```
### img/raw image
The below example uses the "ready to go image" example
```bash
machinectl pull-raw https://cloud-images.ubuntu.com/kinetic/current/kinetic-server-cloudimg-arm64.img
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
![importing an image](importing-image.png)

### img/raw image
```bash
machinectl import-raw dokuwiki.img
```

## View the installed images.
```bash
machinectl list-images --all
machinectl show-image <imagename>
```
![list all images](list-images.png)

# Start container

Technically you can use `machinectl` to start the container but every now and then it doesn't want to work so we will use `nspawn` instead which is like `chroot` but on steroids.

## Directory
Depending if you want a shell or not it might be best to untar it and run it with the `--boot` `-b` flag to let systemd run systemd inside of the container.

```bash
systemd-nspawn -bD dokuwiki/
```
## In a Shell
```bash
sytemd-nspawn -M dokuwiki
```

![run the container](run-container.png)


# Other
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
