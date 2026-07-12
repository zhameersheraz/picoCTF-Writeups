# Disk, disk, sleuth! — picoCTF Writeup

**Challenge:** Disk, disk, sleuth!  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 110  
**Flag:** `picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Use srch_strings from the sleuthkit and some terminal-fu to find a flag in this disk image.

The file is `dds1-alpine.flag.img.gz` — a gzipped raw disk image of an Alpine Linux installation. The flag is *somewhere* in the image, but the challenge does not say where — that is the puzzle. The title and the description both point at one specific tool.

**Hint given in the challenge:**

- `Have you ever used file to determine what a file was?`
- `Relevant terminal-fu in picoGym: https://play.picoctf.org/practice/challenge/85`
- `Mastering this terminal-fu would enable you to find the flag in a single command: https://play.picoctf.org/practice/challenge/48`
- `Using your own computer, you could use qemu to boot from this disk!`

Hint 3 is the giveaway: "find the flag in a single command". That command is `srch_strings | grep`. The other hints are setup (`file` to identify the type) and an alternative path (boot the disk in QEMU).

---

## Background Knowledge (Read This First!)

### What is a disk image?

A **disk image** is a byte-for-byte copy of a storage device. The one in this challenge is `dds1-alpine.flag.img` — 128 MB of raw bytes from an Alpine Linux install. It starts with a 512-byte **MBR** (master boot record) that holds a **partition table** describing where each partition lives, and the rest of the file is the actual partition data.

If you have never worked with disk images before, the mental model is: "imagine the entire contents of `/dev/sda` flattened into a single file". That is exactly what this is.

### What is `strings`?

The classic Unix `strings` command (and its sibling `strings -a`, which scans the whole file) finds runs of printable ASCII characters in a binary blob and prints them out. It is the canonical "I do not know what is in this file, show me the readable parts" tool.

```
$ strings somefile.bin | head
```

For example, given a disk image, `strings` will surface every file path, every error message, every config value, every URL, and every English word the operating system and applications have written to disk. That includes — usually — flags, because CTF authors are not encoding the flag in raw binary, they are just stuffing it somewhere that does not look like a file at first glance.

`strings` is the single most useful forensics tool you will ever meet, and it is preinstalled on every Linux distribution.

### What is `srch_strings`?

`srch_strings` is **The Sleuth Kit**'s version of `strings`. It is built on top of `libewf` and `libtsk`, so it understands disk-image structures (partition tables, filesystems, slack space, unallocated regions) and can be told *where* to look.

The most useful flags:

| Flag | What it does |
|------|--------------|
| `-a` | Scan the entire file (default for plain files, off for disk images). |
| `-o <sector>` | Restrict the search to the ext/NTFS/FAT filesystem at the given 512-byte sector offset. |
| `-s <sector> <length>` | Scan a specific range of sectors. |
| `-t d` | Print the decimal byte offset of each hit. |
| `-t x` | Print the hexadecimal byte offset. |
| `-e l` / `-e b` | 16-bit little-endian / big-endian string encoding. |
| `-n <len>` | Minimum run length (default 4). |

For a disk image, the standard incantation is `srch_strings -o <partition_offset> <image>` — and the challenge's hint 3 is telling you that the offset is *optional*. `srch_strings` will also happily scan the entire image (including the partition table, slack space, deleted files, everything) without `-o`. That is the "single command" path the hint is pointing at.

### Why is `srch_strings` better than plain `strings` on a disk image?

Plain `strings` is fine. It will work. But it has a few rough edges on raw images:

- It will report *every* ASCII string, including the ones in the partition table, the boot sector, the filesystem superblock, and the kernel's compiled-in symbol names. Most of those are noise.
- It does not understand filesystems, so it cannot tell you "this string is in `/etc/passwd`" vs "this string is in the MBR" — it just gives you offsets.

`srch_strings`, on the other hand, knows about partition tables. When you pass `-o 2048` (or whatever the filesystem offset is), it scopes the search to that partition and skips the MBR / boot sector noise. When you do *not* pass `-o`, it scans everything, which is what you want for a CTF where you do not know where the flag is.

Either way works. The "single command" is the `-o`-less version: it is faster to type, and on a 128 MB image the extra noise is not enough to be a problem.

### Where the flag actually is

After solving, the flag lives in `/boot/syslinux/syslinux.cfg` — the **syslinux bootloader configuration file**. The relevant lines are:

```
DEFAULT linux
  SAY Now booting the kernel from SYSLINUX...
  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
 LABEL linux
  KERNEL /boot/vmlinuz-virt
  APPEND ro root=/dev/sda1 rootfstype=ext3 initrd=/boot/initramfs-virt
```

