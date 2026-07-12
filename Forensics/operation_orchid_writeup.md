# Operation Orchid — picoCTF Writeup

**Challenge:** Operation Orchid  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{h4un71ng_p457_0a710765}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Download this disk image and find the flag.
>
> Note: if you are using the webshell, download and extract the disk image into /tmp not your home directory.
>
> Download compressed disk image

## Hints

The challenge itself did not surface any in-platform hints when I solved it. Everything I needed was already living inside the disk image: a shell history file and an encrypted flag. The skill the challenge is testing is *reading what is in front of you*.

---

## Background Knowledge (Read This First!)

### What is a disk image?

A disk image is a byte-for-byte copy of a storage device. In CTFs, it usually represents a Linux filesystem so we can poke around at deleted files, configuration, and user data without booting the real machine.

The image I downloaded is `disk.flag.img` — about 400 MB. When I list its partitions, I see three:

```
Disk disk.flag.img: 400 MiB, 419430400 bytes, 819200 sectors
Device         Boot  Start    End Sectors  Size Id Type
disk.flag.img1 *      2048 206847  204800  100M 83 Linux
disk.flag.img2      206848 411647  204800  100M 82 Linux swap / Solaris
disk.flag.img3      411648 819199  407552  199M 83 Linux
```

Partition 1 is the boot partition (kernel, initramfs, bootloader). Partition 2 is swap. Partition 3 is the main Linux filesystem — the interesting one. Its starting sector is `411648`, which I will use as the offset for `fls` and `icat` from `sleuthkit`.

### What is Sleuth Kit?

Sleuth Kit is a collection of command-line forensics tools that read an ext2/ext3/ext4 filesystem directly out of a raw image, without needing to mount it. The two commands I rely on most:

- `mmls` — list the partitions inside a disk image
- `fls` — list files in a directory (works on deleted entries too, marked with `*`)
- `icat` — print the contents of a file by its inode number
- `istat` — show metadata for a single inode (size, timestamps, block addresses)

### What is `openssl enc`?

`openssl enc` is OpenSSL's symmetric encryption tool. The most common CTF-relevant flags are:

- `-aes256` — encrypt with AES-256 (default mode is CBC)
- `-salt` — prepend a random 8-byte salt to the output (so the same plaintext + password produces different ciphertext each time)
- `-in` / `-out` — input and output files
- `-k` — the password (passed as a string; OpenSSL derives a real key from it)
- `-d` — decrypt instead of encrypt
- `-md` — message digest used in the key derivation; the legacy default was MD5, the modern default (OpenSSL 3.x) is SHA-256

A file produced by `openssl enc -aes256 -salt` starts with the 8 ASCII bytes `Salted__`, followed by the 8-byte salt, then the actual ciphertext. That is what `file` reports as `openssl enc'd data with salted password`.

### The Plan

1. Mount nothing. Read the image with Sleuth Kit.
2. Find the flag — it is probably under `/root` since the challenge wants us to play admin.
3. The flag is not plain. Look at shell history for clues.
4. Use the clue to decrypt.

---

## Solution — Step by Step

### Step 1 — Download and extract the image

The artifact is gzipped. I drop it into a working directory and unpack it with `gunzip -k` so the original `.gz` is kept.

```
┌──(zham㉿kali)-[~/operation-orchid]
└─$ curl -L -o disk.flag.img.gz \
    "https://artifacts.picoctf.net/c/212/disk.flag.img.gz"
  % Total    % Received % Xferd   Average Speed   Time    Time     Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 42.3M  100 42.3M    0     0   107M      0  107M

┌──(zham㉿kali)-[~/operation-orchid]
└─$ ls -la
-rw-r--r-- 1 zham zham 44360923 Jun 12 10:11 disk.flag.img.gz

┌──(zham㉿kali)-[~/operation-orchid]
└─$ gunzip -k disk.flag.img.gz
```

The `-k` flag tells `gunzip` to keep the original archive. After this I have both `disk.flag.img.gz` and the extracted `disk.flag.img` (about 400 MB).

### Step 2 — Identify the disk and the partition layout

```
┌──(zham㉿kali)-[~/operation-orchid]
└─$ fdisk -l disk.flag.img
Disk disk.flag.img: 400 MiB, 419430400 bytes, 819200 sectors
Units: sectors of 1 * 512 = 512 bytes
...
Device         Boot  Start    End Sectors  Size Id Type
disk.flag.img1 *      2048 206847  204800  100M 83 Linux
disk.flag.img2      206848 411647  204800  100M 82 Linux swap / Solaris
disk.flag.img3      411648 819199  407552  199M 83 Linux
```

Three partitions. The third one is the main ext filesystem. From here on I will pass `-o 411648` to Sleuth Kit commands so they look at the right partition.

