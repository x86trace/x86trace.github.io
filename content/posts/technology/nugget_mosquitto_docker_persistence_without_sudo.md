---
title: "Nugget: Enabling Persistence and Logging for your Mosquitto MQTT Broker Docker Container without Super User Privileges"
date: 2020-09-10T22:25:58+02:00
draft: false
author: "Shan"

description: "Save your Mosquitto Logs and Persistence Data from your Docker Container to Host Machine without `sudo`"
categories: ["Technology"]
tags: ["MQTT", "IoT", "Docker", "Docker-Compose", "Mosquitto"]

toc:
  enable: true
  auto: true
---
<!--more-->

## Overview

At work I came across a problem where I needed to deploy a secure MQTT Mosquitto Broker on a server where I did not have
__Super User__ (`sudo`) privileges. The Mosquitto Broker's Docker Image `eclipse-mosquitto` has some __Open Issues__ that 
where developers could not store the logs generated from the docker container on the host machine or store the persistent database
from the container on the host machine without changing the directory ownerships for the logs and data.

## Broker Deployment with Docker

{{< admonition >}}
If you want something out of the box, I created an open-source repository called [tiguitto](https://github.com/shantanoo-desai/tiguitto)
{{</ admonition >}}

1. I created __Self-Signed Certificates__ for the server using [tiguitto/selfsigned case][2]
2. I created the following directory structure:
    ```bash
        |- certs
            |- mqtt
                |- ca.crt
                |- mqtt-server.crt
                |- mqtt-server.key
                |- mqtt-client.crt
                |- mqtt-client.key
        |- mosquitto/
            |- config/
                |- mosquitto.conf
            |- logs/
                |- mosquitto.log
            |- data/
        |- docker-compose.yml
    ```

3. `docker-compose.yml` looks like:
    ```yaml
        version: "3"
        service:
        mosquitto:
            image: eclipse-mosquitto
            container_name: secure_mqtt_broker
            volumes:
                - ./certs/mqtt:/mosquitto/config/certs
                - ./mosquitto/config:/mosquitto/config
                - ./mosquitto/log:/mosquitto/log
                - ./mosquitto/data:/mosquitto/data
            restart: always
            ports:
                - "8883:8883"
            network_mode: host
    ```

### Errors Logs for Mosquitto Broker

Upon looking at logs using:
```bash
docker-compose logs -f 
```
Logs:

    1544689704: Error: Unable to open log file /mosquitto/logs/mosquitto.log for writing.
    1544689704: Error: Unable to open log file /mosquitto/logs/mosquitto.log for writing.
    1552421249: Saving in-memory database to /mosquitto/data/mosquitto.db.
    1552421249: Error saving in-memory database, unable to open /mosquitto/data/mosquitto.db.new for writing.
    1552421249: Error: Permission denied.

If I had `sudo` privileges the problem would be solved using:

```
    sudo chown -R 1883:1883 mosquitto/logs/
    sudo chown -R 1883:1883 mosquitto/data/
```


## Figure the IDs out!

The container's User ID is `docker` and the directories and files had the user ID of my account on the server. Since the
ownership IDs are completely different, the permissions to write to the `mosquitto.log` file as well the `mosquitto/data` directory
aren't possible!

### Solution

- Find out the User `UID` and Group ID `GID` (Group ID)

    ```bash
    $ id -u
    ```
    OR

    ```bash
    $ id -g
    ```

- For me the my account had the `uid` of `1002`

- Add: `user: "1002:1002"` to the `docker-compose.yml` file and restart your container

The error vanished and I am now able to check the logs on the host and the persistence is stored under `mosquitto/data/mosquitto.new.db`


## Final Compose file

```yaml
    version: "3"
    service:
    mosquitto:
        image: eclipse-mosquitto
        container_name: secure_mqtt_broker
        user: "1002:1002"
        volumes:
            - ./certs/mqtt:/mosquitto/config/certs
            - ./mosquitto/config:/mosquitto/config
            - ./mosquitto/log:/mosquitto/log
            - ./mosquitto/data:/mosquitto/data
        ports:
            - "8883:8883"
        restart: always
        network_mode: host
```

You can also pass the User ID and Group ID as Environment Variables as follows:

```bash
$ UID=$(id -u) GID=$(id -g) docker-compose -f docker-compose.yml up
```
and make sure to change the `user` key as follows in your `docker-compose.yml` file: `user: ${UID}:${GID}`

## Conclusions

I went through the [Open Issue 1078 for Eclipse Mosquitto GitHub Repository][3] and figured the solution out.
With a little trial and error on the `UID` and `GID` with the `user` key-value pair in the `docker-compose.yml` makes it possible
to avoid `sudo` usage!


If you have more thoughts, improvements and criticisms then connect with me or send me an E-mail or a __LinkedIn Message__ anytime!


[2]: https://github.com/shantanoo-desai/tiguitto/tree/master/selfsigned
[3]: https://github.com/eclipse/mosquitto/issues/1078