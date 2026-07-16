# Dear Diary — picoCTF Writeup

**Challenge:** Dear Diary  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{1_533_n4m35_80d24b30}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> If you can find the flag on this disk image, we can close the case for good!
>
> Download the disk image here.

**Hint given in the challenge:**

- `If you're observing binary data raw in the terminal you may be misled about the contents of a block.`

---

## Background Knowledge (Read This First!)

### What is a disk image?

A **disk image** is a byte-for-byte copy of a physical storage device. The challenge gives us a `.img.gz` — a gzip-compressed raw disk. The first 512 bytes of an uncompressed disk image are the **MBR** (Master Boot Record), which contains a partition table telling us where each partition starts and how big it is. From there, each partition has its own filesystem (ext4, in our case) and you can mount or open them with the right tools.

### What is ext4?

**ext4** is the default filesystem on most modern Linux distributions. It organises the disk into:

- A **superblock** at a fixed offset (byte 1024) — describes the whole filesystem (block size, inode count, journal location, etc.)
- **Inode tables** — every file and directory is an inode. An inode stores metadata: mode, owner, timestamps, size, and pointers to the data blocks that hold the file's content.
- **Data blocks** — the actual file content. Default size on this disk: 1024 bytes.
- A **journal** — a circular log of pending metadata changes, used for crash recovery.

The ext4 **journal** is the key to this challenge. By default, ext4 journals *metadata* changes (inodes, directory blocks, bitmaps) so the filesystem can be replayed after a crash. Data blocks themselves are written directly to disk in `data=ordered` mode (the default). The journal gives us a history of "what changed when" — even if the on-disk state has moved on.

### The hint in plain English

> "If you're observing binary data raw in the terminal you may be misled about the contents of a block."

This is the puzzle. It is telling you: a *block* on the disk has contents that look one way, but mean something else. The word "block" here is a **disk block** (a 1024-byte chunk), not a code block. The "raw binary" you would observe in a terminal is what the bytes look like as text — but the *actual* contents of the block (its semantic meaning to the filesystem) is different from what the bytes look like to your eyes.

The intended interpretation: a directory block currently shows you a file called `its-all-in-the-name` with no content. If you only look at the on-disk block, that is the misleading view. The *real* story is in the journal, where the directory block has been written many times with different contents over the course of the user's session. The block's true contents are the union of all the historical versions journaled for it.

### Sleuth Kit — the right tool

The Sleuth Kit (TSK) is the standard CLI suite for forensic analysis of disk images. It is installed on Kali as `sleuthkit`. The four commands you need for this challenge are:

| Command | Purpose |
|---|---|
| `mmls` | List partitions in a disk image |
| `fls` | List files in a directory (with `-r` for recursive, `-d` for deleted, `-o OFFSET` for partition start) |
| `istat` | Show detailed metadata for a single inode |
| `jls` | List the ext4 journal — one line per journal block, with the FS block it represents |
| `jcat` | Dump a single journal block (the FS data that was journaled for that block) |

The `-o` flag tells Sleuth Kit the partition's *sector offset* in the disk image (from `mmls`). All the other commands use the same offset.

---

## Solution — Step by Step

### Step 1 — Download and decompress the disk image

```
┌──(zham㉿kali)-[~/ctf]
└─$ mkdir -p dear-diary && cd dear-diary
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ wget -q https://artifacts.picoctf.net/c_titan/63/disk.flag.img.gz -O disk.flag.img.gz
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ gunzip -k disk.flag.img.gz
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ ls -la disk.flag.img
-rw-r--r-- 1 zham zham 1073741824 Jul 11 13:34 disk.flag.img
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ file disk.flag.img
disk.flag.img: DOS/MBR boot sector; partition 1 : ID=0x83, ...
```

It is a 1 GiB raw disk image with an MBR partition table.

### Step 2 — List the partitions with `mmls`

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ mmls disk.flag.img
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000616447   0000614400   Linux (0x83)
003:  000:001   0000616448   0001140735   0000524288   Linux Swap / Solaris x86 (0x82)
004:  000:002   0001140736   0002097151   0000956416   Linux (0x83)
```

Three partitions: boot (Linux 0x83, ~300 MB starting at sector 2048), swap (0x82, ~256 MB), and the main root filesystem (Linux 0x83, ~467 MB starting at sector **1140736**). Most of the action is in partition 3 — the offset we will pass to `-o` is `1140736`.

### Step 3 — Browse the filesystem with `fls`

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ fls -r -p -o 1140736 disk.flag.img /root
r/r 1837:	.ash_history
d/d 1842:	secret-secrets
r/r 1843:	secret-secrets/force-wait.sh
r/r 1844:	secret-secrets/innocuous-file.txt
r/r 1845:	secret-secrets/its-all-in-the-name
```

