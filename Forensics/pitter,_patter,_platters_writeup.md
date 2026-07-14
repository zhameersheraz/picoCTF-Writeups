# Pitter, Patter, Platters — picoCTF Writeup

**Challenge:** Pitter, Patter, Platters  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 250  
**Flag:** `picoCTF{b3_5t111_mL|_<3_f2136893}`  
**Platform:** picoCTF 2020 Mini-Competition  
**Writeup by:** zham  

---

## Description

> 'Suspicious' is written all over this disk image.
>
> Download suspicious.dd.sda1

**Hints given in the challenge:**

- It may help to analyze this image in multiple ways: as a blob, and as an actual mounted disk.
- Have you heard of slack space? There is a certain set of tools that now come with Ubuntu that I'd recommend for examining that disk space phenomenon...

---

## Background Knowledge (Read This First!)

### What is "slack space"?

Filesystems allocate disk space in fixed-size **blocks** (also called clusters on Windows). On this image the block size is **1024 bytes** — `fsstat` confirms it.

When you save a file that is smaller than the block size, the filesystem still gives the file a full block. The bytes between the **end of the file's actual data** and the **end of its last block** are called **slack space** (or file slack / block slack).

```
|<----- one 1024-byte block ----->|
| real data | slack space (zeros)  |
```

