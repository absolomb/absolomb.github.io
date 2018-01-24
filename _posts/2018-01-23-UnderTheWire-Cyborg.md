---
layout: post
title: UnderTheWire Cyborg
tags: [underthewire]
---

The Cyborg challenges are the next set of UnderTheWire challenges following Century. These were a little more advanced 
but nothing too crazy. Below are my solutions.

## Cyborg 2

_The password for cyborg2 is the state that the user Chris Rogers is from as stated within Active Directory._

First try was to do a filter on the full name.

```powershell

PS C:\Users\cyborg1\Documents> Get-ADUser -Filter 'Name -like "Chris Rogers"' -Properties state

```

 That did not work however so let's filter on just last name with some asterisks.

```powershell

PS C:\Users\cyborg1\Documents> Get-ADUser -Filter 'Name -like "*Rogers*"' -Properties state


DistinguishedName : CN=Rogers\,
                    Chris,OU=Southside,OU=Cyborg,DC=UNDERTHEWIRE,DC=TECH
Enabled           : False
GivenName         : Rogers
Name              : Rogers, Chris
ObjectClass       : user
ObjectGUID        : 3251b635-dac5-47c1-b8b9-bb7ee058cde7
SamAccountName    : chris.rogers
SID               : S-1-5-21-1013972110-1198539618-3084840507-2117
State             : kansas
Surname           : Chris
UserPrincipalName : chris.rogers@UNDERTHEWIRE.TECH

```

Bingo! We see that our first command failed due to the way the Name formatting is setup. Our password for cyborg2 is `kansas`

## Cyborg 3

_The password for cyborg3 is the host A record IP address for CYBORG713W104N PLUS the name of the file on the desktop._


Nothing crazy here.

 ```powershell

 PS C:\Users\cyborg2\Documents> Resolve-DnsName -Name CYBORG713W104N -Type A | ft -auto

Name                             Type TTL  Section IPAddress
----                             ---- ---  ------- ---------
CYBORG713W104N.UNDERTHEWIRE.TECH A    3600 Answer  172.31.45.167


PS C:\Users\cyborg2\Documents> Get-ChildItem ..\Desktop


    Directory: C:\Users\cyborg2\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/7/2017  11:07 AM              0 _ipv4
```

 Our password is `172.31.45.167_ipv4`


## Cyborg 4

_The password for cyborg4 is the number of users in the Cyborg group within Active Directory PLUS the name of the file on the desktop._ 

Here we just pipe in the group member cmdlet into `measure` (short for `Measure-Object`) to get our count.

```powershell

PS C:\Users\cyborg3> get-adgroupmember cyborg | measure


Count    : 88
Average  :
Sum      :
Maximum  :
Minimum  :
Property :

PS C:\Users\cyborg3> Get-ChildItem .\Desktop


    Directory: C:\Users\cyborg3\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/7/2017  11:10 AM              0 _objects

```

Password is `88_objects`


## Cyborg 5

 _The password for cyborg5 is the PowerShell module name with a version number of 8.9.8.9 PLUS the name of the file on the desktop._

 For this one we need to use the `–ListAvailable` option otherwise we will only get currently loaded modules. We can filter using `Where-Object` searching for our specific version.

 ```powershell

PS C:\Users\cyborg4\Documents> Get-Module –ListAvailable | Where-Object {$_.Version -eq "8.9.8.9"}


    Directory: C:\Windows\system32\WindowsPowerShell\v1.0\Modules


ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Manifest   8.9.8.9    Grits                               Get-grits


PS C:\Users\cyborg4\Documents> Get-ChildItem ..\Desktop


    Directory: C:\Users\cyborg4\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/7/2017  11:23 AM              0 _eggs

 ```

 Password is `grits_eggs`

## Cyborg 6

_The password for cyborg6 is the last name of the user who has logon hours set on their account PLUS the name of the file on the desktop._

Here we get `logonhours` as a property value and filter on it for anything that is not null by using an asterisk.

