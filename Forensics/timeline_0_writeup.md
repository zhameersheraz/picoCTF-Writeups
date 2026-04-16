# Timeline 0 — picoCTF Writeup

**Challenge:** Timeline 0  
**Category:** Forensics  
**Difficulty:** Medium  
**Flag:** `picoCTF{71m311n3_0u7113r_h3r_43a2e7af}`  

---

## Description

> Can you find the flag in this disk image? Wrap what you find in the picoCTF flag format.
> Download the disk image here.

**Hint shown in challenge:** `Create a Sleuthkit MAC timeline!`
**Attachment:** `partition4_img.gz`

---

## Background Knowledge (Read This First!)

### What is inside the archive?

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip partition4_img.gz
# → partition4.img  (Linux ext4 filesystem)
```

### What is a disk image?

A `.img` file is a **raw byte-for-byte copy of a filesystem**. You can inspect its contents without mounting it using **Sleuthkit** forensic tools.

### What is a MAC Timeline?

MAC stands for **Modified, Accessed, Changed** — the three timestamps recorded for every file on a filesystem. A **MAC timeline** lists every file event in chronological order, making it easy to spot suspicious activity.

### What is Sleuthkit?

**Sleuthkit** is a forensic toolkit. Three tools are used here:

- `fls` — lists files and their metadata from a disk image
- `mactime` — converts that metadata into a readable sorted timeline
- `icat` — extracts the raw content of a file by inode number

### The Trick — Hiding in /bin

In this challenge, the attacker hid a file inside the `/bin` directory disguised among legitimate busybox symlinks. On a typical Alpine Linux system, **every file in `/bin` is a symlink to `/bin/busybox`** — so a real standalone file in `/bin` instantly stands out on a timeline.

### ⚠️ Two Important Notes

**Note 1 — Install Sleuthkit if not present**
```
sudo apt-get install sleuthkit
```

**Note 2 — No extra Python libraries needed**
Only the built-in `base64` command is used for decoding!

---

## Solution — Step by Step

### Step 1 — Extract the image

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip partition4_img.gz

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file partition4.img
partition4.img: Linux rev 1.0 ext4 filesystem data, UUID=7a00e9da-98f8-4f0f-b257-95edf422d902
```

### Step 2 — Create the MAC timeline

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls -r -m "/" partition4.img > body.txt

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ mactime -b body.txt > timeline.txt
```

### Step 3 — Spot the suspicious /bin modification

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep "2025" timeline.txt | grep -v "vim\|usr/share\|usr/lib\|apk"
```

You will see the `/bin` directory itself was modified at a suspicious time:

```
Mon Dec 01 2025 20:41:56   4096 m.c. d/drwxr-xr-x 0   0   22   /bin
```

The `/bin` directory was modified right at activity time — something was added to it.

### Step 4 — List the contents of /bin

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ fls partition4.img 22
```

Among all the normal symlinks (`l/l`) to busybox, one entry is a real file (`r/r`):

```
r/r 4945:   bcab
```

`bcab` is a real file — not a symlink — sitting inside `/bin`. That is the hidden file.

### Step 5 — Extract the file using icat

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat partition4.img 4945
NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK
```

### Step 6 — Decode the Base64

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "NzFtMzExbjNfMHU3MTEzcl9oM3JfNDNhMmU3YWYK" | base64 -d
71m311n3_0u7113r_h3r_43a2e7af
```

Wrap in picoCTF format → `picoCTF{71m311n3_0u7113r_h3r_43a2e7af}` ✅ Got the flag! 🎯

---

## Why /bin/bcab?

Every real binary in `/bin` on an Alpine Linux system is actually a symlink pointing to `/bin/busybox`. The attacker dropped a standalone file named `bcab` to blend in with the crowd. Without a MAC timeline highlighting the `/bin` directory modification, this file would be very easy to overlook.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Sleuthkit `fls` | List all files and metadata from disk image | ⭐⭐ Medium |
| Sleuthkit `mactime` | Build a sorted MAC timeline | ⭐⭐ Medium |
| Sleuthkit `icat` | Extract file contents by inode number | ⭐⭐ Medium |
| `base64` (built-in) | Decode the hidden Base64 string | ⭐ Easy |

---

## Key Takeaways

- **MAC timelines** reveal which directories were tampered with — not just which files
- A modification to `/bin` is a major red flag on any Linux system
- On Alpine Linux, every file in `/bin` should be a symlink to busybox — a real file there is instantly suspicious
- Attackers name hidden files with short, random-looking names (`bcab`) to avoid obvious keyword searches
- **Base64** is the most common encoding used to hide flag data inside binary files
