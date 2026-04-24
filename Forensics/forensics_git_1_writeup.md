# Forensics Git 1 — picoCTF 2026 Writeup

**Challenge:** Forensics Git 1  
**Category:** Forensics  
**Difficulty:** Medium  
**Flag:** `picoCTF{g17_r3m3mb3r5_d4ddf904}`  

---

## Description

> Can you find the flag in this disk image?  
> Download the disk image here.

**Hint 1:** `How can you checkout the files of a previous commit?`

---

## Background Knowledge (Read This First!)

### What is a Disk Image?

A **disk image** is an exact byte-for-byte copy of a storage device — every partition, every file, every sector. `.img` files are common in forensics challenges because they preserve the full filesystem, including git history.

### Git Never Truly Forgets

When you delete a file in git and commit the deletion, the file appears to be gone — but it's not. Git stores **every version of every file** as objects inside the `.git/objects/` folder. Even after a `git rm` + commit, the old file content remains intact in git's object database.

`git log -p` shows the full history including **every diff ever made** — even diffs that removed a file. That deleted content is still right there in the output, just marked with a `-` sign.

### What is `debugfs`?

`debugfs` is a Linux tool for reading **ext4 filesystems** directly from disk images without needing to mount them. It lets us browse directories, read files, and extract entire directory trees with `rdump`.

### What is `gzip`?

The disk image is compressed with **gzip**. We decompress it with `gunzip -c` before working with it.

---

## Solution — Step by Step

### Step 1 — Decompress and mount the disk image

The disk has 3 partitions. The filesystem we need is partition 3, starting at sector `1140736`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip -c disk.img.gz > disk_raw.img
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ mkdir recovered
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo mount -o loop,offset=$((1140736*512)) disk_raw.img recovered/
```

### Step 2 — Find the git repo

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls recovered/home/ctf-player/Code/
secrets
```

### Step 3 — Try to read the secrets folder directly

We try a few tools first to understand what we're dealing with:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool secrets
Error: File not found - secrets

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file recovered/home/ctf-player/Code/secrets
recovered/home/ctf-player/Code/secrets: setgid, directory

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat recovered/home/ctf-player/Code/secrets
cat: recovered/home/ctf-player/Code/secrets: Is a directory

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings recovered/home/ctf-player/Code/secrets
strings: Warning: 'recovered/home/ctf-player/Code/secrets' is a directory

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ less recovered/home/ctf-player/Code/secrets
recovered/home/ctf-player/Code/secrets is a directory
```

It's a directory — not a file we can read directly. We need to go inside it.

### Step 4 — Navigate inside and look around

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cd recovered/home/ctf-player/Code/secrets

┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ ls -la
total 3
drwxr-sr-x 3 zham zham 1024 Nov 19 04:20 .
drwxr-sr-x 3 zham zham 1024 Nov 19 04:20 ..
drwxr-sr-x 8 zham zham 1024 Nov 19 04:20 .git
```

Only a `.git` folder — no other files currently exist. This means the flag was in a file that was **deleted via git**. Let's dig deeper.

### Step 5 — List all files in the git repo

```
┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ find . -type f
./.git/config
./.git/refs/heads/master
./.git/objects/f1/50f47a5dabfb4397706aa18905df936595a86e
./.git/objects/5f/b8194539c770a830b8ba089a50778c07072b03
./.git/objects/a6/2340e078686778969b9a555fc722147cf14e5a
./.git/objects/17/7789af0b300e043ea8f54ea57d6cee352291ae
./.git/objects/4b/825dc642cb6eb9a060e54bf8d69288fbee4904
./.git/logs/refs/heads/master
./.git/logs/HEAD
./.git/COMMIT_EDITMSG
./.git/description
./.git/HEAD
./.git/index
./.git/hooks/applypatch-msg.sample
...
```

We can see git objects stored in `.git/objects/` — these are the compressed file versions git keeps forever, even after deletion.

### Step 6 — Check git status and log

```
┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ git status
On branch master
nothing to commit, working tree clean

┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ git log --oneline
5fb8194 (HEAD -> master) Remove flag
177789a Add flag
```

