---
title: "Exploring the marshland Docker Compose versions with single quotes, special characters and environment variables"
date: "2022-07-15T16:53:18+02:00"
draft: false

author: "Shan"
tags: ["docker", "docker-compose", "vagrant", "ansible", "DevOps"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->

## Problem

Since [Docker's Compose V2 General Availability][1] announcement, a lot of things have changed which might lead to 
Incompatibility in your currently running application stacks.

### Compose V2: what changes?

- instead of `docker-compose` the syntax will be `docker compose`
- `docker-compose` (v1) was written in Python, `docker compose` (v2) is in Go
- Compose V1 is official EOL (End Of Life) aka. Deprecated

There are lot of fine things in Compose V2 but this post isn't about it. This write-up takes you through the murky 
marshland between Compose V1 and Compose V2 and how things might start you break when you decide to jump to the 
_cool V2_ version without doing some thorough checks in your stack application


## Swiss Knife for the Exploration

Tools you will need:

- __Hashicorp Vagrant__
- __Ansible__
- __Different versions of Docker Compose (v1 as well as v2)__

### Vagrant

Instead of setting up a pain-staking Virtual Machine, we will leverage a bit simplified and inexpensive Vagrant Machine
with __Ubuntu 20.04 (focal amd64) base image__.

Within this image we will install the following:

- Docker Engine `20.10.17`
- Docker Compose v1: `v1.25.4 build 8d51620a` and `v1.29.2 build 5b3cea4c`
- Docker Compose v2: `2.6.0`

### Ansible

I have Ansible installed on my host machine

```bash
ansible --version
ansible [core 2.13.1]
```

Ansible will install all the spicy Compose versions into the Vagrant Machine for us

## Why are we doing this?

If you have containers that require passwords at rest e.g. in `.env` environment files to be encrypted and with 
special characters like the `$` in them, we are going for ride which will make you want to bookmark this write-up.

It turns out, I had a system running Compose v1 (`v1.25.4`) and upon upgrading to Compose v2 (`2.6.0`) I wasn't able
to login to my Node-RED container which had authentication setup already.

### Node-RED Container 

Node-RED requires that the administrator password must be encrypted in `bcrypt` type scheme and the password looks
something like your dog walked over your keyboard.

```
$2a$08$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.
```

This looks fairly okay, but the menace is the `$` characters in them.

Let's dive deep shall we!


## Code

### `Vagrantfile`

I am using a Linux Machine with Virtual Box as a provider

```
# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant.configure("2") do |config|
  
  # Use Ubuntu 20.04 as base
  config.vm.box = "ubuntu/focal64"

  config.vm.box_check_update =false

  # Synchronize all the docker-compose related file to `/vagrant` dir in VM
  config.vm.synced_folder ".", "/vagrant"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
  end

  # Setup the VM with all the different variants of Docker Compose in them
  config.vm.provision "ansible" do |a|
    a.verbose = "v"
    a.playbook = "./docker-compose-playbook.yml"
  end
end
```

### Ansible Playbook: `docker-compose-playbook.yml`

We will let Ansible install all the required Docker Compose versions for us in the VM through the playbook.

- Docker Compose v1.25.4 will be available in the VM as CLI `docker-compose-125`
- Docker Compose v1.29.2 will be available in the VM as CLI `docker-compose-129`
- Docker Compose v2.6.0 will be available in the VM as CLI `docker compose`

```yaml
---
- hosts: all
  become: yes
  tasks:
    - name: install prerequisites
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg-agent
          - software-properties-common
        update_cache: yes
    - name: add apt-key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
    - name: add docker repo
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
    - name: install docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
        update_cache: yes
    - name: add userpermissions
      shell: "usermod -aG docker vagrant"
    - name: Install docker-compose v1.29.2 from official github repo
      get_url:
        url : https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Linux-x86_64
        dest: /usr/local/bin/docker-compose-129
        mode: 'u+x,g+x'
    - name: change compose v1.29.2 user permissions
      shell: "chmod +x /usr/local/bin/docker-compose-129"
    - name: Install docker-compose v1.25.4 from official github repo
      get_url:
        url: https://github.com/docker/compose/releases/download/1.25.4/docker-compose-Linux-x86_64
        dest: /usr/local/bin/docker-compose-125
        mode: 'u+x,g+x'
    - name: change compose v1.25.4 user permissions
      shell: "chmod +x /usr/local/bin/docker-compose-125"
```

### Files for reproduction of incompatibility

We wont be spinning too much complex stuff so a simple compose file and the relevant `.env.node-red`
file with the encrypted password should suffice

#### `docker-compose.yml`

```yaml
version: '3.7'
services:
  node-red:
    image: nodered/node-red:2.2.2
    container_name: test-nodered
    env_file:
      - .env.node-red
```

#### `.env.node-red`

```
# Encrypting plain-text password with value `password` with bcyrpt
NODERED_ADMIN_PASSWORD=$2a$08$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.
```

## Compatibility Checks

Let's bring the Vagrant Machine up using:

```bash
vagrant up
```
this will download the base ubuntu image if you previously don't have it on your host machine,
provision it with all the docker compose versions in it.

Login into the Machine:

```bash
vagrant ssh
```

Once into the Machine head to the `/vagrant` directory, and just to be safe we check if all the 
required compose versions are available or not.

```bash
$ cd /vagrant

$ docker-compose-125 --version
$ docker-compose-129 --version
$ docker compose version
```

### Checks: Password not enclosed in Single Quotes

As the `.env.node-red` file currently shows that my encrypted password is not enclosed in single quotes

Let's understand what Compose thinks of the value of the environment variable.

#### Compose v1.25.4 (without single quotes)

```bash
docker-compose-125 config
```

produces

```yaml
services:
  node-red:
    container_name: test-nodered
    environment:
      NODERED_ADMIN_PASSWORD: $$2a$$08$$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.
    image: nodered/node-red:2.2.2
version: '3.7'
```

> NO Problem! My dollar characters are escaped by docker-compose. I can login with password `password`


#### Compose v1.29.2 (without single quotes)

```bash
docker-compose-129 config
```

produces

```yaml
services:
  node-red:
    container_name: test-nodered
    environment:
      NODERED_ADMIN_PASSWORD: $$2a$$08$$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.
    image: nodered/node-red:2.2.2
version: '3.7'
```

> NO Problem! My dollar characters are escaped by docker-compose. I can login with password `password`


#### Compose v2 (without single quotes)

```bash
docker compose config
```

produces

```yaml
name: node-red-app
services:
  node-red:
    container_name: test-nodered
    environment:
      NODERED_ADMIN_PASSWORD: a$$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.
    image: nodered/node-red:2.2.2
    networks:
      default: null
networks:
  default:
    name: node-red-app_default
```

> Oh NO! What happened here? Password looks eaten up. The YAML file looks different too!

### Checks: Password enclosed in single quotes

Now let's enclose our encrypted password between single quotes, so our `.env.node-red` file
should look like:

```
NODERED_ADMIN_PASSWORD='$2a$08$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.'
```

#### Compose v1.25.4 (with single quotes)

```bash
docker-compose-125 config
```

produces

```yaml
services:
  node-red:
    container_name: test-nodered
    environment:
      NODERED_ADMIN_PASSWORD: '''$$2a$$08$$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.'''
    image: nodered/node-red:2.2.2
version: '3.7'
```

> Oh NO! Why are there more single quotes than expected. This is not going to let me login!

### Compose v1.29.2 (with single quotes)

```bash
docker-compose-129 config
```

produces

```yaml
services:
  node-red:
    container_name: test-nodered
    environment:
      NODERED_ADMIN_PASSWORD: $$2a$$08$$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.
    image: nodered/node-red:2.2.2
version: '3.7'
```

> NO problem! here this will work fine as the dollar characters are escaped safely by compose


### Compose v2 (with single quotes)

```bash
docker compose config
```

produces

```yaml
name: node-red-app
services:
  node-red:
    container_name: test-nodered
    environment:
      NODERED_ADMIN_PASSWORD: $$2a$$08$$zZWtXTja0fB1pzD4sHCMyOCMYz2Z6dNbM6tl8sJogENOMcxWV9DN.
    image: nodered/node-red:2.2.2
    networks:
      default: null
networks:
  default:
    name: node-red-app_default
```

> NO Problem! Phew ! This will work just fine ! 

## Result: Compose Compatibility Matrix


| Compose Version / Values| `1.25.4`  | `1.29.2` | `2.6.0` |
|:--:|:--------:|:--------:|:-----------:|
| with Single Quotes |  :x:   | :white_check_mark: | :white_check_mark: |
| without Single Quotes | :white_check_mark: | :white_check_mark: | :x: |

### What did we learn today, kids?

__No Single Quotes__:

- Docker Compose v1.25.4 was able to resolve special characters without having enclose them in single quotes
- Docker Compose v1.29.2 will also work just fine for values not enclosed in single quotes
- Docker Compose v2 will try to go hay-wire (try to resolve the value after the `$` sign) if you do not use values in single quotes

__With Single Quotes__:

- Docker Compose v1.25.4 will _literally_ take things enclosed in single quotes as part of your environment variable and will
  break your Container's Login logic

- Docker Compose v1.29.2 will work well
- Docker Compose v2 will work well


In a nutshell

> DON'T JUMP THE GUN, when moving from different V1 to V2 in Compose. Do some throrough research as well as experiments.

Since Compose V1 is now deprecated, when transitioning to Compose V2 check all your environment variable files and if they require
special characters to be either enclosed in single quotes or not. Estimate your downtimes and conduct isolated tests when required.

Rule of thumb from Compose V2 onwards: _if it got some special char in them password, pack em between single quotes_

but to each their own!


### Github Gist

[GitHub Gist available publicly][2]

## Feedback

If you have some feedback, suggestions, criticisms drop me an E-mail or a message on LinkedIn. I am very happy to help and improve myself.


[1]: https://www.docker.com/blog/announcing-compose-v2-general-availability/
[2]: https://gist.github.com/shantanoo-desai/b5d971c533509fdf978f9e97e584b0d0
