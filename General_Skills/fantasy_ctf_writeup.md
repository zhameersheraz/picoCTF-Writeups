# FANTASY CTF — picoCTF Writeup

**Challenge:** FANTASY CTF  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{m1113n1um_3d1710n_1e2b417a}`

---

## Description

> Play this short game to get familiar with terminal applications and some of the most important rules in scope for picoCTF.
> Connect with: `nc verbal-sleep.picoctf.net 52341`

**Hint shown in challenge:** `When a choice is presented like [a/b/c], choose one, for example: a and then press Enter.`

---

## Background Knowledge (Read This First!)

### What is Netcat?

**Netcat (nc)** is a networking utility for reading from and writing to network connections. Basic usage:

```bash
nc hostname port
```

Common uses: connect to CTF challenge servers, port scanning, file transfers, debugging network services.

### Important picoCTF Rules (what this challenge teaches)

1. **One Account Per Person** — never register multiple accounts
2. **No Account Sharing** — each person must use their own account
3. **No Flag Sharing** — don't share flags with other competitors during the competition
4. **No Cheating** — solve challenges yourself, don't search for flags online
5. **Writeup Timing** — wait until after winners are announced to publish writeups

---

## Solution — Step by Step

### Step 1 — Connect to the Server

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nc verbal-sleep.picoctf.net 52341
```

### Step 2 — Read the Story

The challenge presents an interactive fiction game set in the year 3025. I pressed **Enter** to progress through the story dialogue.

### Step 3 — First Choice: Account Registration

```
Options:
A) *Register multiple accounts*
B) *Share an account with a friend*
C) *Register a single, private account*
[a/b/c] > c
```

I chose **`c`** — Nyx warns that registering multiple accounts has been grounds for disqualification for 1,000 years!

### Step 4 — Second Choice: Finding the Flag

```
Options:
A) *Play the game*
B) *Search the Ether for the flag*
[a/b] > a
```

I chose **`a`** — Nyx says "You never want to share flags or artifact downloads."

### Step 5 — Get the Flag

After playing the game, a progress bar appears and completes. The story concludes:

```
"Thanks, Nyx! Here's the flag I found: picoCTF{m1113n1um_3d1710n_1e2b417a}"
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` (netcat) | Connect to the challenge server |

---

## Key Takeaways

- Follow CTF rules — they exist to keep competitions fair
- One account per person — multiple accounts = disqualification
- Don't share flags — this ruins the competition for others
- Publish writeups responsibly — wait until after the competition
- The challenge is set in 3025, making it the "millennium edition" — hence the flag name
