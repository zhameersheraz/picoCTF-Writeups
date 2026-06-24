# DISKO 3 — picoCTF Writeup

**Challenge:** DISKO 3  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 1  
**Flag:** `picoCTF{n3v3r_z1p_2_h1d3_654235e0}`  
**Platform:** picoCTF 2025 (picoGym Exclusive)  
**Writeup by:** zham  

---

## Description

> Can you find the flag in this disk image? This time, its not as plain as you think it is!
> Download the disk image here.

**Hint 1:** `How will you search and extract files in a partition?`

---

## Background Knowledge (Read This First!)

### What is a Disk Image?

A disk image is a file that contains the exact byte-for-byte contents of a storage device — every file, every folder, every piece of metadata. Common formats:

- `.dd` — Raw disk dump (what we have here)
- `.iso` — CD/DVD image
- `.img` — Generic disk image
- `.vmdk` — Virtual machine disk

The challenge gives us `disko-3.dd.gz` — a gzipped raw FAT32 image.

### Why "strings" Fails This Time

In **DISKO 1**, the flag sat in plaintext inside the disk image, so `strings disko.dd | grep picoCTF` printed it instantly. **DISKO 3 turns that off.** The flag is hidden inside a compressed file (a `.gz` archive). When you `strings` the raw disk, the compressed bytes look like random noise — no human-readable flag appears.

That is exactly what the challenge text means by "its not as plain as you think it is!" — the flag is there, but it is wrapped. To unwrap it you need to actually **parse the filesystem**, find the wrapping file, and decompress it.

### What is FAT32?

**FAT32** is one of the oldest and simplest filesystems still in use. It uses a **File Allocation Table** to track which clusters on disk belong to which file. Key ideas:

- Files are split into **clusters** of fixed size
- The FAT is a linked list — each entry points to the next cluster of the file
- Every regular file (and every directory) is described by a **directory entry**

### What is The Sleuth Kit?

**The Sleuth Kit (TSK)** is a collection of CLI forensics tools for analyzing disk images without mounting them. Three commands we will use:

- `fls` — list files in a filesystem (including deleted ones)
- `istat` — show the metadata of a single file by entry number
- `icat` — output the raw contents of a file by entry number

This is the standard "open the image as a forensic analyst" toolset.

### What is gzip?

**gzip** is a file compression tool. You compress a file with `gzip file.txt` and get `file.txt.gz` — about 3x smaller on text. Decompress with `gunzip file.txt.gz` (which restores the original `file.txt`) or read on the fly with `zcat file.txt.gz`.

Compressing a file does **not** encrypt or hide it. Anyone who finds the `.gz` can decompress it. That is the punchline of this challenge's flag.

---

## Solution — Step by Step

### Step 1 — Download and Extract the Disk Image

The challenge file is `disko-3.dd.gz`. I move into my downloads folder and decompress it:

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip disko-3.dd.gz
```

This leaves me with the raw image `disko-3.dd`.

### Step 2 — Try the Easy Way First (and Watch It Fail)

DISKO 1 trained me to try `strings` first. I try it here so I can show the failure explicitly:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings disko-3.dd | grep -i picoctf
MESSAGE=[system] Activating via systemd: service name='org.bluez' unit='dbus-org.bluez.service' requested by ':1.81' (uid=1000 pid=12105 comm="/usr/share/code/code Desktop/picoctf-2025")
MESSAGE=[system] Activating via systemd: service name='org.bluez' unit='dbus-org.bluez.service' requested by ':1.66' (uid=1000 pid=43129 comm="/usr/share/code/code Desktop/picoctf-2025")
MESSAGE=[system] Activating via systemd: service name='org.bluez' unit='dbus-org.bluez.service' requested by ':1.65' (uid=1000 pid=2141 comm="/usr/share/code/code Desktop/picoctf-2025")
MESSAGE=[system] Activating via systemd: service name='org.bluez' unit='dbus-org.bluez.service' requested by ':1.65' (uid=1000 pid=2584 comm="/usr/share/code/code Desktop/picoctf-2025")
```

