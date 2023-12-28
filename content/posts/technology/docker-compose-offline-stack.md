---
title: "Nugget: Prepare your Docker-Compose Stack as a Tarball for Offline Installations"
date: 2022-03-26T13:54:57+01:00
draft: false

author: "Shan"
tags: ["docker", "docker-compose"]
categories: ["Technology"]

toc:
  enable: false
  auto: false
---
<!--more-->
## Requirements

Many a times you might have requirement to run Docker on a Raspberry Pi or other SBCs that may
have to do run some applications without any internet connections. You might have initially 
developed your application only by keeping in mind a Docker Artifact and was meant to run as 
complete stack using `docker-compose.yml`. 

In almost all cases, bringing the stack up will pull the images from a registry through the Internet.
But when you wish to deploy some SBCs into a space where there is no Internet to begin with you might
have to either develop your code artifact as a package or binary bundle. If this application requires
dependencies on some Databases then you will have to write tedious bash scripts to prepare the device
for some pre-installtion.

Seems like a lot of refactoring / re-thinking to do!

Do not worry `docker` has things sorted out, and provides `docker save` and `docker load` functionalities.

## Scenario

Let's consider an Open-Source Stack [tiguitto](https://github.com/shantanoo-desai/tiguitto) for IoT at Edge.
It comprises of some standard apps like an __MQTT Broker__, __InfluxDB__, __Telegraf__, __Grafana__. All these
components are combined through a `docker-compose.yml` file.

Now, as previously mentioned shipping the stack would have been easy, had we had Internet on the target node.
Simply sending the compose file to the node would have done everything for us.

For cases where we do not have Internet access, we have to ship two things:

1. `docker-compose.yml`
2. Compressed Tar-Ball for services present in the compose file

### Usage Steps
Assume we have our development machine with `docker` and `docker-compose` on them.

1. We make sure to remove all not-used, unnecessary images are cleared from the machine and only the images 
   of our application exists.

   ```bash
    docker images -a # check existing images on dev machine
   ```
   If you wish to remove everything and only pull the required images do:
   
   ```bash
    docker rmi $(docker images | awk "NR>0 {print$3}")
   ```
2. Pull the images with their dedicated tags using `docker pull <image_name>:<tag>`

3. Check using `docker images -a`

4. Now leverage `docker save` along with some format parsing using:

    ```bash
    docker save -o myStack.tar $(docker images --format "{{.Repository}}:{{.Tag}}")
    ```
  `docker save` combines the image into their respective Tar balls however we need to tell `docker`
  that we need to combine all the images existing on the dev machine into a combined Tar ball. This
  is done through `--format "{{.Repository}}:{{.Tag}}"`

5. This should produce `myStack.tar` in your directory

## Unpack your Tarball and Test

If you are doing this on a laptop or PC, try switching your network connection off now.

1. Unpack your Tarball
  
  ```bash
    docker load -i myStack.tar
  ```
2. Check with `docker images -a`

3. You can now `docker-compose up` and your application should work offline

If you use some other ways of using shipping your Docker and Compose applications, get in touch with me and 
would like to learn from you!
