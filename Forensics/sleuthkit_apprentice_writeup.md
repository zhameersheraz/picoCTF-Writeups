# Sleuthkit Apprentice — picoCTF Writeup

**Challenge:** Sleuthkit Apprentice  
**Category:** Forensics  
**Difficulty:** Medium  
**Flag:** `picoCTF{by73_5urf3r_2f22df38}`  
**Platform:** picoCTF (picoGym Exclusive)  
**Writeup by:** zham  

---

## Description

> Download this disk image and find the flag.
>
> Note: if you are using the webshell, download and extract the disk image into `/tmp` not your home directory.

**Hints given in the challenge:**

- No formal hint text — the category name (`Sleuthkit`) and the previous challenge (`Sleuthkit Intro`) hint at the toolset to use
- The `/tmp` note tells you the image is large and you should not fill your home directory

---

## Background Knowledge (Read This First!)

### Recap: The Sleuth Kit (TSK)

TSK is a CLI toolkit for analyzing disk images without mounting them. After learning `mmls` in the **Sleuthkit Intro** challenge, this one steps up to file-level forensics using the rest of the kit.

| Tool | Purpose |
|------|---------|
| `mmls` | Show the partition table |
| `fsstat` | Show filesystem info (type, label, last mount) |
| `fls` | List file and directory names (with `-r` for recursive) |
| `icat` | Print the contents of a file by its **inode number** |
| `istat` | Show inode metadata (timestamps, size, block pointers) |

### What is an inode?

In an ext-family filesystem, every file and directory has an **inode** — a metadata record that stores:

- File type (regular, directory, symlink)
- Size in bytes
- Timestamps (access, modify, change)
- Pointers to the data blocks on disk

When you use `fls` to list files, each entry looks like `r/r 2371: flag.uni.txt`. That `2371` is the inode number. Use `icat` with that number to extract the file contents without mounting the filesystem.

### What does `fls -r` show?

`fls -r` recursively walks every directory in the partition and prints every file/directory entry. Output format:

```
type_code  inode_number:  name
```

Common type codes:

| Code | Meaning |
|------|---------|
| `r/r` | regular file |
| `d/d` | directory |
| `l/l` | symbolic link |
| `r/r * N(realloc)` | reallocated (deleted/replaced) file content |

### What is `/root`?

In a Linux disk image, the superuser's home directory is `/root`. Files placed there (especially `flag.txt`) are a common forensics target. The fact that this disk has a separate `/mnt/boot` mount and a swap partition is also realistic — it looks like a real Alpine Linux install.

---

## Solution — Step by Step

### Step 1 — Extract the disk image to `/tmp`

The image is ~300 MB uncompressed. Always extract large images under `/tmp` to avoid filling your home directory:

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /tmp/sleuthkit2 && cd /tmp/sleuthkit2
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ cp ~/Downloads/disk.flag.img.gz .
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ gunzip disk.flag.img.gz
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ ls -la
-rw-r--r-- 1 root root 314572800 Jul  3 00:35 disk.flag.img
```

### Step 2 — Map out the partitions with `mmls`

```
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ mmls disk.flag.img
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002047   Unallocated
002:  000:000   0000002048   0000206847   0000204800   Linux (0x83)
003:  000:001   0000206848   0000360447   0000153600   Linux Swap / Solaris x86 (0x82)
004:  000:002   0000360448   0000614399   0000253952   Linux (0x83)
```

Three partitions:

| # | Offset (sectors) | Type | Likely role |
|---|------------------|------|-------------|
| 1 | 2048 | ext4 (Linux) | boot |
| 2 | 206848 | swap | swap |
| 3 | 360448 | ext4 (Linux) | root |

### Step 3 — Identify which Linux partition is `/`

Use `fsstat` on each Linux partition and read the `Last mounted on:` line:

```
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ fsstat -o 2048 disk.flag.img | grep "Last mounted on"
Last mounted on: /mnt/boot

┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ fsstat -o 360448 disk.flag.img | grep "Last mounted on"
Last mounted on: /
```

The flag is almost certainly under `/root`, so partition 3 (offset **360448**) is the target.

### Step 4 — Walk the root filesystem with `fls -r`

```
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ fls -r -o 360448 disk.flag.img | grep -iE "flag|root"
+ r/r * 2082(realloc):  flag.txt
d/d 1995:               root
+ r/r 2371:             flag.uni.txt
```

Two interesting files in `/root`:

| Inode | Name | Status |
|-------|------|--------|
| `2082` (realloc) | `flag.txt` | deleted/replaced — contents are junk numeric data |
| `2371` | `flag.uni.txt` | live file — this is the real flag |

The `*` and `(realloc)` on inode 2082 is TSK's way of saying "this file used to exist but its blocks have been overwritten by another file." Classic delete-vs-overwrite evidence.

### Step 5 — Extract the flag with `icat`

The file is UTF-16 little-endian (every ASCII char has a `\0` after it). `icat` prints the decoded text:

```
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ icat -o 360448 disk.flag.img 2371 | tr -d '\0'
picoCTF{by73_5urf3r_2f22df38}
```

Verify by dumping the raw byte integers — the character at offset 22 must be `55` (ASCII `7`), not `116` (ASCII `t`):

```
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ icat -o 360448 disk.flag.img 2371 | od -An -tu1 | head -5
   0 112   0 105   0  99   0 111   0  67   0  84   0  70   0 123
   0  98   0 121   0  55   0  51   0  95   0  53   0 117   0 114
   0 102   0  51   0 114   0  95   0  50   0 102   0  50   0  50
   0 100   0 102   0  51   0  56   0 125   0  10
