---
title: "Use Docker Secret to set your Portainer Admin User's Password"
date: 2022-03-09T22:00:00+01:00
draft: false

author: "Shan"
tags: ["portainer", "docker-compose", "docker", "docker-secrets"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Requirements

I was deep-diving on creating a decent-sized `docker-compose` stack with 5+ services in them. 
I was also on the look out for some Deployment Pattern where I didn't want to dump all my
__Environment Variables__ into a giant `.env` file, neither in Production nor in Development.
I decided to create a structure where each service aka. component would have their own dedicated
directory with their respective _secrets_ (credentials) and their configuration files all bundled
together.

I decided upon the following structure:

```bash
ComponentName
├── config.component
└── secrets
    └── .env.component
```
Here the `ComponentName` could your favourite database, the `secrets` directory within this directory
would contain a `.env.database` file with all necessary admin credentials for the initial stack configuration.
The `config.component` file is a dedicated configuration file (if needed)

I think the structure would work _Okay_ since changes to a component's configuration would be reflected
a bit better in the Git History / Logs

## Problem

If you end up having a lot of containers in the stack, `docker-compose logs` only takes ever so far! You might
decide upon __Portainer__ to make the container management a bit easier through it's UI.

One problem, Portainer doesn't support the __Environment Variables__ Configuration pattern as by many container
based software applications.

It relies on executing `command` in your `docker-compose` file to introduce an password into your Portainer's service.

If you have a bit of Software OCD, you are going to hate it! One service not conforming to your dream stack deployment!

## Approach

Here comes [`docker secret`](https://docs.docker.com/engine/swarm/secrets/) to the rescue!

One massive Caveat, according to the documentation, `docker secret` is only available for __Swarm__ Setting, or it ?!

A [blog post from Michael Irwin](https://blog.mikesir87.io/2017/05/using-docker-secrets-during-development/) about using
docker secrets in Development provides a way to trick docker into accepting files as a docker secret and use it into your
service.

Another thing when going through Portainer's Documentation is that, it accepts admin password also through a 
[password file](https://docs.portainer.io/v/ce-2.11/advanced/cli#configuration-flags-available-at-the-command-line)

We could see some Puzzle Pieces falling into place here!

## Solution

Let's start with our structure mentioned above:

```bash
portainer
└── secrets
    └── portainer-admin-password
|- docker-compose.yml
```
Instead of an `.env` we rename the file to `portainer-admin-password` where you can set your password.

Your `docker-compose.yml` will then look as follows:

```yaml
version: '3.7'
services:
  portainer:
      image: portainer/portainer-ce:latest
      container_name: sys_portainer
      restart: always
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
      command:
        - '--admin-password-file=/run/secrets/portainer_admin_password'
        - '--no-analytics'
      networks:
        - some_network
      ports:
        - "9000:9000"
      secrets:
        - portainer_admin_password

networks:
  some_network:
    external: true

secrets:
  portainer_admin_password:
    file: ./portainer/secrets/portainer_admin_password
```

### Breaking it down functionally!

```yaml
secrets:
  portainer_admin_password:
    file: ./portainer/secrets/portainer_admin_password
```
mounts the file as a docker secret into the compose file. Since it is a file mount, it will be
available in the following directory `/run/secrets/<KEY_NAME_IN_SECRETS_section>` since our key
is the same name as that of the file (in order to avoid any discrepancy), the content of the file
will be consumed by the `--admin-password-file` flag during the stack initialization.

Voilá! You now have a similar structure without having to hardcode passwords or sensitive information
into your `docker-compose.yml` file

## Comments, Suggestions

If you would like to suggest me some changes in my design or a better way to tackle this situation, I am all ears
and would love to improve and discuss aspects. You can contact me on `LinkedIn` or via E-Mail.
