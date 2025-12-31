---
title: "TryHackMe: Block"
author: mitcheka
categories: [TryHackMe]
tags: [pcap, wireshark, lsass, pypykatz, smb]
render_with_liquid: false
media_subpath: /images/tryhackme-block/
image:
  path: blockk.webp
---

Short room on `forensics` where we extract hashes from a given `LSASS` dump and using the data to decrypt SMB3 traffic within a given pcap file on wireshark.

![card index](block-card.webp){: width="300" height="300" }

## Initial Enumeration
From the room description we are given a zip archive file ,a packet capture and a memory dump of the `LSASS` process.

```console
$ zipinfo evidence-1697996360986.zip                     
Archive:  evidence-1697996360986.zip
Zip file size: 43679925 bytes, number of entries: 2
-rw-r--r--  3.0 unx    34544 bx defN 23-Oct-22 20:04 traffic.pcapng
-rw-r--r--  3.0 unx 171558408 bx defN 23-Oct-22 19:54 lsass.DMP
2 files, 171592952 bytes uncompressed, 43679601 bytes compressed:  74.5%
```

## LSASS.DMP 
I start by checking the `lsass.dmp` file and we can see it is a minidump for a process and goes without saying it is `LSASS-Local Security Authority Subsystem Service`

```console
$ file lsass.DMP 
lsass.DMP: Mini DuMP crash report, 16 streams, Sun Oct 22 16:54:50 2023, 0x421826 type
```
Using `pypykatz` I extract some credentials from the memory dump.

```console
$ pypykatz lsa minidump lsass.DMP
INFO:pypykatz:Parsing file lsass.DMP
FILE: ======== lsass.DMP =======
== LogonSession ==
authentication_id 1883025 (1cbb91)
session_id 3
username mrealman
domainname BLOCK
logon_server WIN-2258HHCBNQR
logon_time 2023-10-22T16:53:54.168637+00:00
sid S-1-5-21-3761843758-2185005375-3584081457-1104
luid 1883025
        == MSV ==
                Username: mrealman
                Domain: BLOCK
                LM: NA
                NT: 1f9175a516211660c7a8143b0f36ab44
                SHA1: ccd27b4bf489ffda2251897ef86fdb488f248aef
                DPAPI: 3d618a1fffd6c879cd0b056910ec0c3100000000
        == WDIGEST [1cbb91]==
                username mrealman
                domainname BLOCK
                password None
                password (hex)
        == Kerberos ==
                Username: mrealman
                Domain: BLOCK.THM
        == WDIGEST [1cbb91]==
                username mrealman
                domainname BLOCK
                password None
                password (hex)
        == DPAPI [1cbb91]==
                luid 1883025
                key_guid dceaf1e5-3b90-43fe-b6fb-c686b349c13b
                masterkey 2f3e5ee18382e593671c68d96f289eac03f3f49debf8403902342141b74cc7acd7e374c71e1644da5ba76b7d3aea99bf4baa5478a6f17ab83c6df0f1ba3bcdf5
                sha1_masterkey c8561de62042ed90e7d551d4d316f6f52a37d9e1
...
== LogonSession ==
authentication_id 828825 (ca599)
session_id 2
username eshellstrop
domainname BLOCK
logon_server WIN-2258HHCBNQR
logon_time 2023-10-22T16:46:09.215626+00:00
sid S-1-5-21-3761843758-2185005375-3584081457-1103
luid 828825
        == MSV ==
                Username: eshellstrop
                Domain: BLOCK
                LM: NA
                NT: 3f29138a04aadc19214e9c04028bf381
                SHA1: 91374e6e58d7b523376e3b1eb04ae5440e678717
                DPAPI: 87c8e56bc4714d4c5659f254771559a800000000
        == WDIGEST [ca599]==
                username eshellstrop
                domainname BLOCK
                password None
                password (hex)
        == Kerberos ==
                Username: eshellstrop
                Domain: BLOCK.THM
        == WDIGEST [ca599]==
                username eshellstrop
                domainname BLOCK
                password None
                password (hex)
        == DPAPI [ca599]==
                luid 828825
                key_guid c03cf95e-ff22-4fe9-aac6-93b9586d37c8
                masterkey b0bf26e3ee42acb190007958d5514a966ba87f4425d14688566b9a31e0fd98687bf28c9b5a21bc47f53380967cd871d5b56023b7318cd04d9cf4ba6663cd4d9c
                sha1_masterkey c7bee2cb7cdd6eb62201adda6d034ea41a126c7c
...
```

Basing on the size of the hash i'm guessing its a n md5 hash so I use `crackstation` to crack them and we get one password for the user `mrealman`