```

That `55` is the digit `7`, not the letter `t`. Reading it wrong (as `byt3` instead of `by73`) is a classic mistake — the terminal silently swallows the `\0` bytes and your eyes fill in the most "obvious" letter. Always trust the byte dump for `.uni` files.

Got the flag. For completeness, the reallocated `flag.txt` is just numeric noise:

```
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ icat -o 360448 disk.flag.img 2082
            3.449677            13.056403
```

(no flag there — just leftover floating-point data from another file)

---

## What Happened Internally (Timeline)

1. The challenge gave a 300 MB disk image of a Linux system with a separate boot partition, a swap partition, and a root partition.
2. `mmls` showed three partitions. `fsstat` told me which one was `/` (partition 3, offset 360448) by reading the ext4 superblock's `last mounted on` field.
3. `fls -r -o 360448` walked the entire root filesystem. Most of the noise (`/etc`, `/usr`, `/bin`) was a normal Alpine install.
4. Under `/root` I saw two flag-named files: `flag.txt` (reallocated — deleted) and `flag.uni.txt` (live). The `uni` suffix hints the contents are unicode/UTF-16.
5. `icat -o 360448 disk.flag.img 2371` read the live file's data blocks and printed the flag. No mounting, no filesystem driver needed in the host OS.

---

## Alternative Methods

**Method 1 — Mount the root partition directly**

If your kernel supports ext4 you can loop-mount and read the flag with `cat`:

```bash
sudo mkdir /mnt/root
sudo mount -o ro,offset=$((360448 * 512)) disk.flag.img /mnt/root
cat /mnt/root/root/flag.uni.txt
sudo umount /mnt/root
```

**Method 2 — Plain `strings` + `grep`**

`strings` does not need to understand the filesystem — it just scans the raw image for printable runs. The flag is plaintext, so this finds it too:

```bash
strings disk.flag.img | grep picoCTF
```

This is faster but messier — you will get hundreds of matches from the linux distro's text. TSK is cleaner because it knows where file boundaries are.

**Method 3 — Automate with a tiny script**

```bash
#!/bin/bash
# find_flag.sh — pass the image path
IMG="$1"
OFFSET=$(fsstat -o 2048 "$IMG" >/dev/null 2>&1; echo 2048)
ROOT_OFFSET=$(mmls "$IMG" | awk '/Linux \(0x83\)/ && NR>2 {print $3}' | tail -1)
fls -r -o "$ROOT_OFFSET" "$IMG" | grep -i flag
```

Run it:

```
┌──(zham㉿kali)-[/tmp/sleuthkit2]
└─$ bash find_flag.sh disk.flag.img
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `gunzip` | Decompress the `.gz` disk image |
| `mmls` (Sleuth Kit) | Show partition layout and offsets |
| `fsstat` (Sleuth Kit) | Identify which partition is `/` |
| `fls` (Sleuth Kit) | Recursively list files on the root partition |
| `icat` (Sleuth Kit) | Extract file contents by inode number |
| `grep` | Filter the `fls` listing for flag-named files |

---

## Key Takeaways

- The Sleuth Kit workflow is **mmls → fsstat → fls → icat**. That is the same pattern you will use on most disk forensics challenges.
- `fls -r` is your best friend — one recursive listing beats guessing folders. Pipe it to `grep -i flag` and you cut straight to the target.
- Inode numbers are stable. `fls` gives you the inode, `icat` reads the file. You never need to mount the image.
- A `(realloc)` flag in `fls` output means the file was deleted and its blocks reused. The data you read from those blocks belongs to whoever wrote there next.
- Always use `/tmp` for disk images — `/tmp` is often a tmpfs and avoids burning real disk space.
- Flag wordplay: `by73` → **"byte"** (7→t, 3→e) and `5urf3r` → **"surfer"** (5→s, 3→e). The flag spells **"byte surfer"** — exactly what you are doing when you carve through raw bytes of a disk image with the Sleuth Kit.
- **Trap:** if your terminal renders the UTF-16 null bytes invisibly, the flag can look like `byt3` (because your brain substitutes the most "obvious" letter for the digit `7`). Submitting that variant fails. Always pipe `icat` through `od` or `tr -d '\0'` before pasting the flag.
