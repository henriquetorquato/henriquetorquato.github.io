---
layout: post
title: 'Hack The Box CTF 2025: Tales from Eldoria - Rev - SealedRune'
created-at: 2025-03-23
categories: ["Hack The Box", "WriteUp", "Cybersecurity", "Tales from Eldoria", "CTF"]
---

|Category|Name|Difficulty|Status|
|:---:|:---:|:---:|:---:|
|Rev|SealedRune|Very Easy|Completed ✅|

> Elowen has reached the Ruins of Eldrath, where she finds a sealed rune stone glowing with ancient power. The rune is inscribed with a secret incantation that must be spoken to unlock the next step in her journey to find The Dragon’s Heart.

---

When first running the program it asks for an incantation to reveal it's secret, and if the secret is correct, the flag will be revealed.

![Sealed Rune - Fail](/docs/assets/2025-03/htb-ctf-rev-sealed-rune-1.png)

I ran `checksec` and saw that the binary was not stripped. So I executed `Ghidra` and started decompiling the code.

I found the functions `decode_secret`, `decode_flag` and `check_input` that seem to be the meat and potatos here:

```c
void check_input(char *input)
{
  int is_equal_secret;
  char *secret;
  long flag;
  
  secret = (char *)decode_secret();
  is_equal_secret = strcmp(input,secret);
  if (is_equal_secret == 0) {
    puts(&DAT_00102050);
    flag = decode_flag();
    printf("\x1b[1;33m%s\x1b[0m\n",flag + 1);
  }
  else {
    puts("\x1b[1;31mThe rune rejects your words... Try again.\x1b[0m");
  }
  free(secret);
  return;
}

undefined8 decode_secret(void)
{
  undefined8 secret;
  
  secret = base64_decode(incantation);
  reverse_str(secret);
  return secret;
}

undefined8 decode_flag(void)
{
  undefined8 uVar1;
  
  uVar1 = base64_decode(flag);
  reverse_str(uVar1);
  return uVar1;
}
```

it seems that the `incantation` and `flag` variables are stored reversed and base64 encoded as global variables. And if the input matches the incantation, the flag is printed out.

After some time looking online on how to access globally defined values, I ran "String Search" utility on `Ghidra` and for my surprise, even with default configurations, it found the `incantation` string:

![String Search](/docs/assets/2025-03/htb-ctf-rev-sealed-rune-2.png)

After some online base64 decoding and string reverse tools, I got back the secret incantation `VaelGnuffraz`.

And just like that, we have the flag:

![Flag acquired](/docs/assets/2025-03/htb-ctf-rev-sealed-rune-3.png)