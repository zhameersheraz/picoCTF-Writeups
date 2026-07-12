# Disk, disk, sleuth! II — picoCTF Writeup

**Challenge:** Disk, disk, sleuth! II  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 130  
**Flag:** `picoCTF{f0r3ns1c4t0r_n0v1c3_4bd721f2}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> All we know is the file with the flag is named down-at-the-bottom.txt...

The download is `dds2-alpine.flag.img.gz` — a gzipped raw disk image of an Alpine Linux filesystem. Somewhere inside that image, inside a file called `down-at-the-bottom.txt`, the flag is sitting.

**Hint given in the challenge:**

- `The sleuthkit has some great tools for this challenge as well.`
- `Sleuthkit docs here are so helpful: TSK Tool Overview`
- `This disk can also be booted with qemu!`

The hints scream "use The Sleuth Kit" — specifically `mmls`, `fls`, and `icat`. The QEMU hint is an alternative way to read the file, but it is heavy for a 5-second TSK solve, so I will get to it later as an "Alternative Methods" entry.

---

## Background Knowledge (Read This First!)

### What is a disk image?

A **disk image** is a raw byte-for-byte copy of a storage device. It is what you would get if you ran `dd if=/dev/sda of=disk.img` on a whole drive. The file we are working with — `dds2-alpine.flag.img` — is exactly that, 128 MB of raw bytes, no compression, no fancy container format. It starts with a 512-byte **MBR** (master boot record) that holds a **partition table** describing where each partition lives, and the rest of the file is the actual partition data.

The first thing a forensic tool does is parse that MBR to find the partitions. That is the job of `mmls`.

### What is The Sleuth Kit (TSK)?

**The Sleuth Kit** is a collection of small command-line utilities for analyzing disk images *without* mounting them. It understands ext2/3/4, FAT, NTFS, HFS+, and several other filesystems. Each tool does one job and they compose like Lego bricks:

| Tool | Job |
|------|-----|
| `mmls` | Parse the partition table of a raw image. Report where each partition starts and ends. |
| `fsstat` | Print filesystem-level statistics (block size, label, etc.) for a given partition. |
| `fls` | List files inside a filesystem (optionally recursive). Reports each file's **inode number**. |
| `istat` | Print metadata (size, timestamps, block pointers) for a single inode. |
| `icat` | Read and print the contents of a file given its inode number. |
| `ils` | List unallocated/deleted inodes (useful for recovering deleted files). |

Every TSK command that touches a partition takes `-o <offset>` to tell it where the filesystem begins. The offset is reported in 512-byte sectors by `mmls`.

### The "find a file in a disk image" recipe

This is the workflow you will reach for in 90% of disk-image forensics challenges:

1. `mmls image.img` — find the partition offset.
2. `fls -o <offset> -r image.img` — list every file, recursively. Note the inode of the one you want.
3. `icat -o <offset> image.img <inode>` — print that file's contents.

Three commands, no mounting, no root required, read-only by default. This challenge is exactly that recipe.

### Why not just `mount` the image?

You can — Linux's loop-mount lets you mount a raw image at a byte offset:

```
mount -o ro,loop,offset=1048576 dds2-alpine.flag.img /mnt/dds2
```

(`1048576` = `2048` sectors × `512` bytes/sector, the partition start.) That works, but:

- It needs `sudo`.
- It is read-write by default (you can clobber your evidence).
- Some challenge images use filesystems Linux's `mount` cannot read, and TSK handles them fine.

For a single 900-byte file, TSK is the lighter tool. I will show the mount path as an alternative at the end.

### A note on `gunzip` and the `.gz` wrapper

`dds2-alpine.flag.img.gz` is a regular gzip file. `gunzip` will restore it to a `.img` automatically. Use `gunzip -k` to keep the original `.gz` around (handy if you want to re-extract cleanly later). The two tools you might confuse it with are `tar` and `unzip` — wrong, this is a single-stream gzip, not a tarball and not a zip.

---

## Solution — Step by Step

### Step 1 — Make a working directory and decompress the image

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/disk-sleuth-ii && cd ~/disk-sleuth-ii
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ cp ~/Downloads/dds2-alpine.flag.img.gz .
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ ls -la
total 29M
-rw-r--r-- 1 zham zham 29M dds2-alpine.flag.img.gz
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ gunzip -k dds2-alpine.flag.img.gz
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ ls -la
total 157M
-rw-r--r-- 1 zham zham 128M dds2-alpine.flag.img
-rw-r--r-- 1 zham zham  29M dds2-alpine.flag.img.gz
```

