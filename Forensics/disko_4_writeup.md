# DISKO 4 — picoCTF Writeup

**Challenge:** DISKO 4  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{d3l_d0n7_h1d3_w3ll_c2fcb641}`  
**Platform:** picoCTF 2026 (picoGym Exclusive)  
**Writeup by:** zham  

---

## Description

> Can you find the flag in this disk image? This time I deleted the file! Let see you get it now!
> Download the disk image here.

**Hint 1:** `How would you look for deleted files?`

---

## Background Knowledge (Read This First!)

### What is a Disk Image?

A disk image is a file that contains the exact byte-for-byte contents of a storage device — every file, every folder, every piece of metadata. Common formats:

- `.dd` — Raw disk dump (what we have here)
- `.iso` — CD/DVD image
- `.img` — Generic disk image
- `.vmdk` — Virtual machine disk

The challenge gives us `disko-4.dd.gz` — a gzipped raw FAT32 image.

### What is FAT32?

**FAT32** is one of the oldest and simplest filesystems still in use. It uses a **File Allocation Table** (the FAT) to track which clusters on disk belong to which file. Key ideas:

- Files are split into **clusters** of fixed size
- The FAT is a linked list — each entry points to the next cluster of the file
- The root directory and subdirectories are just regular files containing **directory entries**

### What Happens When You "Delete" a File on FAT32?

Here is the part most beginners get wrong: **deleting a file does not erase its data**. All the operating system does is:

1. Change the first byte of the directory entry from `0x00` to `0xE5` (a "tombstone" marker)
2. Mark all the file's clusters in the FAT as "free"

The actual file contents are still sitting on the disk, untouched, until something else overwrites those clusters. This is why deleted files are often **fully recoverable** — and exactly why this challenge is solvable.

### What is The Sleuth Kit?

**The Sleuth Kit (TSK)** is a collection of CLI forensics tools for analyzing disk images without mounting them. Two commands we will use heavily:

- `fls` — list files in a filesystem (including deleted ones)
- `icat` — output the raw contents of a file by its inode number

`fls` output marks deleted entries with a `*` after the type column. So when you see `r/r * 532021: dont-delete.gz`, the `*` is your signal that this file is deleted but still recoverable.

### What is `gunzip`?

`gunzip` decompresses files that were compressed with gzip (extension `.gz`). It removes the `.gz` extension when it finishes.

### What is `strings | grep`?

This is a two-step pipeline:

- `strings <file>` — pulls every readable ASCII run (at least 4 printable chars) out of any binary file
- `grep picoCTF` — keeps only the lines that mention the flag prefix

Piping (`|`) feeds the first command's output into the second. We will use this to confirm the flag after recovery.

---

## Solution — Step by Step

### Step 1 — Download and Extract the Disk Image

The challenge file is `disko-4.dd.gz`. I move into my downloads folder and decompress it:

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip disko-4.dd.gz
```

This leaves me with the raw image `disko-4.dd`.

### Step 2 — Identify the Filesystem

Before I touch any forensics tools, I want to know what kind of image this is:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file disko-4.dd
disko-4.dd: DOS/MBR boot sector, code offset 0x58+2, OEM-ID "mkfs.fat", Media descriptor 0xf8, sectors/track 32, heads 8, sectors 204800 (volumes > 32 MB), FAT (32 bit), sectors/FAT 1576, serial number 0x49838d0b, unlabeled
```

`FAT (32 bit)` — confirmed FAT32. Now I know which Sleuth Kit commands to expect output from.

### Step 3 — List ALL Files Including Deleted Ones

`fls` lists files. The `-r` flag recurses into every subdirectory. The deleted files show up with a `*` after the type:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -r disko-4.dd
d/d 4:	log
+ d/d 22:	private
+ d/d 24:	sysstat
+ d/d 28:	mysql
+ d/d 30:	inetsim
+ d/d 32:	installer
+ r/r 519123:	vmware-vmsvc-root.2.log
+ r/r 519125:	kern.log.4.gz
+ r/r 522627:	daemon.log
+ r/r * 522629:	messages
+ r/r * 532021:	dont-delete.gz
+ r/r 522632:	alternatives.log.2.gz
...
```

I grep for the `*` to see only the deleted entries — there are exactly two:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -r disko-4.dd | grep '*'
+ r/r * 522629:	messages
+ r/r * 532021:	dont-delete.gz
```

The second one — `dont-delete.gz` — is suspiciously named. Irony is usually a hint in picoCTF.

### Step 4 — Inspect the Deleted File Metadata

Let me peek at the deleted entry with `istat` to confirm it is really there:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ istat disko-4.dd 532021
Directory Entry: 532021
Not Allocated
File Attributes: File, Archive
Size: 87
Name: _ONT-D~1.GZ

Directory Entry Times:
Written:	2026-02-04 21:54:54 (UTC)
Accessed:	2026-02-04 00:00:00 (UTC)
Created:	2026-02-04 21:54:54 (UTC)

Sectors:
36434
```

`Not Allocated` confirms it is deleted. Size is 87 bytes — tiny. Name shows the classic FAT short-name format `_ONT-D~1.GZ` (the `~1` part is how FAT32 fits long names into 8.3 slots).

### Step 5 — Extract the Deleted File with `icat`

