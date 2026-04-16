# Riddle Registry — picoCTF Writeup

**Challenge:** Riddle Registry  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{puzzl3d_m3tadata_f0und!_3578739a}`  

---

## Description

> This challenge provides a PDF file that appears to contain only garbled nonsense. The description hints that there's a hidden treasure within the file's metadata. The goal is to extract and decode the hidden flag from the PDF's metadata.

---

## Background Knowledge (Read This First!)

### What is PDF Metadata?

PDF files store metadata separately from visible content. This includes fields like Author, Title, Creator, and more. This metadata is often overlooked but can be extracted with tools like `exiftool`.

### How to Identify Base64?

Base64 strings:
- End with `=` (padding character)
- Only contain A-Z, a-z, 0-9, +, /, =
- No spaces or unusual characters

---

## Solution — Step by Step

### Step 1 — Download the PDF

I downloaded `confidential.pdf` from the challenge link provided.

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Extract Metadata

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool confidential.pdf
...
Author: cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV8zNTc4NzM5YX0=
...
```

The Author field immediately stood out — it looked like Base64, not a real name.

### Step 3 — Decode Base64

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV8zNTc4NzM5YX0=" | base64 -d
picoCTF{puzzl3d_m3tadata_f0und!_3578739a}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `exiftool` | Extract metadata from files |
| `base64` | Decode Base64 strings |

---

## Key Takeaways

- Always check metadata in forensics challenges
- Learn to recognize common encodings like Base64, Hex, and ROT13
- PDF files can hide data in metadata fields not visible when opening the document normally
- Forensics is about looking beyond the obvious