A bunch of systemd log messages mention `picoctf-2025` (the IDE used to build the challenge), but no `picoCTF{...}` flag. The hint told us to expect this:

> "How will you search and extract files in a partition?"

Time to actually open the filesystem.

### Step 3 — Identify the Filesystem

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file disko-3.dd
disko-3.dd: DOS/MBR boot sector, code offset 0x58+2, OEM-ID "mkfs.fat", Media descriptor 0xf8, sectors/track 32, heads 8, sectors 204800 (volumes > 32 MB), FAT (32 bit), sectors/FAT 1576, serial number 0x49838d0b, unlabeled
```

`FAT (32 bit)` — confirmed FAT32. Now I can use The Sleuth Kit.

### Step 4 — List Every File in the Image

`fls -r` recurses through every directory:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -r disko-3.dd | head -40
d/d 4:	log
+ d/d 22:	private
+ d/d 24:	sysstat
+ d/d 26:	stunnel4
++ r/r 70:	stunnel.log
+ d/d 28:	mysql
+ d/d 30:	inetsim
+ d/d 32:	installer
+ r/r 519123:	vmware-vmsvc-root.2.log
+ r/r 519125:	kern.log.4.gz
+ r/r 522627:	daemon.log
+ r/r 522628:	flag.gz
+ r/r * 522629:	_ESSAGES
+ r/r 522632:	alternatives.log.2.gz
+ r/r 522634:	debug
+ d/d 522636:	lightdm
...
```

There it is — entry `522628: flag.gz`, sitting in `/log/` right next to the deleted `_ESSAGES` (the famous Debian `messages` log file, with the leading `M` overwritten to `0xE5`).

The filename is a deliberate hint. The flag is gzipped, which is why `strings` could not see it.

### Step 5 — Inspect the File Metadata

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ istat disko-3.dd 522628
Directory Entry: 522628
Allocated
File Attributes: File, Archive
Size: 78
Name: flag.gz

Directory Entry Times:
Written:	2025-07-17 15:06:44 (UTC)
Accessed:	2025-07-17 00:00:00 (UTC)
Created:	2025-07-17 15:06:44 (UTC)

Sectors:
36434
```

`Allocated` — it is a live file, not deleted. Size 78 bytes — tiny. Lives in cluster starting at sector 36434.

### Step 6 — Extract the File with `icat`

`icat` reads a file by its entry number and writes the raw bytes to stdout. Redirect into a file:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat disko-3.dd 522628 > flag.gz
```

Verify what came out:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file flag.gz
flag.gz: gzip compressed data, was "flag", last modified: Thu Jul 17 15:06:44 2025, from Unix, original size modulo 2^32 53
```

Still a valid gzip — original was a 53-byte plaintext file called `flag`.

### Step 7 — Decompress and Read the Flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip flag.gz

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat flag
Here is your flag
picoCTF{n3v3r_z1p_2_h1d3_654235e0}
```

The flag is `picoCTF{n3v3r_z1p_2_h1d3_654235e0}`.