The OS reports the file size as the "real data" length, so the slack bytes look unused. But the disk still physically holds whatever was written there before — and crucially, anyone who can read the raw block (without going through the file's logical size) can see those bytes.

Slack space is a classic place to hide small pieces of data — a flag, a fragment of a message, etc. It survives normal file copies and even most "delete" operations because the bytes are never reallocated until the filesystem decides that whole block is free.

### The Sleuth Kit family of tools

The Sleuth Kit (TSK) is a CLI toolkit for analyzing disk images without mounting them. For this challenge the two relevant tools are:

| Tool | Purpose |
|------|---------|
| `fsstat` | Show filesystem info (type, block size, last mount) |
| `fls` | List file and directory names |
| `istat` | Show a file's inode metadata (size, timestamps, **direct blocks**) |
| `blkcat` | Print the raw contents of a block by its block number |
| `blkls` | Print unallocated blocks. With `-s` it prints **slack space** of allocated files |

A few of these are the "certain set of tools that now come with Ubuntu" the hint is talking about. TSK ships in the standard `sleuthkit` package, so on Ubuntu / Kali it is a one-liner away:

```bash
sudo apt install sleuthkit
```

### What `istat` tells you about a file

When you run `istat` on an inode, the bottom of the output lists the **Direct Blocks** the file occupies on disk. Those numbers are physical block addresses inside the image, not the same as inode numbers. You can feed them straight into `blkcat` to dump the raw block.

### What `strings -e l` does

Plain `strings` reads ASCII / UTF-8 runs of printable characters. The `-e l` switch tells it to also scan for **little-endian 16-bit** (UTF-16LE) runs — pairs of bytes where the second is a null. That is a very common way to hide short strings inside an otherwise-binary blob, and it is exactly the encoding used to stash this flag.

### About the title

"Pitter, Patter, Platters" is a tongue-twister riff on "Peter Piper picked a peck of pickled peppers." The "platter" half is the giveaway — a hard disk is made of one or more spinning **platters**, and the title is nudging you to think about how data lives on those platters, down to the byte level.

---

## Solution — Step by Step

### Step 1 — Move the image somewhere safe and confirm what it is

Disk images are big and I do not want to fill my home directory, so I work out of `/tmp`.

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /tmp/pitter && cd /tmp/pitter
┌──(zham㉿kali)-[/tmp/pitter]
└─$ cp ~/Downloads/suspicious.dd.sda1 .
┌──(zham㉿kali)-[/tmp/pitter]
└─$ ls -la
total 32108
-rw-r--r-- 1 zham zham 32868864 ... suspicious.dd.sda1
┌──(zham㉿kali)-[/tmp/pitter]
└─$ file suspicious.dd.sda1
suspicious.dd.sda1: Linux rev 1.0 ext3 filesystem data, UUID=fc168af0-183b-4e53-bdf3-9c1055413b40 (needs journal recovery)
```

`file` already tells us this is a raw `ext3` filesystem image, not a full disk with a partition table. That is why the challenge name ends in `sda1` — it is the first partition of a disk, dumped directly.

### Step 2 — Look at the filesystem metadata

```
┌──(zham㉿kali)-[/tmp/pitter]
└─$ fsstat suspicious.dd.sda1
FILE SYSTEM INFORMATION
--------------------------------------------
File System Type: Ext3
Volume ID: 403b4155109cf3bd534e3b18f08a16fc
Last Mounted at: 2020-09-30 09:26:26 (UTC)
Last mounted on: /mnt/sda1
...
CONTENT INFORMATION
--------------------------------------------
Block Range: 0 - 32095
Block Size: 1024
```

Two things to remember: it is ext3, and **block size = 1024 bytes**.

### Step 3 — List the files and read the obvious-looking one

The description says "Suspicious is written all over this disk image." I list everything and look for the obvious decoy.

```
┌──(zham㉿kali)-[/tmp/pitter]
└─$ fls suspicious.dd.sda1
d/d 11:    lost+found
d/d 2009:  boot
d/d 4017:  tce
r/r 12:    suspicious-file.txt
V/V 8033:  $OrphanFiles
```

`suspicious-file.txt` is the bait. I pull it out by inode number with `icat`.

```
┌──(zham㉿kali)-[/tmp/pitter]
└─$ icat suspicious.dd.sda1 12
Nothing to see here! But you may want to look here -->
```

Yep, a decoy. But it confirms two things: the file is 55 bytes long, and the message itself ("...look here -->") is a hint that the flag is somewhere near the file, not in it. Hint 2 was already screaming **slack space** at us.

### Step 4 — Find which block the file lives in

I run `istat` on inode 12 to get the direct block pointer.

```
┌──(zham㉿kali)-[/tmp/pitter]
└─$ istat suspicious.dd.sda1 12
inode: 12
Allocated
Group: 0
mode: rrw-r--r--
size: 55
num of links: 1
Inode Times:
  Accessed:    2020-09-30 13:15:59 (UTC)
  File Modified:   2020-09-30 05:17:23 (UTC)
  Inode Modified:  2020-09-30 13:15:54 (UTC)
Direct Blocks:
2049
```

Block **2049**. That single 1024-byte block holds the entire file. The file uses 55 bytes; the remaining 1024 − 55 = **969 bytes** of that block are slack space.

### Step 5 — Dump the raw block and look past the file's end

`blkcat` reads raw block contents ignoring the file's logical size, so it shows me every byte in that block — including the slack.

```
┌──(zham㉿kali)-[/tmp/pitter]
└─$ blkcat suspicious.dd.sda1 2049 | head -c 200
Nothing to see here! But you may want to look here -->
}      3       9       8       6       3       1       2       f       _
3       <       _       |       L       m       _       1       1       1
t       5       _       3       b       {       F       T       C       o
c       i       p
```

There it is, but it is **garbled by null bytes**. Each printable character is followed by a `\0`. That is the classic signature of **UTF-16 little-endian** encoding.

### Step 6 — Decode the UTF-16LE string

I send the slack-region dump through `strings -e l`, which understands UTF-16LE and just prints the readable text:

```
┌──(zham㉿kali)-[/tmp/pitter]
└─$ blkcat suspicious.dd.sda1 2049 | strings -e l
Nothing to see here! But you may want to look here -->
}3986312f_3<_|Lm_111t5_3b{FTCocip
```

I now have a clean flag-like string — but it reads **backwards**. It starts with `}` and ends with `picoCTF{`. A simple `rev` fixes that.

```
┌──(zham㉿kali)-[/tmp/pitter]
└─$ echo '}3986312f_3<_|Lm_111t5_3b{FTCocip' | rev
picoCTF{b3_5t111_mL|_<3_f2136893}
```

**Flag:** `picoCTF{b3_5t111_mL|_<3_f2136893}`

### Step 7 (optional) — One-liner with `blkls -s`

The Sleuth Kit has a dedicated slack-space mode: `blkls -s` prints just the slack bytes of every allocated file in one stream. I could have skipped `istat` and `blkcat` entirely and just used this:

```
┌──(zham㉿kali)-[/tmp/pitter]
└─$ blkls -s suspicious.dd.sda1 | strings -e l | grep -E '^\}|pico'
}3986312f_3<_|Lm_111t5_3b{FTCocip
```

Same string, same `rev`, same flag.

---

## What Happened Internally (Timeline)

1. The challenge shipped a **raw ext3 partition image** named `suspicious.dd.sda1`. It was a standalone filesystem, not a full disk with a partition table, so I did not need `mmls` to find partitions — I could point TSK tools at the file directly.
2. `fsstat` told me the **block size was 1024 bytes**. That number is critical: every regular file lives in 1024-byte chunks, and any file smaller than 1024 bytes leaves room in its last block.
3. `fls` listed a single suspicious-looking file: `suspicious-file.txt` at inode 12. `icat` printed only "Nothing to see here! But you may want to look here -->" — a 55-byte decoy that even tells you the flag is not in the file, it is **next to** the file.
4. `istat 12` showed the file occupied exactly one block: **block 2049**.
5. `blkcat 2049` dumped the raw 1024-byte block. The first 55 bytes matched the file content; the remaining bytes were the **slack** of the block, and they were filled with the flag encoded as **UTF-16 little-endian** (each ASCII byte followed by `\x00`).
6. `strings -e l` (the `-e l` switch is the key) decoded those UTF-16LE runs and exposed the printable string `}3986312f_3<_|Lm_111t5_3b{FTCocip` — the flag written in reverse byte order.
7. Reversing the string with `rev` produced the real flag: `picoCTF{b3_5t111_mL|_<3_f2136893}`.

---

## Alternative Methods

### Method 1 — `blkls -s` for a faster workflow

`blkls -s` extracts only the slack bytes of every allocated file in one go, so I do not have to look up the inode, find the block, and dump that single block. After that it is the same `strings -e l | rev` finish:

```bash
blkls -s suspicious.dd.sda1 | strings -e l | rev
```

This is the closest match to the hint's "certain set of tools that now come with Ubuntu" — it is the literal slack-space tool, and it ships with `sleuthkit` on every modern Ubuntu / Kali.

### Method 2 — Mount the image as a real disk (the hint's other suggestion)

The first hint says to also try it "as an actual mounted disk". On a normal machine I can loop-mount the image read-only and inspect it the usual Linux way, then read slack with `icat` / `blkcat` separately. I could not actually `mount` inside the sandbox, but the commands would be:

```bash
mkdir -p /mnt/case
sudo mount -o ro,loop,noexec,nodev,noload suspicious.dd.sda1 /mnt/case
ls -la /mnt/case/
sudo cat /mnt/case/suspicious-file.txt
# confirm: still a decoy

# then drop back to TSK to read the slack
istat /mnt/case suspicious-file.txt   # not a real tool, illustrative
```

`mount` only gives you the **logical** view of the files (the 55 bytes the filesystem "knows" about), so it cannot expose the slack on its own. The hint is really telling you to do both views, then trust TSK to get the hidden bytes.

### Method 3 — `foremost` or `binwalk` to carve the block

If I had not known exactly which block to look at, carving tools would also find the flag while sweeping the image:

```bash
# binwalk: walk the whole image looking for embedded files / strings
binwalk -e suspicious.dd.sda1

# foremost: carve by header / footer signatures
foremost -i suspicious.dd.sda1 -o carved/
```

The flag is plain UTF-16LE, not wrapped in a file header, so these tools may not isolate it as cleanly as `strings -e l` does, but they do confirm the image has nothing else suspicious inside.

### Method 4 — Pure `dd` + `xxd` (no TSK)

If I had no TSK installed, I can reproduce `blkcat 2049` with a `dd` skip and inspect the hex with `xxd` / `od`:

```bash
# block 2049, each block is 1024 bytes -> skip 2049*1024 = 2097152 bytes
dd if=suspicious.dd.sda1 bs=1 skip=$((2049*1024)) count=1024 2>/dev/null \
  | od -c | head -20
```

You will see the same null-interleaved characters that `strings -e l` later decodes.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify the image as a raw ext3 filesystem |
| `fsstat` (Sleuth Kit) | Confirm the filesystem type and the **1024-byte block size** |
| `fls` (Sleuth Kit) | List files in the image and locate `suspicious-file.txt` at inode 12 |
| `icat` (Sleuth Kit) | Read the (decoy) contents of inode 12 |
| `istat` (Sleuth Kit) | Find the **direct block** number (2049) of inode 12 |
| `blkcat` (Sleuth Kit) | Dump the raw 1024 bytes of block 2049, including the slack tail |
| `blkls -s` (Sleuth Kit) | Optional shortcut: dump slack space of every file at once |
| `strings -e l` | Decode the **UTF-16 little-endian** runs hidden in the slack |
| `rev` | Reverse the extracted string so it reads `picoCTF{...}` |

---

## Key Takeaways

- **Slack space is real, and it is the first place to look** when a forensics challenge mentions "between" things, "extra bytes", or any of the standard "Pitter / Patter / Platter" type hints. The filesystem hides slack from `cat`, `cp`, and even from `mount`, but it is fully visible with raw block readers.
- **`blkls -s` is the dedicated slack-space tool** — it is faster than `istat` + `blkcat` when you do not yet know which file is interesting, because it streams the slack of every allocated file in one pass.
- **The `strings -e l` switch is the unlock** for this challenge. The flag is encoded as UTF-16LE (each ASCII byte followed by `\x00`), so plain `strings` ignores it. `-e l` tells `strings` to also recognize 16-bit little-endian runs.
- **Block size matters.** A 55-byte file in a 1024-byte block leaves almost a kilobyte of slack. Knowing the block size up front (from `fsstat`) tells you how much slack to expect and keeps you from over-searching.
- **Decoys are part of the puzzle.** `suspicious-file.txt` literally tells you to "look here" — and the only thing here that matters is the bytes right after the file's logical end. The challenge name itself (platters, the spinning disks in a hard drive) is a second nudge toward raw-block thinking.
- **Flag wordplay decode:** `b3_5t111_mL|_<3_f2136893` reads in leet as **"be still mL I love"** — `b3` = "be", `5t111` = "still" (5→s, 1→i, 1→l, 1→l), `mL` is the abbreviated "my love" tucked into the underscores, and `|<3` is text-art for **"I <3"** = "I love". The trailing `f2136893` is just a numeric tail to make every flag unique, same trick picoCTF uses on most of their challenge flags.
