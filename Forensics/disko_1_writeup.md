# DISKO 1 - picoCTF Writeup

**Challenge:** DISKO 1  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{1t5_ju5t_4_5tr1n9_be6031da}`

---

## Description

Can you find the flag in this disk image?

Download the disk image here.

---

## Hints

1. Maybe Strings could help? If only there was a way to do that?

---

## Solution

### Step 1: Download the Disk Image

I downloaded the file `disko-1.dd.gz` from the challenge.

### Step 2: Extract the Compressed File

The disk image is compressed with gzip, so I extracted it:
```bash
cd /media/sf_downloads
gunzip disko-1.dd.gz
```

This created the file `disko-1.dd` (the actual disk image).

### Step 3: Use strings to Find the Flag

Following the hint about "strings", I used the `strings` command to extract all printable text from the disk image:
```bash
strings disko-1.dd | grep picoCTF
```

**Output:**
```
picoCTF{1t5_ju5t_4_5tr1n9_be6031da}
```

Found the flag! 🎯

---

## Why This Works

* **Disk images** contain raw data from storage devices
* The `strings` command extracts all **human-readable text** from binary files
* Even if data is embedded in a disk image, strings can find it
* The `grep` command filters the output to show only lines containing "picoCTF"
* The flag was stored as plain text somewhere in the disk image

---

## What is a Disk Image?

A **disk image** is a file that contains the exact contents and structure of a storage device:

**Common formats:**
- `.dd` - Raw disk dump (what we have here)
- `.iso` - CD/DVD image
- `.img` - Disk image
- `.vmdk` - Virtual machine disk

**Common uses:**
- Forensics investigations
- Backup and recovery
- Virtual machines
- Evidence preservation

---

## Alternative Methods I Tried

### Method 1: exiftool (Didn't work)
```bash
exiftool disko-1.dd
```
This shows file metadata but didn't reveal the flag.

### Method 2: cat (Didn't work well)
```bash
cat disko-1.dd
```
This outputs binary data that's unreadable.

### Method 3: Mount the disk (Could work but unnecessary)
```bash
sudo mkdir /mnt/disko
sudo mount -o loop disko-1.dd /mnt/disko
ls /mnt/disko
```
This would mount the disk image but strings was simpler.

---

## Terminal Commands
```bash
# Navigate to downloads
cd /media/sf_downloads

# Extract the compressed disk image
gunzip disko-1.dd.gz

# Extract strings and search for flag (MAIN SOLUTION)
strings disko-1.dd | grep picoCTF

# Alternative: Save all strings to file for analysis
strings disko-1.dd > output.txt
cat output.txt | grep picoCTF
```

---

## Flag

`picoCTF{1t5_ju5t_4_5tr1n9_be6031da}`

---

## Tools Used

* **gunzip** - Decompress gzip files
* **strings** - Extract printable strings from binary files
* **grep** - Search for patterns in text

---

## Key Takeaways

* The `strings` command is extremely powerful for forensics
* Disk images may contain plain text data that's easily extractable
* Always follow the hints - they guide you to the right tool
* The flag name "it's just a string" confirms this is the intended solution
* Not all forensics challenges require complex tools - sometimes simple Unix commands work best
* `strings` is one of the first tools to try on any binary file in forensics
