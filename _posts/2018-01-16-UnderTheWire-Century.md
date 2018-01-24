---
layout: post
title: UnderTheWire Century
tags: [underthewire]
---

If you aren't familiar with the famous OverTheWire challenges do check them out. They are fantastic exercises for learning some basic Linux exploitation skills. I recently found out that there was a similar thing for Powershell called UnderTheWire. So I decided to check it out.

Century is the first in the series and a great introduction to doing some simple things in Powershell. Below are my solutions to each challenge.

## Century 2

_The password for Century2 is the build version of the instance of PowerShell installed on this system._

Easy one here, we can call a system variable to do the job.

```powershell

PS C:\Users\century1\Documents> $PSVersionTable

Name                           Value
----                           -----
PSVersion                      5.1.14409.1012
PSEdition                      Desktop
PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0...}
BuildVersion                   10.0.14409.1012
CLRVersion                     4.0.30319.42000
WSManStackVersion              3.0
PSRemotingProtocolVersion      2.3
SerializationVersion           1.1.0.1

```
The password is `10.0.14409.1012`

## Century 3

_The password for Century3 is the name of the built-in cmdlet that performs the wget like function within PowerShell PLUS the name of the file on the desktop._

We grab the filename off the desktop. The wget equivalent in Powershell is `Invoke-WebRequest`.

```powershell

PS C:\Users\century2\Documents> Get-ChildItem ..\desktop\


    Directory: C:\Users\century2\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/8/2017   4:05 PM              0 80


PS C:\Users\century2\desktop> Invoke-WebRequest

```


Password is `invoke-webrequest80`

## Century 4

_The password for Century4 is the number of files on the desktop._

We can use `Get-ChildItem` to list everything on the desktop and pipe it into `Measure-Object` to give us a count.

```powershell

PS C:\Users\century3\Documents> Get-ChildItem ..\desktop\ | Measure-Object


Count    : 517
Average  :
Sum      :
Maximum  :
Minimum  :
Property :

```

Password is `517`

## Century 5

_The password for Century5 is the name of the file within a directory on the desktop that has spaces in its name._

Here we just add a `Recurse` option to get inside of the folders and list all files.

```powershell

PS C:\Users\century4\Documents> get-childitem ..\desktop\ -Recurse


Directory: C:\Users\century4\desktop\500
                                        501


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/8/2017   4:19 PM              0 65536

```

Password is `65536`

## Century 6

_The password for Century6 is the short name of the domain in which this system resides in PLUS the name of the file on the desktop._ 

We can grab environment variables through `env:` and filter the name to only show `USERDOMAIN` which is the shortname, the long name of the domain would be under `USERDNSDOMAIN`

```powershell

PS C:\Users\century5\Documents> Get-ChildItem env: | where-object {$_.Name -eq 'USERDOMAIN'}

Name                           Value
----                           -----
USERDOMAIN                     UNDERTHEWIRE

PS C:\Users\century5\Documents> Get-ChildItem ..\desktop\


    Directory: C:\Users\century5\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/8/2017   4:20 PM              0 _4321

```

Password is `underthewire_4321`

## Century 7

_The password for Century7 is the number of folders on the desktop._

Similar to what we did earlier except now we can add the `Directory` option to only give us folders back.

```powershell

PS C:\Users\century6> get-childitem .\Desktop\ -Directory | Measure-Object


Count    : 416
Average  :
Sum      :
Maximum  :
Minimum  :
Property :

```

Password is `416`

## Century 8

_The password for Century8 is in a readme file somewhere within the contacts, desktop, documents, downloads, favorites, music, or videos folder in the user's profile._ 

Here we are going to recursively search and set a filter to match any file starting with readme.

```powershell

PS C:\Users\century7\Documents> Get-ChildItem ..\ -Recurse -File -Filter readme* | get-content

human_versus_computer

```

Password is `human_versus_computer`

## Century 9

_The password for Century9 is the number of unique entries within the file on the desktop._ 

Command line here is pretty self explanatory.

```powershell

PS C:\Users\century8\Desktop> Get-Content .\Unique.txt | Sort-Object | Get-Unique | Measure-Object


Count    : 511
Average  :
Sum      :
Maximum  :
Minimum  :
Property :

```

Password is `511`

## Century 10

