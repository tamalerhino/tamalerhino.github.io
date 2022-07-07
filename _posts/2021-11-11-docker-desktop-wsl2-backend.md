---
title: Docker Desktop WSL2 Backend
date: 2021-11-11 12:00:00 -500
categories: [Containerization]
tags: [containerization,wsl,wsl2,docker,docker desktop,windows]
---

Recently with the usage of Docker Desktop, there has been a need to use WSL2 to run Docker Desktop on Windows. However because of the various security implications, missing security kernel modules, agents, etc. The idea of possibly creating or rolling our own docker solution by creating a WSL2 Linux image that we control, with all our monitoring and logging tools needed has increased. This blog will explain one possible way to do this.

# Docker to create a WSL2 image...for docker...
WSL2 image needs a few things, a created and tar'd up filesystem as well as the binaries and everything a distro needs. One thing to mention is that the Distros that Windows uses ARE CONTAINERS, they're System Containers not Application containers like Docker. So the simplest way to go about this in my opinion is to turn an app container into a system container!

This is where docker comes in, I will use docker to create a container then I will export that container plus the filesystem and import it into WSL2 as my own distribution. This will allow us to do two things, create a Golden Image Docker Container, and let us use the same container as a full-fledged distro, to run anything...including docker!

## Prerequisites 
- Windows System
- WSL2 installed and configured on that System
- Docker Desktop 
- Admin rights probably...

Note: I just realized I could have done this on a Linux system and saved myself some trouble but oh well...
```powershell
PS C:\Users\tamalerhino> wsl --list
```
Should return:
IMAGE_PLACEHOLDER

## Download and Run the container you will want to turn into the Linux distro.
In this case, I will choose RedHat since I know that it does not exist in the Microsoft store and we use oracle so it will work the same.

Where it says tamalerhino-distro feels free to name it whatever this doesn't matter and it's only to make my next command easier.
```powershell
PS C:\Users\tamalerhino> docker run -d --name tamalerhino-distro registry.access.redhat.com/ubi8/ubi-init
```
Should return:
IMAGE_PLACEHOLDER

## Export container
Next, let's export the container plus the filesystem created. replace `tamalerhino-distro` for whatever your container name is.
```powershell
PS C:\Users\tamalerhino> docker export -o tamalerhino-distro.tar.gz tamalerhino-distro

PS C:\Users\tamalerhino> dir


    Directory: C:\Users\tamalerhino


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        11/11/2021   1:29 PM                .docker
d-r---          7/8/2021  11:05 AM                3D Objects
d-r---          7/8/2021  11:05 AM                Contacts
d-r---        11/11/2021   1:28 PM                Desktop
d-r---         7/23/2021   4:48 AM                Documents
d-r---          8/2/2021   3:36 PM                Downloads
d-r---          7/8/2021  11:05 AM                Favorites
d-----          8/2/2021   4:03 PM                getting-started
d-r---          7/8/2021  11:05 AM                Links
d-r---          7/8/2021  11:05 AM                Music
d-r---          8/3/2021  12:04 PM                OneDrive
d-r---         11/4/2021   5:20 AM                Pictures
d-r---          7/8/2021  11:05 AM                Saved Games
d-r---          7/8/2021  11:05 AM                Searches
d-r---          8/2/2021   3:58 PM                Videos
-a----        11/11/2021   1:44 PM      227159552 tamalerhino-distro.tar.gz
```
## Import into WSL
ow we can share this or create a script in order to import it as our own distro

Where it says tamalerhino-distro , change it to what you want to name it on the system. Also, I told it to mount the distro here "./tamalerhino-distro" ideally you will want to mount it to a location you can lock down.
```powershell
PS C:\Users\tamalerhino> wsl --import tamalerhino-distro ./tamalerhino-distro .\tamalerhino-distro.tar.gz
```
You can use the following command to show that it created it correctly. 
```powershell
PS C:\Users\tamalerhino> wsl --list
```
IMAGE_PLACEHOLDER
## Profit
And finally login to it by using the following command.
```powershell
PS C:\Users\tamalerhino> wsl -d tamalerhino-distro
```
IMAGE_PLACEHOLDER
Here I'm just showing you that its redhat.
IMAGE_PLACEHOLDER

## Adding Stuff
One of the main issues that we have is that the docker distro isn't locked down and the only user is the root user, let's change that
```bash
yum update -y && yum install passwd sudo -y
adduser -G wheel tamalerhino
echo -e "[user]\ndefault=tamalerhino" >> /etc/wsl.conf
passwd $myUsername
```
You will need to exit and terminate the running distro
```powershell
wsl --terminate tamalerhino-distro
wsl -d tamalerhino-distro
```
As we can see we have defaulted to a non-root user.
IMAGE_PLACEHOLDER

There now you have your very own linus distro runing inside of WLS!

# Do we even need docker of containers on our desktop?...
I still think Docker is going to go away, podman is much easier to use, doesn't run as root by default, and lets systemd handle containers.
```bash
$podman --version
```