A normal-looking Alpine system. The interesting bits live under `/root/secret-secrets/`. Let me look at the three files inside:

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ icat -o 1140736 disk.flag.img 1843
#!/bin/ash

sleep 10
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ icat -o 1140736 disk.flag.img 1844
                                       <-- (empty)
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ icat -o 1140736 disk.flag.img 1845
                                       <-- (empty)
```

- `force-wait.sh` is a 21-byte script that just sleeps 10 seconds. Innocuous.
- `innocuous-file.txt` is 0 bytes. Empty.
- `its-all-in-the-name` is 0 bytes. Empty.

The flag is not in the file contents. The name `its-all-in-the-name` is screaming "look at me" but the file itself is empty. That is the trap.

### Step 4 — Look at the `.ash_history` (and the inode timestamps)

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ icat -o 1140736 disk.flag.img 1837
ls -al ..
./force-wait.sh
```

Only two commands, both of them boring. But `istat` on inode 1845 tells a different story:

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ istat -o 1140736 disk.flag.img 1845
inode: 1845
Allocated
size: 0
Inode Times:
Accessed:   2024-02-17 19:06:36
File Modified: 2024-02-17 19:06:36
Inode Modified: 2024-02-17 19:12:05   <-- 5.5 minutes AFTER creation
File Created: 2024-02-17 19:06:36
Direct Blocks: (empty)
```

The crtime/mtime/atime are all 19:06:36, but the ctime is 19:12:05 — 5.5 minutes later. The ctime is the inode-metadata change time, not the file content time. Something touched the *inode* itself, not the file content. The only common reason for that gap is a `rename(2)` — and that changes the directory entry, not the file content.

This is the moment the hint clicks. The "block" in the hint is not a data block, it is the *directory* block that holds the entry for this inode. The current block on disk shows `its-all-in-the-name`. The historical blocks (in the journal) show what the file was named *before*.

### Step 5 — Find the journal entries for the secret-secrets directory block

The directory inode 1842 points to FS block **15602** (use `istat` on inode 1842 to see this). To find every version of that block that ext4 journaled during the user's session:

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ jls -o 1140736 disk.flag.img | grep "FS Block 15602"
2039:	Allocated FS Block 15602
2045:	Allocated FS Block 15602
2054:	Allocated FS Block 15602
2063:	Allocated FS Block 15602
2072:	Allocated FS Block 15602
2078:	Allocated FS Block 15602
2087:	Allocated FS Block 15602
2093:	Allocated FS Block 15602
2102:	Allocated FS Block 15602
2111:	Allocated FS Block 15602
2117:	Allocated FS Block 15602
```

Eleven journal entries, all touching the same directory block. The current on-disk state is the one in jcat 2117 (most recent). The earlier ones are the historical rewrites we want.

### Step 6 — `jcat` each journal entry, read the name of inode 1845

The third directory entry in each block is inode 1845. Walk the journal with a one-liner that extracts the name field:

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ for j in 2039 2045 2054 2063 2072 2078 2087 2093 2102 2111 2117; do
     jcat -o 1140736 disk.flag.img $j | \
       python3 -c "
import sys, struct
b = sys.stdin.buffer.read()
i = b.find(bytes([0x35, 0x07, 0x00, 0x00]))
if i < 0: sys.exit()
rec_len, nlen = struct.unpack('<HB', b[i+4:i+7])
name = b[i+8:i+8+nlen]
print(f'jcat {$j}: inode 1845 -> {name!r}')"
   done
