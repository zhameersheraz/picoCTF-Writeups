# Riddle Registry - picoCTF Writeup

**Challenge:** Riddle Registry  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{puzzl3d_m3tadata_f0und!_3578739a}`

---

## Description
This challenge provides a PDF file that appears to contain only garbled nonsense.  
The description hints that there's a hidden treasure within the file's metadata.  
The goal is to extract and decode the hidden flag from the PDF's metadata.

---

## Approach
The challenge name "Riddle Registry" and the mention of metadata suggested that the flag might be hidden in the PDF's file properties rather than the visible content.  
PDF files can store metadata like author name, creation date, and custom fields — perfect places to hide information.

---

## Solution

### Step 1: Download the PDF
I downloaded `confidential.pdf` from the challenge link provided.

### Step 2: Extract Metadata
I used `exiftool` to examine hidden metadata:
```bash
exiftool confidential.pdf
```

**Key Output:**
```
Author: cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV8zNTc4NzM5YX0=
```

The Author field immediately stood out. It looked like encoded data, not a normal name.

### Step 3: Identify the Encoding
The string appeared to be **Base64** because:
* Ends with `=` (padding character)
* Only contains A-Z, a-z, 0-9, +, /, =
* No spaces or unusual characters

Base64 is common in CTFs for hiding text because it's reversible and looks obfuscated.

### Step 4: Decode Base64
```bash
echo "cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV8zNTc4NzM5YX0=" | base64 -d
```

**Result:**
```
picoCTF{puzzl3d_m3tadata_f0und!_3578739a}
```

---

## Why This Works
PDF files store metadata separately from visible content. This metadata is often overlooked but can be extracted with tools like `exiftool`.

In this challenge, the flag was Base64-encoded in the Author field to make it less obvious.

---

## Common Encoding Types Reference

| Encoding Type | Characteristics | Example | Decode Command |
|--------------|----------------|---------|----------------|
| Base64 | A-Z, a-z, 0-9, +, /, = | `cGljb0NURg==` | `echo "..." \| base64 -d` |
| Hexadecimal | 0-9, a-f (or A-F) | `70696e6f435446` | `echo "..." \| xxd -r -p` |
| MD5 Hash | 32 hex characters | `5f4dcc3b5aa765d61d8327deb882cf99` | (cannot decode - one-way hash) |
| Binary | Only 0 and 1 | `01101001` | Custom conversion |
| ROT13 | Letters shifted 13 positions | `cvpbPG` | `echo "..." \| tr 'A-Za-z' 'N-ZA-Mn-za-m'` |
| URL Encoding | Contains %XX codes | `pico%20CTF` | `urldecode` |

---

## Tools Used
* `exiftool` - Extracts metadata from files
* `base64` - Decodes Base64 strings

---

## Key Takeaways
* Always check metadata in forensics challenges.
* Learn to recognize common encodings like Base64, Hex, and ROT13.
* PDF files can hide data in metadata fields not visible normally.
* Forensics is about looking beyond the obvious.
