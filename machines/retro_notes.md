# THM - Retro

#### Ip: 10.10.195.75
#### Name: Retro
#### Difficulty: Hard

----------------------------------------------------------------------

![retro.jpeg](../assets/retro_assets/retro.jpeg)

### Enumeration

Lets scan the target using Nmap. Here I will use the `-p-` flag to scan all TCP ports, as well as the `-sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/THM/Retro]
└─$ sudo nmap -p-  --min-rate 10000 10.10.198.104 -sC -sV -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-19 13:40 CDT
Nmap scan report for 10.10.198.104
Host is up (0.21s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: RETROWEB
|   NetBIOS_Domain_Name: RETROWEB
|   NetBIOS_Computer_Name: RETROWEB
|   DNS_Domain_Name: RetroWeb
|   DNS_Computer_Name: RetroWeb
|   Product_Version: 10.0.14393
|_  System_Time: 2023-09-19T18:40:29+00:00
|_ssl-date: 2023-09-19T18:40:33+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=RetroWeb
| Not valid before: 2023-09-18T18:38:09
|_Not valid after:  2024-03-19T18:38:09
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.82 seconds
```

Heading to the site on port 80 we find a default IIS landing page:

iis.png

Using Feroxbuster for directory fuzzing we find a `/retro` page, which appears to be running WordPress

ferox.png

Checking out the site we find a blog about retro games and movies:

retro.png

Looking around the page we find a Recent Comments section with an interesting comment from user wade:

comment.png

### Exploitation

Because these look like credentials, and because we know RDP is running, lets try them there:

```text
┌──(ryan㉿kali)-[~/THM/Retro]
└─$ xfreerdp /u:wade /p:'parzival' /w:1275 /h:650 /v:10.10.198.104:3389 /cert:ignore
```

Nice, that worked!

We can now grab the user.txt flag:

user_flag.png

### Privilege Escalation

Looking in the Recycle Bin we see an executable hhupd.

bin.png

Also of interest in Chrome is the bookmarked site https://nvd.nist.gov/vuln/detail/CVE-2019-1388

This is an interesting exploit, and took a bit to get figured out.

First we'll open up Chrome and IE and then drag the file from the Recycling Bin back onto the desktop.

Then we can right click on the file and select "Run as Administrator" and click on Show More Details

admin.png

From here we can select "Show information about the publisher's certificate"

info.png

We'll now get a warning that the page cant be displayed, but if we hit `ctrl + s` and navigate to `C:\Windows\System32\*.*`. We can now scroll down until we find cmd:

cant.png

cmd.png

If we right click on cmd a command prompt will open for running as SYSTEM and we can grab the final flag:

root_flag.png

Thanks for following along!

-Ryan 

---------------------------------------
