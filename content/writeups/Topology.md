---
title: Topology (HTB) / Easy
image: /assets/writeups/Trends International Call of Duty_ Modern Warfare 2 - Ghost Tarot Card Wall Poster, 22_375_ x 34_, Silver Framed Version.jpeg
description: Write-up of Topology Machine (Hackthebox * Hacker's Wrath) 
date: 2023-11-02
categories: [hackthebox]
tags: [hackthebox, htb, writeup, easy, machine, linux, code-injection, hashcracking]
author: trace
---

## Introduction

Welcome to the HackTheBox machine "Topology" writeup. In this writeup, we will guide you through the process of exploiting this Linux-based HackTheBox machine, including port scanning, enumeration, user privilege escalation, and root privilege escalation.

**Machine Information:**
- Machine Name: Topology
- Platform: HackTheBox
- Difficulty: Easy

## Table of Contents

1. [Port Scan](#port-scan)
2. [Enumeration](#enumeration)
3. [Getting the User Flag](#getting-the-user-flag)
4. [Privilege Escalation](#privilege-escalation)
5. [Getting the Root Flag](#getting-the-root-flag)

## Port Scan <a name="port-scan"></a>

To start our journey, we need to perform a port scan to identify open ports on the target machine. We add the IP of the Topology machine (10.10.11.217) to our `/etc/hosts` file as `topology.htb` and run an initial nmap port scan.

```bash
$ nmap -p- --open -sS --min-rate 5000 -n -vvv -oA enumeration/nmap1 10.10.11.217
```

The results show that two ports are open:

- Port 22: SSH
- Port 80: HTTP

We continue with a more detailed scan to gather additional information about these services.

```bash
$ nmap -sCV -p 22,80 -oA enumeration/nmap2 10.10.11.217
```

The detailed scan reveals the following information:

- Port 22: OpenSSH 8.2p1 (Ubuntu)
- Port 80: Apache HTTP Server 2.4.41 (Ubuntu)

## Enumeration <a name="enumeration"></a>

After identifying the open ports, we access the web portal on port 80 and find a page related to Miskatonic University's Topology Group.

![Miskatonic University Page](https://byte-mind.net/wp-content/uploads/2023/06/topology.png)

Upon further investigation, we discover a link to the `latex.topology.htb` domain on the page. We add this domain to our hosts file and access it, revealing a page related to LaTeX software.

The portal uses LaTeX, a document creation software known for its high typographic quality. Researching LaTeX hacking, we find information on abusing the service. We manage to perform an LFI (Local File Inclusion) on the `/etc/passwd` file with a specific payload.

```latex
\newread\file
\openin\file=/etc/passwd
\read\file to\line
\text{\line}
\closein\file
```

Although this payload only returns one line of the file, we discover a way to exploit the service by adding a file with PHP code and reading the complete `passwd` file.

```latex
\begin{filecontents}[overwrite]{kekma.php}
<?php print_r(file_get_contents("/etc/passwd")); ?>
\end{filecontents*}
```

With this, we successfully retrieve some credentials from the file.

## Getting the User Flag <a name="getting-the-user-flag"></a>

To gain user access, we use the obtained credentials to SSH into the machine.

```bash
$ ssh vdaisley@topology.htb
```

Once inside, we proceed to crack the user's password hash using John the Ripper.

```bash
$ john hash -w=/usr/share/wordlists/rockyou.txt
```

After successfully cracking the password, we can access the user's home directory and retrieve the user flag.

## Privilege Escalation <a name="privilege-escalation"></a>

During enumeration, we discover an interesting process running on the machine. This process, launched by root, utilizes gnuplot, a program for generating graphs. It runs existing plt templates in the `/opt/gnuplot` directory.

Although we cannot read the directory, we can write to it. We create a template that adds suid permissions to the `bash` binary.

We copy our template to the `/opt/gnuplot` directory and wait for it to run again. When we check the permissions, it has worked.

With escalated privileges, we can access the root flag.

## Getting the Root Flag <a name="getting-the-root-flag"></a>

Now, with root permissions, we retrieve the root flag.

```bash
$ cat /root/root.txt
```
