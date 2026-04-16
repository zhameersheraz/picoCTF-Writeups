# Corrupted File — picoCTF Writeup

**Challenge:** Corrupted File  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{r3st0r1ng_th3_by73s_684e09bc}`  

---

## Description

> This file seems broken... or is it? Maybe a couple of bytes could make all the difference. Can you figure out how to bring it back to life?
> Download the file here.

**Hints shown in challenge:**
1. Try checking the file's header.
2. JPEG
3. Tools like xxd or hexdump can help you inspect and edit file bytes.

---

## Background Knowledge (Read This First!)

### What is a File Header / Magic Bytes?

Every file type has a **magic number** — a specific sequence of bytes at the very beginning that identifies what type of file it is. Programs use these to decide how to open the file, not just the extension.

### JPEG Magic Bytes

JPEG files must always start with:
```
FF D8 FF E0
```

If these bytes are wrong, the file is unrecognizable and shows as "data".

### Common File Signatures

| File Type | Magic Bytes (Hex) |
|-----------|-------------------|
| JPEG | `FF D8 FF E0` |
| PNG | `89 50 4E 47 0D 0A 1A 0A` |
| GIF | `47 49 46 38 39 61` |
| PDF | `25 50 44 46` |
| ZIP | `50 4B 03 04` |

---

## Solution — Step by Step

### Step 1 — Check the File Type

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file file
file: data

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool file
Error: Unknown file type
```

The file is corrupted — unrecognizable.

### Step 2 — Open in a Hex Editor

I went to https://hexed.it/ and uploaded the file. Looking at the hex values at the beginning:

```
00000000: 00 00 00 00
00000004: 5C 78 FF E0
```

The file had `5C 78` where it should have `FF D8`.

### Step 3 — Fix the Bytes

In hexed.it, I:

1. Found the bytes at offset `00000004`
2. Changed `5C` → `FF`
3. Changed `78` → `D8`

Now it looks like:
```
00000004: FF D8 FF E0
```

### Step 4 — Export and Verify

I clicked **Export** in hexed.it and saved it. Then:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ mv file file.jpg

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file file.jpg
file.jpg: JPEG image data, JFIF standard 1.01
```

### Step 5 — Open and Read the Flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xdg-open file.jpg
```

The image displayed the flag with `[` and `]` instead of `{` and `}` — corrected to proper picoCTF format.

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| https://hexed.it/ | Online hex editor to fix the file header |
| `file` | Check file type |
| `exiftool` | Check file metadata |
| `xdg-open` | Open the fixed image |

---

## Key Takeaways

- Always check the file header when dealing with corrupted files
- JPEG files must start with `FF D8`
- Hex editors let you modify individual bytes
- Sometimes just 2 bytes can make the difference between a broken file and a working one
- File extensions don't matter — the magic bytes determine the actual file type
