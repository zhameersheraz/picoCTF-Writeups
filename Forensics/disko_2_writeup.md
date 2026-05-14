# DISKO 2 — picoCTF Writeup

**Challenge:** DISKO 2  
**Category:** Forensics  
**Difficulty:** Medium  
**Flag:** `picoCTF{4_P4Rt_1t_i5_a93c3ba0}`  
**Platform:** picoGym Exclusive  
**Writeup by:** zham  

---

## Description

> Can you find the flag in this disk image? The right one is Linux! One wrong step and its all gone!
> Download the disk image here.

**Hint 1:** `How can you extract/isolate a partition?`

---

## Background Knowledge (Read This First!)

### What is a disk image with multiple partitions?

A disk image is a byte-for-byte copy of a physical drive. A real drive can contain **multiple partitions** — separate sections each with their own filesystem. In this challenge the disk has two partitions:

- **Partition 1** — `ID=0x83` → Linux ext4 filesystem
- **Partition 2** — `ID=0x0b` → FAT32 filesystem

The hint says "The right one is Linux" — so the flag is in partition 1 (ext4).

### What is `file` command?

The `file` command identifies what type of data a file contains by reading its header bytes — not just the extension. On a disk image it shows the MBR and partition table info.

### What is `dd`?

**`dd`** is a raw copy tool. You can use it to extract a specific byte range from a disk image — perfect for isolating one partition out of many:

```bash
dd if=disk.img of=partition.img bs=512 skip=<start_sector> count=<sector_count>
```

### What is `strings`?

**`strings`** extracts all readable text from any binary file. Since the flag is stored as plaintext inside the ext4 partition, `strings` finds it instantly — no mounting needed.

---

## Solution — Step by Step

### Step 1 — Identify the disk structure

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file disko-2_dd
disko-2_dd: DOS/MBR boot sector; partition 1: ID=0x83, startsector 2048, 51200 sectors; partition 2: ID=0xb, startsector 53248, 65536 sectors
```

Two partitions found. `ID=0x83` is Linux ext4 — that's the target.

### Step 2 — Extract the Linux partition with `dd`

Partition 1 starts at sector `2048` and is `51200` sectors long. Each sector is 512 bytes:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ dd if=disko-2_dd of=part1.img bs=512 skip=2048 count=51200
```

Verify it's ext4:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file part1.img
part1.img: Linux rev 1.0 ext4 filesystem data, UUID=9e5121c2...
```

### Step 3 — Search for the flag with `strings`

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings part1.img | grep picoCTF
picoCTF{4_P4Rt_1t_i5_a93c3ba0}
```

✅ Got the flag! 🎯

---

## Alternative Method — Mount the partition

```bash
sudo mount -o ro part1.img /mnt/disko2
find /mnt/disko2 -type f
cat /mnt/disko2/flag.txt
sudo umount /mnt/disko2
```

---

## Sector Math Explained

| Value | Source | Meaning |
|-------|--------|---------|
| `2048` | `file` output | Start sector of partition 1 |
| `51200` | `file` output | Number of sectors in partition 1 |
| `512` | Standard | Bytes per sector |
| `skip=2048` | dd flag | Skip first 2048 sectors (MBR + empty) |
| `count=51200` | dd flag | Copy exactly 51200 sectors |

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Identify disk structure and partition table | ⭐ Easy |
| `dd` | Extract the Linux partition from the full disk image | ⭐⭐ Medium |
| `strings \| grep` | Find the plaintext flag in the partition | ⭐ Easy |

---

## Key Takeaways

- **Disk images can have multiple partitions** — always run `file` first to understand the layout
- **`ID=0x83` = Linux ext4**, **`ID=0x0b` = FAT32** — knowing partition type IDs is essential for disk forensics
- **`dd skip=N count=M`** extracts any byte range from a raw image — the go-to tool for partition isolation
- The flag `4_P4Rt_1t_i5` → "a partition it is" — you found the right partition