The `SAY` directive is an obscure syslinux feature that uses the BIOS's text-to-speech hardware (or the firmware's beep-pattern encoder) to "say" the text aloud during boot. Real syslinux configs use it for accessibility — voice prompts for visually-impaired users. The challenge author exploited it as a hiding spot because:

- It is in `/boot/`, which is rarely inspected by forensics tools that focus on `/etc`, `/home`, and `/var`.
- The `SAY` prefix is misleading — it looks like a Bash command (`echo "..."` written in shorthand) but is actually a syslinux directive.
- The file is small (a few hundred bytes), so a casual `find /boot -type f` lands on it but most people never read it.

This is the kind of place flags live in the real world too: a config file you would not normally look at, hiding a string that does not belong there.

---

## Solution — Step by Step

### Step 1 — Decompress the image

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/disk-sleuth && cd ~/disk-sleuth
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ cp ~/Downloads/dds1-alpine.flag.img.gz .
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ gunzip -k dds1-alpine.flag.img.gz
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ ls -la dds1-alpine.flag.img
-rw-r--r-- 1 zham zham 134217728 Jul 13 00:16 dds1-alpine.flag.img
```

128 MB of raw disk image, ready to inspect.

### Step 2 — Confirm what it is (hint 1)

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ file dds1-alpine.flag.img
dds1-alpine.flag.img: DOS/MBR boot sector; partition 1 : ID=0x83, start-CHS (0x3ff,254,63), end-CHS (0x3ff,254,63), startsector 2048, 260096 sectors
```

`file` immediately tells us it is a DOS/MBR-style disk image with a single Linux partition starting at sector 2048. That number is the offset every TSK command wants.

### Step 3 — The single-command solve (hint 3)

The hint says "you can find the flag in a single command". That command is `srch_strings | grep pico`. We do not even need the partition offset, because the flag is in an allocated ext file and `srch_strings` will find it scanning the whole image:

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ srch_strings dds1-alpine.flag.img | grep -i pico
  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
```

One pipeline, one hit, one flag. That is the entire challenge.

(The leading spaces and the `SAY` prefix are noise from the bootloader config — more on that in the "What Happened Internally" section below.)

### Step 4 — Cross-check with plain `strings`

`srch_strings` is just `strings` with disk-image awareness. Plain `strings` will find the same flag, with the same context:

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ strings dds1-alpine.flag.img | grep -i picoctf
  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
```

Same answer, different tool. Use whichever you have.

### Step 5 — (Optional) Scope to just the filesystem

If we want to skip the MBR / boot-sector noise and search only the ext partition, pass `-o 2048`:

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ srch_strings -o 2048 dds1-alpine.flag.img | grep -i pico
  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
```

Identical result. The flag is inside the filesystem, so scoping to it does not change anything in this case — but for a 100 GB image with hundreds of thousands of strings, scoping the search is the difference between a 2-second pipeline and a 2-minute one.

### Step 6 — (Optional) Identify the file containing the flag

For a CTF solve this is not needed, but it is satisfying to close the loop and see which file the flag actually lives in. The flag is in `/boot/syslinux/syslinux.cfg`, the syslinux bootloader config:

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ fls -r -o 2048 dds1-alpine.flag.img | grep syslinux.cfg
+ r/r 8138:   syslinux.cfg

┌──(zham㉿kali)-[~/disk-sleuth]
└─$ icat -o 2048 dds1-alpine.flag.img 8138
DEFAULT linux
  SAY Now booting the kernel from SYSLINUX...
  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
 LABEL linux
  KERNEL /boot/vmlinuz-virt
  APPEND ro root=/dev/sda1 rootfstype=ext3 initrd=/boot/initramfs-virt
```

The `SAY` directive is syslinux's text-to-speech command. The author snuck the flag into a file you would normally never read end-to-end, in a section (`SAY`) that is essentially documentation.

### The flag

```
picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
```

---

## What Happened Internally (Timeline)

1. We started with a gzipped disk image. `gunzip -k` gave us the raw 128 MB file. `file` confirmed it was a DOS/MBR disk image with a single Linux partition starting at sector 2048.
2. We ran `srch_strings dds1-alpine.flag.img | grep -i pico`. `srch_strings` scanned the entire image (262144 sectors of 512 bytes each), reading every byte and looking for runs of printable ASCII ≥ 4 characters long. Every time it found one, it printed it to stdout.
3. Most of the output was noise — kernel symbol names (`pirq_pico_get`, `pico_router_probe` — the `pico` here is the PicOS router, not a CTF flag), build strings, library paths, ext4 superblock constants. The `| grep -i pico` filter cut all of that to a single line: `SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}`.
4. The `SAY` prefix is syslinux's text-to-speech directive. It tells the bootloader to "say" the line aloud (through the BIOS's PC speaker beep pattern) during boot. The challenge author used it as a steganography trick: hide the flag in a config file that lives in `/boot/`, under a directive name that makes the line look like a shell command.
5. We confirmed the file with `fls` (it was inode 8138, named `syslinux.cfg` in `/boot/syslinux/`) and read it back with `icat` to see the full context.

