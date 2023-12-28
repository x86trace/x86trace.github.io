---
title: "Generating Mosquitto MQTT Broker Credentials with Ansible"
date: "2023-08-16T13:00:00+02:00"
draft: false

author: "Shan"
tags: ["devops", "ansible", "mosquitto", "mqtt", "jinja2", "python3", "IIoT", "IoT"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Challenge

This is a development note from a personal, open-source project called [__Komponist__](https://github.com/shantanoo-desai/komponist).
The aim of the project is to provide IoT / IIoT Developers a platform that can be used locally as well as in production using commonly used containers 
in the Edge computing space such as:
- Time-series Databases (InfluxDBv1/v2, QuestDB, TimescaleDB)
- Data processing tools (Node-RED, Telegraf)
- Dashboard visualization / monitoring (Grafana)
- Message Broker (Eclipse-Mosquitto)
There are standard tools / software used quite ubiquitously with containerization technology like __Docker / Compose v2__.

One major challenge that developers face is to maintain a lot of files that are needed to bring these tools / software up on initialization. 
These files mostly contain user credentials, initialization scripts and configurations needed to start these containers correctly and according to user specifications.
Additionally, each container of these tools / software are very distinct and may rely on various setups, markup languages (YAML, TOML, INI) etc.
and there is rarely any coherence available between one tool against the other.

Komponist solves this issue by add a logical configuration generation tool over the tools and relies only on two core files
that describes the complete stack along side the respective credentials and configurations. 
It uses the easy to use `ansible` configuration management tool to generate the respective docker-compose files, credentials file, YAML and TOML files for the software
and prepares the stack within a couple of minutes either locally or within a group of IoT edge devices.

This writeup focuses more on the Mosquitto MQTT Broker. The writeup describes how to leverage Ansible and Jinja2 powered Templating engine to generate Mosquitto Broker's
credentials.

NOTE: This writeup requires some basic to intermediate knowledge of [Ansible](https://ansible.com) as a tool.
## Mosquitto Broker's Credentials Generation

### `mosquitto_passwd` CLI native

According to the [mosquitto broker's authentication methods documentation](https://mosquitto.org/man/mosquitto_passwd-1.html)
it is possible to generate password files when one either has the `mosquitto_passwd` CLI installed locally / remotely on the IoT devices.
For example, the following plain-text `passwd.txt` file:

```
user1:superSecretPass1
user2:superSecretPass2
```

will encrypt the plain-text passwords into hashes that are acceptable by the mosquitto broker via the `mosquitto-passwd` CLI using:

```bash
mosquitto_passwd -U /path/to/passwd.txt
```

However, it relies on the existence of `mosquitto_passwd` tool to be available locally.

### `mosquitto_passwd` CLI via Docker Container

The same tool can be used via a Docker container where the plain-text password `passwd.txt` file can be encrypted via spinning a docker container
locally and mounting the plain-text file to a dedicated path into it. An example with the same `passwd.txt` can be:

```bash
docker run --rm -v /path/to/passwd.txt:/mosquitto/config/users \
eclipse-mosquitto:2.0.15 \
mosquitto_passwd -U /mosquitto/config/users
```

Since the file is mounted into the container, any changes within the container will be reflected on the `passwd.txt` on the host i.e., the plain-text passwords will be encrypted accordingly.

> Both these methods, require dependency on either the `mosquitto_passwd` CLI or the `eclipse-mosquitto` Docker Image to exist locally beforehand.

## Independent Credential Generation for Broker

Upon observing the code base for [eclipse/mosquitto](https://github.com/eclipse/mosquitto/blob/master/src/password_mosq.h) it turns out the password hash generation relies on two methods:
1. SHA512 + additional salting and base64 encryption logic
2. SHA512 type PBKDF2 encryption logic

The details of the these two methods are beyond the scope of the write up but the implementations for them are quite common and can be found in every programming language.

Since __Komponist__ relies on Ansible and Jinja2 templating engine which are directly reliant on Python3.x as a programming language, the implementation is rather trivial thanks to [`passlib`](https://passlib.readthedocs.io/en/stable/index.html) python package.
`passlib` provides the implementation of __PBKDF2_SHA512__ logic with all the necessary encoding logic needed for the hashing of plain-text passwords that are acceptable by the Broker.

### Templates for Password File

As previously seen a `passwd.txt` an acceptable structure is:

```
user1:password1
user2:password2
userN:passwordN
```
A `users.j2` template file can be created as follows:

```jinja2
{%- for user in mosquitto.users %}
{{ user.username }}:{{ user.password }}
{% endfor %}
```

where the `mosquitto.users` is an array of dictionaries:

```
mosquitto:
  users:
    - username: user1
      password: password1
    - username: user2
      password: password2
```

To test this logic we can create an Ansible Playbook  called `mosquitto_users.yml` as follows:

```yaml
- name: Mosquitto Users (plain-text) file generation
  hosts: localhost
  gather_facts: false
  vars:
    mosquitto:
      users:
        - username: testuser1
          password: password1
        - username: testuser2
          password: password2

  tasks:
    - name: generate the Mosquitto users file
      ansible.builtin.template:
        src: users.j2
        dest: users
```

Execute the playbook using:

```bash
$ tree
├── mosquitto_users.yml
└── users.j2
$ ansible-playbook mosquitto_users.yml
```
upon execution of the playbook a `users` file will be generated that will fill the values of the credentials accordingly.

### Custom Jinja2 Filter for Hash Generation

The template file `users.j2` currently fulfills the criteria of structure required for the Broker.
We now create a custom Jinja2 filter that will consume the plain-text passwords and generate the acceptable hashes for the Broker.

A Jinja2 filter in Ansible is rather a very simple python script that Ansible executes to generate the respective logic.
As previously mentioned, we know that `passlib` library provides a [__PBKDF2_SHA512___](https://passlib.readthedocs.io/en/stable/lib/passlib.hash.pbkdf2_digest.html)
implementation and upon inspecting the code base for eclipse mosquitto we can also determine how many rounds(iterations) are required and what the salt length should be.
The values are as follows:
1. Iterations are 101
2. salt length is 12 bytes (random bytes)
Based on the `passlib` API this information can be passed into a function call rather easily.

Let's call the filter the same as `mosquitto_passwd`  . Here is the implementation:

1. Create a folder called `filter_plugins` and create `mosquitto_passwd.py` under it
2. install the `passlib` package using `pip install --user passlib`
3. Add the following code into the `mosquitto_passwd.py` 
   
```python
from ansible.errors import AnsibleError

def mosquitto_passwd(passwd):
    try:
        import passlib.hash
    except Exception as e:
        raise AnsibleError('mosquitto_passlib custom filter requires the passlib pip package installed')
    
    SALT_SIZE = 12
    ITERATIONS = 101

    digest = passlib.hash.pbkdf2_sha512.using(salt_size=SALT_SIZE, rounds=ITERATIONS) \
                                        .hash(passwd) \
                                        .replace("pbkdf2-sha512", "7") \
                                        .replace(".", "+")
    
    return digest + "=="


class FilterModule(object):
    def filters(self):
        return {
            'mosquitto_passwd': mosquitto_passwd,
        }   
```

Few things to note in the implementation:
- Hash digest generated by `passlib.hash.pbkdf2_sha512` is the of the following structure:
   `$pbkdf2-sha256$6400$0ZrzXitFSGltTQnBWOsdAw$Y11AchqV4b0sUisdZd0Xr97KWoymNE0LNNrnEgY4H9M`
- Mosquitto Broker requires that the first part of the digest either be `7` for PBKDF2_SHA512 or `6` for the SHA-512. This is done by the `.replace("pbkdf2_sha512", "7")` in the code
- `passlib` PBKDF2_SHA512 implementation has a [shortened base64 format](https://passlib.readthedocs.io/en/stable/lib/passlib.utils.binary.html#passlib.utils.binary.ab64_encode) which omits padding and whitespaces, which will not be acceptable by the Broker.
In order to make the digest compatible we replace the instances of `.` with `+`  in the digest (via `.replace(".", "+")`)
- we add the `==` characters at the end of the digest to make up for the shortened base64 encoding implementation when it comes to the overall length of the digest

The final step should is to add the custom `mosquitto_passwd` filter into our `users.j2` file as follows:

```jinja2
{%- for user in mosquitto.users %}
{{ user.username }}:{{ user.password | mosquitto_passwd }}
{% endfor %}
```
the template during generation will pass the `user.password` to the `mosquitto_passwd` filter and provide the hashed value as opposed to the plain-text values in a `users` file.

## Results

For the following structure:
```bash
$ tree .
.
├── filter_plugins
│   └── mosquitto_passwd.py
├── mosquitto_users.yml
└── users.j2
```

executing the playbook `mosquitto_users.yml`:

```bash
ansible-playbook mosquitto_users.yml
```
will generate a `users` file in the directory. Upon viewing the contents of the file:

```bash
cat users
testuser1:$7$101$9h4jJKT0HiMEgDAm$3ynyYne/t4KJyjDqOyCVzF0G0Oa71nu0K9iaa1NNfzCNc5c71iVnUVqV+zxDjGkaDy6gSAWAyk6KmqgTaE1IDQ==
testuser2:$7$101$HwNAKGVsjbE2JkRI$Nk+xnkjOTyllT7hD4N5kd9kGarMAQJMJVgWraa0VxmKZgyRyfUjG0+kUCPE1ZuLaGPrbeq0H80Pl6tHd+Qm2Lw==
```

(password hashes may vary, since salt lengths are randomly generated)

### Verifying with the Broker

The best way to confirm that the hashes are acceptable is to generate a `mosquitto.conf` file with the following contents:

```
# MQTT Port Listener
listener    1883
protocol    mqtt

# Authentication
allow_anonymous     false
password_file       /mosquitto_users
```

and then using the following `docker-compose.yml` file:

```yaml
services:
  mosquitto:
    image: docker.io/eclipse-mosquitto:2.0.15
    container_name: test_broker
    configs:
      - mosquitto_conf
      - mosquitto_users
    entrypoint: mosquitto -c /mosquitto_conf
    ports:
      - 1883:1883
configs:
  mosquitto_conf:
    file: ./mosquitto.conf
  mosquitto_users:
    file: ./users
```

upon performing:

```bash
docker compose up
```

You will be able to see the logs of the broker without any erroneous information:

```
test_broker  | 1692181531: mosquitto version 2.0.15 starting
test_broker  | 1692181531: Config loaded from /mosquitto_conf.
test_broker  | 1692181531: Opening ipv4 listen socket on port 1883.
test_broker  | 1692181531: Opening ipv6 listen socket on port 1883.
test_broker  | 1692181531: mosquitto version 2.0.15 running
```

Use an MQTT client and try to connect to the broker `<IP_ADDRESS>:1883`  with either of the credentials values mentioned in `mosquitto.users` and it should work.

Now you have a functioning configuration management tool for your eclipse Mosquitto Broker using a popular DevOps / Configuration Management Tool (Ansible) 
which provides you a guarantee of authentication without having any Mosquitto related software installed beforehand on your IoT Devices / Servers / Industrial Hardware.

## Potential Extensions

You can now generate Jinja2 based template files for the Access-Control Lists (ACLs)  or other related configurations with having to maintain multiple versions of them rather let Ansible generate them for you.

Similar, logic has been developed for [_Komponist_](https://github.com/shantanoo-desai/komponist)
where a user can define how many users and what ACL permissions each user has on topics by just maintaining a single credentials file for the Mosquitto Broker as well as other containers such as InfluxDB, node-RED etc.
