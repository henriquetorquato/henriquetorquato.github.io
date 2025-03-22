---
layout: post
title: 'Hack The Box CTF 2025: Tales from Eldoria - Web - Whispers of the Moonbeam'
created-at: 2025-03-21
categories: ["Hack The Box", "WriteUp", "Cybersecurity", "Tales from Eldoria", "CTF"]
---

|Category|Name|Difficulty|Points|Status|
|:---:|:---:|:---:|:---:|:---:|
|Web|Whispers of the Moonbeam|Very Easy|900|Completed ✅|

> In the heart of Valeria's bustling capital, the Moonbeam Tavern stands as a lively hub of whispers, wagers, and illicit dealings. Beneath the laughter of drunken patrons and the clinking of tankards, it is said that the tavern harbors more than just ale and merriment—it is a covert meeting ground for spies, thieves, and those loyal to Malakar's cause. The Fellowship has learned that within the hidden backrooms of the Moonbeam Tavern, a crucial piece of information is being traded—the location of the Shadow Veil Cartographer, an informant who possesses a long-lost map detailing Malakar’s stronghold defenses. If the fellowship is to stand any chance of breaching the Obsidian Citadel, they must obtain this map before it falls into enemy hands.

---

This one didn't even deserve a writeup 😅.

This challenge is basically a browser text based game, where you can give commands and your character will execute actions during the game.

The commands not only describe the action your character is doing, it also return some data from the machine. Right away after typing `gossip`, the directory content is dumped showing the `flag.txt` file sitting there:

![Flag menacingly just sitting there](/docs/assets/2025-03/htb-ctf-web-whispers-of-the-moonbeam-1.png)

None of the base commands that are displayed when typing `help` can print contents of a file.

But I did notice right at the bottom of the screen:
> Tip: Use ↑↓ for history, Tab for completion, ; for command injection

";" for command injection? 🤔

![Flag contents - just like that](/docs/assets/2025-03/htb-ctf-web-whispers-of-the-moonbeam-2.png)

There it was.