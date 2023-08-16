# THM - MD2PDF

#### Ip: 10.10.83.71
#### Name: MD2PDF
#### Rating: Easy

----------------------------------------------------------------------

md2pdf.png

```text
Hello Hacker!

TopTierConversions LTD is proud to announce its latest and greatest product launch: MD2PDF.

This easy-to-use utility converts markdown files to PDF and is totally secure! Right...?
```


I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/THM/MD2PDF]
└─$ sudo nmap -p-  --min-rate 10000 10.10.83.71
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-16 13:59 CDT
Nmap scan report for 10.10.83.71
Host is up (0.13s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 8.21 seconds

```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/THM/MD2PDF]
└─$ sudo nmap -sC -sV 10.10.83.71 -p 22,80                     
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-16 14:00 CDT
WARNING: Service 10.10.83.71:80 had already soft-matched rtsp, but now soft-matched sip; ignoring second value
Nmap scan report for 10.10.83.71
Host is up (0.13s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 6ee150ba8724c987f1b4433fe2fa0256 (RSA)
|   256 8ec25b6ab040f4627471f6b6a1043497 (ECDSA)
|_  256 f450193371afa8b98faa006a7c1645b0 (ED25519)
80/tcp open  rtsp
|_http-title: MD2PDF
|_rtsp-methods: ERROR: Script execution failed (use -d to debug)
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 404 NOT FOUND
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 232
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
|     <title>404 Not Found</title>
|     <h1>Not Found</h1>
|     <p>The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.</p>
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 2660
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8" />
|     <meta
|     name="viewport"
|     content="width=device-width, initial-scale=1, shrink-to-fit=no"
|     <link
|     rel="stylesheet"
|     href="./static/codemirror.min.css"/>
|     <link
|     rel="stylesheet"
|     href="./static/bootstrap.min.css"/>
|     <title>MD2PDF</title>
|     </head>
|     <body>
|     <!-- Navigation -->
|     <nav class="navbar navbar-expand-md navbar-dark bg-dark">
|     <div class="container">
|     class="navbar-brand" href="/"><span class="">MD2PDF</span></a>
|     </div>
|     </nav>
|     <!-- Page Content -->
|     <div class="container">
|     <div class="">
|     <div class="card mt-4">
|     <textarea class="form-control" name="md" id="md"></textarea>
|     </div>
|     <div class="mt-3
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Content-Type: text/html; charset=utf-8
|     Allow: OPTIONS, GET, HEAD
|     Content-Length: 0
|   RTSPRequest: 
|     RTSP/1.0 200 OK
|     Content-Type: text/html; charset=utf-8
|     Allow: OPTIONS, GET, HEAD
|_    Content-Length: 0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port80-TCP:V=7.93%I=7%D=8/16%Time=64DD1CB9%P=aarch64-unknown-linux-gnu%
SF:r(GetRequest,AB5,"HTTP/1\.0\x20200\x20OK\r\nContent-Type:\x20text/html;
SF:\x20charset=utf-8\r\nContent-Length:\x202660\r\n\r\n<!DOCTYPE\x20html>\
SF:n<html\x20lang=\"en\">\n\x20\x20<head>\n\x20\x20\x20\x20<meta\x20charse
SF:t=\"utf-8\"\x20/>\n\x20\x20\x20\x20<meta\n\x20\x20\x20\x20\x20\x20name=
SF:\"viewport\"\n\x20\x20\x20\x20\x20\x20content=\"width=device-width,\x20
SF:initial-scale=1,\x20shrink-to-fit=no\"\n\x20\x20\x20\x20/>\n\n\x20\x20\
SF:x20\x20<link\n\x20\x20\x20\x20\x20\x20rel=\"stylesheet\"\n\x20\x20\x20\
SF:x20\x20\x20href=\"\./static/codemirror\.min\.css\"/>\n\n\x20\x20\x20\x2
SF:0<link\n\x20\x20\x20\x20\x20\x20rel=\"stylesheet\"\n\x20\x20\x20\x20\x2
SF:0\x20href=\"\./static/bootstrap\.min\.css\"/>\n\n\x20\x20\x20\x20\n\x20
SF:\x20\x20\x20<title>MD2PDF</title>\n\x20\x20</head>\n\n\x20\x20<body>\n\
SF:x20\x20\x20\x20<!--\x20Navigation\x20-->\n\x20\x20\x20\x20<nav\x20class
SF:=\"navbar\x20navbar-expand-md\x20navbar-dark\x20bg-dark\">\n\x20\x20\x2
SF:0\x20\x20\x20<div\x20class=\"container\">\n\x20\x20\x20\x20\x20\x20\x20
SF:\x20<a\x20class=\"navbar-brand\"\x20href=\"/\"><span\x20class=\"\">MD2P
SF:DF</span></a>\n\x20\x20\x20\x20\x20\x20</div>\n\x20\x20\x20\x20</nav>\n
SF:\n\x20\x20\x20\x20<!--\x20Page\x20Content\x20-->\n\x20\x20\x20\x20<div\
SF:x20class=\"container\">\n\x20\x20\x20\x20\x20\x20<div\x20class=\"\">\n\
SF:x20\x20\x20\x20\x20\x20\x20\x20<div\x20class=\"card\x20mt-4\">\n\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20<textarea\x20class=\"form-control\"\x2
SF:0name=\"md\"\x20id=\"md\"></textarea>\n\x20\x20\x20\x20\x20\x20\x20\x20
SF:</div>\n\n\x20\x20\x20\x20\x20\x20\x20\x20<div\x20class=\"mt-3\x20")%r(
SF:HTTPOptions,69,"HTTP/1\.0\x20200\x20OK\r\nContent-Type:\x20text/html;\x
SF:20charset=utf-8\r\nAllow:\x20OPTIONS,\x20GET,\x20HEAD\r\nContent-Length
SF::\x200\r\n\r\n")%r(RTSPRequest,69,"RTSP/1\.0\x20200\x20OK\r\nContent-Ty
SF:pe:\x20text/html;\x20charset=utf-8\r\nAllow:\x20OPTIONS,\x20GET,\x20HEA
SF:D\r\nContent-Length:\x200\r\n\r\n")%r(FourOhFourRequest,13F,"HTTP/1\.0\
SF:x20404\x20NOT\x20FOUND\r\nContent-Type:\x20text/html;\x20charset=utf-8\
SF:r\nContent-Length:\x20232\r\n\r\n<!DOCTYPE\x20HTML\x20PUBLIC\x20\"-//W3
SF:C//DTD\x20HTML\x203\.2\x20Final//EN\">\n<title>404\x20Not\x20Found</tit
SF:le>\n<h1>Not\x20Found</h1>\n<p>The\x20requested\x20URL\x20was\x20not\x2
SF:0found\x20on\x20the\x20server\.\x20If\x20you\x20entered\x20the\x20URL\x
SF:20manually\x20please\x20check\x20your\x20spelling\x20and\x20try\x20agai
SF:n\.</p>\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.12 seconds
```

Lets use Feroxbuster to scan for any directories we can find:

ferox.png

Cool, looks like there's an `/admin` directory.

Navigating to the page we get an access denied/ forbidden error. But more importantly, the error message is really verbose:

local.png

Looks like we can only access the directory from localhost:5000

Lets try using an `iframe` to access this directory. iframes essentally allow us to display or embed other documents inside of html code. We can try to access the directory with:

```text
 <iframe src="http://localhost:5000/admin">
 </iframe>
```

iframe.png

Cool, that worked!

flag.png

Thanks for following along!

-Ryan

-------------------------------------------