```powershell

PS C:\Users\cyborg5\Documents> get-aduser -Properties logonhours -Filter {logonhours -like '*'}


DistinguishedName : CN=Rowray\, Benny  \
                    ,OU=Southside,OU=Cyborg,DC=UNDERTHEWIRE,DC=TECH
Enabled           : False
GivenName         : Benny
logonhours        : {255, 255, 255, 255...}
Name              : Rowray, Benny
ObjectClass       : user
ObjectGUID        : 23501b6d-a0ec-4048-bd51-82f84c7945d3
SamAccountName    : Benny.Rowray
SID               : S-1-5-21-1013972110-1198539618-3084840507-1978
Surname           : Rowray
UserPrincipalName : Benny.Rowray



PS C:\Users\cyborg5\Documents> Get-ChildItem ..\Desktop


    Directory: C:\Users\cyborg5\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/7/2017  11:13 AM              0 _timer

``` 

Password is `rowray_timer`

## Cyborg 7 

_The password for cyborg7 is the decoded text of the string within the file on the desktop._

 ```powershell

PS C:\Users\cyborg6\Desktop> Get-ChildItem


    Directory: C:\Users\cyborg6\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/16/2016  12:01 PM             78 cypher.txt


PS C:\Users\cyborg6\Desktop> Get-Content .\cypher.txt
VGhlIHBhc3N3b3JkIGlzIGN5YmVyZ2VkZG9u

```
This looks exactly like base64 encoding. Let's decode it.  

```powershell

PS C:\Users\cyborg6\Desktop> $base64 = Get-Content .\cypher.txt
PS C:\Users\cyborg6\Desktop> [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($base64))
The password is cybergeddon

 ```

 Alright so this one is a little tricky. First we will set a variable named `$base64` with the contents of our file. 
 Next we will have to use .NET classes and methods to get the rest of the job done since there is not cmdlet currently that will do the leg work for us. 
 
 First the `[System.Text.Encoding]`class is called, followed by two colons. The double-colon accesses methods on a class, which for the first part is the `UTF8.GetString` method. Inside of this method we are calling the `System.Convert` class along with it's `FromBase64String` method to decode our variable properly. 
 
 So you may be wondering why you can't just call the `System.Convert` class and the method `FromBase64String` to get our answer.

 Well let's break this down for further understanding and do just that.

 ```powershell

PS C:\Users\cyborg6\Desktop> $a = [System.Convert]::FromBase64String($base64)
PS C:\Users\cyborg6\Desktop> $a
84
104
101
32
112
97
115
115
119
111
114
100
32
105
115
32
99
121
98
101
114
103
101
100
100
111
110
 ```
 If we just call our the `FromBase64String` method we see the output is still not human readable. So what is it?

 ```powershell

PS C:\Users\cyborg6\Desktop> $a.GetType()

IsPublic IsSerial Name                                     BaseType
-------- -------- ----                                     --------
True     True     Byte[]                                   System.Array

 ```

 We can see that we have a byte array. So to decode this further we have to call the `UTF8.GetString` method to do just that.

 ```powershell

PS C:\Users\cyborg6\Desktop> [System.Text.Encoding]::UTF8.GetString($a)
The password is cybergeddon

 ```

Hopefully this makes a little more sense now, rather than just copying and pasting a long command in. 

## Cyborg 8

_The password for cyborg8 is the executable name of a program that will start automatically when cyborg7 logs in._ 

`Get-ChildItem` won't do the job here. We'll have to use `Get-ItemProperty` instead.

```powershell

PS C:\Users\cyborg7\Documents> Get-ItemProperty "hkcu:\Software\Microsoft\Windows\CurrentVersion\Run"


SKYNET       : C:\Program Files\Cyberdyne Systems\Skynet.exe
PSPath       : Microsoft.PowerShell.Core\Registry::HKEY_CURRENT_USER\Software\M
               icrosoft\Windows\CurrentVersion\Run
PSParentPath : Microsoft.PowerShell.Core\Registry::HKEY_CURRENT_USER\Software\M
               icrosoft\Windows\CurrentVersion
PSChildName  : Run
PSDrive      : HKCU
PSProvider   : Microsoft.PowerShell.Core\Registry

 ```

