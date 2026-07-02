# Sleuthkit Intro — picoCTF Writeup

**Challenge:** Sleuthkit Intro  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** (picoCTF standard scoring)  
**Flag:** `picoCTF{mm15_f7w!}`  
**Platform:** picoCTF (picoGym Exclusive)  
**Writeup by:** zham  

---

## Description

> Download the disk image and use `mmls` on it to find the size of the Linux partition. Connect to the remote checker service to check your answer and get the flag.
>
> Note: if you are using the webshell, download and extract the disk image into `/tmp` not your home directory.

**Hints given in the challenge:**

- No formal hint, but the description itself names the exact tool: `mmls`
- The note about `/tmp` is a hint that the image is large and you should avoid filling your home directory

---

## Background Knowledge (Read This First!)

### What is The Sleuth Kit?

**The Sleuth Kit (TSK)** is an open-source collection of command-line digital forensics tools. It is the go-to toolkit for analyzing disk images without mounting them. Each command in TSK handles a specific forensic job:

| Tool | Purpose |
|------|---------|
| `mmls` | Display the partition layout of a disk image |
| `fls` | List file and directory names in a filesystem |
| `icat` | Extract a file by its inode number |
| `fsstat` | Show filesystem statistics |
| `img_stat` | Show image metadata |

If you are on Kali Linux, TSK is **preinstalled**. Otherwise install with:

```bash
sudo apt install sleuthkit
```

### What is `mmls`?

`mmls` (Media Management Layer Show) reads the partition table of a disk image and prints every partition it finds. For each partition it shows:

- **Slot** — the index in the partition table
- **Start** — the first sector of the partition
- **End** — the last sector of the partition
- **Length** — the total number of sectors in the partition
- **Description** — what kind of partition it is (Linux, FAT32, NTFS, etc.)

Output is given in **sectors** (units of 512 bytes by default). This is exactly what we need for the challenge — the size of the Linux partition in sectors.

### Partition Type IDs

When `mmls` describes a partition you will often see a hex ID like `0x83`. Common IDs:

| ID | Type |
|----|------|
| `0x83` | Linux |
| `0x82` | Linux swap |
| `0x07` | NTFS / exFAT |
| `0x0b` | FAT32 (CHS) |
| `0x0c` | FAT32 (LBA) |
| `0xee` | GPT protective MBR |

In this challenge we are looking for the `Linux (0x83)` partition.

### Why `/tmp`?

Disk images are big — this one is 100 MB. If you extract it in your home directory you risk filling the disk and breaking your session. `/tmp` is the safe place.

---

## Solution — Step by Step

### Step 1 — Move the disk image to `/tmp` and extract it

The downloaded file is gzip-compressed (`.gz`). I extracted it under `/tmp`:

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /tmp/sleuthkit && cd /tmp/sleuthkit
┌──(zham㉿kali)-[/tmp/sleuthkit]
└─$ cp ~/Downloads/disk.img.gz .
┌──(zham㉿kali)-[/tmp/sleuthkit]
└─$ gunzip disk.img.gz
┌──(zham㉿kali)-[/tmp/sleuthkit]
└─$ ls -la
total 102408
drwxr-xr-x 2 root root      4096 Jul  3 00:27 .
drwxr-xrwt 1 root root      4096 Jul  3 00:27 ..
-rw-r--r-- 1 root root 104857600 Jul  3 00:27 disk.img
```

Confirmed it is a real disk image (raw `.dd`-style dump):

```
┌──(zham㉿kali)-[/tmp/sleuthkit]
└─$ file disk.img
disk.img: DOS/MBR boot sector; partition 1: ID=0x83, startsector 2048, 202752 sectors
```

### Step 2 — Run `mmls` to inspect the partition layout

```
┌──(zham㉿kali)-[/tmp/sleuthkit]
└─$ mmls disk.img
DOS Partition Table
Offset Sector: 0
Units are in 512-byte sectors

      Slot      Start        End          Length       Description
000:  Meta      0000000000   0000000000   0000000001   Primary Table (#0)
001:  -------   0000000000   0000002047   0000002048   Unallocated
002:  000:000   0000002048   0000204799   0000202752   Linux (0x83)
```

The Linux partition:

- Starts at sector **2048**
- Ends at sector **204799**
- **Length = 202752 sectors** ← this is the answer

### Step 3 — Connect to the checker service

The checker asks for the size in sectors. Just pipe the number into `nc`:

```
┌──(zham㉿kali)-[/tmp/sleuthkit]
└─$ echo "202752" | nc saturn.picoctf.net 51533
202752
What is the size of the Linux partition in the given disk image?
Length in sectors: Great work!
picoCTF{mm15_f7w!}
```

Got the flag.

---

## What Happened Internally (Timeline)

1. The challenge gave a compressed disk image and explicitly named the tool: `mmls`.
2. I extracted `disk.img.gz` → `disk.img` (100 MB raw dump) inside `/tmp` so I would not fill my home directory.
3. I ran `mmls disk.img`. The Sleuth Kit read the MBR partition table at offset 0 and decoded one real partition entry: `000:000  Linux (0x83)` with `Length = 202752`.
4. The checker service simply wants the raw number from the **Length** column. Sending `202752` is accepted.
5. The flag is returned directly from the checker — no further decoding needed.

---

## Alternative Methods

**Method 1 — Read the answer from `file` output**

`file` already prints the partition size in sectors inside its description line. You can grep it instead of running `mmls`:

```bash
file disk.img | grep -oP '\d+(?= sectors)' | head -1
```

**Method 2 — Manually walk the MBR with `dd` and inspect bytes**

The MBR partition entries live at offsets `0x1BE`, `0x1CE`, `0x1DE`, `0x1EE`. Each 16-byte entry contains start LBA at offset `+8` and number of sectors at offset `+12`, both little-endian 32-bit. Reading them manually is overkill here, but it works:

```bash
dd if=disk.img bs=1 skip=$((0x1BE+8)) count=4 | xxd -e -g4
dd if=disk.img bs=1 skip=$((0x1BE+12)) count=4 | xxd -e -g4
```

`mmls` is just doing this internally.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `gunzip` | Decompress the `.gz` disk image |
| `file` | Identify the disk image type and quick partition hint |
| `mmls` (Sleuth Kit) | Display the full partition table and read partition lengths |
| `nc` (netcat) | Send the answer to the remote checker service |

---

## Key Takeaways

- **`mmls` is the fastest way to inspect a disk image's partitions** — it does not mount anything and does not need root.
- The answer to this challenge is the raw **Length** column from `mmls`, expressed in **sectors** (each sector = 512 bytes). If the checker wanted bytes you would multiply by 512: `202752 × 512 = 103,809,024 bytes` (~99 MiB).
- Always extract big disk images under `/tmp` to avoid filling your home directory.
- Knowing common partition type IDs (`0x83` = Linux, `0x0b`/`0x0c` = FAT32, `0x07` = NTFS) saves time across many disk forensics challenges.
- Flag wordplay: `mm15` reads as **"mmls"** (1→l, 5→s) and `f7w` reads as **"ftw"** (7→t). So the flag spells **"mmls ftw!"** — mmls for the win, exactly the tool that solved it.