Optional sanity check with `strings` on the recovered file:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings flag | grep picoCTF
picoCTF{n3v3r_z1p_2_h1d3_654235e0}
```

---

## Alternative Methods

### Method 1 — Mount the Image and Pull the File Out

If you prefer a "real filesystem" approach, mount it read-only and copy `flag.gz`:

```bash
sudo mkdir /mnt/disko3
sudo mount -o ro,loop disko-3.dd /mnt/disko3
ls /mnt/disko3/log/
cp /mnt/disko3/log/flag.gz .
gunzip flag.gz
cat flag
sudo umount /mnt/disko3
```

Same result. The mount path just lets you browse files with normal `ls`/`cat` instead of Sleuth Kit's entry-number scheme.

### Method 2 — `icat` Directly to stdout, Pipe Through `gunzip -c`

Skip the intermediate file with a one-liner:

```bash
icat disko-3.dd 522628 | gunzip -c
```

`gunzip -c` writes the decompressed data to stdout without touching the input file. Nice for a quick capture.

### Method 3 — `binwalk` to Find and Extract the gzip Stream

`binwalk` scans for embedded files by magic bytes. Useful if you do not want to install The Sleuth Kit:

```bash
binwalk -e disko-3.dd
```

`binwalk` finds the gzip at sector 36434 and extracts it into `_disko-3.dd.extracted/flag.gz`. Then `gunzip` as usual.

---

## What Happened Internally (Timeline)

1. **Filesystem creation** — `mkfs.fat` formatted the volume as FAT32. The OEM ID `mkfs.fat` shows up in `file` output.
2. **Files copied in** — a Debian `/var/log/` directory tree was written to the image. Real logs like `daemon.log`, `alternatives.log.2.gz`, etc. were placed there for camouflage.
3. **Flag was placed** — someone wrote the plaintext `Here is your flag\npicoCTF{...}` (53 bytes) into `/log/flag` and then ran `gzip flag`, producing `/log/flag.gz` (78 bytes). The gzip wrapper is what made the bytes unreadable to `strings`.
4. **A neighbouring file was deleted** — the original `messages` log was removed (`rm` or equivalent). Its directory entry now shows as `_ESSAGES` in `fls`. The flag file itself was NOT deleted — only its neighbour was, as a red herring.
5. **We extract with `icat`** — Sleuth Kit reads cluster starting at sector 36434 (per `istat`), pulls all 78 raw bytes of the gzip stream, and writes them to `flag.gz`. Nothing is mounted, nothing is modified.
6. **We decompress** — `gunzip` inflates the stream back to the 53-byte plaintext. The flag drops out.

The reason `strings` failed in step 5 is simple: gzip's DEFLATE algorithm produces output that has very few printable ASCII runs longer than 4 characters, so the entire flag (plus its wrapping `Here is your flag` line) stays invisible until you actually decode the stream.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `gunzip` | Decompress the disk image and the recovered `flag.gz` | Easy |
| `file` | Identify the filesystem type (FAT32) and the recovered file type (gzip) | Easy |
| `fls` (Sleuth Kit) | List every file in the image; spot `flag.gz` | Easy |
| `istat` (Sleuth Kit) | Confirm the file is allocated, size 78 bytes, sector 36434 | Easy |
| `icat` (Sleuth Kit) | Extract the file's raw bytes by entry number | Easy |
| `strings \| grep` | Sanity-check the recovered flag | Easy |
| `mount` (alt) | Mount the image read-only and copy `flag.gz` manually | Easy |
| `binwalk` (alt) | Carve the gzip stream out of the raw image by magic bytes | Easy |

To install The Sleuth Kit on Kali:

```bash
sudo apt update
sudo apt install sleuthkit
```

---

## Key Takeaways

- **Compression is not hiding.** `gzip` only shrinks bytes — it does not protect them. The flag inside `flag.gz` is in plaintext once you inflate it. If you can see the `.gz`, you can read it.
- **`strings` works only on plaintext (or lightly-encoded) data.** Compressed, encrypted, or binary files defeat it. When `strings | grep picoCTF` returns nothing, switch to a real filesystem tool.
- **Always try `fls -r` next** when the hint mentions "files in a partition." It scans the filesystem directory entries directly, so file names like `flag.gz` pop out regardless of whether their contents are plaintext.
- **`icat <image> <entry>` is the safest extraction command.** It pulls a single file by metadata entry without ever mounting the disk — no risk of accidentally writing to the image.
- **Watch out for neighbours.** The deleted `_ESSAGES` next to `flag.gz` is bait — easy to assume the deleted file is the target because DISKO 4 had the same trick. Always read filenames, not just inode numbers.
- The flag wordplay **`n3v3r_z1p_2_h1d3` decodes to "never zip to hide"** — gzipping a secret does not hide it. If someone finds the archive, they get the secret. Real protection needs encryption (e.g. `gpg`), not just compression.