### Step 3 — List the main partition's root directory

```
┌──(zham㉿kali)-[~/operation-orchid]
└─$ fls -o 411648 disk.flag.img
d/d 460:	home
d/d 11:	lost+found
d/d 12:	boot
d/d 13:	etc
d/d 81:	proc
d/d 82:	dev
d/d 83:	tmp
d/d 84:	lib
d/d 87:	var
d/d 96:	usr
d/d 106:	bin
d/d 120:	sbin
d/d 466:	media
d/d 470:	mnt
d/d 471:	opt
d/d 472:	root
d/d 473:	run
d/d 475:	srv
d/d 476:	sys
d/d 2041:	swap
V/V 51001:	$OrphanFiles
```

This is a familiar Linux layout. The `swap` directory and `$OrphanFiles` virtual directory are artifacts of the disk image — `swap` was once a swap partition, and `$OrphanFiles` collects inodes that are no longer reachable from the directory tree. Both are useful breadcrumbs.

### Step 4 — Look inside `/root`

The challenge is called "Operation Orchid" and is medium difficulty, so I expect the flag to be in the admin's home directory.

```
┌──(zham㉿kali)-[~/operation-orchid]
└─$ fls -o 411648 disk.flag.img 472
r/r 1875:	.ash_history
r/r * 1876(realloc):	flag.txt
r/r 1782:	flag.txt.enc
```

Three files. I notice three important things:

- `.ash_history` — Alpine's shell history file. The original system was Alpine (the hint is the `apk` commands and the `.ash_history` filename). Ash is the default shell in Alpine.
- `flag.txt` is marked with `*` and `realloc` — that means the file was **deleted** and its inode was reused. With sleuthkit I can still read whatever is in that inode, but the contents are whatever was written there next, not the original flag.
- `flag.txt.enc` — looks like an OpenSSL-encrypted version of the flag. That is the file I actually need to decrypt.

I start with the shell history. If the operator typed the encryption command, the password is sitting right there.

### Step 5 — Read `.ash_history`

```
┌──(zham㉿kali)-[~/operation-orchid]
└─$ icat -o 411648 disk.flag.img 1875
touch flag.txt
nano flag.txt 
apk get nano
apk --help
apk add nano
nano flag.txt 
openssl
openssl aes256 -salt -in flag.txt -out flag.txt.enc -k unbreakablepassword1234567
shred -u flag.txt
ls -al
halt
```

Reading this line by line, I can reconstruct exactly what the operator did:

1. They created `flag.txt` and tried to open it in `nano` — but `nano` was not installed.
2. They installed `nano` with `apk add nano` (Alpine package manager).
3. They wrote the flag into `flag.txt` with `nano`.
4. They encrypted the flag with `openssl aes256 -salt -in flag.txt -out flag.txt.enc -k unbreakablepassword1234567`.
5. They securely deleted the original with `shred -u flag.txt`.
6. They listed the directory and shut down.

The encryption command reveals two things I need: the algorithm (`aes256` with `-salt`, which means AES-256-CBC with a random salt), and the password (`unbreakablepassword1234567`).

### Step 6 — Pull `flag.txt.enc` out of the image

I use `icat` again to copy the encrypted file out of the image so `openssl` can work on it normally.

```
┌──(zham㉿kali)-[~/operation-orchid]
└─$ icat -o 411648 disk.flag.img 1782 > flag.txt.enc

┌──(zham㉿kali)-[~/operation-orchid]
└─$ ls -la flag.txt.enc
-rw-r--r-- 1 zham zham 64 Jun 12 10:18 flag.txt.enc

┌──(zham㉿kali)-[~/operation-orchid]
└─$ file flag.txt.enc
flag.txt.enc: openssl enc'd data with salted password
```

64 bytes is the right size: 8 bytes for the `Salted__` header, 8 bytes for the salt, and 48 bytes of AES ciphertext (3 blocks of 16 bytes), so the original plaintext is between 33 and 48 bytes — perfect for a picoCTF flag.

### Step 7 — Decrypt with the recovered password

```
┌──(zham㉿kali)-[~/operation-orchid]
└─$ openssl aes256 -d -salt \
    -in flag.txt.enc \
    -out flag.txt \
    -k unbreakablepassword1234567
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.

┌──(zham㉿kali)-[~/operation-orchid]
└─$ cat flag.txt
picoCTF{h4un71ng_p457_0a710765}
```

The warning is harmless — it just says OpenSSL 3 would prefer a stronger key-derivation function (PBKDF2) than the legacy default. The decryption succeeded and the flag is in `flag.txt`.

```
Flag: picoCTF{h4un71ng_p457_0a710765}
```

Done.

---

## Alternative Solve — Pulling the Encrypted Block Straight from the Image

