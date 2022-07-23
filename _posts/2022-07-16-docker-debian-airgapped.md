---
title: Install Docker Air Gapped
date: 2022-07-16 12:00:00 -500
categories: [Containerization]
tags: [containerization,docker,docker desktop,kali]
img_path: /assets/img/posts/
image:
  path: pexels-brett-sayles-3803517.jpg
  width: 1000
  height: 800
  alt: pc
---

As part of a competition i am taking part of i needed the ability to run a simple self hosted wiki page on my Kali box. the fastest and best alternative was to run this as a container.
Although i tried to export the image to run with systemd-namespaces i ran into many issues. So i decided to find a way to find a way to install docker air-gapped along with my wiki image.

# Getting the files
In order to install docker you will need 3 different files (4 if you want docker compose).
Per Dockers [documentation](https://docs.docker.com/engine/install/debian/#install-from-a-package) you will need the following files.
- containerd
- docker-ce
- docker-ce-cli
- docker-compose-plugin #optional

Go to this page and download the files for your system: https://download.docker.com/linux/debian/dists/bullseye/pool/stable/amd64/
Note: replace `amd64` with your own architecture if for example you need to run on a Raspberry Pi.
or
Just run the following commands to download your deb packages.
```bash
wget https://download.docker.com/linux/debian/dists/bullseye/pool/stable/amd64/containerd.io_1.6.6-1_amd64.deb
wget https://download.docker.com/linux/debian/dists/bullseye/pool/stable/amd64/docker-ce-cli_20.10.16~3-0~debian-bullseye_amd64.deb
wget https://download.docker.com/linux/debian/dists/bullseye/pool/stable/amd64/docker-ce_20.10.16~3-0~debian-bullseye_amd64.deb
wget https://download.docker.com/linux/debian/dists/bullseye/pool/stable/amd64/docker-compose-plugin_2.6.0~debian-bullseye_amd64.deb
```

Now save these to a flash drive or however you will be able to get to it later.

# Install Docker
Although the documentaton says to use `dpkg` i had a few issues with installing it so i just used `apt`.
```bash
sudo apt install ./containerd.io_1.6.6-1_amd64.deb
sudo apt install ./docker-ce-cli_20.10.16~3-0~debian-bullseye_amd64.deb
sudo apt install ./docker-ce_20.10.16~3-0~debian-bullseye_amd64.deb
sudo apt install ./docker-compose-plugin_2.6.0\~debian-bullseye_amd64.deb
```
## Additional post install
If you want your `kali` user to be able to run the commands without `sudo` run the following commands.

Create a `docker` group.
```bash
sudo groupadd docker
```
Add your user to the `docker` group.
```bash
sudo usermod -aG docker $USER
```
Log out and log back in so that your group membership is re-evaluated.
If testing on a virtual machine, it may be necessary to restart the virtual machine for changes to take effect.

# Get a Docker Image
So docker is useless by itself so its time to get your docker images ready so you can load them in the airgapped system later

## Pull the image
First on a internet connected machine pull your image using docker.
`docker pull my_docker_image`
Example:
```bash
docker pull lscr.io/linuxserver/dokuwiki:latest
```
## Save your image
Then its time to save it in a format that we can import later
`save -o my_docker_image.docker my_docker_image`
Example:
```bash
docker save -o dokuwiki.docker lscr.io/linuxserver/dokuwiki
```
Note: if you dont know the name of the your image just run `docker images` and it will be the full name under `REPOSITORY`

Save this image in your usb drive or use SCP or  like with your other files wherever you can get to them later.
# Run the docker image on an airgapped sytem.
Now that you have docker installed and running, import your image.
`docker load -i my_image_name`
Example:
```bash
docker load -i dokuwiki.docker
```
Now you can run your container just like you would normally.
Example:
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
