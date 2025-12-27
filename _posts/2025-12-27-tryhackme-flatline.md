---
title: "TryHackMe: Flatline"
author: mitcheka
categories: [TryHackMe]
tags: [Privilege Escalation, windows, token impersonation, command execution]
render_with_liquid: false
media_subpath: /images/tryhackme-flatline/
image:
  path: flatline.jpeg
---

Flatline was a simple room where I used a vulnerability exploit from `exploit-db` to gain an initial foothold through `command execution` and eventually escalate privileges by abusing `token impersonation`.

![card index](flatline-card.png){: width="300" height="300" }

## Initial Enumeration
### Nmap scan

```console
$ nmap -sC -sV -vv -Pn -T4 -p- 10.81.142.210
PORT     STATE SERVICE          REASON          VERSION
3389/tcp open  ms-wbt-server    syn-ack ttl 126 Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: WIN-EOM4PK0578N
|   NetBIOS_Domain_Name: WIN-EOM4PK0578N
|   NetBIOS_Computer_Name: WIN-EOM4PK0578N
|   DNS_Domain_Name: WIN-EOM4PK0578N
|   DNS_Computer_Name: WIN-EOM4PK0578N
|   Product_Version: 10.0.17763
|_  System_Time: 2025-12-27T11:46:14+00:00
|_ssl-date: 2025-12-27T11:46:19+00:00; +3s from scanner time.
| ssl-cert: Subject: commonName=WIN-EOM4PK0578N
| Issuer: commonName=WIN-EOM4PK0578N
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-12-26T11:34:14
| Not valid after:  2026-06-27T11:34:14
| MD5:   0435:38f5:8b5e:f392:6158:d8fe:4a10:f856
| SHA-1: 8b3d:f7a0:ddf2:3a31:2092:e9ee:2422:321d:1a93:548f
| -----BEGIN CERTIFICATE-----
| MIIC4jCCAcqgAwIBAgIQYwYITwDDV7pLMC+O02dkNzANBgkqhkiG9w0BAQsFADAa
| MRgwFgYDVQQDEw9XSU4tRU9NNFBLMDU3OE4wHhcNMjUxMjI2MTEzNDE0WhcNMjYw
| NjI3MTEzNDE0WjAaMRgwFgYDVQQDEw9XSU4tRU9NNFBLMDU3OE4wggEiMA0GCSqG
| SIb3DQEBAQUAA4IBDwAwggEKAoIBAQDZ98I2j8Mhb2zwNVmElqqOgiKWJoWQtl1v
| 7rigKt+M63ZX3bPdaAbeTpU4rTkSIyumnItd3KINe3aS798gn30tzSwVM4/3WT3N
| /av82JgKfsi3zYKZIQPy9xvQnWsY2pd6y3EvZ0ZPI4vc3q4hyNTFQMRuFAakefD5
| j84sNwmOdE/aayEJp8P9fIQQaPP9+3dObW7DsQo7pV1nF2Ah7rD38nf+3qtPit1Y
| jFU6wWljpGibmLHtKHjU1/WNxTVAJsYwtkzcQJin+KppOeuHSVHPJS6VvEetKn9q
| 3LzULzet6nyejdnWXN15/Q8/5ISTxCMrW7a47kTqD34WsEhMC9DhAgMBAAGjJDAi
| MBMGA1UdJQQMMAoGCCsGAQUFBwMBMAsGA1UdDwQEAwIEMDANBgkqhkiG9w0BAQsF
| AAOCAQEAdHP9lVdWGMhPQbShLV68KXYQIoZ8eo5OfAzv8bqgXXzwK6t778f0Rchb
| 2bsrE0RkT8s0ifBaY4igBVA3dW47kpSJNIU1Ex+g5x4LRzp13gExAEiGlDyqv8Xh
| 4TYvccU0eCBcj+DbtV0T3/nfjAMi4zuOumAHtchTpOLN29bUd+of7mkQkje3gIaK
| gTy/Vhk/8iyA5zN7Vj1SpowfpxjwbCgdK6bkxOvRkOvbkLXuDxiL0y4uF/xnPQzk
| y2yq+Bpg19R5wkgG44CNDJ7WhDhDOtKBawOGt4mxa9Zd3p4xAlzxplGQItd5SDL+
| YgnAfVSBmvdjukxMUu9K9aZmlZdGOg==
|_-----END CERTIFICATE-----
8021/tcp open  freeswitch-event syn-ack ttl 126 FreeSWITCH mod_event_socket
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Based on the nmap scan there are two ports;
- **Port 3389** `RDP`
- **Port 8021** `FreeSWITCH`

## OSINT
My first action is to research the `freeSWITCH` service which looks interesting and I manage to get a vulnerability associated with it on `exploit-db` which is `EDB-ID:47799`.

![exploit-db index](exploit-db.webp){: width="1200" height="600" }

I extract the python script and create a file on my attackbox `flatline.py`.

## Initial foothold

One way to find out if the script works based on the comments is to test it and this yields results as we get the name of the user..

![enum index](enum1.webp){: width="1200" height="600" }

My plan now is to try and get a reverse shell using the same exploit.
I go back to my attackbox to create a payload using `msfvenom`

![msfvenom index](msfvenom.webp){: width="1200" height="600" }

Next I  setup a webserver on my attackbox and use `certutil` on the target machine to transfer the payload from my attackbox to the target via the server.

![flatline transfer index](flatline-transfer.webp){: width="1200" height="600" }

![webserver index](py-server2.webp){: width="1200" height="600" }

I then use the exploit to run the payload which ultimately gives a foothold.

![foothold index](flatline-exe.webp){: width="1200" height="600" }
![foothold2 index](initial-access.webp){: width="1200" height="600" }

Looking around we are able to acquire the first flag.

![flag index](user.txt.webp){: width="1200" height="600" }

## Post-exploitation
### Privilege Escalation

First step is to check for user privileges by determining rights,permissions and integrity level of the current user using `whoami /priv`.

![privesc index](privesc-vectors.webp){: width="1200" height="600" }

We find out that the user has `SeImpersonatePrivilege` rights.

The `printspoofer` exploit can be used to escalate our privilege.First things first I have to get the executable on our compromised machine using `certutil`.
Executing the `printspoofer.exe` using the following command we get the highest level of privilege `NT AUTHORITY\SYSTEM`.Also we are able to capture the second and last flag of the room.

![root index](root.webp){: width="1200" height="600" }



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
