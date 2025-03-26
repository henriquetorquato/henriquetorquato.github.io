---
layout: post
title: 'Hack The Box CTF 2025: Tales from Eldoria - Web - Cyber Attack'
created-at: 2025-03-21
categories: ["Hack The Box", "WriteUp", "Cybersecurity", "Tales from Eldoria", "CTF"]
---

|Category|Name|Difficulty|Status|
|:---:|:---:|:---:|:---:|
|Web|Cyber Attack|Easy|Not Completed ❌|

> Welcome, Brave Hero of Eldoria. You’ve entered a domain controlled by the forces of Malakar, the Dark Ruler of Eldoria. This is no place for the faint of heart. Proceed with caution: The systems here are heavily guarded, and one misstep could alert Malakar’s sentinels. But if you’re brave—or foolish—enough to exploit these defenses, you might just find a way to weaken his hold on this world. Choose your path carefully: Your actions here could bring hope to Eldoria… or doom us all. The shadows are watching. Make your move.

---

This looks like some sort of DDoS platform interface. It expects a name and a domain name/ip to attack:

![Landing page](/docs/assets/2025-03/htb-ctf-web-cyber-attack-1.png)

It seems that only the "Attack Domain" option lights up, and after confirming the attack, the page refreshes and shows the following message:

![Successful attack example](/docs/assets/2025-03/htb-ctf-web-cyber-attack-2.png)

And if I try to add an invalid domain name as target, it shows the following message:

![Failed attack example](/docs/assets/2025-03/htb-ctf-web-cyber-attack-3.png)

Firing up BurpSuite, I can see that clicking the "Attack a Domain" option makes the following request:

> GET /cgi-bin/attack-domain?target=facebook.com&name=Malakias

In turn it receives a HTTP 302 code (Found) redirecting the request back to that same landing page. But now with some extra information on the url:

```
HTTP/1.1 302 Found
Date: Fri, 21 Mar 2025 ... GMT
Server: Apache/2.4.54 (Debian)
Location: ../?result=Succesfully attacked facebook.com!
Content-Length: 310
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="../?result=Succesfully attacked facebook.com!">here</a>.</p>
<hr>
<address>Apache/2.4.54 (Debian) Server at ...</address>
</body></html>
```

Checking the page's source code, there is a bit of JS there:

```js        
// Check if the user's IP is local
const isLocalIP = (ip) => {
    return ip === "127.0.0.1" || ip === "::1" || ip.startsWith("192.168.");
};

// Get the user's IP address
const userIP = "<my machine's ip>";

// Enable/disable the "Attack IP" button based on the user's IP
const attackIPButton = document.getElementById("attack-ip");

// Enable buttons if required fields are filled
const enableButtons = () => {
    const playerName = document.getElementById("user-name").value;
    const target = document.getElementById("target").value;
    const attackDomainButton = document.getElementById("attack-domain");
    const attackIPButton = document.getElementById("attack-ip");

    if (playerName && target) {
        attackDomainButton.disabled = false;
        attackDomainButton.removeAttribute("data-hover");
        if (isLocalIP(userIP)) {
            attackIPButton.disabled = false;
        }
    } else {
        attackDomainButton.disabled = true;
        attackIPButton.disabled = true;
    }
};

document.getElementById("user-name").addEventListener("input", enableButtons);
document.getElementById("target").addEventListener("input", enableButtons);

// Attack Domain Button Click Handler
document.getElementById("attack-domain").addEventListener("click", async () => {
    const target = document.getElementById("target").value;
    const name = document.getElementById("user-name").value;
    if (target) {
        window.location.href = `cgi-bin/attack-domain?target=${target}&name=${name}`;
    }
});

// Attack IP Button Click Handler
document.getElementById("attack-ip").addEventListener("click", async () => {
    const target = document.getElementById("target").value;
    const name = document.getElementById("user-name").value;
    if (target) {
        window.location.href = `cgi-bin/attack-ip?target=${target}&name=${name}`;
    }
});
```

Looks like only ip's that fulfill the `isLocalIP` conditions are allowed to click the "Attack an IP" option. Kinda like some sort of very unsafe access level check to only allow this for people requesting the action from the same network.

If I try:

> GET /cgi-bin/attack-ip?target=127.0.0.1&name=Malakias

I get HTTP 403 (Forbidden) back.

## Failing at it

Honestly, I have no idea on how to proceed with ths one.