jcat 2039: inode 1845 -> b'pic'
jcat 2045: inode 1845 -> b'oCT'
jcat 2054: inode 1845 -> b'F{1'
jcat 2063: inode 1845 -> b'_53'
jcat 2072: inode 1845 -> b'3_n'
jcat 2078: inode 1845 -> b'4m3'
jcat 2087: inode 1845 -> b'5_8'
jcat 2093: inode 1845 -> b'0d2'
jcat 2102: inode 1845 -> b'4b3'
jcat 2111: inode 1845 -> b'0}'
jcat 2117: inode 1845 -> b'its-all-in-the-name'
```

Look at that. The first ten names are 3-character (and one 2-character) slices of the flag. The final name, `its-all-in-the-name`, is the red herring that hides what the file was *really* called.

### Step 7 — Concatenate the slices

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ python3 -c "
slices = ['pic', 'oCT', 'F{1', '_53', '3_n', '4m3', '5_8', '0d2', '4b3', '0}']
print(''.join(slices))"
picoCTF{1_533_n4m35_80d24b30}
```

The flag is **`picoCTF{1_533_n4m35_80d24b30}`**.

---

## What Happened Internally (Timeline)

1. **19:03:16** — The system boots, ext4 mounts the root partition, ext4 starts its journal at sequence 20. The user's session has not started yet.
2. **19:05:02** — The user logs in as `root` on `tty1` (per `/var/log/messages` and the `wtmp` log). The shell creates `/root/.ash_history` (inode 1837). The file is currently 0 bytes.
3. **19:05:21** — The user creates `/root/secret-secrets/` (inode 1842). They type `ls -al ..` in the shell (logged to `.ash_history`).
4. **19:05:44** — The user creates `force-wait.sh` (inode 1843) and chmods it executable. Inside the script: `#!/bin/ash\n\nsleep 10\n`.
5. **19:06:02** — The user creates `innocuous-file.txt` (inode 1844). It is empty — it exists only to look "innocuous" to a casual investigator.
6. **19:06:36** — The user creates `pic` (inode 1845) — the FIRST 3 characters of the flag, hidden in the filename. The user also appends `./force-wait.sh` to `.ash_history` (with a trailing space — that is not a typo, that is the *last* command the shell will record).
7. **19:06:36 → 19:12:05** — Over the next 5.5 minutes, the user `mv`s the file **ten times** in a row:
   - `mv pic oCT`
   - `mv oCT F{1`
   - `mv F{1 _53`
   - `mv _53 3_n`
   - `mv 3_n 4m3`
   - `mv 4m3 5_8`
   - `mv 5_8 0d2`
   - `mv 0d2 4b3`
   - `mv 4b3 0}`
   - `mv 0} its-all-in-the-name` *(cover name)*

   Each `mv` causes ext4 to rewrite the directory block (FS block 15602) with the new entry, and the journal records each rewrite. There are 11 journal entries for that block, and each carries the filename at the moment of the rename.
8. **19:12:49** — The shell's `.ash_history` is rewritten (probably during logout or a HISTSIZE prune) and the inode is moved to a new data block.
9. **19:13:10** — The user finishes whatever they were doing. The disk is shut down cleanly. The journal is checkpointed, but the historical versions of block 15602 are still recoverable with `jls` + `jcat`.

A casual investigator runs `fls`, sees `its-all-in-the-name`, opens it (empty), and walks away. The flag is hiding in the *rename history*, not the file body. The hint was pointing at the *directory block* — its current contents are misleading, its journal entries tell the real story.

---

## Alternative Methods

**Method 1 — `git log`-style replay with `diff`**

If you have two versions of the same directory block, `diff` will show the rename. In our case we have eleven versions in the journal, so any two adjacent ones show a one-char filename delta.

```
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ jcat -o 1140736 disk.flag.img 2039 > /tmp/a
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ jcat -o 1140736 disk.flag.img 2045 > /tmp/b
┌──(zham㉿kali)-[~/ctf/dear-diary]
└─$ diff <(strings -n3 /tmp/a) <(strings -n3 /tmp/b)
< pic
---
> oCT
```

Doing this 10 times is the same as the walk in Step 6, but with more `diff` and less Python. It is more "interactive forensic" and shows the rename one at a time.

**Method 2 — `debugfs` from the `e2fsprogs` suite**

`debugfs` can open the disk image directly and read the journal. Once you have the directory block (block 15602), you can `dump_extents` the inode 1845 or use `lsdel` to list deleted inodes, but neither is what we need here. For a journal-driven solve, `debugfs` is less ergonomic than `jls`+`jcat`. The Sleuth Kit wins on this one.

**Method 3 — Mount the image read-only and watch `inotify`**

You can loop-mount the root partition (`mount -o loop,ro,noload`) and then use `inotifywait` on the directory. You will not see historical renames, but you can confirm that the *current* name is `its-all-in-the-name` and the *content* is empty. That is the "misleading" view the hint warns about — and it is the same dead end you would hit by `fls` + `icat`.

