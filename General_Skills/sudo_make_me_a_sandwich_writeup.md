# SUDO MAKE ME A SANDWICH — picoCTF Writeup

**Challenge:** SUDO MAKE ME A SANDWICH  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{ju57_5ud0_17_2feb37e6}`  

---

## Description

> Can you read the flag? I think you can!
>
> `ssh -p 50774 ctf-player@green-hill.picoctf.net` using password `f068c7da`

**Hint 1:** `What is sudo?`  
**Hint 2:** *(unlockable)*

---

## Background Knowledge (Read This First!)

### What is `sudo`?

**`sudo`** stands for **"superuser do"**. It lets a normal user run a specific command **as root** (the administrator). Think of it like asking for temporary admin privileges for just one command.

The challenge title is a reference to a classic Linux joke:

> User: "Make me a sandwich."  
> Computer: "No."  
> User: "**sudo** make me a sandwich."  
> Computer: "Okay."

The joke means: with `sudo`, you have the power to do things that are normally restricted.

### What is file permission?

On Linux, every file has permissions that control who can read, write, or execute it. When you try to open a file and get:

```
cat: flag.txt: Permission denied
```

It means your current user account does not have permission to read that file. Only **root** (or a user with elevated privileges) can open it.

### What is `sudo -l`?

`sudo -l` lists all the commands your current user is **allowed to run with sudo**. This is critical in CTF challenges — if you can't run `cat` with sudo, something else on the list might still let you read the file.

### What is `emacs`?

**Emacs** is a powerful text editor for Linux. It can open and display files — which means if you're allowed to run it with sudo, you can use it to read files that are normally restricted. To exit emacs, press `Ctrl + X` then `Ctrl + C`.

### ⚠️ Important Note

**Passwords are invisible in SSH** — when prompted, type `f068c7da` and press Enter even though nothing appears on screen.

---

## Solution — Step by Step

### Step 1 — SSH into the challenge server

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 50774 ctf-player@green-hill.picoctf.net
```

Type `yes` to accept the fingerprint, then enter password `f068c7da`:

```
ctf-player@green-hill.picoctf.net's password:
Welcome to Ubuntu 20.04.3 LTS...
ctf-player@challenge:~$
```

### Step 2 — Look around

```
ctf-player@challenge:~$ ls
flag.txt
```

There's one file: `flag.txt`. Let's try to read it directly:

```
ctf-player@challenge:~$ cat flag.txt
cat: flag.txt: Permission denied
```

As expected — the file is locked. Your normal user account can't read it.

### Step 3 — Try sudo cat

The obvious next move is to try reading it as root with `sudo`:

```
ctf-player@challenge:~$ sudo cat flag.txt
[sudo] password for ctf-player:
Sorry, user ctf-player is not allowed to execute '/usr/bin/cat flag.txt' as root on challenge.
```

Still blocked — `cat` is not on the allowed list. Before giving up, check what commands **are** allowed with sudo.

### Step 4 — Check sudo permissions

```
ctf-player@challenge:~$ sudo -l
Matching Defaults entries for ctf-player on challenge:
    env_reset, mail_badpass, ...
User ctf-player may run the following commands on challenge:
    (ALL) NOPASSWD: /bin/emacs
```

Only **`emacs`** is allowed — and with `NOPASSWD`, meaning it won't even ask for a password. Emacs is a text editor, and a text editor can open and display any file. That's our way in.

### Step 5 — Open the flag in emacs as root

```
ctf-player@challenge:~$ sudo emacs flag.txt
```

Emacs opens with root privileges and displays the contents of `flag.txt`:

```
picoCTF{ju57_5ud0_17_2feb37e6}
```

✅ Got the flag! 🎯

### Step 6 — Exit emacs

To close emacs and return to the terminal:

```
Ctrl + X, then Ctrl + C
```

---

## Alternative Methods

### Alternative 1 — Read the file from inside emacs without the GUI

If you're on a terminal without a graphical display, emacs still works in text mode. The flag will appear right in the terminal window just the same.

### Alternative 2 — Use emacs to run a shell command

Emacs has a built-in way to execute shell commands. Once inside `sudo emacs`, you can press:

```
Meta + ! (Alt + !)
```

Then type:
```
cat flag.txt
```

This runs `cat` **from within emacs as root**, bypassing the sudo restriction on `cat` directly. The flag appears in the emacs minibuffer at the bottom.

### Alternative 3 — GTFOBins

**GTFOBins** (https://gtfobins.github.io/) is a reference site listing Linux binaries that can be abused to escalate privileges or bypass restrictions. If you search for `emacs` on GTFOBins, it shows multiple ways to exploit sudo access to emacs — including reading files and even spawning a root shell. This is the first place to check in any CTF where `sudo -l` shows an unexpected binary.

---

## Why Could We Use Emacs?

The server administrator restricted `cat` but forgot that **any program that can read files** is effectively equivalent to `cat` when given sudo access. Emacs, `vim`, `less`, `more`, `nano` — all of them can display file contents. This is a classic **sudo misconfiguration**: the intention was to block direct file reading, but the allowed binary list wasn't properly locked down.

The flag name says it all: `ju57_5ud0_17` → "just sudo it."

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `ssh` | Connect to the remote challenge server | ⭐ Easy |
| `cat` | Attempt to read the flag (fails) | ⭐ Easy |
| `sudo -l` | Discover what commands are allowed with sudo | ⭐ Easy |
| `sudo emacs` | Open the restricted file as root | ⭐ Easy |

---

## Key Takeaways

- **Always run `sudo -l`** when you can't access a file — it shows exactly which commands you can escalate with
- **Any file-reading program with sudo is as powerful as `sudo cat`** — editors, pagers, and viewers all expose file contents
- **GTFOBins** is your best friend for sudo privilege escalation — bookmark it
- `NOPASSWD` in sudo rules means the command runs without asking for a password — a bigger risk than it looks
- The challenge title is the hint: when something says "Permission denied", think **sudo**
