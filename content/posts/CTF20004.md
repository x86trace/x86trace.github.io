---

title: CTF-200-04 (OffSec. Proving Grounds)
image: /assets/writeups/trace7.jpg
description: Write-up of CTF-200-01 Machine
date: 2023-12-27 09:00:00 -3000
categories: [proving grounds]
tags: [FUXA, CVE, linux]
author: trace

---

## Enumeration:

```bash
rustscan -a <IP>
```

and we see 2 ports open, port `80` & port `1881`

#### Port 1881:

It was running FUXA web-based Process Visualization, 

![](../../assets/writeups/2023-12-27-CTF-200-04/exploringport1881.png)

Quick googling around and found a [POC](https://github.com/rodolfomarianocy/Unauthenticated-RCE-FUXA-CVE-2023-33831) to pop a reverse shell,

![](../../assets/writeups/2023-12-27-CTF-200-04/gotroot.png)

As root user too lol