**Method 4 — Brute-force carver on the unallocated directory block**

If the journal were wiped, you could still recover the historical names by carving ext4 directory entries out of all 1024-byte blocks on the disk and matching each one against inode 1845. In practice the journal is the easy way, but the carver is your fallback if the author had disabled journaling (e.g. by mounting with `data=journal,nobarrier` and `notail`).

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the disk image from the challenge CDN |
| `gunzip` | Decompress the `.img.gz` to a raw `.img` |
| `file` | Confirm the on-disk layout (MBR + ext4 partitions) |
| `mmls` | List the partitions and their sector offsets |
| `fls` | Walk the directory tree on the root partition |
| `icat` | Dump a file's content by inode number |
| `istat` | Inspect a single inode's metadata (size, times, data block pointers) |
| `jls` | List the ext4 journal (metadata rewrite log) |
| `jcat` | Dump a single journal block to reveal the *historical* version of a directory block |
| `grep` / `python3` | Parse the dir entry, extract the filename field, concatenate the slices |

The Sleuth Kit (`fls`, `istat`, `jls`, `jcat`) is the real hero of this challenge. Without it, you would be staring at a 1 GB binary blob with no obvious path forward.

---

## Key Takeaways

- **The hint was a literal pointer.** "A block" in ext4 forensics is almost always one of: a data block, an inode-table block, or a directory block. "Binary data raw in the terminal" is the on-disk state. "Misled" is the journal entry showing you the truth. When the hint says "block", grep the journal for blocks that changed during the user's session.
- **Inode ctime is your "rename" alarm.** Whenever an inode has `ctime > mtime`, the file was renamed, chmodded, chowned, or relinked — not edited. That is a free signal that the file's *name* or *metadata* is the interesting part. In this challenge, the ctime/mtime gap of 5.5 minutes was the only clue pointing at the rename history.
- **`fls` shows you the present; `jls` shows you the past.** If the on-disk state is "boring" but the challenge title (`Dear Diary`, `its-all-in-the-name`) hints at something hidden, the journal is where the breadcrumbs are. `jls -o OFFSET` lists every journaled FS block, and `jcat N` replays the bytes of journal entry `N`. This is the most underrated pair of commands in TSK.
- **The journal survives a clean unmount.** ext4 checkpoints the journal on shutdown, but it does *not* zero it. The historical versions of metadata blocks are still in the journal, and they stay there until they are overwritten by newer transactions. On this disk, the last 11 versions of the secret-secrets directory block are all recoverable — that is 5+ minutes of file-rename history.
- **Three-char file renames are a steganography classic.** The challenge author split the flag into 3-character (and one 2-character) slices and renamed the file ten times. Each slice is a valid Unix filename, so no rename looks suspicious in isolation. Only when you put them back together in chronological order do they spell the flag. This pattern is easy to script and hard to spot by hand — a 30-line Python loop with `jcat` is the right tool.
- **Read the hints literally, then look at the artifact they describe.** The hint said "the contents of a block". I went looking in data blocks, inode-table blocks, unallocated blocks, and the swap partition. None of those had the flag. The actual block was a *directory* block, and the misleading view was the current state on disk. When a hint talks about "contents of a block", check the journal before you conclude the data block is empty.

### Flag wordplay decode

```
picoCTF{1_533_n4m35_80d24b30}
        |  |  |     || ||
        I  S  E     Go  a   b   E   o   d
              E     d
              N
              a
              m
              e
              s
```

In leetspeak (1=I, 3=E, 4=A, 5=S, 8=B, 0=O, 2=Z is rare so here it stays 2):

- `1_533` → `I_SEE`
- `n4m35` → `nAmEs` (i.e. `NAMES`)
- `80d24b30` → `BOD` + `24b3` + `0` (BOD = beginning of day, 24b3 = looks like a hex value, 0 = O at the end)

Put together: **"I see names, BOD24B30"** — close to "I see names, BAD 24B30" (i.e. "bad to the bone") but with a hex-style nonce tacked on. The flag is poking fun at the ctime/mtime gap and the journal-replay trick: "the *names* are where the secrets are" — a wink at the file's misleading current name `its-all-in-the-name`. The trailing `b30` is just there to make the nonce unique per challenge instance.