_The password for Century10 is the 161st element within the file on the desktop._

Again nothing crazy, just adding the `Index` option and specifying exactly where we want to return.

```powershell

PS C:\Users\century9\Desktop> Get-ChildItem


    Directory: C:\Users\century9\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        6/25/2017   5:40 PM          97622 words.txt


PS C:\Users\century9\Desktop> Get-Content .\words.txt | Select-Object -Index 161

shark

```

Password is `shark`

## Century 11

_The password for Century11 is the 10th and 8th word of the Windows Update service description combined PLUS the name of the file on the desktop._

For whatever reason `Get-Service` as of now won't return the description of a service, so we have to look to WMI to grab it.

```powershell

PS C:\Users\century10> Get-WmiObject win32_Service -Filter "DisplayName = 'Windows Update'" | Select-Object -Property Description | ft -Wrap

Description
-----------
Enables the detection, download, and installation of updates for Windows and
other programs. If this service is disabled, users of this computer will not
be able to use Windows Update or its automatic updating feature, and programs
will not be able to use the Windows Update Agent (WUA) API.

PS C:\Users\century10> Get-ChildItem .\Desktop


    Directory: C:\Users\century10\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/8/2017   4:57 PM              0 _4u

``` 

Password is `windowsupdates_4u`

## Century 12

_The password for Century12 is the name of the hidden file within the contacts, desktop, documents, downloads, favorites, music, or videos folder in the user's profile._

First we will use `Get-ChildItem`  to grab all the folders in the user profile that aren't hidden, otherwise we'll be searching through our AppData folder and that will return a lot of results. Then we can search for all hidden files, where the name does not equal `desktop.ini` to cut down on unnecessary results.

```powershell

PS C:\Users\century11> Get-ChildItem | Get-ChildItem -Recurse -File -Hidden | Where-Object {$_.Name -ne 'desktop.ini'}


    Directory: C:\Users\century11\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a-h--         6/8/2017   4:59 PM              0 secret_sauce


    Directory: C:\Users\century11\Searches


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-arh--         6/3/2017   4:12 PM            248 Everywhere.search-ms
-arh--         6/3/2017   4:12 PM            248 Indexed Locations.search-ms


```

Password is `secret_sauce`

## Century 13

_The password for Century13 is the description of the computer designated as a Domain Controller within this domain PLUS the name of the file on the desktop._

`Get-ADDomainController` will not return the AD attribute description so we will have to feed the computer name into Get-ADComputer and make sure we specify we want the `Description` property returned.

```powershell

PS C:\Users\century12> Get-ADDomainController | Select-Object name

name
----
CENTURY

PS C:\Users\century12> Get-ADComputer CENTURY -Properties Description


Description       : i_authenticate
DistinguishedName : CN=CENTURY,OU=Domain Controllers,DC=UNDERTHEWIRE,DC=TECH
DNSHostName       : Century.UNDERTHEWIRE.TECH
Enabled           : True
Name              : CENTURY
ObjectClass       : computer
ObjectGUID        : e1248e0f-ed89-42a4-86ef-687303e886a5
SamAccountName    : CENTURY$
SID               : S-1-5-21-3968311752-1263969649-2303472966-1002
UserPrincipalName :


PS C:\Users\century12> Get-ChildItem .\Desktop


    Directory: C:\Users\century12\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/8/2017   5:09 PM              0 _things

```

Password is `i_authenticate_things`

## Century 14

_The password for Century14 is the number of words within the file on the desktop._

Another similar challenge from earlier however this time we specify the `Word` option. If you don't you'll see the line count is returned instead giving you a 1.

```powershell

PS C:\Users\century13> Get-ChildItem .\Desktop | get-content | Measure-Object -Word

Lines  Words Characters Property
-----  ----- ---------- --------
      475361

```

Password is `475361`

## Century 15

_The password for Century15 is the number of times the word "polo" appears within the file on the desktop_

We use `Select-String` to filter our pattern of `polo` and then pipe it to get our count.

```powershell

PS C:\Users\century14\Desktop> get-content .\stuff.txt | Select-String -Pattern "polo" | Measure-Object


Count    : 10
Average  :
Sum      :
Maximum  :
Minimum  :
Property :

```

Password is `10`

And that's it! Overall a quick set of challenges. 

