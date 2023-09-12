# THM - Wgel CTF

#### Ip: 10.10.177.82
#### Name: Wgel CTF
#### Difficulty: Easy

----------------------------------------------------------------------

wgel.png

```text
Can you exfiltrate the root flag?
```

### Enumeration

Lets scan the target using Nmap. Here I will use the `-p-` falg to scan all TCP ports, as well as the `-sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/THM/Wgel_CTF]
└─$ sudo nmap -p-  --min-rate 10000 10.10.177.82 -sC -sV
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-12 15:01 CDT
Warning: 10.10.177.82 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.177.82
Host is up (0.21s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 94961b66801b7648682d14b59a01aaaa (RSA)
|   256 18f710cc5f40f6cf92f86916e248f438 (ECDSA)
|_  256 b90b972e459bf32a4b11c7831033e0ce (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.25 seconds
```

Heading over to the site we find a Apache landing page:

site.png

But in the page source we see a comment left behind, revealing a potential username of jessie:

jessie.png

Doing some directory scanning we find an interesting directory:

ferox.png

map.png

Wow, looks like we've got a private SSH key here:

### Exploitation

Lets try to logon as user Jessie:

```text
┌──(ryan㉿kali)-[~/THM/Wgel_CTF]
└─$ chmod 600 id_rsa
                                                                                                                             
┌──(ryan㉿kali)-[~/THM/Wgel_CTF]
└─$ ssh -i id_rsa jessie@10.10.177.82                   
The authenticity of host '10.10.177.82 (10.10.177.82)' can't be established.
ED25519 key fingerprint is SHA256:6fAPL8SGCIuyS5qsSf25mG+DUJBUYp4syoBloBpgHfc.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.177.82' (ED25519) to the list of known hosts.
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-45-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


8 packages can be updated.
8 updates are security updates.

jessie@CorpOne:~$ hostname
CorpOne
```

From here we can grab the user.txt flag:

user_flag.png

### Privilege Escalation

Running `sudo -l` to see what jessie can run with elevated permissions, we find wget:

```text
jessie@CorpOne:~/Documents$ sudo -l
Matching Defaults entries for jessie on CorpOne:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jessie may run the following commands on CorpOne:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/wget
```

Lets head over to https://gtfobins.github.io/gtfobins/wget/ for the commands we'll need to exploit this:

gtfo.png

Setting up a NetCat listener and running the commands, we are able to exfiltrate the root_flag.txt file:

root_flag.png

Thanks for following along!

-Ryan

--------------------------------------------