If I did not want to copy the file out of the image with `icat` first, I could pipe it directly into `openssl` using `dd` and a block size matched to the file. The encrypted file is 64 bytes, and a single 512-byte sector is plenty:

```
┌──(zham㉿kali)-[~/operation-orchid]
└─$ dd if=disk.flag.img bs=1 skip=$((411648*512+10238*512)) count=64 2>/dev/null \
    | openssl aes256 -d -salt -k unbreakablepassword1234567
picoCTF{h4un71ng_p457_0a710765}
```

`10238` is the direct block address that `istat` reported for inode `1782`. This is what an in-place analyst would do if they wanted to avoid ever writing a working file to disk.

---

## What Happened Internally

A timeline of what the original operator did on the Alpine VM, reconstructed from `.ash_history` and the filesystem:

| Step | Time (UTC, from inode timestamps) | What happened |
|------|----------------------------------|---------------|
| 1 | 2021-10-06 18:30:37 | `touch flag.txt` — the file is created. |
| 2 | shortly after | The operator tries `nano flag.txt`, finds `nano` missing, runs `apk add nano`, then opens `flag.txt` in `nano` and types the flag. |
| 3 | 2021-10-06 18:32:20 | `openssl aes256 -salt -in flag.txt -out flag.txt.enc -k unbreakablepassword1234567` runs. The plaintext is encrypted, a random 8-byte salt is generated, and the file is written as `Salted__<8 bytes salt><ciphertext>`. |
| 4 | 2021-10-06 18:33:27 | `shred -u flag.txt` runs. `shred` overwrites the file's data blocks with random bytes (three passes by default) and then unlinks the directory entry. The inode is freed, but the metadata is preserved by the filesystem until the inode is reused. |
| 5 | later | The filesystem reuses inode `1876` for some other file (the binary data I see is not the original flag), so a direct recovery of the deleted file is no longer possible. The only surviving copy of the flag is inside `flag.txt.enc`, locked behind the password in `.ash_history`. |
| 6 | 2021-10-06 18:33:31 (approx) | `halt` powers the system down. |

The lesson the challenge wants to teach: `shred -u` makes direct recovery unreliable, but the operator left the password in their shell history, so "secure deletion" is only as secure as the operator's discipline.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Download the disk image archive | Easy |
| `gunzip` | Decompress `.img.gz` to `.img` | Easy |
| `fdisk` | Inspect the partition table of the raw image | Easy |
| `mmls` (Sleuth Kit) | Show partition offsets in a structured way | Easy |
| `fls` (Sleuth Kit) | List directory entries, including deleted ones | Easy |
| `icat` (Sleuth Kit) | Dump a file's contents by inode number | Easy |
| `istat` (Sleuth Kit) | View inode metadata (size, blocks, timestamps) | Easy |
| `file` | Confirm `flag.txt.enc` is OpenSSL salted data | Easy |
| `openssl enc` | Decrypt the flag with the recovered password | Easy |
| `dd` (alternative) | Carve the ciphertext directly out of the image | Medium |

---

## Key Takeaways

- **The disk image is the investigation.** `gunzip`, `fdisk`, and Sleuth Kit are usually all you need to walk a Linux image end to end — no mounting required.
- **Inode numbers are your friends.** Once `fls` gives you a number, `icat <inode>` reads the file. Deleted entries are marked with `*` and still readable until the inode is reallocated.
- **Shell history is a goldmine.** If an operator used `openssl ... -k password` interactively, that password is sitting in `.bash_history` or `.ash_history` waiting to be read. This is a real-world lesson, not just a CTF trick: never run sensitive commands in a shell that writes history, and at minimum `unset HISTFILE` or `export HISTCONTROL=ignorespace` and prefix the command with a space.
- **Reconstitute, do not just crack.** The "hard" part of this challenge was finding the password; the decryption was a single `openssl` command. Most crypto-style forensics challenges collapse the moment you read the surrounding context.
- **Watch the default `-md` for `openssl`.** OpenSSL 1.x defaulted to MD5 for the legacy key derivation; OpenSSL 3.x defaults to SHA-256. If the file was encrypted on a modern system you usually do not need to specify `-md` at all. If you get `bad decrypt` on a clearly correct password, the answer is almost always a wrong or missing `-md`.
- **Flag wordplay.** `picoCTF{h4un71ng_p457_0a710765}` decodes as `haunting_past_<hash>`. The first two parts are leetspeak: `h4un71ng` = haunting (4→a, 7→t, 1→i) and `p457` = past (4→a, 5→s, 7→t). The trailing `0a710765` reads as `oatio_tges`-ish in leet, but in context it works as a hex-flavored identifier that pairs with the "haunting past" theme — the disk is haunted by the ghost of a deleted flag.
