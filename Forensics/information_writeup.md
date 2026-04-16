# information — picoCTF Writeup

**Challenge:** information  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{the_m3tadata_1s_modified}`  

---

## Description

> Files can always be changed in a secret way. Can you find the flag?
> Download: cat.jpg

**Hints shown in challenge:**
1. Look at the details of the file
2. Make sure to submit the flag as picoCTF{YOURFLAG}

---

## Background Knowledge (Read This First!)

### What is EXIF Metadata?

EXIF metadata stores extra information about image files — camera info, date/time, GPS, copyright, and custom fields. The flag in this challenge was hidden inside the **License** metadata field as Base64.

### What is Base64?

Base64 converts binary data into readable ASCII text. It ends with `=` or `==` and only uses letters, numbers, `+`, `/`.

---

## Solution — Step by Step

### Step 1 — Download the File

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Examine the File with exiftool

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool cat.jpg
ExifTool Version Number         : 13.36
File Name                       : cat.jpg
...
License                         : cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9
Rights                          : PicoCTF
...
```

The **License** field contains a suspicious Base64-encoded string!

### Step 3 — Decode the Base64 String

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9" | base64 -d
picoCTF{the_m3tadata_1s_modified}
```

Got the flag! 🎯

---

## Alternative Methods

**Method 1 — strings Command**
```bash
strings cat.jpg | grep pico
```

**Method 2 — One-liner**
```bash
exiftool cat.jpg | grep License | cut -d: -f2 | xargs echo | base64 -d
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `exiftool` | Read EXIF metadata |
| `base64` | Decode Base64 strings |

---

## Key Takeaways

- Always check metadata in forensics challenges
- `exiftool` is essential for examining image files
- Base64 encoding is commonly used to hide data in metadata text fields
- Images can contain hidden information completely beyond what is visible
- Always strip metadata from photos before sharing publicly
