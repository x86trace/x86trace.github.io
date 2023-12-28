---
title: "Customized Ubuntu Images using Packer + QEMU + Cloud-Init & UEFI bootloading"
date: "2022-08-21T15:44:28+02:00"
draft: false

author: "Shan"
tags: ["devops", "packer", "qemu", "IaC", "ubuntu"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
# Custom Ubuntu Images

> NOTE: This post requires more than intermediate knowledge of the tools being used.

In current times, you might want to spin up a customized, well-known Distribution like Ubuntu, Debian, CentOS etc.
without having to write large Shell Scripts. This post sheds light on how to create images using tools like
__Hashicorp's Packer__ and __QEMU (Quick Emulator)__.

## Cloud-Init

[Cloud-Init][1] is a _standardized_ way to configure your Images without having to write shell scripts. It is a set of YAML
files that tells the image what needs to be done on the first-boot of the OS. We will use Cloud-Init to create the following:

- Create 2 Users (`admin` and `user1`)
- Set the Bootloader Sequence when trying to boot the Image using __UEFI__

## Packer

[Hashicorp Packer][2] provides a nice wrapper / abstraction over the QEMU in order to boot the image and use it to set it up on first-boot.
Instead of writing really long commands in order to boot up the image using QEMU, Packer provided a nice Configuration Template in a more
readable fashion.

## QEMU
[QEMU][3] is one of the most renowned emulator. We will use it to actually boot the ISO image, which Packer will download for us and use it 
customize our Ubuntu Image. We will use the `X86_64` architecture.

## Ubuntu Live-Server

You can use either the Cloud Images or Live-Server from Ubuntu depending on your use. We will be using Ubuntu's Live-Server.

## Implementation

> NOTE: The configurations used here will work for Ubuntu 20.04 LTS (Focal Fossa) as well as Ubuntu 22.04 (Jammy Jellyfish)

### Pre-Requisites

Make sure to install:

1. `qemu` (Optionally `kvm`)
2. `packer`

### Cloud-Init: `user-data` File

The `user-data` file is where we will be add configuration that is needed for first boot. As a clarification, Ubuntu Live Server uses a tool
called __Autoinstall / Subiquity__ Installer wherein Cloud-Init configuration is a subset.

As previously mentioned, we will use this file to setup:

1. Create Two Users: `admin` and `user1`
2. Setup the bootloader logic in order to quickly boot the ISO after the first-boot

```yaml
#cloud-config
autoinstall:
  version: 1
  locale: en_US
  keyboard:
    layout: us
  ssh:
    install-server: true
    allow-pw: true
  packages:
    - qemu-guest-agent
  late-commands:
    - |
      if [ -d /sys/firmware/efi ]; then
        apt-get install -y efibootmgr
        efibootmgr -o $(efibootmgr | perl -n -e '/Boot(.+)\* ubuntu/ && print $1')
      fi
  user-data:
    preserve_hostname: false
    hostname: packerubuntu
    package_upgrade: true
    timezone: Europe/Berlin
    chpasswd:
      expire: true
      list:
        - user1:packerubuntu
    users:
      - name: admin
        passwd: $6$xyz$74AlwKA3Z5n2L6ujMzm/zQXHCluA4SRc2mBfO2/O5uUc2yM2n2tnbBMi/IVRLJuKwfjrLZjAT7agVfiK7arSy/
        groups: [adm, cdrom, dip, plugdev, lxd, sudo]
        lock-passwd: false
        sudo: ALL=(ALL) NOPASSWD:ALL
        shell: /bin/bash
      - name: user1
        plain_text_passwd: packerubuntu
        lock-passwd: false
        shell: /bin/bash
```

The above mentioned file does the following:

- the `autoinstall` is Ubuntu's AutoInstall / Subiquity configuration section which will set:
   - Keyboard locale to `en_US` and layout to US
   - Install SSH Server and allow Password login (will be used by Packer)
   - `packages` will install some necessary packages on first-boot. Here we install `qemu-guest-agent` to help out
     with login
   - `late-commands` will be triggered at the end of the installation. Here we install the Bootloader Manager (`efibootmgr`).
     We also define the sequence that tells the Boot Manager how it should setup the boot sequence. This will tell the manager
     where the OS is and when should it be loaded
   - `user-data` is the actual section where the Cloud-Init configuration takes place.

- The cloud-init configuration above should do the following:
   - change the hostname to `packerubuntu`
   - set the timezone `Europe/Berlin`
   - Force change the password for the user called `user1` through `chpasswd`
   - Describe which users need to be created:
     - `admin` should be create with the password `packerubuntu` (the encrypted password is created using `openssl passwd -6 -salt xyz`)
     - `admin` should be granted sudo access without requirements for password
     - `user1` should be created with `packerubuntu`

### Packer File

Packer configuration will set all the necessary values to download the ISO Image from Ubuntu Artifacts Repository, give the boot command
options when the ISO is first booted, tell Ubuntu that it will do an automatic installation rather than anticipating the user to intervene.

I am using the Hashicorp Language to define my Packer template file but one can also JSON to setup the configuration. The file is called
`ubuntu.pkr.hcl`.

```hcl

variable "vm_template_name" {
  type    = string
  default = "ubuntu-22.04"
}

variable "ubuntu_iso_file" {
  type    = string
  default = "ubuntu-22.04.1-live-server-amd64.iso"
}

source "qemu" "custom_image" {
  
  # Boot Commands when Loading the ISO file with OVMF.fd file (Tianocore) / GrubV2
  boot_command = [
    "<spacebar><wait><spacebar><wait><spacebar><wait><spacebar><wait><spacebar><wait>",
    "e<wait>",
    "<down><down><down><end>",
    " autoinstall ds=nocloud-net\\;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/",
    "<f10>"
  ]
  boot_wait = "5s"
  
  http_directory = "http"
  iso_url   = "https://releases.ubuntu.com/22.04.1/${var.ubuntu_iso_file}"
  iso_checksum = "file://https://releases.ubuntu.com/22.04.1/SHA256SUMS"
  memory = 4096
  
  ssh_password = "packerubuntu"
  ssh_username = "admin"
  ssh_timeout = "20m"
  shutdown_command = "echo 'packerubuntu' | sudo -S shutdown -P now"

  headless = false # to see the process, In CI systems set to true
  accelerator = "kvm" # set to none if no kvm installed
  format = "qcow2"
  disk_size = "30G"
  cpus = 6

  qemuargs = [ # Depending on underlying machine the file may have different location
    ["-bios", "/usr/share/OVMF/OVMF_CODE.fd"]
  ] 
  vm_name = "${var.vm_template_name}"
}

build {
  sources = [ "source.qemu.custom_image" ]
  provisioner "shell" {
    inline = [ "while [ ! -f /var/lib/cloud/instance/boot-finished ]; do echo 'Waiting for Cloud-Init...'; sleep 1; done" ]
  }
}
```

The Template file tells Packer where to find and download the ISO file. The `boot_command` is quite important because it tells 
Packer how to navigate through the initial Boot Loaders Interface in order to enter the Grub Settings where we mention:

```
  autoinstall ds=cloud-net\\;s=http://{{ .HTTPIP}}:{{ .HTTPPort }}/ 
```

This will tell ubuntu to install the Live server without manual intervention and obtain the cloud-init configuration files
from an HTTP File Server (setup by Packer itself).

> NOTE: please check beforehand where the dedicated OVMF.fd file is located on your build machine. In Majaro Linux it is located
>       at `/usr/share/OVMF/OVMF_CODE.fd`

The `headless = false` is very useful because Packer will open a VNC Viewer window that will completely show all the necessary
Boot loader UI + Setups + Logs. If using this file in a CI Pipeline, please set the value to `true`

The `build` section of the template file will just check whether the Cloud-Init Steps have been finished or not.

Run the following command:

```bash
packer build -force ubuntu.pkr.hcl
```

This will do all the magic for you! At the end you will have `qcow2` image which you can use to load it on your bare-metal machine
or use `qemu` commands to simply boot it up and test it.


Voilà ! You now have a custom Ubuntu Image which you can load on your devices, servers etc. and do not have to painstakingly configure
anything by hand!

Don't let this stop you here, Packer let's you provision your Images with all necessary software packages using your favorite tools like
Ansible, Chef, Puppet. This implies you can even fine tune your images further to make your images exactly the way you want it to be.

## Repository

The code for this post can be found on [GitHub][4].

Get in touch via LinkedIn, Email if you have queries, suggestions or criticism about this post! 

## Resources

[Julien Brochet's Blog Post on Using Packer + Proxmox for Ubuntu 22.04][5]

[Dogukan Cagatay's QEMU VM Template Packer Repo][6]
[Pupeteers.net Blog on Ubuntu 20.04 qemu images with Packer][7]

[1]: https://cloud-init.io
[2]: https://packer.io
[3]: https://www.qemu.org
[4]: https://github.com/shantanoo-desai/packer-ubuntu-server-uefi
[5]: https://www.aerialls.eu/posts/ubuntu-server-2204-image-packer-subiquity-for-proxmox/
[6]: https://github.com/dogukancagatay/qemu-vm-template-packer
[7]: https://www.puppeteers.net/blog/building-ubuntu-20-04-qemu-images-with-packer/