![crackstation index](crackstation.webp){: width="800" height="600" }

## Packet Capture File
Opening the `traffic.pcapng` file in `wireshark` and check the statistics we can deduct most of the traffic is SMB.

Checking the traffic manually we see `NTLM_AUTH` which shows login for the user `mrealman`.

![realman index](mrealman-log.webp){: width="800" height="600" }

Enumerating further we also find a login for the user `eshellstrop`.

![eshellstrop index](eshellstrop-log.webp){: width="800" height="600" }

And lastly we see some encrypted SMB3 traffic.

![smb traffic](encrypted=smb.webp){: width="800" height="600" }

## Decrypting SMB3 Traffic

I do enumeration on ways to decrypt the smb3 traffic in the packet capture and find an article that helps us [link](https://medium.com/maverislabs/decrypting-smb3-traffic-with-just-a-pcap-absolutely-maybe-712ed23ff6a2).
We can use the `Key Exchange Key` to decrypt the `Encrypted Session Key` to get the `Random Session Key` and use it to decrypt the SMB3 traffic.
We will use the `username`,`domain`,`NT hash or password` and `NTProofStr` to calculate the `Key Exchange Key`.

From our previous enumeration we have everything we need 
The `username`,`domain` and `NTProofStr` are found in the packet capture while `NT hash and Password` are found in the lsass dump.
We also need the `Encrypted Session Key` which is in the packet capture file aswell.
A python script is provided to calculate the `Key Exchange Key` and decrypt the `Encrypted Session Key` but I make an adjustment to run on `python3` and also a functionality to accept `NTLM hash` as an argument.

```console
import hashlib
import hmac
import argparse
from Cryptodome.Cipher import ARC4
from Cryptodome.Cipher import DES
from Cryptodome.Hash import MD4

def generateEncryptedSessionKey(keyExchangeKey, exportedSessionKey):
    cipher = ARC4.new(keyExchangeKey)
    sessionKey = cipher.encrypt(exportedSessionKey)
    return sessionKey

parser = argparse.ArgumentParser(description="Calculate the Random Session Key based on data from a PCAP (maybe).")
parser.add_argument("-u", "--user", required=True, help="User name")
parser.add_argument("-d", "--domain", required=True, help="Domain name")
credential = parser.add_mutually_exclusive_group(required=True)
credential.add_argument("-p", "--password", help="Password of User")
credential.add_argument("-H", "--hash", help="NTLM Hash of User")
parser.add_argument("-n", "--ntproofstr", required=True, help="NTProofStr. This can be found in PCAP (provide Hex Stream)")
parser.add_argument("-k", "--key", required=True, help="Encrypted Session Key. This can be found in PCAP (provide Hex Stream)")
parser.add_argument("-v", "--verbose", action="store_true", help="increase output verbosity")

args = parser.parse_args()

#Upper Case User and Domain
user = args.user.upper().encode("utf-16le")
domain = args.domain.upper().encode("utf-16le")

if args.password:
  # If password is supplied create 'NTLM' hash of password
  passw = args.password.encode("utf-16le")
  hash1 = hashlib.new("md4", passw).digest()
else:
  hash1 = bytes.fromhex(args.hash)

# Calculate the ResponseNTKey
h = hmac.new(hash1, digestmod=hashlib.md5)
h.update(user + domain)
respNTKey = h.digest()

# Use NTProofSTR and ResponseNTKey to calculate Key Excahnge Key
NTproofStr = bytes.fromhex(args.ntproofstr)
h = hmac.new(respNTKey, digestmod=hashlib.md5)
h.update(NTproofStr)
KeyExchKey = h.digest()

# Calculate the Random Session Key by decrypting Encrypted Session Key with Key Exchange Key via RC4
RsessKey = generateEncryptedSessionKey(KeyExchKey, bytes.fromhex(args.key))

if args.verbose:
    print("USER WORK: " + user.hex() + "" + domain.hex())
    print("PASS HASH: " + hash1.hex())
    print("RESP NT:   " + respNTKey.hex())
    print("NT PROOF:  " + NTproofStr.hex())
    print("KeyExKey:  " + KeyExchKey.hex())
print("Random SK: " + RsessKey.hex())
```

So now we gather values we need to decrypt the session for `mrealman`

![ntproofstr mrm index](ntproofstr-mrealman.webp){: width="800" height="600" }

Username: `mrealman`
Domain: `WORKGROUP`
NTProofStr: `16e816dead16d4ca7d5d6dee4a015c14`
Encrypted Session Key: `fde53b54cb676b9bbf0fb1fbef384698`

We also have the user password based on the hash crack.
Also take note of the `Session ID` 

![sessid index](mrealman-sessid.webp){: width="800" height="600" }
Session ID: `0x0000100000000041`

Having everything we need we run the python script to get the `Random Session Key`.

```console
$ python3 random-session-key.py -u 'mrealman' -d 'WORKGROUP' -H '1f9175a516211660c7a8143b0f36ab44
' -n '16e816dead16d4ca7d5d6dee4a015c14' -k 'fde53b54cb676b9bbf0fb1fbef384698' -v
USER WORK: 4d005200450041004c004d0041004e0057004f0052004b00470052004f0055005000
PASS HASH: 1f9175a516211660c7a8143b0f36ab44
RESP NT:   110fd571fec8b2d44728e3d4d6f32f0a
NT PROOF:  16e816dead16d4ca7d5d6dee4a015c14
KeyExKey:  17e09b2c9b92045329a4382898f50159
Random SK: 20a642c086ef74eee26277bf1d0cff8c
```

We then add it to wireshark under protocole preferences>secret session keys for decryption to decrypt the traffic.
The session ID bytes are reversed due to endianness.

![sesskey index](sesskeydecrypt-mrm.webp){: width="800" height="600" }

Going back to the traffic on the pcap we see that the SMB3 traffic is now decrypted and also see the user accessing the file `clients156.csv`.

![decrypt index](decrypted-smb-mrm.webp){: width="800" height="600" }

We can use wireshark to export the file through `SMB objects`.

![object 1 index](smb-object1.webp){: width="800" height="600" }

Reading the extracted file we get the first flag.

```console
$ cat %5cclients156.csv                                        
first_name,last_name,password
Jewell,Caseri,eS8/y*t?8$
Abey,Sigward,yB0{g_>KezO
Natassia,Freeth,tS2<1Fef9tiF
Verina,Wainscoat,kT8/2uEMH
Filia,Sommerling,oE9.2c?Sce
Farris,Busst,THM{SmB_DeCrypTing_who_Could_Have_Th0ughT}
Bat,Oakes,gE0%f@'qw}s%
Verina,Jedrachowicz,wK4~4L\O
Caril,Wolfarth,yQ3$Ji0~f7aB>F{
Bordie,Baume,iM1}"x)yP'`2|S
,,
```                                     

Now we have to do the same process for the user `eshellstrop`.

Finding values for random session key script.

![eshellstrop index](ntproofstr-eshellstrop.webp){: width="800" height="600" }
![eshellstrop sess index](eshellstrop-sessid.webp){: width="800" height="600" }

Username: `eshellstrop`
Domain: `WORKGROUP`
NTProofStr: `0ca6227a4f00b9654a48908c4801a0ac`
Encrypted Session Key: `c24f5102a22d286336aac2dfa4dc2e04`
Session ID: `0x0000100000000045`

Running the script we get the `Random Session Key`

```console
$ python3 random-session-key.py -u 'eshellstrop' -d 'WORKGROUP' -H '3f29138a04aadc19214e9c04028bf381' -n '0ca6227a4f00b9654a48908c4801a0ac' -k 'c24f5102a22d286336aac2dfa4dc2e04' -v
USER WORK: 45005300480045004c004c005300540052004f00500057004f0052004b00470052004f0055005000
PASS HASH: 3f29138a04aadc19214e9c04028bf381
RESP NT:   f48087e449d58b400e283a27914209b9
NT PROOF:  0ca6227a4f00b9654a48908c4801a0ac
KeyExKey:  9754d7acae384644b196c05cda5315df
Random SK: facfbdf010d00aa2574c7c41201099e8
```

We add the key to the list of SMB session keys on `Wireshark`

![sesskey2 index](sesskeydecrypt-eshell.webp){: width="800" height="600" }

And once again the SMB traffic is decrypted and now the user is accessing the file `clients978.csv`

![decrypt index](decrypted-smb-eshell.webp){: width="800" height="600" }

We export the file and read the `clients978.csv` we get our second and last flag of the room.

![object2 index](smb-object2.webp){: width="800" height="600" }

```console
$ cat %5cclients978.csv 
first_name,last_name,password
Fran,McCane,vP5{|r$IYDDu
Fredrika,Delea,qU2!&Bev
Josefa,Keir,hX0)gq54I"%d
Joannes,Greatham,vS1)N,z1X1rc
Courtenay,Keble,lV6|0aiSZL@@`bbM
Tonye,Risebrow,THM{No_PasSw0Rd?_No_Pr0bl3m}
Joleen,Balog,tK9'ZapdU.'igGs
Clementia,Kilsby,uC6!Bx}`Xe
Mason,Woolvett,eL0$NO)FRY1IT
Rozele,Izachik,wA8>11$,'0,b+
,,
```                                            


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
