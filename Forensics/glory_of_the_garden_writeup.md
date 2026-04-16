# Glory of the Garden — picoCTF Writeup

**Challenge:** Glory of the Garden  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{more_than_m33ts_the_3y339cbe6dc}`  

---

## Description

> This file contains more than it seems.
> Get the flag from garden.jpg.

**Hint shown in challenge:** `What is a hex editor?`

---

## Background Knowledge (Read This First!)

### How can text hide inside an image?

Image files (JPEG, PNG, etc.) are binary files. Text can be appended or embedded inside the binary data **without affecting how the image looks** when opened. The `strings` command extracts all printable ASCII strings from any file — revealing hidden messages.

### What is Steganography?

Steganography is the practice of hiding data inside other files. In this challenge, text was simply appended to the end of a JPEG image — a basic but effective hiding technique.

---

## Solution — Step by Step

### Step 1 — Download and Check the File

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file garden.jpg
garden.jpg: JPEG image data, JFIF standard 1.01
```

It's a valid JPEG image file.

### Step 2 — Use strings Command

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings garden.jpg | grep "picoCTF"
Here is a flag: picoCTF{more_than_m33ts_the_3y339cbe6dc}
```

Got the flag! 🎯

---

## Alternative Method — Hex Editor

As the hint suggests, you could also use a hex editor:

1. Open `garden.jpg` in a hex editor (https://hexed.it/)
2. Scroll to the **bottom** of the file
3. Look for readable ASCII text near the end
4. Find: `Here is a flag: picoCTF{more_than_m33ts_the_3y339cbe6dc}`

```bash
# Using hexedit
hexedit garden.jpg
# Press Ctrl+E to go to the end and look for readable text
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Check file type |
| `strings` | Extract printable strings from binary files |
| Hex editor (optional) | View raw binary data |

---

## Key Takeaways

- Image files can contain hidden text data that is invisible when the image is opened normally
- The `strings` command is essential for forensics challenges
- Always check files for embedded text even if they appear to be just images
- Steganography often hides data at the end of files
- Always try basic commands (`strings`, `file`, `exiftool`) before complex tools
- The flag name references "more than meets the eye" — also a Transformers reference 🤖
