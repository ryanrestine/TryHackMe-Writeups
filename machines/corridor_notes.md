# THM - Corridor

#### Ip: 10.10.146.177
#### Name: Corridor
#### Rating: Easy

----------------------------------------------------------------------

corridor.png

```text
You have found yourself in a strange corridor. Can you find your way back to where you came?

In this challenge, you will explore potential IDOR vulnerabilities. Examine the URL endpoints you access as you navigate the website and note the hexadecimal values you find (they look an awful lot like a hash, don't they?). This could help you uncover website locations you were not expected to access.
```

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I will also use the `--min-rate 10000` flag to speed the scan up.

```text
┌──(ryan㉿kali)-[~/THM/Corridor]
└─$ sudo nmap -p-  --min-rate 10000 10.10.146.177                                                        
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-02 16:12 CDT
Warning: 10.10.146.177 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.146.177
Host is up (0.13s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 25.77 seconds
```

Ok, looks like just port 80 is open. Lets go ahead and also use the `-sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/THM/Corridor]
└─$ sudo nmap -sC -sV -T4 10.10.146.177 -p 80                                                            
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-02 16:14 CDT
Nmap scan report for 10.10.146.177
Host is up (0.13s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Werkzeug httpd 2.0.3 (Python 3.10.2)
|_http-title: Corridor
|_http-server-header: Werkzeug/2.0.3 Python/3.10.2

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.13 seconds
```

Navigating to the site, we have a background of a hall or corridor, and each of the doors is a link. Each of the links appear to be a hash of somekind.

link.png

Inspecting the page source (CTRL + U) we can get a list of all the links/ hashes. 

I can copy all of these (with quotes and all) into a file called messy_links, and then use awk and sed to cut out everything but the hashes:

```text
┌──(ryan㉿kali)-[~/THM/Corridor]
└─$ cat messy_links | sed 's/title=//g'| sed 's/"//g'| awk '{print $4}' > hashes.txt
```

only_hash.png

We can now paste these into crackstation to try and crack the hashes.

cracked.png

Interesting, the hashes just appear to be md5 and are sequential numbers going from 1-12. 

Because we know from the box instructions we're looking for IDOR vulnerabilites, my first thought is to try to check if there is a page for 0 (zero) hashed that we can access. 

Simply going to http://10.10.146.177/0 won't work because the numbers/ links were hashed:

nope.png

So we'll need to generate the md5 hash of the number 0. We can do this in the terminal:

```text
┌──(ryan㉿kali)-[~/THM/Corridor]
└─$ echo -n 0 | md5sum                                     
cfcd208495d565ef66e7dff9f98764da
```

Cool, we can now navigate to http://10.10.146.177/cfcd208495d565ef66e7dff9f98764da and sure enough we find our flag!

flag.png

Thanks for following along!

-Ryan

-----------------------------------------------------