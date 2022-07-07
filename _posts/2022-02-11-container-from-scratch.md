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


# Resources
- https://ericchiang.github.io/post/containers-from-scratch/
- https://lwn.net/Articles/531114/
- https://blog.nicolasmesa.co/posts/2018/08/container-creation-using-namespaces-and-bash/
- https://www.redhat.com/sysadmin/net-namespaces
- https://en.wikipedia.org/wiki/Linux_namespaces
- https://wiki.archlinux.org/title/Linux_Containers
