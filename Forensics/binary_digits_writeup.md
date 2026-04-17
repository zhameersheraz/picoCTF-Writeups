# Binary Digits — picoCTF Writeup

**Challenge:** Binary Digits  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{h1dd3n_1n_th3_b1n4ry_3d2e65ba}`  

---

## Description

> This file doesn't look like much... just a bunch of 1s and 0s. But maybe it's not just random noise. Can you recover anything meaningful from this?
> Download the file here.

**Attachment:** `digits.bin`

---

## Background Knowledge (Read This First!)

### What is inside the file?

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool digits.bin
File Type    : TXT
MIME Type    : text/plain
Line Count   : 1
Word Count   : 1
```

Running `strings digits.bin` reveals a single long string of only 1s and 0s — raw binary text.

### What is binary encoding?

Binary encoding is how computers represent data at the lowest level. Every character has an ASCII value, and that value can be written as an 8-bit binary number. For example:

| Character | ASCII | Binary |
|-----------|-------|--------|
| `p` | 112 | `01110000` |
| `J` | 74 | `01001010` |

If someone saves a file — like a JPEG image — and writes out every byte as its 8-digit binary representation, the result looks like meaningless 1s and 0s. That is exactly what `digits.bin` is.

### The Trick — A JPEG Hidden as Binary Text

The first 4 bytes of any JPEG file are always `FF D8 FF E0`. Converting those to binary gives:

```
11111111 11011000 11111111 11100000
```

The binary string inside `digits.bin` starts with exactly this pattern — confirming the entire file is a **JPEG image encoded as ASCII binary text**.

---

## Solution — Step by Step

### Step 1 — Inspect the file

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool digits.bin
File Type    : TXT

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings digits.bin
1111111111011000111111111110000000000000...
```

It's one long binary string — no spaces, no newlines.

### Step 2 — Create the decode script using nano

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano solve.py
```

Type the following inside nano:

```python
with open('digits.bin', 'r') as f:
    binary_text = f.read().strip()

data = bytearray()
for i in range(0, len(binary_text) // 8 * 8, 8):
    data.append(int(binary_text[i:i+8], 2))

with open('output.jpg', 'wb') as f:
    f.write(data)

print(f"Saved {len(data)} bytes to output.jpg")
```

Save and exit nano:

- Press `Ctrl + O` → then `Enter` to confirm the filename
- Press `Ctrl + X` to exit

### Step 3 — Run the script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 solve.py
Saved 8875 bytes to output.jpg
```

### Step 4 — Open the image

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xdg-open output.jpg
```

The image displays the flag in plain text:

```
picoCTF{h1dd3n_1n_th3_b1n4ry_3d2e65ba}
```

✅ Got the flag! 🎯

---

## Alternative — CyberChef (No Coding Required!)

You can solve this in your browser using [CyberChef](https://gchq.github.io/CyberChef/) with just two operations:

1. Go to [CyberChef](https://gchq.github.io/CyberChef/)
2. Add the **"From Binary"** operation — set Delimiter to `None`, Byte Length to `8`
3. Add the **"Render Image"** operation — set Input Format to `Raw`
4. Paste the contents of `digits.bin` into the Input box
5. Click **BAKE!**

The flag appears rendered directly in the Output panel. No terminal, no Python, no installation needed — pure browser magic. 🧙

---

## Why It Works

The binary string in `digits.bin` is the raw bytes of a JPEG image written out as 1s and 0s, 8 characters per byte. Converting them back in groups of 8 reconstructs the original JPEG perfectly. The image itself contains the flag written as visible text.

The key insight: **the file extension `.bin` is a red herring**. `exiftool` correctly identifies it as a plain text file. The real question is — what does the text *represent*? The JPEG magic bytes at the start (`FFD8FFE0`) answer that immediately.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `exiftool` | Identify the true file type | ⭐ Easy |
| `strings` | View the raw binary text content | ⭐ Easy |
| `nano` | Write the Python decode script | ⭐ Easy |
| Python `int(x, 2)` | Convert 8-bit binary chunks to bytes | ⭐ Easy |
| `xdg-open` | Open and read the decoded JPEG | ⭐ Easy |
| CyberChef (alternative) | From Binary → Render Image in browser | ⭐ Easy |

---

## Key Takeaways

- **File extensions lie** — always verify with `file` or `exiftool`; `.bin` here is just a plain text file
- **Binary strings encode real data** — 8 bits at a time maps directly to ASCII/bytes
- **JPEG magic bytes** (`FFD8FFE0`) are a fast way to identify a hidden image before decoding
- **CyberChef's "From Binary" + "Render Image"** solves this entire challenge in two clicks
- When you see a file full of only 1s and 0s, always ask: *what are these bits actually encoding?*
