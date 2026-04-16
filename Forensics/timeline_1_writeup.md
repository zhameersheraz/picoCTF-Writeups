# Timeline 1 — picoCTF Writeup

**Challenge:** Timeline 1  
**Category:** Forensics  
**Difficulty:** Medium  
**Flag:** `picoCTF{573417h13r_7h4n_7h3_1457_58527bb222}`  

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

### Step 3 — Look at the most recent activity

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tail -100 timeline.txt | grep -v "vim\|usr/share\|usr/lib"
```

You will see a suspicious file modified right before system shutdown:

```
Mon Dec 01 2025 21:50:07    49 macb r/rrw-r--r-- 0   0   32716   /etc/chat
```

`/etc/chat` is not a standard Linux config file — it stands out immediately.

### Step 4 — Extract the file using icat

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ icat partition4.img 32716
NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK
```

### Step 5 — Decode the Base64

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "NTczNDE3aDEzcl83aDRuXzdoM18xNDU3XzU4NTI3YmIyMjIK" | base64 -d
573417h13r_7h4n_7h3_1457_58527bb222
```

Wrap in picoCTF format → `picoCTF{573417h13r_7h4n_7h3_1457_58527bb222}` ✅ Got the flag! 🎯

---

## Why /etc/chat?

The attacker hid the file inside `/etc/chat` — a filename that looks like a legitimate system config file (chat scripts are real in old PPP dial-up configs). By placing it in `/etc/`, it blends in with real config files and is easy to miss without a timeline.

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

- **MAC timelines** are one of the most powerful forensic techniques — always build one first
- Files modified **right before shutdown** are the most suspicious entries to investigate
- Attackers hide data in **legitimate-looking filenames** inside `/etc/` to avoid detection
- `/etc/chat` is not a standard modern Linux file — always Google unfamiliar filenames
- **Base64** is the most common encoding used to wrap binary data into printable characters
