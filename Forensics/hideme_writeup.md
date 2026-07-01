# hideme — picoCTF Writeup

**Challenge:** hideme  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{Hiddinng_An_imag3_within_@n_ima9e_82101824}`  
**Platform:** picoCTF (2023)  
**Writeup by:** zham  

---

## Description

> Every file gets a flag.
>
> The SOC analyst saw one image been sent back and forth between two people. They decided to investigate and found out that there was more than what meets the eye here.

**Hint shown in challenge:** `steganography`

---

## Background Knowledge (Read This First!)

### What is File Appending / Concatenation?

A PNG file ends with a special chunk called `IEND`. Image viewers stop reading as soon as they hit `IEND`, so **anything written after it is ignored** by the viewer but still lives inside the file.

This means an attacker can literally glue a second file (a ZIP, another image, even an executable) to the end of a PNG. The image still opens normally, but the hidden file is sitting right there in the bytes.

### What is binwalk?

`binwalk` is a forensics tool that scans a file for **embedded file signatures** (also called magic bytes). It can tell you "hey, your PNG is actually a PNG + a ZIP archive in one file" by matching known patterns.

Useful flags:
- `binwalk <file>` — scan and list embedded signatures with their byte offsets
- `binwalk -e <file>` — extract everything it finds into a folder

### The IEND Trick in Practice

A normal PNG looks like this in bytes:

```
[PNG header] [pixel data...] [IEND chunk] [EOF]
```

An "appended" PNG looks like this:

```
[PNG header] [pixel data...] [IEND chunk] [ZIP data...] [ZIP EOF]
```

The viewer reads up to `IEND` and calls it done. The ZIP lives in the dead space, invisible until you scan for it.

---

## Solution — Step by Step

### Step 1 — Download the Challenge File

I grabbed `flag.png` from the challenge link and moved into my downloads folder.

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ wget https://artifacts.picoctf.net/c/259/flag.png
--2026-07-01 12:34:56--  https://artifacts.picoctf.net/c/259/flag.png
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 172.64.155.42
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|172.64.155.42|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 43098 (42K) [image/png]
Saving to: 'flag.png'

flag.png              100%[===================>]  42.10K  --.-K/s    in 0.05s

2026-07-01 12:35:01 (786 KB/s) - 'flag.png' saved [43098/43098]
```

### Step 2 — Confirm the File Type

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file flag.png
flag.png: PNG image data, 512 x 504, 8-bit/color RGBA, non-interlaced
```

Looks like a normal PNG. The plot thickens later.

### Step 3 — Open the Image (Spoiler: It's Just the picoCTF Logo)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ eog flag.png
```

Just the picoCTF logo on a white background. No flag text in the image itself. The description said "more than what meets the eye" — so something is hiding behind it.

### Step 4 — Scan for Hidden Content with binwalk

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ binwalk flag.png

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 512 x 504, 8-bit/color RGBA, non-interlaced
41            0x29            Zlib compressed data, compressed
39739         0x9B3B          Zip archive data, at least v1.0 to extract, name: secret/
39804         0x9B7C          Zip archive data, at least v2.0 to extract, compressed size: 2997, uncompressed size: 3152, name: secret/flag.png
43036         0xA81C          End of Zip archive, footer length: 22
```

There it is. A ZIP archive starting at byte 39739, with a `secret/flag.png` tucked inside. The image is only ~43KB but the ZIP keeps going for another ~3KB past the visible PNG — that's the hidden payload.

### Step 5 — Extract the Embedded Files

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ binwalk -e flag.png

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 512 x 504, 8-bit/color RGBA, non-interlaced
41            0x29            Zlib compressed data, compressed
39739         0x9B3B          Zip archive data, at least v1.0 to extract, name: secret/
39804         0x9B7C          Zip archive data, at least v2.0 to extract, compressed size: 2997, uncompressed size: 3152, name: secret/flag.png
43036         0xA81C          End of Zip archive, footer length: 22
```

`binwalk` created a new folder with the carved-out pieces.

### Step 6 — List the Extracted Folder

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -la _flag.png.extracted/
total 12
drwxr-xr-x 3 zham zham 4096 Jul  1 12:36 .
drwxr-xr-x 2 zham zham 4096 Jul  1 12:36 29
drwxr-xr-x 2 zham zham 4096 Jul  1 12:36 29.zlib
-rw-r--r-- 1 zham zham 3298 Jul  1 12:36 9B3B.zip
drwxr-xr-x 2 zham zham 4096 Jul  1 12:36 secret
```

### Step 7 — Find the Hidden flag.png

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -la _flag.png.extracted/secret/
total 8
drwxr-xr-x 2 zham zham 4096 Jul  1 12:36 .
drwxr-xr-x 3 zham zham 4096 Jul  1 12:36 ..
-rw-r--r-- 1 zham zham 3152 Jul  1 12:36 flag.png
```

