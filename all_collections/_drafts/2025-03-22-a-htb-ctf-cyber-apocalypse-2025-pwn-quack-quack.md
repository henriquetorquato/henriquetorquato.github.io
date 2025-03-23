---
layout: post
title: 'Hack The Box CTF 2025: Tales from Eldoria - Pwn - Quack Quack'
created-at: 2025-03-21
categories: ["Hack The Box", "WriteUp", "Cybersecurity", "Tales from Eldoria", "CTF"]
---

|Category|Name|Difficulty|Status|
|:---:|:---:|:---:|:---:|
|Pwn|Quack Quack|Very Easy||

> On the quest to reclaim the Dragon's Heart, the wicked Lord Malakar has cursed the villagers, turning them into ducks! Join Sir Alaric in finding a way to defeat them without causing harm. Quack Quack, it's time to face the Duck!

---

The target here is a `quack_quack` binary file. When invoking it, it prints out the picture of a duck, and expects you to quack it.

![Mr. Quack Quack](/docs/assets/2025-03/htb-ctf-pwn-quack-quack-1.png)

But it seems like it needs a proper greeting.

![Mr. Quack Quack - Angry](/docs/assets/2025-03/htb-ctf-pwn-quack-quack-2.png)

Attempting to print out the contents of the binary with `cat` shows some interesting things:

![cat result](/docs/assets/2025-03/htb-ctf-pwn-quack-quack-3.png)

It seems that there is a conversation programmed.

I then attempted to run the extract the binary strings using the `strings` utility, but didn't get anything important back.

I then uploaded the binary to a online decompiler to see how effective it would be, it looks like the names were not stripped during compiling:

![Online decompiler](/docs/assets/2025-03/htb-ctf-pwn-quack-quack-4.png)

So I fired up `Ghidra` and started decompiling the code.

Looks like `main` calls the `duckling` function, that not only prints out the majestic duck, it also handles the conversation.

There I could see that it expected what I think is a name prefixed with "Quack Quack " as a greeting:

![duckling function](/docs/assets/2025-03/htb-ctf-pwn-quack-quack-5.png)

And with that, I was able to get him talking. But, looking at the code, it doesn't really matter what I say to him, he'll always win.

There is a `duck_attack` method that is responsible for reading and displaying the target flag:

![duck_attack function](/docs/assets/2025-03/htb-ctf-pwn-quack-quack-6.png)

At this point, I decided to run the target machine to see what are the limitations on executing the exploit, since one of the solution I thought for this was the debug the executable and force call the `duck_attack` function call.

It was a little complicated and weird since it gave me a IP/port. But I found [this](https://medium.com/@rahulhoysala07/hack-the-box-pwn-challenge-getting-started-54acc706afa7) post on Medium by [Rahul Hoysala](https://medium.com/@rahulhoysala07) that explained how to go through with accessing the challenge.

```python
from pwn import *

r = remote('83.136.251.145', 43092)
r.stream()

r.sendline(b'Quack Quack test')
```