Password is `skynet`.

## Cyborg 9 

_The password for cyborg9 is the Internet zone that the picture on the desktop was downloaded from._

To accomplish this we will need to look at the file streams. We can use `Get-Item` to query the `Zone.Identifier` stream which
specifies the zone it was downloaded from. Then to actually see the value we pipe into `Get-Content` to get our zone number, which
is 4, the Restricted Sites Zone.

```powershell

PS C:\Users\cyborg8\Desktop> Get-Item .\picture1.jpg -Stream Zone.Identifier | Get-Content
[ZoneTransfer]
ZoneId=4

```

Password is `4`

## Cyborg 10

_The password for cyborg10 is the first name of the user with the phone number of 867-5309 listed in Active Directory PLUS the name of the file on the desktop._ 

This one is similar to one we did earlier. We just need to filter on the `telephoneNumber` attribute in Active Directory.

```powershell

PS C:\Users\cyborg9\Documents> Get-ADUser -Properties telephoneNumber -Filter {t
elephoneNumber -like '867-5309'}


DistinguishedName : CN=Conner\,
                    John,OU=Northside,OU=Cyborg,DC=UNDERTHEWIRE,DC=TECH
Enabled           : False
GivenName         : John
Name              : Conner, John
ObjectClass       : user
ObjectGUID        : 61af13ae-3258-4661-b5a3-dee78ac6f659
SamAccountName    : john.conner
SID               : S-1-5-21-1013972110-1198539618-3084840507-2119
Surname           : Conner
telephoneNumber   : 867-5309
UserPrincipalName : john.conner@UNDERTHEWIRE.TECH

PS C:\Users\cyborg9\Documents> Get-ChildItem ..\Desktop


    Directory: C:\Users\cyborg9\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/7/2017   4:39 PM              0 72


```

Password is `john72`

## Cyborg 11

_The password for cyborg11 is the description of the Applocker Executable deny policy for ill_be_back.exe PLUS the name of the file on the desktop._ 

This one is pretty interesting. Since `Get-AppLockerPolicy` only outputs the actual rules into XML format, we will need to do some XML filtering
to get what we want. You could take the easy way out and skim the XML but there are better ways.

```powershell

PS C:\Users\cyborg10\Documents> $xml = [xml](Get-AppLockerPolicy -Effective -Xml)
PS C:\Users\cyborg10\Documents> $xml

AppLockerPolicy
---------------
AppLockerPolicy

```

First we will create a variable and cast it as an XML object so we can manipulate it properly.

Now we can essentially move down the XML structure as such.

```powershell

PS C:\Users\cyborg10\Documents> $xml.AppLockerPolicy

Version RuleCollection
------- --------------
1       {RuleCollection, RuleCollection, RuleCollection, RuleCollection...}

```

To expand on the rules, we can use `select` the `childnodes` property with the `ExpandProperty` option to fully
list out all the child items under RuleCollection. To get exactly what we want we can filter using `where` on Name 
for our exe.

```powershell

PS C:\Users\cyborg10\Documents> $xml.AppLockerPolicy.RuleCollection | select -ExpandProperty childnodes | 
where {$_.name -eq 'ill_be_back.exe'}


Id             : 5d6eb575-3e78-4cc1-a6ac-38260a101d8d
Name           : ill_be_back.exe
Description    : terminated!
UserOrGroupSid : S-1-1-0
Action         : Deny
Conditions     : Conditions

PS C:\Users\cyborg10\Documents> Get-ChildItem ..\Desktop


    Directory: C:\Users\cyborg10\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/7/2017   4:34 PM              0 99

```

Voila! The password is `terminated!99`

## Cyborg 12

_The password for cyborg12 is located in the IIS log. The password is not Mozilla or Opera._

