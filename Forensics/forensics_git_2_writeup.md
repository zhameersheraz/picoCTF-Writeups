# Forensics Git 2 — picoCTF 2026 Writeup

**Challenge:** Forensics Git 2  
**Category:** Forensics  
**Difficulty:** Medium  
**Flag:** `picoCTF{g17_r35cu3_16ac6bf3}`

---

## Description

The agents interrupted the perpetrator's disk deletion routine. Can you recover this git repo?

**Hint 1:** We think the deletion was interrupted before any git objects were touched.

---

## Background Knowledge (Read This First!)

### What is a Disk Image?

A **disk image** is a file that contains an exact copy of a storage device — partitions, filesystem, and all files. `.img` files are common in forensics challenges because they preserve everything including deleted files.

### What is Git History?

Git keeps a full history of every commit ever made — even if files are deleted later, the content still exists in the `.git` folder as objects. A `git show <commit-hash>` lets you read any past commit, including ones that were "removed."

### The Key Forensics Trick

If someone commits a secret then deletes it in a later commit, the secret is **still in the git history**. The `.git/logs/HEAD` file shows every commit ever made — we can find the deleted commit hash and read it directly.

---

## Solution — Step by Step

### Step 1 — Decompress the disk image

The downloaded file is gzip compressed. Decompress it first:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip -c disk.img > disk_raw.img
```

### Step 2 — Mount the disk partition

The disk has 3 partitions. The filesystem we need is partition 3, starting at sector `1140736`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ mkdir recovered
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo mount -o loop,offset=$((1140736*512)) disk_raw.img recovered/
```

### Step 3 — Find the git repo

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls recovered/home/ctf-player/Code/
> killer-chat-app

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cd recovered/home/ctf-player/Code/killer-chat-app
┌──(zham㉿kali)-[/media/…/Code/killer-chat-app]
└─$ ls -la
> drwxr-sr-x 8 zham zham 1024 Nov 19 05:47 .git
> -rwxr-xr-x 1 zham zham   25 Nov 19 05:47 client
> drwxr-sr-x 2 zham zham 1024 Nov 19 05:47 logs
> -rwxr-xr-x 1 zham zham   25 Nov 19 05:47 server
```

The `.git` folder is intact — deletion was interrupted before it was touched. ✅

### Step 4 — Fix git ownership error

```
┌──(zham㉿kali)-[/media/…/Code/killer-chat-app]
└─$ git config --global --add safe.directory '*'
┌──(zham㉿kali)-[/media/…/Code/killer-chat-app]
└─$ git config --global --add safe.directory /media/sf_downloads/recovered/home/ctf-player/Code/killer-chat-app
```

### Step 5 — Read the git log from HEAD

```
┌──(zham㉿kali)-[/media/…/Code/killer-chat-app]
└─$ cat .git/logs/HEAD
> 0000000000000000000000000000000000000000 2c0a9b2b15dce92f800393d5030c7454efc278ae ctf-player commit (initial): Add netcat scripts
> 2c0a9b2b15dce92f800393d5030c7454efc278ae 26b809e0c41d8421f1126ed3a4eb06ad66e6d90a ctf-player commit: Add video game chat log
> 26b809e0c41d8421f1126ed3a4eb06ad66e6d90a 5827632e046a80a1e0d7b4fc5c7800dd539baeaf ctf-player commit: Add TV show chat log
> 5827632e046a80a1e0d7b4fc5c7800dd539baeaf e80b38b3322a5ba32ac07076ef5eeb4a59449875 ctf-player commit: Add secret hideout chat log  ← 🎯
> e80b38b3322a5ba32ac07076ef5eeb4a59449875 2151ef0ccc15aed1ab88e1afdc7484aaeff211c4 ctf-player commit: Remove secret hideout log
> 2151ef0ccc15aed1ab88e1afdc7484aaeff211c4 01533f718556a0e59f1467dae4fa462eed82c2a1 ctf-player commit: Add random chat log
```

The suspicious commit is **"Add secret hideout chat log"** which was immediately removed in the next commit. That's where the flag is hidden.

### Step 6 — Show the deleted commit

```
┌──(zham㉿kali)-[/media/…/Code/killer-chat-app]
└─$ git show e80b38b3322a5ba32ac07076ef5eeb4a59449875
> commit e80b38b3322a5ba32ac07076ef5eeb4a59449875
> Author: ctf-player <ctf-player@example.com>
> Date:   Wed Nov 19 10:47:20 2025 +0000
>     Add secret hideout chat log
>
> diff --git a/logs/3.txt b/logs/3.txt
> new file mode 100644
> --- /dev/null
> +++ b/logs/3.txt
> @@ -0,0 +1,3 @@
> +Rex: Meet at the old arcade basement for the secret hideout.
> +Jay: Ask Rusty at the door and use password picoCTF{g17_r35cu3_16ac6bf3}.
> +Rex: Bring the decoder map so we can plan the route.
```

✅ Got the flag! 🎯

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `gunzip` | Decompress the gzip disk image | ⭐ Easy |
| `mount` | Mount the ext4 partition from the disk image | ⭐ Medium |
| `cat .git/logs/HEAD` | Read full git commit history including deleted commits | ⭐ Medium |
| `git show <hash>` | Display the contents of any commit by its hash | ⭐ Easy |

---

## Key Takeaways

- **Git never truly deletes history** — even after `git rm` and a new commit, the old content lives in `.git` objects forever
- **`.git/logs/HEAD`** shows every commit hash ever made — even ones not visible in `git log` due to missing refs
- **Disk images preserve everything** — mounting a raw `.img` file gives you full access to the filesystem as it was
- The flag `g17_r35cu3` → "git rescue" — you rescued the flag from deleted git history