`icat` reads a file's raw contents by its inode/entry number and writes them to stdout. I redirect that into a new file:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat disko-4.dd 532021 > dont-delete.gz
```

Let me verify what I pulled out:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file dont-delete.gz
dont-delete.gz: gzip compressed data, was "dont-delete", last modified: Wed Feb  4 21:54:54 2026, from Unix, original size modulo 2^32 55
```

Still a gzip. The clusters are intact.

### Step 6 — Decompress and Read the Flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip dont-delete.gz

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat dont-delete
Here is your flag
picoCTF{d3l_d0n7_h1d3_w3ll_c2fcb641}
```

Done. The flag is `picoCTF{d3l_d0n7_h1d3_w3ll_c2fcb641}`.

I like to double-check with `strings | grep` so I do not miss it later:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings disko-4.dd | grep picoCTF
picoCTF{d3l_d0n7_h1d3_w3ll_c2fcb641}
```

The flag is also sitting in plaintext in the raw image, so a quick `strings` sweep would have found it too — but the Sleuth Kit route is the proper forensic way and tells us exactly which file it came from.

---

## Alternative Methods

### Method 1 — Mount the Image and Hunt Manually

If you prefer GUI/file-manager style recovery, mount the image read-only and look at `/log/`:

```bash
sudo mkdir /mnt/disko4
sudo mount -o ro,loop disko-4.dd /mnt/disko4
ls -la /mnt/disko4/log/
sudo umount /mnt/disko4
```

Deleted files will not show up here (they are gone from the directory listing), but the intact files are. Good for getting your bearings, not for finding the flag.

### Method 2 — `testdisk` / `photorec`

`testdisk` is the interactive big-brother tool from the same family as The Sleuth Kit. It can walk the FAT and let you undelete interactively. `photorec` ignores the filesystem entirely and carves files by their magic bytes.

```bash
sudo photorec /d ./recovered disko-4.dd
```

`photorec` will dump hundreds of recovered chunks into `recovered/`. Grep them for the flag:

```bash
grep -r picoCTF ./recovered
```

This works, but it is overkill — `fls` + `icat` already pointed us straight at the right file in two commands.

### Method 3 — `foremost` Carving

`foremost` is another carver, similar to `photorec`:

```bash
sudo foremost -t gzip -i disko-4.dd -o ./foremost-out
grep -r picoCTF ./foremost-out
```

Useful when you cannot install The Sleuth Kit, but again slower than `fls`/`icat` because it has to scan everything.

---

## What Happened Internally (Timeline)

Here is the chain of events that made this challenge solvable, in order:

1. **Filesystem creation** — someone ran `mkfs.fat` to format the volume as FAT32. The OEM ID `mkfs.fat` in `file` output is the giveaway.
2. **Files were written** — the Debian `/var/log/` directory tree was copied onto the volume. `messages` and `dont-delete.gz` were regular files at this point.
3. **The flag was placed** — `dont-delete.gz` was created containing `Here is your flag\npicoCTF{...}` and gzipped. Original size: 55 bytes.
4. **The file was "deleted"** — the operator (probably you, in another universe) ran `rm dont-delete.gz`. The FAT entries for the file's clusters were marked free, and the directory entry's first byte became `0xE5`. The data on disk was NOT touched.
5. **We extract with `icat`** — Sleuth Kit reads sector 36434 (per `istat` output), pulls the 87 raw bytes of the gzip stream straight off disk, and writes them to `dont-delete.gz`. No filesystem was mounted; nothing was modified.
6. **We decompress** — `gunzip` inflates the stream back to the 55-byte plaintext and the flag drops out.

The whole challenge hinges on step 4 → step 5: that "delete" never overwrote anything, so the bytes were still recoverable.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `gunzip` | Decompress the disk image and the recovered gzip file | Easy |
| `file` | Identify the filesystem type (FAT32) | Easy |
| `fls` (Sleuth Kit) | List every file in the image, marking deleted ones with `*` | Medium |
| `istat` (Sleuth Kit) | Show metadata for the deleted directory entry | Medium |
| `icat` (Sleuth Kit) | Extract the deleted file's raw bytes by entry number | Medium |
| `strings \| grep` | Confirm the flag is present in the raw image | Easy |
| `mount` (alt) | Optionally mount the image read-only for browsing | Easy |

To install The Sleuth Kit on Kali:

```bash
sudo apt update
sudo apt install sleuthkit
```

---

## Key Takeaways

- **Deleting a file does not erase it** — on FAT32 (and most filesystems), only the directory entry is marked free. The data lingers until overwritten. This is the foundation of file-recovery forensics.
- **`fls -r` is the first command to try** when a challenge says "I deleted the file." The `*` next to an entry is your green light.
- **`icat <image> <inode>` pulls any file by its metadata entry number** — works for both live and deleted files. It is the `dd` of the forensic world, but indexed.
- **`istat` gives you the cluster locations** if you ever need to extract data with raw `dd` instead of `icat`.
- **Always confirm with `strings | grep picoCTF`** as a sanity check that what you recovered is the real flag and not some other plaintext string.
- The filename `dont-delete.gz` is the joke: the attacker named the file "don't delete" — and then deleted it anyway. The flag wordplay **`d3l_d0n7_h1d3_w3ll` decodes to "del don't hide well"** — because deleting a FAT32 file leaves the data wide open for anyone with `fls` to find.