A second `flag.png` is sitting inside `secret/`. This is the real flag carrier.

### Step 8 — Open the Hidden Image

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ eog _flag.png.extracted/secret/flag.png
```

The second image is plain white with the flag text printed on it.

### Step 9 — Read the Flag (Optional, for the Terminal-Only Crowd)

If you want the flag without opening a GUI, run OCR on the carved image. Tesseract works great for this.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt install tesseract-ocr -y
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ tesseract _flag.png.extracted/secret/flag.png stdout
picoCTF{Hiddinng_An_imag3_within_@n_ima9e_82101824}
```

The flag is: `picoCTF{Hiddinng_An_imag3_within_@n_ima9e_82101824}`

---

## Alternative Solve — Manual Carving with `dd` + `unzip`

If you don't have `binwalk` (or want to understand what's really happening under the hood), you can carve the ZIP out by hand using `dd`.

### Step 1 — Slice the ZIP Out by Byte Offset

`binwalk` told us the ZIP starts at offset 39739 and the original PNG is 43098 bytes. We just slice from 39739 to EOF.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ dd if=flag.png of=extracted.zip bs=1 skip=39739
3298+0 records in
3298+0 records out
3298 bytes (3.3 kB, 3.2 KiB) copied, 0.013 s, 254 kB/s
```

### Step 2 — Unzip the Carved File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip extracted.zip
Archive:  extracted.zip
   creating: secret/
  inflating: secret/flag.png
```

### Step 3 — Open the Inner flag.png

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ eog secret/flag.png
```

Same flag as before. Both methods work — `binwalk` is just the automated shortcut.

---

## What Happened Internally (Timeline)

1. **Challenge author crafted the bait image** — a normal 512x504 picoCTF logo PNG (~42KB).
2. **They generated a second image** containing the flag text on a white background (~3KB).
3. **They zipped the second image** into `secret/flag.png.zip`, producing a valid ZIP archive.
4. **They appended the ZIP to the end of the PNG** using a simple `cat` or similar concatenation. The PNG's `IEND` marker is at byte ~39738, and the ZIP data starts immediately after.
5. **Image viewers ignore the ZIP** because they stop reading at `IEND`. The file looks and acts like a normal PNG.
6. **binwalk scans for magic bytes** — the ZIP's local file header signature `PK\x03\x04` (hex `50 4B 03 04`) gives it away at offset 0x9B3B.
7. **binwalk -e extracts the ZIP**, dropping the inner `secret/flag.png` into a folder.
8. **Opening the carved flag.png** reveals the flag as overlay text on a white background.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the challenge file from the picoCTF artifacts server |
| `file` | Confirm the file is really a PNG (and not something else with a .png extension) |
| `eog` | Open the image in a GUI to see what the attacker wants you to see |
| `binwalk` | Scan the PNG for embedded file signatures and list their byte offsets |
| `binwalk -e` | Carve out (extract) every embedded file binwalk found |
| `ls -la` | Browse the extracted folder and find the hidden `flag.png` |
| `tesseract` (alt) | OCR the flag text out of the inner image without opening a GUI |
| `dd` (alt) | Manually carve the ZIP out of the PNG using the known byte offset |
| `unzip` (alt) | Unpack the carved ZIP archive |

---

## Key Takeaways

- **PNG files end at the `IEND` chunk.** Anything after it is ignored by image viewers but still part of the file. This makes PNGs a favorite cover for hiding extra payloads.
- **`binwalk` is the go-to tool** for finding embedded files inside other files. Always run it as a first step when an image "looks too normal" in a forensics challenge.
- **The `file` command only reports the outer format.** A file can be a PNG *and* a ZIP at the same time. Trust binwalk, not extensions.
- **Manual carving with `dd` + `unzip` is the fallback** when binwalk isn't available, and it teaches you exactly what's happening at the byte level.
- **The hint "steganography" + "more than meets the eye"** was pointing at file-in-file hiding, not pixel-level stego (LSB / steghide). Read the description carefully — different stego challenges need different tools.
- **Real-world connection:** malware and data exfiltration kits regularly append encrypted blobs to images posted on social media or attached to emails. SOC analysts use exactly this binwalk workflow to detect them.

**Flag wordplay decoded:** `Hiddinng_An_imag3_within_@n_ima9e` = "Hiding An image within an image" — written in leet-speak (`3` → `e`, `9` → `g`, `@` → `a`) and with a typo on "Hiding" (extra `n`). The challenge name `hideme` was telling us the whole time.