The "hard" part of the challenge is recognising that `srch_strings` is the right tool. The actual execution is one pipeline.

---

## Alternative Methods

**Method 1 — Plain `strings` without sleuthkit**

`sleuthkit` is overkill if you just want to find a flag. Plain `strings` works identically:

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ strings dds1-alpine.flag.img | grep -i picoctf
  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
```

`strings` is on every Linux install. It will scan the same bytes, just without the partition-awareness. For this challenge the difference is invisible. For a real forensic exam, `srch_strings -o <offset>` is strictly better because it lets you scope the search.

**Method 2 — `grep -a` (no strings command at all)**

`grep -a` treats binary input as text and applies the regex:

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ grep -aoE "picoCTF\{[^}]+\}" dds1-alpine.flag.img
picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
```

`grep -a` (treat binary as text), `grep -o` (print only the matching part, not the whole line), `grep -E` (extended regex), and the pattern `picoCTF\{[^}]+\}` (the flag prefix, an opening brace, anything-not-brace, and the closing brace). This is the shortest possible solve — one command, no dependencies beyond GNU grep, and the output is the flag with no other noise.

**Method 3 — `binwalk` + carve + grep**

If you wanted to be thorough and find the flag *and* identify which file it lives in, binwalk is the right tool:

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ binwalk dds1-alpine.flag.img | head -5
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             DOS/MBR boot sector
512           0x200           Linux EXT filesystem
...
```

`binwalk` would not find the flag itself (it is not a different file format — it is text inside an existing file), but it confirms the partition layout. Then `strings | grep` does the actual flag-finding. Binwalk is the right tool for "what's hidden in this image", `strings` is the right tool for "find the text I am looking for".

**Method 4 — Boot in QEMU (hint 4)**

Hint 4 lets you boot the image in QEMU. The flag will be visible on screen during boot because syslinux's `SAY` directive shows the text on the console as it speaks it:

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ qemu-system-x86_64 -m 512 -drive file=dds1-alpine.flag.img,format=raw,if=virtio
```

Watch the boot screen. The text `Now booting the kernel from SYSLINUX...` will appear, followed by `picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}`. The flag literally plays aloud. (Well, "aloud" in the BIOS-beep sense — modern QEMU does not actually emit audio by default, but the text is printed on the bootloader screen.) This is a fun "see the flag in its natural habitat" solve, but it is the slow path.

**Method 5 — Mount the filesystem and read the config**

