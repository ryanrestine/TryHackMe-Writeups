# THM - Intermediate Nmap 

#### Ip: 10.10.148.214
#### Name: Intermediate Nmap
#### Rating: Easy

----------------------------------------------------------------------

nmap.png

```text
Can you combine your great nmap skills with other tools to log in to this machine?

You've learned some great nmap skills! Now can you combine that with other skills with netcat and protocols, to log in to this machine and find the flag? This VM 10.10.148.214 is listening on a high port, and if you connect to it it may give you some information you can use to connect to a lower port commonly used for remote access!
```

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/Intermediate_Nmap]
└─$ sudo nmap -p-  --min-rate 10000 10.10.148.214
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-10 15:24 CDT
Nmap scan report for 10.10.148.214
Host is up (0.13s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
2222/tcp  open  EtherNetIP-1
31337/tcp open  Elite

Nmap done: 1 IP address (1 host up) scanned in 8.18 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/THM/Intermediate_Nmap]
└─$ sudo nmap -sC -sV -T4 10.10.148.214 -p 22,2222,31337
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-10 15:27 CDT
Nmap scan report for 10.10.148.214
Host is up (0.12s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 7ddceb90e4af33d99f0b219afcd577f2 (RSA)
|   256 83a74a61ef93a3571a57385c482aeb16 (ECDSA)
|_  256 30bfef9408860700f7fcdfe8edfe07af (ED25519)
2222/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c982b176c875f5831ba92f84e70a21d (RSA)
|   256 8456be4894e82b60e8f3c89c4f88650c (ECDSA)
|_  256 4a47d729df01435192622cb98dd6c001 (ED25519)
31337/tcp open  Elite?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, Kerberos, LANDesk-RC, LDAPBindReq, LDAPSearchReq, LPDString, NULL, RPCCheck, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, X11Probe: 
|     In case I forget - user:pass
|_    ubuntu:Dafdas!!/str0ng
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port31337-TCP:V=7.93%I=7%D=8/10%Time=64D54834%P=aarch64-unknown-linux-g
SF:nu%r(NULL,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\nubuntu:Dafda
SF:s!!/str0ng\n\n")%r(GetRequest,35,"In\x20case\x20I\x20forget\x20-\x20use
SF:r:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(SIPOptions,35,"In\x20case\x20I\x
SF:20forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(GenericLines
SF:,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0n
SF:g\n\n")%r(HTTPOptions,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\n
SF:ubuntu:Dafdas!!/str0ng\n\n")%r(RTSPRequest,35,"In\x20case\x20I\x20forge
SF:t\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(RPCCheck,35,"In\x20
SF:case\x20I\x20forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(D
SF:NSVersionBindReqTCP,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\nub
SF:untu:Dafdas!!/str0ng\n\n")%r(DNSStatusRequestTCP,35,"In\x20case\x20I\x2
SF:0forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(Help,35,"In\x
SF:20case\x20I\x20forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r
SF:(SSLSessionReq,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\nubuntu:
SF:Dafdas!!/str0ng\n\n")%r(TerminalServerCookie,35,"In\x20case\x20I\x20for
SF:get\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(TLSSessionReq,35,
SF:"In\x20case\x20I\x20forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\
SF:n")%r(Kerberos,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\nubuntu:
SF:Dafdas!!/str0ng\n\n")%r(SMBProgNeg,35,"In\x20case\x20I\x20forget\x20-\x
SF:20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(X11Probe,35,"In\x20case\x20
SF:I\x20forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(FourOhFou
SF:rRequest,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\nubuntu:Dafdas
SF:!!/str0ng\n\n")%r(LPDString,35,"In\x20case\x20I\x20forget\x20-\x20user:
SF:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(LDAPSearchReq,35,"In\x20case\x20I\
SF:x20forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n")%r(LDAPBindReq
SF:,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\nubuntu:Dafdas!!/str0n
SF:g\n\n")%r(LANDesk-RC,35,"In\x20case\x20I\x20forget\x20-\x20user:pass\nu
SF:buntu:Dafdas!!/str0ng\n\n")%r(TerminalServer,35,"In\x20case\x20I\x20for
SF:get\x20-\x20user:pass\nubuntu:Dafdas!!/str0ng\n\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.98 seconds
```

Interesting, looks like Nmap picked up some potential credentials while fingerprinting port 31337.

strings.png

Lets confirm that by connecting to the port using Netcat:

```text
┌──(ryan㉿kali)-[~/THM/Intermediate_Nmap]
└─$ nc -nv 10.10.148.214 31337
(UNKNOWN) [10.10.148.214] 31337 (?) open
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```
Yep, same credentials as Nmap discovered.

Lets try these on SSH:

```text
┌──(ryan㉿kali)-[~/THM/Intermediate_Nmap]
└─$ ssh ubuntu@10.10.148.214                            
The authenticity of host '10.10.148.214 (10.10.148.214)' can't be established.
ED25519 key fingerprint is SHA256:8VuYGtc5lO2sXK+MVsdbgQV9nF+EVHf8wJcrMAEWg10.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:36: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.148.214' (ED25519) to the list of known hosts.
ubuntu@10.10.148.214's password: 
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.13.0-1014-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

$ whoami && hostname
ubuntu
f518fa10296d
```

That was easy!

Lets grab the flag to complete the room:

flag.png

That was an incredibly short challenge.

Thanks for following along!

-Ryan

-------------------------------------------------
