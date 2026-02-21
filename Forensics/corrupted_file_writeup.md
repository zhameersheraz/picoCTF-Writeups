# Corrupted file - picoCTF Writeup

**Challenge:** Corrupted file  
**Category:** Forensics  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{r3st0r1ng_th3_by73s_684e09bc}`

---

## Description

This file seems broken... or is it? Maybe a couple of bytes could make all the difference. Can you figure out how to bring it back to life?

Download the file here.

---

## Hints

1. Try checking the file's header.
2. JPEG
3. Tools like xxd or hexdump can help you inspect and edit file bytes.

---

## Solution

### Step 1: Check what type of file this is

I downloaded the file and tried to see what it was:

```bash
cd /media/sf_downloads

file file
```

**Output:**
```
file: data
```

So it's just showing as data, which means it's corrupted. I also tried:

```bash
exiftool file
```

**Output:**
```
Error: Unknown file type
```

Yeah, definitely corrupted.

### Step 2: Open it in a hex editor

The hint says to check the file's header, so I went to https://hexed.it/ and uploaded the file there.

Looking at the hex values, I saw this at the beginning:

```
00000000: 00 00 00 00
00000004: 5C 78 FF E0
```

### Step 3: Figure out what file type it should be

I Googled "JPEG file signature" because from the file size (8.8 KB) and context, it seemed like it might be an image.

JPEG files should start with these bytes:
```
FF D8 FF E0
```

But the file had:
```
5C 78 FF E0
```

So the first two bytes (`5C 78`) are wrong! They should be `FF D8`.

### Step 4: Fix the bytes in hexed.it

In hexed.it, I:
1. Found the bytes at offset `00000004`
2. Changed `5C` to `FF`
3. Changed `78` to `D8`

Now it looks like:
```
00000004: FF D8 FF E0
```

### Step 5: Export and open the file

I clicked "Export" in hexed.it and saved it. Then I renamed it to `file.jpg`:

```bash
mv file file.jpg

file file.jpg
```

**Output:**
```
file.jpg: JPEG image data, JFIF standard 1.01
```

It works! I opened the image and saw the flag:

```
picoCTF[r3st0r1ng_th3_by73s_684e09bc]
```

Changed the `[` to `{` and `]` to `}` to get the proper flag format.

---

## Why This Works

Every file type has a "magic number" or signature at the start. JPEG files always start with `FF D8`. The challenge corrupted these bytes to `5C 78`, making the file unrecognizable. By fixing just these 2 bytes, the whole file becomes readable again.

---

## Common File Signatures

| File Type | Magic Bytes (Hex) |
|-----------|-------------------|
| JPEG | `FF D8 FF E0` or `FF D8 FF E1` |
| PNG | `89 50 4E 47 0D 0A 1A 0A` |
| GIF | `47 49 46 38 39 61` |
| PDF | `25 50 44 46` |
| ZIP | `50 4B 03 04` |

---

## Terminal Commands

```bash
# Check file type
file file

# After fixing in hexed.it, rename it
mv file file.jpg

# Verify it worked
file file.jpg

# Open the image
xdg-open file.jpg
```

---

## Flag

`picoCTF{r3st0r1ng_th3_by73s_684e09bc}`

---

## Tools Used

* https://hexed.it/ - Online hex editor
* `file` - Check file types
* `exiftool` - Check file metadata

---

## Key Takeaways

* Always check the file header when dealing with corrupted files
* JPEG files must start with `FF D8`
* Hex editors let you modify individual bytes
* Sometimes just 2 bytes can make the difference between a broken file and a working one
