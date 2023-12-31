---
title: AVenger (THM)
image: assets/writeups/trace1.jpeg
description: Write-up of AVenger Machine 
date: 2023-11-26
categories: [tryhackme]
tags: [tryhackme, thm, writeup, medium, machine, windows, fileupload, privesc ]
author: trace
---

![](https://tryhackme-images.s3.amazonaws.com/room-icons/07fe26c8113c521c23f979ce7829147a.png)

## Reconnaissance

We begin with the basic Nmap scan:

```bash
┌──(trace㉿0xtrace)-[~/Documents/machines/avenger]
└─$ nmap -sC -sV 10.10.164.125 
Starting Nmap 7.80 ( https://nmap.org ) at 2023-11-26 14:30 IST
Stats: 0:00:56 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 85.71% done; ETC: 14:31 (0:00:03 remaining)
Nmap scan report for 10.10.164.125 (10.10.164.125)
Host is up (0.21s latency).
Not shown: 993 closed ports
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Apache httpd 2.4.56 (OpenSSL/1.1.1t PHP/8.0.28)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
|_http-title: Index of /
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp  open  ssl/http      Apache httpd 2.4.56 (OpenSSL/1.1.1t PHP/8.0.28)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.0.28
|_http-title: Index of /
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
445/tcp  open  microsoft-ds?
3306/tcp open  mysql         MySQL 5.5.5-10.4.28-MariaDB
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.4.28-MariaDB
|   Thread ID: 10
|   Capabilities flags: 63486
|   Some Capabilities: Speaks41ProtocolNew, Speaks41ProtocolOld, InteractiveClient, DontAllowDatabaseTableColumn, IgnoreSigpipes, LongColumnFlag, Support41Auth, IgnoreSpaceBeforeParenthesis, SupportsTransactions, SupportsLoadDataLocal, FoundRows, ConnectWithDatabase, ODBCClient, SupportsCompression, SupportsMultipleResults, SupportsAuthPlugins, SupportsMultipleStatments
|   Status: Autocommit
|   Salt: voHZm5]VvqR'i_|vY=bW
|_  Auth Plugin Name: mysql_native_password
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: GIFT
|   NetBIOS_Domain_Name: GIFT
|   NetBIOS_Computer_Name: GIFT
|   DNS_Domain_Name: gift
|   DNS_Computer_Name: gift
|   Product_Version: 10.0.17763
|_  System_Time: 2023-11-26T09:02:08+00:00
| ssl-cert: Subject: commonName=gift
| Not valid before: 2023-06-29T08:09:48
|_Not valid after:  2023-12-29T08:09:48
|_ssl-date: 2023-11-26T09:02:17+00:00; +24s from scanner time.
Service Info: Hosts: localhost, www.example.com; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 23s, deviation: 0s, median: 23s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-11-26T09:02:11
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 69.95 seconds
Segmentation fault (core dumped)
```

We observe several open ports, but we will delve into them later. First, let's visit the site and utilize `gobuster` to enumerate web directories.

```bash
┌──(trace㉿0xtrace)-[~/Documents/machines/avenger]
└─$ gobuster -u http://10.10.164.125/ -w /usr/share/wordlists/Seclists/common.txt 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.164.125/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/Seclists/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2023/11/26 14:31:18 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htpasswd (Status: 403)
/.htaccess (Status: 403)
/aux (Status: 403)
/cgi-bin/ (Status: 403)
/com2 (Status: 403)
/com3 (Status: 403)
/com4 (Status: 403)
/com1 (Status: 403)
/con (Status: 403)
/dashboard (Status: 301)
/favicon.ico (Status: 200)
/gift (Status: 301)
/img (Status: 301)
/licenses (Status: 403)
/lpt1 (Status: 403)
/lpt2 (Status: 403)
/nul (Status: 403)
/phpmyadmin (Status: 403)
/prn (Status: 403)
/server-info (Status: 403)
/server-status (Status: 403)
/webalizer (Status: 403)
/wordpress (Status: 301)
=====================================================
2023/11/26 14:33:06 Finished
=====================================================
```

We find several 301 and 403 responses. Let's visit the 301 `dir.` which appears to be the most interesting - `/gift`.

![](https://imgur.com/Fy7E0Hh.png)

Upon visiting `/gift`, we discover a blog-like page. Scrolling down, we find a contact page with an upload button.

![](https://imgur.com/LNTR2W0.png)

![](https://imgur.com/vikzzWG.png)

Now, it's time to obtain a shell. Despite encountering various issues, persistence pays off. After numerous attempts, a breakthrough occurs when considering that the server may be running something like ```cmd.exe /c "payload_goes_here"```. Referencing documentation and blogs, a helpful [blog post on command execution to shell](https://mayfly277.github.io/posts/GOADv2-pwning-part7/#command-execution-to-shell) guides the creation of a payload for upload.

![](https://i.imgur.com/cmC9Hus.png)

We create a `.cmd` file, upload it, and patiently wait for execution.

![](https://imgur.com/HmuJoIa.png)

We successfully obtain our first flag located in `/Users/hugo/Desktop`.

## Foothold

Now that we have the shell, navigating to `Administrator` reveals obscured content due to UAC restrictions. The solution involves acquiring Hugo's credentials to RDP into the machine, as observed from the open RDP port during the initial Nmap scan.

Executing `PrivescCheck.ps1` yields the necessary credentials.

![](https://i.imgur.com/IKNYADu.png?1)

Using the obtained credentials, RDP into the machine as user Hugo:

```bash
xfreerdp /u:hugo /v:10.10.116.150
```

Subsequently, run CMD as `Administrator`, accept the UAC prompt, and obtain the root flag.

![](https://i.imgur.com/HwOk6xp.png?1)





