# Forensics Git 0 — picoCTF 2026 Writeup

**Challenge:** Forensics Git 0  
**Category:** Forensics  
**Difficulty:** Medium  
**Flag:** `picoCTF{g17_1n_7h3_d15k_041217d8}`  

---

## Description

> Can you find the flag in this disk image?  
> Download the disk image here.

**Hint 1:** `How can you extract the directory from the disk image?`

---

## Background Knowledge (Read This First!)

### What is a Disk Image?

A **disk image** is an exact byte-for-byte copy of a storage device — every partition, every file, every deleted sector. `.img` files are a staple of forensics challenges because they preserve everything that was on the original disk, including git repositories.

### What is a Git Commit Message?

Every time someone makes a commit in git, they write a **commit message** — a short description of what changed. These messages are stored permanently in the `.git/logs/HEAD` file. Unlike file contents which can be deleted, commit messages stay as long as the `.git` folder exists.

Sometimes, a developer accidentally puts sensitive information directly into a commit message — and this challenge is exactly that scenario.

### What is `debugfs`?

`debugfs` is a Linux tool for directly reading **ext4 filesystems** from disk images without needing to mount them. It lets us:

- Browse the directory structure (`ls`)
- Read individual files (`cat`)
- Extract entire directories (`rdump`)

It's like a file explorer for disk images — even without root mount access.

### What is `gzip`?

The disk image is compressed with **gzip** (`.gz` extension). Before we can work with it, we need to decompress it using `gunzip`. The `-c` flag writes the output to stdout and we redirect it to a new file, leaving the original compressed file untouched.

---

## Solution — Step by Step

### Step 1 — Decompress the disk image

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip -c disk.img.gz > disk_raw.img
```

Breaking this down:
- `gunzip` — the decompression tool for `.gz` files
- `-c` — write output to stdout instead of overwriting the original
- `> disk_raw.img` — redirect the decompressed data into a new file

### Step 2 — Identify the partitions

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file disk_raw.img
disk_raw.img: DOS/MBR boot sector;
  partition 1: ID=0x83, startsector 2048, 614400 sectors;
  partition 2: ID=0x82, startsector 616448, 524288 sectors;
  partition 3: ID=0x83, startsector 1140736, 956416 sectors
```

There are 3 partitions. Partition 3 is the main Linux filesystem (ext4) starting at sector `1140736`.

### Step 3 — Mount the disk partition

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ mkdir recovered
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo mount -o loop,offset=$((1140736*512)) disk_raw.img recovered/
```

### Step 4 — Find the git repo

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls recovered/home/ctf-player/Code/
secrets
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -la recovered/home/ctf-player/Code/secrets
total 3
drwxr-sr-x 3 zham zham 1024 Nov 19 04:20 .
drwxr-sr-x 3 zham zham 1024 Nov 19 04:20 ..
drwxr-sr-x 8 zham zham 1024 Nov 19 04:20 .git
```

Only a `.git` folder — no other files currently visible. Let's read the git history.

### Step 5 — Fix git ownership and read the commit log

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ git config --global --add safe.directory '*'
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat recovered/home/ctf-player/Code/secrets/.git/logs/HEAD
0000000000000000000000000000000000000000 327681bb38cf467cec328eec9707b240e3e74ced ctf-player commit (initial): Wrap this phrase in the flag format: g17_1n_7h3_d15k_041217d8
```

The developer accidentally put the flag phrase **directly into the commit message**! 🤦

The commit message says: `Wrap this phrase in the flag format: g17_1n_7h3_d15k_041217d8`

Wrapping it in `picoCTF{}`:

```
picoCTF{g17_1n_7h3_d15k_041217d8}
```

✅ Got the flag! 🎯

> **Fun detail:** `g17_1n_7h3_d15k` is leetspeak for "git in the disk" — a nod to exactly where we found it.

---

## Alternative Methods

### Alternative 1 — Use `git log` directly

After navigating into the repo:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cd recovered/home/ctf-player/Code/secrets
┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ git log --oneline
327681b (HEAD -> master) Wrap this phrase in the flag format: g17_1n_7h3_d15k_041217d8
```

Same flag visible directly in the commit message.

### Alternative 2 — Use `debugfs` instead of mounting (no sudo needed)

If you don't have sudo access, use `debugfs` to extract the repo without mounting:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ debugfs -R "rdump /home/ctf-player/Code/secrets recovered/" disk_raw.img
debugfs 1.47.0 (5-Feb-2023)
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat recovered/secrets/.git/logs/HEAD
0000000000000000000000000000000000000000 327681bb38cf467cec328eec9707b240e3e74ced ctf-player commit (initial): Wrap this phrase in the flag format: g17_1n_7h3_d15k_041217d8
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `gunzip` | Decompress the gzip disk image | ⭐ Easy |
| `file` | Identify partition layout | ⭐ Easy |
| `mount` | Mount the ext4 partition from the disk image | ⭐ Medium |
| `cat .git/logs/HEAD` | Read the full git commit history | ⭐ Easy |

---

## Key Takeaways

- **Commit messages are permanent** — never put sensitive data in a git commit message; it stays in `.git/logs/HEAD` forever
- **`mount -o loop,offset=`** lets you mount a specific partition from a raw disk image by calculating the byte offset from the sector number
- The challenge name says it all — the flag was hiding "git in the disk" (`g17_1n_7h3_d15k`)
