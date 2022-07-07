---
title: Docker Desktop WSL2 Deep Dive
date: 2021-10-01 12:00:00 -500
categories: [Containerization]
tags: [containerization,wsl,wsl2,docker,docker desktop,windows]
---

In the last few years theres been a big push for Docker Desktop on developer workstations, many of those workstations being Mac and Windows not Linux. But since containers need a Linux kernel to run how does Docker Desktop handle this? Unfortunately the documentation out there is very light particularly using WSL2 so i have taken the time to do research and document my findings.

## Windows History
Before we talk about WSL it's important to understand a bit about Windows, specifically Windows NT(kernel) and its subsystems. Since the beginning, Windows was designed with the ability to use several different subsystems such as POSIX, OS/2, and Win32 which is the most commonly used one today. This would give a programmatic interface to the applications without having to worry about how to implement them in the WindowsNT kernel itself.

Each subsystem was implemented as "user mode" modules that issued appropriate NT system calls based on the API they presented to the applications for that subsystem. All applications were PE/COFF executables, a set of libraries and services to implement the subsystem API, and NTDLL to perform the NT system call. This means when a user launched a program it would indicate which subsystem to use and load all of its dependencies along with it based on the executable header. The predecessor to WSL, Subsystem for Unix-based Applications (SUA) was a replacement for POSIX, it was created to encourage developers to port their applications over by not having to worry about major rewrites to their code, instead, they implemented the POSIX user-mode APIs using NT constructs. However because the components were constructed in user mode, this model relied on the need for programs to be recompiled and required ongoing feature porting and was a maintenance burden.

Eventually, most of those initial subsystems were retired but the Windows NT kernel maintained the ability to support multiple subsystems.

## WSL1
WSL1 makes use of this ability by deploying a Linux Kernel Emulator Subsystem alongside the Win32 Subsystem. It is a collection of user-mode and kernel-mode components that give the ability to run native Linux ELF64 binaries to run on Windows. The main components are the LXSS Manager service which is a user-mode session manager service that handles the Linux instance life cycle. The Pico Processes that host the unmodified user mode Linux (e.g. /bin/bash). The Pico Provider Drivers (lxss.sys, lxcore.sys) emulates the Linux kernel by translating Linux system calls into WindowsNT system calls such as `fork()`. Again this was an emulated Linux kernel and not an actual Linux Kernel as shown below.