128 MB is the un-gzipped size (262144 sectors × 512 bytes/sector). That number is a strong hint it is an MBR-style image with a 1 MiB alignment gap before the partition.

### Step 2 — Find the partition with `mmls`

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ mmls dds2-alpine.flag.img
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000262143   0000260096   Linux (0x83)
```

The Linux partition starts at sector **2048** and runs to 262143. The 2048 sectors before it (1 MiB) are the standard alignment gap modern partitioners leave for SSD/Advanced Format geometry — `mmls` labels it `Unallocated`.

The number I will be passing to every TSK command for the rest of this challenge: `-o 2048`.

### Step 3 — Find the file with `fls` (recursive, filtered)

A plain `fls` shows the root of the partition. `-r` recurses the whole tree, and I pipe through `grep` to jump to the file the challenge told us about:

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ fls -o 2048 dds2-alpine.flag.img
d/d 26417:    home
d/d 11:       lost+found
r/r 12:       .dockerenv
d/d 20321:    bin
d/d 4065:     boot
d/d 6097:     dev
d/d 2033:     etc
d/d 8129:     lib
d/d 14225:    media
d/d 16257:    mnt
d/d 18289:    opt
d/d 16258:    proc
d/d 18290:    root
d/d 16259:    run
d/d 18292:    sbin
d/d 12222:    srv
d/d 16260:    sys
d/d 18369:    tmp
d/d 12223:    usr
d/d 14229:    var
V/V 32513:    $OrphanFiles

┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ fls -o 2048 -r dds2-alpine.flag.img | grep -iE "down-at-the-bottom|\.txt"
+ r/r 18291:  down-at-the-bottom.txt
```

Found it. The `+` at the start of the line just means TSK already visited the parent directory in this recursive walk — purely informational. The number I care about is the inode: **18291**.

If you want to confirm the file is really at the root of the partition (not buried somewhere else), drop the `grep` and look at the path column:

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ fls -o 2048 -r dds2-alpine.flag.img | grep -B0 "18291"
+ r/r 18291:  down-at-the-bottom.txt
```

(In this challenge TSK's recursive output does not print directory paths, but the inode-and-filename output is enough to confirm it is a single, regular file at the filesystem root.)

### Step 4 — Read it with `icat`

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ icat -o 2048 dds2-alpine.flag.img 18291
   _     _     _     _     _     _     _     _     _     _     _     _     _  
  / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \ 
 ( p ) ( i ) ( c ) ( o ) ( C ) ( T ) ( F ) ( { ) ( f ) ( 0 ) ( r ) ( 3 ) ( n )
  \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/ 
   _     _     _     _     _     _     _     _     _     _     _     _     _  
  / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \ 
 ( s ) ( 1 ) ( c ) ( 4 ) ( t ) ( 0 ) ( r ) ( _ ) ( n ) ( 0 ) ( v ) ( 1 ) ( c )
  \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/ 
   _     _     _     _     _     _     _     _     _     _     _     _     _  
  / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \ 
 ( 3 ) ( _ ) ( 4 ) ( b ) ( d ) ( 7 ) ( 2 ) ( 1 ) ( f ) ( 2 ) ( } )
  \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/  
```

It is a flag, just rendered as ASCII-art balloons. Each `( x )` is one character of the flag. Read them in order:

```
p i c o C T F { f 0 r 3 n s 1 c 4 t 0 r _ n 0 v 1 c 3 _ 4 b d 7 2 1 f 2 }
```

So the flag is `picoCTF{f0r3ns1c4t0r_n0v1c3_4bd721f2}`.

### Step 5 — (Optional) Save the file and extract the flag string cleanly

If you want to keep the ASCII art around as evidence and also pull the flag out as a plain string, the same `grep` trick works:

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ icat -o 2048 dds2-alpine.flag.img 18291 > down-at-the-bottom.txt

┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ grep -oE "\( . \)" down-at-the-bottom.txt | tr -d '() ' | tr -d '\n' ; echo
picoCTF{f0r3ns1c4t0r_n0v1c3_4bd721f2}

┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ wc -c down-at-the-bottom.txt
900 down-at-the-bottom.txt
```

`grep -oE "\( . \)"` prints one `( x )` group per line, `tr -d '() '` strips the parens and spaces, and the final `tr -d '\n'` collapses everything into a single line. The trailing `; echo` adds the newline that `tr` removed so the prompt lands on its own line. 900 bytes — exactly the file's `istat` size.

For full paranoia, also pull the inode metadata with `istat`:

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ istat -o 2048 dds2-alpine.flag.img 18291
inode: 18291
Allocated
Group: 9
Generation Id: 2892579770
uid / gid: 0 / 0
mode: rrw-r--r--
size: 900
num of links: 1

Inode Times:
Accessed:       2021-02-16 18:21:20 (UTC)
File Modified:  2021-02-16 18:21:20 (UTC)
Inode Modified: 2021-02-16 18:21:20 (UTC)

Direct Blocks:
39993
```

Owned by `root:root`, mode `644`, single direct data block (#39993) holding 900 bytes. The Feb 2021 timestamp lines up with picoCTF 2021, so this file was added to the image when the challenge was created. No surprises, no hidden alternate data streams, no second block.

### The flag

```
picoCTF{f0r3ns1c4t0r_n0v1c3_4bd721f2}
```

---

## What Happened Internally (Timeline)

1. The challenge gave us a `.gz` archive containing a 128 MB raw disk image. `gunzip -k` decompressed it back to the bare `.img`.
2. `mmls` parsed the MBR at sector 0 of the image. The MBR is a 512-byte structure whose first 446 bytes are boot code and whose last 64 bytes hold a 4-entry partition table. TSK read those entries and reported a single Linux partition starting at sector 2048 (1 MiB into the disk — the standard 1 MiB alignment gap).
3. `fls -o 2048 -r` opened the ext filesystem at that offset, walked it from the root inode, and printed every directory entry. For each regular file it printed the **inode number** (e.g. `18291`) and the filename. The challenge's target file showed up exactly once.
4. `istat` (optional sanity check) showed inode 18291 is 900 bytes, owned by root, last touched 2021-02-16, with a single direct data block at block 39993.
5. `icat -o 2048 dds2-alpine.flag.img 18291` re-opened the filesystem, jumped to inode 18291, read its data block (#39993) and streamed those 900 bytes to stdout. The bytes turned out to be ASCII-art balloons, one character per balloon, spelling the flag.

The whole thing is three TSK commands plus a `gunzip` and you are done. The "hard" part is recognising that the answer to "I gave you a disk image, find this file" is "use TSK, and here is the exact three-command incantation".

---

## Alternative Methods

**Method 1 — Loop-mount the partition and `cat` the file**

If you have `sudo`, the entire TSK dance is replaceable with a single mount:

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ sudo mkdir -p /mnt/dds2
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ sudo mount -o ro,loop,offset=1048576 dds2-alpine.flag.img /mnt/dds2
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ cat /mnt/dds2/down-at-the-bottom.txt
   _     _     _     _     _     _     _     _     _     _     _     _     _  
  / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \   / \ 
 ( p ) ( i ) ( c ) ( o ) ( C ) ( T ) ( F ) ( { ) ( f ) ( 0 ) ( r ) ( 3 ) ( n )
  \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/   \_/ 
...
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ sudo umount /mnt/dds2
```

`offset=1048576` is the partition start in *bytes* (2048 sectors × 512 bytes). `-o ro` makes the mount read-only so you cannot accidentally write to the evidence image. Same answer, but it requires root and a kernel module that understands the filesystem (ext4 in this case).

**Method 2 — Boot the image in QEMU (Hint 3)**

Hint 3 literally tells you to do this. The image is bootable Alpine Linux, so you can run it as a VM:

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ qemu-system-x86_64 -m 512 -drive file=dds2-alpine.flag.img,format=raw,if=virtio
```

Log into the Alpine VM (default `root` / no password on most picoCTF images) and read the file with `cat /down-at-the-bottom.txt`. Same flag, same balloons. This is the most "realistic" workflow — the challenge author actually wanted you to pretend the disk was a real computer — but it is the slowest path. Save it for when TSK is not available (e.g. exotic filesystem, encrypted volume) or when you want to actually *interact* with the OS to look at logs, processes, or shell history.

**Method 3 — Use `tsk_recover` to dump everything**

If you do not know the filename, TSK can extract every file in one go:

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ tsk_recover -o 2048 dds2-alpine.flag.img ./recovered/
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ ls recovered/
bin  boot  dev  down-at-the-bottom.txt  etc  home  ...
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ cat recovered/down-at-the-bottom.txt
...same ASCII art...
```

`tsk_recover` walks the entire filesystem and writes every allocated file into a directory tree. Useful when the challenge does not tell you the filename. For this challenge it is overkill — the filename is given to you in the description — but it is the right tool when you are staring at an image with no idea what is inside.

**Method 4 — `binwalk` for carved files**

If the image were corrupted or the filesystem were unreadable, `binwalk` can sometimes carve files out by magic-byte signatures:

```
┌──(zham㉿kali)-[~/disk-sleuth-ii]
└─$ binwalk -e dds2-alpine.flag.img
```

It would find the ext superblock, the inode table, the MBR, and a pile of other structures. With a 900-byte text file in the middle of a 128 MB image, this approach is hopeless (the text has no magic bytes to grep for). Skip it unless you are working with a known-corrupted image.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `gunzip` | Decompress the `.gz` wrapper to recover the raw `.img` disk image. |
| `mmls` (TSK) | Parse the MBR partition table and report the start sector of the Linux partition. |
| `fls` (TSK) | List files inside the ext filesystem (recursive), with their inode numbers. |
| `icat` (TSK) | Read and print the contents of a file given its inode number. |
| `istat` (TSK) | Optional: print inode metadata (size, timestamps, block pointers) for confirmation. |
| `grep` / `tr` | Extract the flag string from the ASCII-art balloons. |
| `mount` (alt) | Optional: loop-mount the partition at byte offset 1048576 and `cat` the file. |
| `qemu-system-x86_64` (alt) | Optional: boot the image as a VM and read the file from inside Alpine. |
| `tsk_recover` (alt) | Optional: dump the entire filesystem to a directory in one go. |

The whole primary toolchain is `gunzip`, `mmls`, `fls`, `icat`. All four are on a stock Kali install (the TSK package is called `sleuthkit` in `apt`).

---

## Key Takeaways

- **`mmls` is the first command you should run on any disk image.** It tells you the partition layout in seconds. Without it you are poking around in raw bytes guessing where the filesystem starts. With it you have an exact offset to feed into the next tool.
- **The 3-step TSK recipe is the "find a file in a disk image" muscle memory:** `mmls` for the offset, `fls` for the inode, `icat` for the contents. Memorise it. It will serve you in every disk-image forensics challenge from picoCTF all the way up to DFRWS corpora.
- **Always pass the same partition offset to every TSK command.** The flag is "find the inode with `fls -o 2048 ...`, then read it with `icat -o 2048 ...`" — change the offset and you are reading a different region of the disk, almost certainly not the partition you want.
- **`fls -r` (recursive) + `grep` is faster than browsing.** Most CTF challenges tell you the filename in the description; let `grep` skip 2000 irrelevant files for you.
- **A single 900-byte text file with no compression, sitting in one direct data block, is the simplest possible file layout.** The challenge is not "can you find a needle in a haystack" — it is "do you know the right three tools". The forensics skill on display is toolchain knowledge, not recovery wizardry.
- **`gunzip -k` keeps the original archive.** Cheap habit. If you ever corrupt the extracted image (bad `dd`, wrong offset, whatever) you can re-extract from the clean `.gz` in one second.
- **Read the hints, take them literally.** "The sleuthkit has some great tools for this challenge" is the challenge author spelling out the solution. Same for "this disk can also be booted with qemu" — it is an alternative, not a red herring, but TSK is the intended path because it is the one that needs three commands and zero setup.

### Flag wordplay decode

```
picoCTF{f0r3ns1c4t0r_n0v1c3_4bd721f2}
         |     |     |    |          |
         f0r3n s1c4  t0r  n0v1c3     4bd721f2
```

Read in chunks it becomes **"f0r3ns1c4t0r n0v1c3 4bd721f2"** — a leet-speak rendering of **"forensicator novice"**, with a hex nonce (`4bd721f2`) at the end:

- `f0r3ns1c4t0r` → `forensicator` (a playful portmanteau of "forensics operator" — literally the toolchain user). `0` for `o`, `3` for `e`, `1` for `i`, `4` for `a`.
- `n0v1c3` → `novice` (`0` for `o`, `1` for `i`, `3` for `e`).
- `4bd721f2` → 8 hex chars (4 bytes) — a per-instance nonce, the standard "make the flag unique to this challenge generation" trick.

The challenge author is signing the flag with a wink: *"forensicator novice"*. The category is Forensics, the difficulty is Medium, and the entire solution boils down to knowing three TSK commands. Calling the solver a "forensicator novice" is the author's way of saying *"this is your first real intro to disk-image forensics, and you are now a forensicator — even if a novice one."*