Since we know what we aren't looking for, we can filter to exclude everything we don't want using `notLike` operators.

```powershell

PS C:\inetpub\logs\LogFiles\W3SVC1> Get-Content .\u_ex160413.log | where {($_ -notLike "*Mozilla*") -and ($_ -notLike "*Opera*")}
#Software: Microsoft Internet Information Services 8.5
#Version: 1.0
#Date: 2016-04-13 04:14:01
#Fields: date time s-sitename s-computername s-ip cs-method cs-uri-stem cs-uri-query s-port cs-username c-ip cs-version cs(User-
Agent) cs(Cookie) cs(Referer) cs-host sc-status sc-substatus sc-win32-status sc-bytes cs-bytes time-taken
2016-04-13 04:14:12 W3SVC1 Century 172.31.45.65 GET / - 80 - 172.31.45.65 HTTP/1.1 
LordHelmet/5.0+(CombTheDesert)+Password+is:spaceballs - - century.underthewire.tech 200 0 0 925 118 0
PS C:\inetpub\logs\LogFiles\W3SVC1>

```

Password is `spaceballs`

## Cyborg 13

_The password for cyborg13 is the first four characters of the base64 encoded fullpath to the file that started the i\_heart\_robots service PLUS the name of the file on the desktop._ 

First we will need to grab our path, again we'll have to use WMI, `Get-Service` is pretty limited.

```powershell

PS C:\Users\cyborg12\Documents> Get-WmiObject win32_service | where {$_.Name -eq
 "i_heart_robots"} | select Pathname

Pathname
--------
C:\windows\system32\abc.exe

```

Now let's put our path into a variable and basically do the reverse of what we did earlier when we base64 decoded when we
called .NET classes and methods. 

```powershell

PS C:\Users\cyborg12\Documents> $path = 'C:\windows\system32\abc.exe'

PS C:\Users\cyborg12\Documents> [Convert]::ToBase64String([System.Text.Encoding]
::UTF8.GetBytes($path))
Qzpcd2luZG93c1xzeXN0ZW0zMlxhYmMuZXhl

PS C:\Users\cyborg12\Documents> Get-ChildItem ..\Desktop


    Directory: C:\Users\cyborg12\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/7/2017   4:58 PM              0 _heart

```
Password is `qzpc_heart`


## Cyborg 14

_The password cyborg14 is the number of days the refresh interval is set to for DNS aging for the underthewire.tech zone PLUS the name of the file on the desktop._ 

Pretty easy here, a cmdlet that pulls exactly what we want.

```powershell

PS C:\Users\cyborg13\Documents> Get-DNSServerZoneAging underthewire.tech


ZoneName             : underthewire.tech
AgingEnabled         : False
AvailForScavengeTime :
RefreshInterval      : 16.00:00:00
NoRefreshInterval    : 16.00:00:00
ScavengeServers      :



PS C:\Users\cyborg13\Documents> Get-ChildItem ..\Desktop


    Directory: C:\Users\cyborg13\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        6/12/2017   7:59 PM              0 _days

```

Password is `16_days`

## Cyborg 15

_The password for cyborg15 is the caption for the DCOM application setting for application ID {59B8AFA0-229E-46D9-B980-DDA2C817EC7E} PLUS the name of the file on the desktop._ 

Once again we'll have to turn to querying WMI, this time querying for `win32_DCOMApplicationSetting` and filtering for our 
specified AppID and finally selecting `Caption`.

```powershell

PS C:\Users\cyborg14> Get-WmiObject win32_DCOMApplicationSetting | where {$_.App
ID -eq "{59B8AFA0-229E-46D9-B980-DDA2C817EC7E}"} | select Caption

Caption
-------
propshts

PS C:\Users\cyborg14> Get-ChildItem .\Desktop


    Directory: C:\Users\cyborg14\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        6/12/2017   8:03 PM              0 _objects

```

Password is `propshts_objects`

That's it! Another set of challenges down!
