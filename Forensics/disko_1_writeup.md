# DISKO 1 — picoCTF Writeup

**Challenge:** DISKO 1  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{1t5_ju5t_4_5tr1n9_be6031da}`  

---

## Description

> Can you find the flag in this disk image?
> Download the disk image here.

**Hint shown in challenge:** `Maybe Strings could help? If only there was a way to do that?`

---

## Background Knowledge (Read This First!)

### What is a Disk Image?

A disk image is a file that contains the exact contents and structure of a storage device. Common formats:

- `.dd` — Raw disk dump (what we have here)
- `.iso` — CD/DVD image
- `.img` — Disk image
- `.vmdk` — Virtual machine disk

### What does strings do?

The `strings` command extracts all human-readable (printable ASCII) text from any binary file — including disk images. Even if data is embedded deep in raw storage, `strings` can find it.

---

## Solution — Step by Step

### Step 1 — Download and Extract

The disk image is compressed with gzip, so I extracted it first:

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ gunzip disko-1.dd.gz
```

This created `disko-1.dd` (the actual disk image).

### Step 2 — Use strings to Find the Flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings disko-1.dd | grep picoCTF
picoCTF{1t5_ju5t_4_5tr1n9_be6031da}
```

Got the flag! 🎯

---

## Alternative Methods

**Method 1 — Mount the disk**
```bash
sudo mkdir /mnt/disko
sudo mount -o loop disko-1.dd /mnt/disko
ls /mnt/disko
```

**Method 2 — Save strings output to file**
```bash
strings disko-1.dd > output.txt
cat output.txt | grep picoCTF
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `gunzip` | Decompress gzip files |
| `strings` | Extract printable strings from binary files |
| `grep` | Search for patterns in text |

---

## Key Takeaways

- The `strings` command is extremely powerful for forensics challenges
- Disk images may contain plain text data that is easily extractable
- Always follow the hints — they guide you to the right tool
- The flag name "it's just a string" confirms this is the intended solution
- `strings` is one of the first tools to try on any binary file in forensics