The boring, reliable path: loop-mount the partition, read the file, unmount.

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ sudo mount -o ro,loop,offset=1048576 dds1-alpine.flag.img /mnt/dds1
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ cat /mnt/dds1/boot/syslinux/syslinux.cfg
DEFAULT linux
  SAY Now booting the kernel from SYSLINUX...
  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
 LABEL linux
  KERNEL /boot/vmlinuz-virt
  APPEND ro root=/dev/sda1 rootfstype=ext3 initrd=/boot/initramfs-virt
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ sudo umount /mnt/dds1
```

`offset=1048576` is the partition start in *bytes* (2048 sectors × 512 bytes). `-o ro` makes the mount read-only. Works on any Linux box. Slowest of the bunch because it needs root and a filesystem-supporting kernel, but always available.

**Method 6 — `tsk_recover` to dump the whole filesystem, then grep**

```
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ tsk_recover -o 2048 dds1-alpine.flag.img ./recovered/
┌──(zham㉿kali)-[~/disk-sleuth]
└─$ grep -r "picoCTF" recovered/
recovered/boot/syslinux/syslinux.cfg:  SAY picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
```

`tsk_recover` walks the entire ext filesystem and writes every allocated file to a directory tree. Then `grep -r` walks the recovered tree and finds the flag. Useful when you do not trust `strings` to find what you want (e.g. the flag is split across non-contiguous bytes, or wrapped in some non-ASCII encoding). For this challenge it is overkill — `strings` already worked — but it is the "industrial" path that scales to bigger problems.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `gunzip` | Decompress the `.gz` wrapper to recover the raw `.img` disk image. |
| `file` | Identify the file as a DOS/MBR disk image and report the partition offset (2048). |
| `srch_strings` (TSK) | Scan the disk image for printable-ASCII strings, optionally scoped to a partition with `-o`. |
| `grep` | Filter the strings output to just the `pico` / `picoCTF` line. |
| `strings` (alt) | Plain Unix `strings` works identically for a one-shot flag hunt. |
| `grep -a` (alt) | Treat the image as text and apply a regex — shortest possible solve. |
| `fls` / `icat` (TSK) | (Optional) Locate and read the file that contains the flag (`/boot/syslinux/syslinux.cfg`). |
| `qemu-system-x86_64` (alt) | (Optional) Boot the image and watch syslinux say the flag aloud. |
| `mount` (alt) | (Optional) Loop-mount the partition and `cat` the config file. |
| `tsk_recover` (alt) | (Optional) Dump the entire filesystem to disk, then `grep -r` the result. |

The primary toolchain is `gunzip`, `srch_strings`, and `grep`. All three are on a stock Kali install (`sleuthkit` is in the default metapackage).

---

## Key Takeaways

- **`strings` is the first tool to reach for when you have a binary file and a clue.** It is fast, it is universal, and it scales from kilobytes to terabytes. For CTFs, the move is `strings file | grep -i <keyword>` — that is the muscle memory. Every file format is, at the byte level, "stuff that encodes human-readable data" plus padding, and `strings` exposes the human-readable part.
- **`srch_strings` is `strings` with disk-image awareness.** It is the right tool when the file is a raw disk image and you want to scope your search to a specific partition (or skip the MBR noise). The flag is usually in the partition, not in the boot sector, so `-o <offset>` is your friend.
- **For CTFs, the "single command" is `srch_strings file | grep -i picoctf`.** That is the entire solution to this challenge, and it is the same solve for ~30% of "find the hidden flag" forensics challenges. The `grep -i` is important — CTF authors love mixing case.
- **`grep -aoE "picoCTF\{[^}]+\}"` is even shorter.** If you have GNU grep and you trust the flag format, this one-liner prints the flag directly with no other output. It is the technique to fall back on when `strings` is not installed (rare on Kali, common on barebones boxes).
- **The `SAY` syslinux directive is a real thing, not a CTF invention.** Syslinux supports text-to-speech output during boot, and `SAY` is the keyword. The author exploited an *obscure* feature, not a *fake* one. Reading the docs of unfamiliar tools is a habit that pays off in CTFs — the more you know, the more you spot.
- **Hiding a flag in `/boot/syslinux/syslinux.cfg` is clever because `/boot` is rarely examined.** Forensics tools often focus on user data (`/home`, `/etc`, `/var`), and `/boot` is "the place the kernel lives, nothing interesting here". Real-world flags hide in similar places: GRUB configs, initramfs hooks, EFI variables, kernel command-line arguments, etc. When you do not know where the flag is, search the *whole* filesystem, not just the parts that look data-rich.
- **A 4-step TSK workflow for "I have a disk image, find me the flag":**
  1. `file` — confirm it is a disk image, get the partition offset.
  2. `srch_strings -o <offset> <image> | grep <keyword>` — find the flag in the filesystem.
  3. (Optional) `fls -r -o <offset>` — locate the file by name.
  4. (Optional) `icat -o <offset> <inode>` — read the file to confirm.
  Steps 1 and 2 are enough to solve. Steps 3 and 4 are for the writeup.

### Flag wordplay decode

```
picoCTF{f0r3ns1c4t0r_n30phyt3_5e56e786}
         |     |       |         |
         f0r3n s1c4t0r n30phyt3  5e56e786
```

Read in chunks it becomes **"f0r3ns1c4t0r n30phyt3 5e56e786"** — leet-speak for **"forensicator neophyte"**, with a hex nonce (`5e56e786`) at the end:

- `f0r3ns1c4t0r` → `forensicator` (the same portmanteau as the `dds2` flag — "forensics operator", the user of forensic tools). `0` for `o`, `3` for `e`, `1` for `i`, `4` for `a`.
- `n30phyt3` → `neophyte` (a beginner, a newcomer, a person who has just been initiated). `3` for `e`, `0` for `o`, `3` for `e`, `0` for `o`. The `ph` is a hint — "neophyte" is the only common English word with `ph` after `neo`, so the `ph` gives away the spelling.
- `5e56e786` → 8 hex chars (4 bytes) — the standard per-instance nonce that makes the flag unique to this challenge generation.

The pairing is deliberate: `dds1` says *neophyte* (you are new to this), `dds2` says *novice* (you are still new, but you have progressed). The two challenges form a mini-arc: solve the first one as a neophyte, and you are a novice by the time you reach the second. The author is grading your growth across the two challenges in the flag itself.

The technique progression is the real lesson: `dds1` is a `strings` solve, `dds2` is a `fls`+`icat` solve. Both teach a different TSK muscle. Solve both and you have the basic TSK toolkit for the rest of the disk-image forensics challenges in your CTF career.
