---
title: "Nugget: User Management in MQTT Mosquitto Broker with Docker"
date: 2020-08-19T20:20:03+02:00
draft: false
author: "Shan"

description: "How to manage Users for your Mosquitto MQTT broker docker container without restarting the container"
categories: ["Technology"]
tags: ["MQTT", "IoT", "Docker", "docker-compose", "User-Management"]

toc:
  enable: true
  auto: true
---
<!--more-->

## Overview

If you use `docker` or `docker-compose` often to setup your IoT Stacks, you might want be at crossroads where Security and User Management
becomes the next necessary step to improve the Stack. This post builds upon [tiguitto][1] which provides
different set of security measures to the __TIG (Telegraf, InfluxDB, Grafana) + Mosquitto MQTT Broker__ stack.

During the learning phase of setting up the stacks, there came a crucial requirement about how to *add/remove* user management to the MQTT Mosquitto Broker. In the stack, there are only two services - __Grafana__ and __Mosquitto MQTT Broker__, which would require user management. Grafana, tends to provide an easy way via UI to manage users. To the best of my knowledge, no UI for Mosquitto Broker since it is mostly a CLI based service.

> I was very skeptical whether one can manage Mosquitto Broker users without _restarting_ the `mosquitto` container.
> I wanted to avoid downtime in the stack

Apparently, there is a way to avoid a downtime to add/remove users to the Broker. Let's look into it!

## Solution

According to [mosquitto's man page][2] `mosquitto` responds to a __SIGNAL__ called `SIGHUP`. The documentation mentions:

{{< admonition >}}
SIGHUP

Upon receiving the SIGHUP signal, mosquitto will attempt to reload configuration file data, assuming that the -c argument was provided when mosquitto was started. Not all configuration parameters can be reloaded without restarting. See mosquitto.conf(5) for details.
{{< /admonition >}}

The next step was to check how to send signals to a specific `docker` container. This is also possible via `docker` CLI as answered in [this StackOverflow Query][3].

If your container is named `mosquitto` in your `docker-compose.yml` file, the the following will work:

```bash
docker kill --signal=SIGHUP mosquitto
# OR
docker kill --signal=SIGHUP <container_id>
```

## Getting Hands Greasy!

As a simple example if you use the [tiguitto][1] repository you can simply following the documentation in the repository to setup a Stack easily based on your requirement. For the sake of understanding, we will assume you are using the most simple use-case [`PROTOTYPE`][4] case from the repository.

If you use the case _out-of-the-box_, there are already two users added to the Broker:

```
    # mosquitto/config/passwd file in the Prototype Case from tiguitto
    # username:password

    pubclient:tiguitto
    subclient:tiguitto
```
The `pubclient` username can be used by an Sensor Node to publish data to the broker.

### Adding Users to Mosquitto MQTT Broker

Let's add a user `pubclient1` with password `tiguitto`.

Open the `mosquitto/config/passwd` file and add a new username for a new Sensor Node:

```
pubclient:$6$ITRzMkBDHn7MKJ95$i/Ea2KniGMmV9jomR0+jyl40c0YK1eTwTeyKms95obREDFZzhEIQhTntncJeX3PS9eoj9u
R+ATtS8kGowA==
subclient:$6$0lEyFrntsYvkzU7N$24t0vyKVSeEBiSOdvOExaMG67F5xfHeVw8zBxdPtb9BVjchKFNdiI7WaJApVkVZ/DS+MMk
xWycgOeizTmA==
pubclient1:tiguitto
```
The `pubclient` and `subclient` have passwords encrypted, hence the garbled string.

#### Encrypt the Password for the New User

Assuming, you are in `tiguitto/prototype` directory we can encrypt the password using the following the command:

```bash

docker run -it --rm -v $(pwd)/mosquitto/config:/mosquitto/config eclipse-mosquitto mosquitto_passwd -U /mosquitto/config/passwd
```

If you do not get a response on the terminal, your password is encrypted. Check the `mosquitto/config/passwd` file:

```
pubclient:$6$ITRzMkBDHn7MKJ95$i/Ea2KniGMmV9jomR0+jyl40c0YK1eTwTeyKms95obREDFZzhEIQhTntncJeX3PS9eoj9u
R+ATtS8kGowA==
subclient:$6$0lEyFrntsYvkzU7N$24t0vyKVSeEBiSOdvOExaMG67F5xfHeVw8zBxdPtb9BVjchKFNdiI7WaJApVkVZ/DS+MMk
xWycgOeizTmA==
pubclient1:$6$ue8f+Z9OlNRoPM6C$ay+mX+UMKfLWMvZqc9+4s+cFT7NOXhFfQ6iNW1dIuxnxVRDngWSnKBgkCC5h3l1M3b2Uq
KMw73dKXS05xQ==
```

#### Reload the Configuration File

from the same directory perform:

```bash
docker kill --signal=SIGHUP mosquitto
# OR
docker-compose -f docker-compose.prototype.yml -s SIGHUP mosquitto
```

This should provide a log in the `mosquitto` container as following:

    mosquitto    | 1597864573: Reloading config.

Tada! 👏👏👏 New publishing client `pubclient1` added!!

Use an MQTT Client to publish data to the broker with the new credentials.


### Deleting A User from the MQTT Broker

Let's remove `pubclient1` from the `mosquitto/config/passwd` by executing the following:

```bash
docker run -it --rm -v $(pwd)/mosquitto/config:/mosquitto/config/ eclipse-mosquitto  mosquitto_passwd -D /mosquitto/config/passwd pubclient1
```
The `-D <password_file> <username_to_remove>` will remove the username from the password file.

#### Reload the Configuration File

```bash
docker kill --signal=SIGHUP mosquitto
# OR
docker-compose -f docker-compose.prototype.yml -s SIGHUP mosquitto
```
Will then remove the username from the MQTT Broker


## Conclusions

A bit of tweaking on the command line with `docker` and some documentation reading + a dash of StackOverflow resolved queries helped me overcome the worry I previously had for downtimes in the Stack.

## Future Works

I aware of an [Authentication Plugin on GitHub for Mosquitto which handles User Management via different Databases][5] but the repository is archived and there are so many forks to dapple with!

> However, I might look into creating a RESTful API for the User Management described above but I am still in doubt as to if it makes sense to actually issue Shell Commands via REST APIs, on servers which might get messy at times!

If you have some thoughts on this idea, I would like to hear them. You can send me a message on my __LinkedIn Profile__ anytime.



[1]: https://github.com/shantanoo/tiguitto
[2]: https://mosquitto.org/man/mosquitto-8.html#
[3]: https://stackoverflow.com/questions/25687131/how-to-send-signal-to-program-run-in-a-docker-container
[4]: https://github.com/shantanoo-desai/tiguitto/tree/master/prototype
[5]: https://github.com/jpmens/mosquitto-auth-plug