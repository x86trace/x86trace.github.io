---
title: "Build a Bootable OS with Docker Containers using Hashicorp Packer"
date: "2022-09-18T12:59:53+02:00"
draft: false

author: "Shan"
tags: ["DevOps", "IaC", "Packer", "Docker"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Why do this?

I was mightly impressed by [Ivan Velichko's Blogpost][1] on using Docker Containers
to create bootable Disk images. He even has a [GitHub Repository][2] that still gets a lot of
attention from developers who have tried and tested things out.

I am already using [Hashicorp Packer][3] at work and for personal projects and I wanted to test
this idea out by wrapping it a single Packer Template file. This reduces the level of maintaining
a lot of small scripts, Dockerfiles and configurations and the user can simply trigger a couple of 
commands to get a minimalist OS at the end of the process.

Also just sheer curiousity from side. Why not try things out!

### What else is there in the market?

Don't get me wrong there are some projects already out there doing things similarly, and a lot of products
leverage these tools to ship out custom, minimalist OS images for Edge Computing Devices / Cloud Platforms.

- [LF-Edge EVE project][4] leverages [Linuxkit][5] to create custom OSs for Edge Devices which in turn leverages
  Containers as Lego Blocks

Albeit these projects do much more than just create a Lego Tower of Docker Containers to create a bootable image,
but if it peeks your curiosity you should check them out.

## Presenting: PockerISO

[PockerISO][6] is just Packer + Docker to create bootable images. The difference here is that Packer does the
heavy-lifting of doing the following things:
  - pulling the base images
  - bringing them up
  - executing the respective shell scripts within the container
  - saving/exporting the current filesystem state into a tarball 
  - creating an isolated container to execute the steps to create the bootable image without disturbing the host machine fs

> Packer is the Maestro and Docker is the Orchestra that plays the symphony, and at the end the end-user cheers with
> a nice minimalist OS image

### How does PockerISO work?

### Filesystem Generation

I strongly recommend reading Ivan Velichko's blogpost mentioned previously to get some greate understanding on what needs
to be done. He does some insanely cool visualizations too!

In a nutshell, containers __DO NOT__ have a kernel (they leverage the kernel of the host machine), and they __DO NOT__ have
system manager, containers always run with Process ID 1.

So in order to make an operating system, we will install these two things in the container base image to atleast create a
filesystem tarball which we can use to create an image.

Once the container installs the kernel, generates the respective `initramfs` for us and installs `systemd` via APTITUDE 
package manager, we tell Packer to finish of this process by creating a tarball of the container's filesystem.

#### Bootable Image creation

Once the tarball is created, we extract it on the host for the simple reason that extracting the tarball within a docker 
container generally leads to a lot of Failures when done through via Packer.

We then spin up a container of the same base image, and copy the necessary files and folders into it. The main thing to note here
is this container is brought up with `privileged` mode. Remember we will still need the host device to use Operating System
Loop devices, to mount filesystems and create the final base image.

Within this phase we do the following via a dedicated shell script:
  1. Create an empty image file using `dd` of approximately 1GB
  2. Make Partitions for the disk image
  3. Create a filesystem on the image with `ext4` format
  4. Copy our filesystem with the kernel, systemd and initramfs
  5. Install a bootloader and create a boot partition on the image
  6. Mount / Unmount the dedicated image

The final result will be an `.img` as well as `.qcow2` image that can be tinkered with using QEMU.

As a user you only need to execute a simple `make` command:

```
make ubuntu # or `make debian`
```
and at the end of the magical automation rainbow you will have a minimalist OS you can play with.

## What base images does PockerISO offer?

At this point the following:
  - __Ubuntu__: 20.04, 22.04
  - __Debian__: __bullseye__, __bookworm__

## Potential Updates

- I am planning to add Alpine Images with it's own Packer Template
- I wish to make this repository compatible also for `linux/arm64` images since Docker lets you cross-compile images via BuildKit.
  The only caveat is that for devices like the Raspberry Pi, the bootloader logic might be a tad bit tedious to figure out.


If you would like to give some feedback, criticisms, suggestions on improving the project or this write-up leave me a message on 
LinkedIn and I will happily revert back.

[1]: https://iximiuz.com/en/posts/from-docker-container-to-bootable-linux-disk-image/
[2]: https://github.com/iximiuz/docker-to-linux
[3]: https://packer.io
[4]: https://www.lfedge.org/projects/eve/
[5]: https://github.com/linuxkit/linuxkit
[6]: https://github.com/shantanoo-desai/PockerISO
