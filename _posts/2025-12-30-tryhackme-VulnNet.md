---
title: "TryHackMe: VulnNet:Roasted"
author: mitcheka
categories: [TryHackMe]
tags: [windows, Active directory, Kerberos]
render_with_liquid: false
media_subpath: /images/tryhackme-vulnnet/
image:
  path: vuln.webp
---

VulnNet is a purely active directory box where we find usernames from an `SMB` share and using `lookupsids` script from impacket to perform `RID Bruteforce` getting usernames from the server.Next `ASREPRoast` is used to get a hash thats cracked getting a password that accesses a **NETLOGON** share containing a `vbs` script which has credentials for a user in the domain admins group where we use `secretsdump` to dump  NTLM hashes of the Administrator and gain access using `impacket-wmiexec` and `Pass-the-Hash`.

![room card index](vuln-card.webp){: width="300" height="300" }
## Initial Enumeration
### Nmap scan

```console
$ nmap -sC -sV -vv -Pn -T4 -p- 10.80.159.82
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 126 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 126 Microsoft Windows Kerberos (server time: 2025-12-30 14:21:11Z)
135/tcp   open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 126 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 126 Microsoft Windows Active Directory LDAP (Domain: vulnnet-rst.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 126
464/tcp   open  kpasswd5?     syn-ack ttl 126
593/tcp   open  ncacn_http    syn-ack ttl 126 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 126
3268/tcp  open  ldap          syn-ack ttl 126 Microsoft Windows Active Directory LDAP (Domain: vulnnet-rst.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 126
5985/tcp  open  http          syn-ack ttl 126 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        syn-ack ttl 126 .NET Message Framing
49666/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49673/tcp open  ncacn_http    syn-ack ttl 126 Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49677/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49703/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
Service Info: Host: WIN-2BO8M1OE1M1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 0s
| smb2-time: 
|   date: 2025-12-30T14:22:06
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 54861/tcp): CLEAN (Timeout)
|   Check 2 (port 39066/tcp): CLEAN (Timeout)
|   Check 3 (port 11686/udp): CLEAN (Timeout)
|   Check 4 (port 55931/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
```

Based on the nmap scan we find alot of open ports which might suggest we are dealing with a domain controller.

### SMB
As always we start with enumerating the `SMB` shares.
Using `smbclient` we get two shares that are non-standard.

![shares index](smbc-shares.webp){: width="800" height="600" }

This doesn't show whether we have read access to these shares though so we use `netexec` to enumerate shares on the box and it shows we have read access to `VulnNet-Business-Anonymous` and `VulnNet-Enterprise-Anonymous` shares

![smb guest index](smb-guest-shares.webp){: width="800" height="600" }

Before I proceed I add the domain name `vulnnet-rst.local` to `/etc/hosts`

Using `spider_plus` module on `netexec` I was able to see all the files present in those shares but I won't go into that cause I did not find valuable information.

### Impacket
I resort to use `impacket-lookupsid` so that we can try to do an `RID` bruteforce to get a list of usernames.We get a hit and I save the usernames to a file.

![lookupsid index](rid-brute.webp){: width="800" height="600" }

Next we have to test if any of the users was `ASREPRoastable` meaning a user didn't require credentials to request for a    `Ticket Granting Ticket` from the `Key Distribution Centre`.The tool used was `GetNPUsers` from impackets suite.

![asrep index](asrep.webp){: width="800" height="600" }

We get a hash from the user `t-skid`
Cracking the hash using john the ripper we get a valid password.

![john index](john-asp.webp){: width="800" height="600" }

I try accessing SMB share using these credentials and we have access to alot more shares.

![skid share index](t-skid-shares.webp){: width="800" height="600" }

Next I move to check `NETLOGON`  and `SYSVOL`  shares but NETLOGON has a file that in turn unravels credentials for the user `a-whitehat`

```console
$ smbclient \\\\10.82.170.114\\NETLOGON -U vulnnet-rst.local\\t-skid
Password for [VULNNET-RST.LOCAL\t-skid]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Mar 17 02:15:49 2021
  ..                                  D        0  Wed Mar 17 02:15:49 2021
  ResetPassword.vbs                   A     2821  Wed Mar 17 02:18:14 2021

                8771839 blocks of size 4096. 4530915 blocks available
smb: \> get ResetPassword.vbs
getting file \ResetPassword.vbs of size 2821 as ResetPassword.vbs (3.7 KiloBytes/sec) (average 3.7 KiloBytes/sec)
smb: \> 
```

This section is where we get the credentials after opening the vbs script

![reset index](reset-pass.webp){: width="800" height="600" }

Testing the credentials using `netexec`,we get a `pwned` signed meaning we can use `impacket-wmiexec` to get a shell on the box

```console
$ netexec smb 10.82.170.114 -u a-whitehat -p "bNdKVkjv3RR9ht"
SMB         10.82.170.114   445    WIN-2BO8M1OE1M1  [*] Windows 10 / Server 2019 Build 17763 x64 (name:WIN-2BO8M1OE1M1) (domain:vulnnet-rst.local) (signing:True) (SMBv1:False)
SMB         10.82.170.114   445    WIN-2BO8M1OE1M1  [+] vulnnet-rst.local\a-whitehat:bNdKVkjv3RR9ht (Pwn3d!)
```
## Exploitation
Using the `impacket-wmiexec` command we get a shell as user `a-whitehat` and subsequently our `user.txt` flag

![shell index](wmiexec-whitehat.webp){: width="800" height="600" }

## Post-exploitation

Running `whoami /groups` we can see the account is part of `domain admins` group which means we can dump the entire SAM database and get access to all the hashes from any user.

![enum index](group-enum.webp){: width="800" height="600" }

Using `secretsdump` from impackets suite I dump all the hashes and the one hash I am interested in is the administrators **NTLM hash**

![hashdump index](hashdump.webp){: width="800" height="600" }

We can use `Pass-the-Hash` technique together with `impacket-wmiexec` to gain access to the administrator account.Also the final flag `system.txt` is also captured.

![system index](wmiexec-admin.webp){: width="800" height="600" }

On a sidenote ,tryhackme's vm crashed at some point had to start again which would explain difference in IP towards the end

`Thats's a wrap!!!`



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
