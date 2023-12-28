---
title: "Telegraf Deployment Strategies with Docker Compose"
date: "2023-07-20T11:52:00+02:00"
draft: false

author: "Shan"
tags: ["docker", "docker-compose", "telegraf", "DevOps"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->

## InfluxData's Telegraf

Telegraf is widely used as a metric aggregation tool thanks to the diverse amount of plugins it provides which interface with a multitude of systems without having to write complex software logic. With the advent of the Low-Code / No-Code paradigm in system operations; Telegraf sees itself as a front-runner.

However, with Low-Code/No-Code tools it might get complicated to maintain a lot of additional yet necessary information like credentials to other subsystems. Although standard practice dictates the use of Environment Variables within Telegraf - with version 1.27 and beyond Secret Stores will come in handy when passing credentials into Telegraf plugins without having to pass environment variables explicitly, especially when using a containerization tool like Docker.

We will go through some strategies of deploying a Telegraf Docker Container using Docker Compose v2 tool in this post that may help intermediate to advanced users decide upon how to organize their stack configurations appropriately

### Using Docker Secrets

Docker Compose v2 specifications provide a useful [Secrets feature][1] which may also be used for standalone Compose Application Stacks and not just in Docker Swarm mode. With Docker Secrets the environment variables that contain credentials for other subsystems are mounted into the Telegraf Container as files. These secret files are read through the Docker Secret Store plugin and passed to the respective plugins in a relatively safe manner. By using the Docker Secret Store Plugin, one can also avoid credentials that were previously visible via environment variables, to be now hidden behind runtime secret files within the container.
Standard Method with Environment Variables
As an example, it is possible to pass the credentials to a plugin via the environment variable placeholder in a telegraf configuration file where the credentials for a plugin exist in a `.env` file (e.g. MQTT input Plugin)

```
MQTT_USERNAME=telegraf
MQTT_PASSWORD=superSecurePasswordMQTT
```
A telegraf configuration file for the MQTT Input plugin can be as follows:

```toml
[[inputs.mqtt_consumer]]

  servers = [ "tcp://<broker_ip>:1883" ]
  topics = [ "test1/#", "test2/+/topic" ]
  username = "${MQTT_USERNAME}"
  password = "${MQTT_PASSWORD}"
```

A Docker Compose File for the example can be as follows:

```yaml
services:
  telegraf:
    image: docker.io/telegraf:latest
   container_name: telegraf
   environment:
     - MQTT_USERNAME=${MQTT_USERNAME}
     - MQTT_PASSWORD=${MQTT_PASSWORD}
   volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
```
When the container is brought up, the values of the environment variables are interpolated by telegraf in order to connect to the MQTT Broker. This is with an assumption that the `.env` file and the `docker-compose.yml` file are in the same directory. This works well however a simple inspection of the running container may yield the credential values via the environment variable by a command such as:


```bash
docker compose exec telegraf env
```
OR

```bash
docker inspect -f “{{ .Config.Env }}” <telegraf_container_name / telegraf_container_id>
```

With Docker Secrets and Telegraf Docker Secret Store Plugin
With Docker Secrets we can change the Compose file to pick the environment variables up from the `.env` file and let Docker Compose mount these values as files under the `/run/secrets` directory in the telegraf container as follows:

```yaml
services: 
  image: docker.io/telegraf:latest
  container_name: telegraf
  secrets:
    - source: mqtt_username
      mode: 0444
   - source: mqtt_password
     mode: 0444
  volumes: 
./telegraf.conf:/etc/telegraf/telegraf.conf:ro

secrets: 
  mqtt_username: 
    environment: MQTT_USERNAME
  mqtt_password: 
    environment: MQTT_PASSWORD
```

Docker Compose will mount the values of `MQTT_USERNAME` and `MQTT_PASSWORD` into the `/run/secrets/mqtt_username` and `/run/secrets/mqtt_password` file within the container at runtime. We also set the file permissions to world-readable (0444) so that all users within the container may be able to read the values

Similarly we adapt the telegraf configuration file with the Docker Secret Store plugin and the MQTT credentials as follows:

```toml
[[ inputs.mqtt_consumer ]]
  servers = [ “tcp://<broker_ip>:1883” ]
  topics = [ "test1/#", "test2/+/topic" ]
  username = “@{custom_secretstore:mqtt_username}”
  password = “@{custom_secretstore:mqtt_password}”

[[ secretstores.docker ]]
  id = “custom_secretstore”
```

Bringing the container up with `docker compose up` will let Telegraf connect to the MQTT Broker and subscribe to the required topics.

A benefit with this method is that the previously visible environment variables within the docker container are now safely mounted into the container at runtime and upon inspection via

```bash
docker compose exec telegraf env
```
Or

```bash
docker inspect -f “{{ .Config.Env }}” <telegraf_container_name/ telegraf_container_id>
```

The environment variables won’t be visible and provide a bit more safe usage of our credentials.

### Splitting Telegraf Configuration Files

There may be situations where a Telegraf Configuration file starts to grow in size and the number of plugin configurations would be better maintained if they were to be split into individual files. Telegraf is able to consume multiple configuration plugin files and is able to process them one-by-one to run the telegraf agent as if these files were merged together (it is not a simple merge however, there is some more effort that the source code has to undertake).

It is possible to mount a directory with split configuration files to the telegraf Compose service as follows:

```
telegraf/
| - inputs.<plugin_name#1>.conf
| - inputs.<plugin_name#2>.conf
| - outputs.<plugin_name#1>.conf
| - outputs.<plugin_name#2>.conf
| - processors.<plugin_name#1>.conf
| - processors.<plugin_name#2>.conf
| - secretstores.<plugin_name>.conf
```

The respective Compose file is as follows:

```yaml
services:
  telegraf:
    image: telegraf:latest
    container_name: telegraf
    volumes:
      - ./telegraf:/etc/telegraf/telegraf.d/
```

We notify telegraf that the configuration is to be picked up from a directory i.e. `/etc/telegraf/telegraf.d/` and not a standalone configuration file here. 

> NOTE: It is recommended to use the `order` parameter in processor plugins to let telegraf know in which order the data coming in from input plugins needs to be manipulated.

Of course, the split configuration files can be used together with Docker Secrets.

### Docker Config for Telegraf Configuration File

There is also a possibility where a telegraf configuration file can be mounted via [Docker Configs feature][2] instead of volume mounts. 
According to the [Compose specification document][3] for Docker Configs:

> Configs allow services to adapt their behaviour without the need to rebuild a Docker image.


Mounting Telegraf Configurations files via Docker Config is very seldomly used in deployments with stand alone Compose applications, however they may be beneficial when working with custom Docker images for Telegraf.

A common Compose file for Telegraf

```yaml
services:
  image: telegraf:latest
  container_name: telegraf
  Volumes:
./telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro
```

Can be transformed to let Compose mount the telegraf configuration file via Docker Config as follows:

```yaml
services:
  image: telegraf:latest
  container_name: telegraf
  command: --config /telegraf_conf
  configs:
    - telegraf_conf

configs:
  telegraf_conf:
    file: ./telegraf/telegraf.conf
```
By default, the configuration is mounted in the root of the container with the name of the docker config mapped to the file of the configuration file. When using such a configuration it is essential to mention within the `command` parameter where should telegraf load the configuration file from i.e. `command: –-config /telegraf_conf`


## Inference

This guide should provide a baseline for how Engineers can plan to design their Telegraf Configuration file(s) with Docker Compose v2 as a deployment tool. This is not an exhaustive guide by any means, but it explores some new features in Telegraf (Docker Secret Store) and some less familiar Docker Compose v2 specifications that may not be often used in deployments.

Resources / Links

- [Telegraf’s Secretstores Plugin implementation on GitHub][4]
- [Telegrafs’ Docker Secret Store Plugin Implementation on GitHub][5]
- [Compose Specification][6]
- [Additonal Information on Docker Secrets with Docker Compose via Environment Variables - Source Docker Blog][7]

[1]: https://github.com/compose-spec/compose-spec/blob/master/09-secrets.md
[2]: https://github.com/compose-spec/compose-spec/blob/master/08-configs.md
[3]: https://github.com/compose-spec/compose-spec/blob/master/08-configs.md#configs-top-level-element
[4]: https://github.com/influxdata/telegraf/tree/master/plugins/secretstores
[5]: https://github.com/influxdata/telegraf/tree/master/plugins/secretstores/docker
[6]: https://compose-spec.io/
[7]: https://www.docker.com/blog/new-docker-compose-v2-and-v1-deprecation/