Two commits — "Add flag" then immediately "Remove flag". The flag is hiding in the history!

### Step 7 — View the full diff history with `git log -p`

```
┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ git log -p
commit 5fb8194539c770a830b8ba089a50778c07072b03 (HEAD -> master)
Author: ctf-player <ctf-player@example.com>
Date:   Wed Nov 19 09:20:05 2025 +0000

    Remove flag

diff --git a/flag.txt b/flag.txt
deleted file mode 100644
index f150f47..0000000
--- a/flag.txt
+++ /dev/null
@@ -1 +0,0 @@
-picoCTF{g17_r3m3mb3r5_d4ddf904}
\ No newline at end of file

commit 177789af0b300e043ea8f54ea57d6cee352291ae
Author: ctf-player <ctf-player@example.com>
Date:   Wed Nov 19 09:20:05 2025 +0000

    Add flag

diff --git a/flag.txt b/flag.txt
new file mode 100644
index 0000000..f150f47
--- /dev/null
+++ b/flag.txt
@@ -0,0 +1 @@
+picoCTF{g17_r3m3mb3r5_d4ddf904}
\ No newline at end of file
```

Even though `flag.txt` was deleted, `git log -p` shows us the full diff of every commit — including the line that was removed. The flag is right there prefixed with `-` in the "Remove flag" commit and `+` in the "Add flag" commit.

✅ Got the flag! 🎯

> **Fun detail:** `g17_r3m3mb3r5` is leetspeak for "git remembers" — a perfect name for a challenge where git's memory of deleted files is the key to solving it.

---

## Alternative Methods

### Alternative 1 — `git show` on the first commit

Instead of `git log -p` (which shows all commits), you can target just the specific commit that added the flag:

```
┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ git show 177789af
commit 177789af0b300e043ea8f54ea57d6cee352291ae
Author: ctf-player <ctf-player@example.com>
Date:   Wed Nov 19 09:20:05 2025 +0000

    Add flag

diff --git a/flag.txt b/flag.txt
new file mode 100644
index 0000000..f150f47
--- /dev/null
+++ b/flag.txt
@@ -0,0 +1 @@
+picoCTF{g17_r3m3mb3r5_d4ddf904}
```

### Alternative 2 — Checkout the old commit to restore the file

```
┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ git checkout 177789af
Note: switching to '177789af...'
HEAD is now at 177789a Add flag
┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ cat flag.txt
picoCTF{g17_r3m3mb3r5_d4ddf904}
```

This actually restores `flag.txt` back to disk so you can read it like a normal file.

### Alternative 3 — Read the git blob object directly

Every file version is stored as a blob in `.git/objects/`. The blob hash `f150f47` appears in the diff output — you can read it directly:

```
┌──(zham㉿kali)-[/media/…/home/ctf-player/Code/secrets]
└─$ git cat-file -p f150f47
picoCTF{g17_r3m3mb3r5_d4ddf904}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `gunzip` | Decompress the gzip disk image | ⭐ Easy |
| `mount` | Mount the ext4 partition from the disk image | ⭐ Medium |
| `ls -la` | Reveal hidden `.git` folder | ⭐ Easy |
| `find . -type f` | List all files including git objects | ⭐ Easy |
| `git log --oneline` | See commit history at a glance | ⭐ Easy |
| `git log -p` | Show full diffs of every commit — including deleted content | ⭐ Easy |

---

## Key Takeaways

- **Git never truly deletes file content** — even after `git rm` + commit, the old data lives in `.git/objects/` forever
- **`git log -p`** shows the full diff of every commit including removed lines — deleted file content is still visible prefixed with `-`
- **`git log --oneline`** is a fast way to spot suspicious commit names like "Remove flag" that signal something was hidden then erased
- **`find . -type f`** inside a `.git` folder reveals all the stored object files — proof that git keeps everything
- The challenge hint said it all: "How can you checkout the files of a previous commit?" — `git checkout <hash>` brings deleted files back to life
