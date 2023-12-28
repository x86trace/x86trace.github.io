---
title: "Set the Hostname of a Docker Container same as your Host Machine"
date: "2022-06-08T20:31:18+02:00"
draft: false

author: "Shan"
tags: ["docker", "docker-compose", "alpine", "bash"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## __UPDATE__

With [Docker Compose v2.15.1](https://docs.docker.com/compose/release-notes/#2151)
the solution is now trivial by simply using `uts: host` in a compose file as follows:

```yaml
services:
  test:
    image: alpine:latest
    container_name: hostname-tester
    uts: host
    command:
      - hostname
```

Or by using the Docker CLI:

```bash
docker run --uts=host alpine:latest hostname
```

## Setting Container Hostname to be same as your Host Machine

Sounds trivial, but it turns out you need to do some tweaks in order to set the hostname
to be the same as that of Host Machine.

### Scenario
I had a requirement at work, where I needed to set the hostname of a specific container
to be that of the Machine it was supposed to run on.

The Linux Distribution under consideration was __Ubuntu 20.04 LTS__.

### Ubuntu's mysterious `HOST` and `HOSTNAME` environment variables

If you are on an Ubuntu machine right now, try `echo $HOS` and press the <kbd>TAB</kbd>
key to let the bash completion fill out. High chances that you will see `HOST` and `HOSTNAME`
as available Environment Variables already available to your bash shell's session.

Fairly Simple, you could simply use either one of them in your Compose file as an environment
variable to the `hostname` key and should work out of the box! Not quite!

### Investigation via a simple Example

Take the following `docker-compose.yml` file

```yaml
services:
  test:
    image: alpine:latest
    container_name: hostname-tester
    hostname: alpine-${HOST}
    command:
      - hostname
```

The following `test` service should be able to retrieve the `HOST` variable value and set it as 
the hostname of the alpine container. The `command` should be able to print something like 
`alpine-my-ubuntu` as an example.

Let's see what happens when we run `docker compose up`

```bash
WARN[0000] The "HOST" variable is not set. Defaulting to a blank string. 
[+] Running 1/1
 ⠿ Container hostname-tester  Recreated                                    0.1s
Attaching to hostname-tester
hostname-tester  | alpine-
hostname-tester exited with code 0
```
As you see the `HOST` variable although available to the shell is not available within the 
Compose file.

Most of the shell environment variables are visible via using `printenv` on the host machine, 
so a quick search for `HOST` or `HOSTNAME` reveals that these specific environment variables
are not in the `env` of th shell

```bash
printenv | grep -i "host"
```

### Solution

The most standard way to make a variable available for a shell session is by exporting it.

Let's export `HOST` using:

```bash
export HOST
```

Let's try bringing the container back up again and see if it picks up the variable values

```bash
❯ export HOST
❯ docker compose up
[+] Running 1/0
 ⠿ Container hostname-tester  Recreated                                    0.0s
Attaching to hostname-tester
hostname-tester  | alpine-shan-pc
hostname-tester exited with code 0
```

for my machine (__Manjaro Linux (Rolling)__) I could now set the hostname of the container
to be the same as that of the Host Machine.

Sure enough searching through `printenv` again you will find the export `HOST` variable.

If you have on-prem servers or cloud images where you might need such a configuration a 
solution would be to add `export HOST` to your user's `~/.bashrc` or `~/.zshrc` (if using ZSH)
and the value will be available when bringing the corresponding Compose Stack up.

Hope this information helps people trying to find a similar solutions when it comes to 
Docker and Docker Compose based environments.

If you have a better solution reach out to me (on LinkedIn) and would be happy to learn and improve!
