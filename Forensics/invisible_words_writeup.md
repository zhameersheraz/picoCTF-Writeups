# Invisible WORDs — picoCTF Writeup

**Challenge:** Invisible WORDs  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 300  
**Flag:** `picoCTF{w0rd_d4wg_y0u_f0und_5h3113ys_m4573rp13c3_e4f8c8f0}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Do you recognize this cyberpunk baddie? We don't either. AI art generators are all the rage nowadays, which makes it hard to get a reliable known cover image. But we know you'll figure it out. The suspect is believed to be trafficking in classics. That probably won't help crack the stego, but we hope it will give motivation to bring this criminal to justice!
>
> Download the image [here](https://artifacts.picoctf.net/c/403/output.bmp)

## Hints

> 1. Something doesn't quite add up with this image...
> 2. How's the image quality?

---

## Background Knowledge (Read This First!)

### What is a BMP file?

A `.bmp` (Windows Bitmap) is one of the simplest image formats — the file is literally a header, then rows of raw pixel data. For a 32-bit BMP, each pixel takes **4 bytes** in **BGRA** order: **B**lue, **G**reen, **R**ed, **A**lpha (transparency). For most BMPs, the alpha channel is unused (always `0xFF` = fully opaque) — so 3 of the 4 bytes per pixel are visually meaningful, and 1 is throwaway.

The BMP header stores the byte offset of the pixel array at bytes `0x0A`–`0x0D` (a 32-bit little-endian integer). For this file it is `0x8A` = 138. The first 138 bytes are the header — skip them when you walk the pixel data.

### What is a ZIP file?

A ZIP file starts with the magic bytes `PK\x03\x04` (`P` `K` = Phil Katz, the inventor of the format). A ZIP archive can contain one or more files that are usually **deflate-compressed** and recovered by tools like `unzip`, `7z`, or `binwalk -e`.

### What is binwalk?

`binwalk` is a tool for **scanning binary files for embedded file signatures**. It walks the file, looks for known magic bytes (PNG, ZIP, PDF, JPEG, etc.), and reports each match's offset. With `binwalk -e` it will also **carve out** every embedded file it finds and save them into a directory called `_<filename>.extracted/`. It is the go-to first move for any "something is hidden in this binary blob" challenge.

### Putting it all together for this challenge

The challenge hands you a 32-bit BMP. The visible cyberpunk character is just cover art — but the **bottom rows of the red and alpha channels are not normal image data**. They contain the bytes of a ZIP archive. The blue and green channels are clean. The challenge title "Invisible WORDs" is a hint: the *words* (text) are hidden in the R and A bytes of each pixel, where your eye never looks. To recover the flag:

1. Open the BMP and walk the pixel array.
2. For each pixel, **skip the first 2 bytes** (Blue and Green) and **keep the last 2 bytes** (Red and Alpha — the hidden payload).
3. Concatenate all those bytes into a file.
4. Use `binwalk` (or scan manually for `PK\x03\x04`) to find the embedded ZIP.
5. Unzip the archive.
6. The archive contains a text file (the full text of *Frankenstein* by Mary Shelley) — `grep picoCTF` on it and the flag is on one of the lines.

**Critical gotcha:** the picoctfsolutions.com writeup and the YouTube walkthrough disagree on which 2 bytes to keep. The picoctfsolutions script `out.append(data[i]); out.append(data[i+1])` (keeps B and G) produces a non-ZIP blob for this challenge. The YouTube walkthrough script `skip=f.read(2); keep=f.read(2); g.write(keep); skip=f.read(2)` (keeps R and A — bytes 2 and 3 of each 4-byte pixel) is the one that works. Always try both if the first attempt doesn't yield a valid ZIP.

---

## Solution — Step by Step

I solved this on Windows + Python, but it works identically on Kali.

### Step 1 — Get the image

The challenge page's "here" link points to `https://artifacts.picoctf.net/c/403/output.bmp` (the documented path `c/371/Invisible WORDs.bmp` may 403 depending on the CDN's false-positive block — try both, see the troubleshooting section below).

Use the picoCTF in-browser **Webshell** (the green button on the left sidebar of the challenge page) to download the file. The webshell has direct CDN access that `wget` from your own machine often does not. Once the file is on your local machine, place it in your working directory as `output.bmp`.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -la output.bmp
-rwxrwxrwx 1 root root 2073738 Jul 23 23:42 output.bmp
```

It should be about **2 MB** (2,073,738 bytes for the 960×540 version, or 2,211,840 bytes for the 1024×540 version — both are valid depending on which instance you got).

### Step 2 — Confirm the file is a 32-bit BMP

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file output.bmp
output.bmp: PC bitmap, Windows 98/2000 and newer format, 960 x 540 x 32, cbSize 2073738, bits offset 138
```

Key facts to note:
- `960 x 540` (or `1024 x 540` for some instances) — the image dimensions
- `32` — bits per pixel (so 4 bytes per pixel, BGRA layout)
- `bits offset 138` — the pixel array starts at byte 138

### Step 3 — Look at the color channels

The hint says "something doesn't quite add up with this image" and "How's the image quality?". Open the BMP in any image tool that lets you view per-channel data (CyberChef "Split Colour Channels", GIMP, Photoshop, stegsolve.jar) and compare the channels:

- **Blue channel** — looks normal everywhere, smooth gradients
- **Green channel** — looks normal everywhere, smooth gradients
- **Red channel** — normal at the top, but the **bottom rows are random noise** ← hidden data
- **Alpha channel** — same as red: normal at the top, noise at the bottom ← hidden data

The red and alpha channels are where the stego lives. The pixel data is in **BGRA** order, so for each pixel, the byte at offset `i` is the blue byte, the byte at `i+1` is the green byte, the byte at `i+2` is the red byte, and the byte at `i+3` is the alpha byte. We **skip the first 2** (B and G) and **keep the last 2** (R and A).

### Step 4 — Extract red and alpha bytes with Python

Save this as `extract.py` (this is the exact script from the picoCTF 2023 official walkthrough video by Martin Carlisle — confirmed working with the actual challenge file):

```python
g = open("output_ra.zip", "wb")
with open("output.bmp", "rb") as f:
    hdr = f.read(0x8a)        # skip the 138-byte BMP header
    skip = f.read(2)          # skip the first 2 bytes of each 4-byte pixel group (B, G)
    while skip:
        keep = f.read(2)      # keep the next 2 bytes (R, A — the hidden payload)
        g.write(keep)
        skip = f.read(2)      # skip the next 2 bytes
g.close()
```

Run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 extract.py
```

This writes a file called `output_ra.zip` to your working directory. Each 4-byte group in the pixel array (B, G, R, A) has the first 2 bytes skipped and the last 2 bytes written — the kept half (R and A) is the hidden ZIP payload.

### Why this works

The 960×540 image has `960 * 540 = 518,400` pixels. Each pixel is 4 bytes (BGRA), so the pixel array is `518,400 * 4 = 2,073,600` bytes. We keep 2 of every 4 bytes → `2,073,600 / 2 = 1,036,800` bytes of output. The 2 bytes we drop are visual cover (the eye barely notices if you mess with two channels of a four-channel image), and the 2 bytes we keep are the hidden ZIP.

### Step 5 — Unzip and read the embedded file

The Python script wrote `output_ra.zip` directly, so you can unzip it right away:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip output_ra.zip
Archive:  output_ra.zip
  inflating: ZnJhbmtlbnN0ZWluLXRlc3QudHh0

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls
output.bmp  extract.py  output_ra.zip  ZnJhbmtlbnN0ZWluLXRlc3QudHh0
```

The unzipped file is `ZnJhbmtlbnN0ZWluLXRlc3QudHh0`. That is just the filename **base64-encoded**: decode it to see the real name:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo ZnJhbmtlbnN0ZWluLXRlc3QudHh0 | base64 -d
frankenstein-test.txt
```

So the embedded file is `frankenstein-test.txt` — the full text of *Frankenstein* by Mary Shelley (the "trafficking in classics" line in the description was the hint that the hidden content is a public-domain classic novel).

### Step 6 — Find the flag

The flag is somewhere inside the 448,642-byte Frankenstein text. `grep` for it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep -n "picoCTF" ZnJhbmtlbnN0ZWluLXRlc3QudHh0
<picoCTF{w0rd_d4wg_y0u_f0und_5h3113ys_m4573rp13c3_e4f8c8f0}>
```

**That is the flag.**

### Step 7 — Submit

Paste the flag into the picoCTF challenge page and submit.

---

## Alternative Method (use binwalk instead of the loop)

The YouTube script above does the extraction and writing in one pass — but if you want to inspect the raw extracted bytes before unzipping, you can write them to `output.bin` instead and let `binwalk` find the ZIP for you:

```python
g = open("output.bin", "wb")
with open("output.bmp", "rb") as f:
    hdr = f.read(0x8a)
    skip = f.read(2)
    while skip:
        keep = f.read(2)
        g.write(keep)
        skip = f.read(2)
g.close()
```

Then:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ binwalk -e output.bin
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Zip archive data, at least v2.0 to extract, compressed size: 169391, uncompressed size: 448642, name: ZnJhbmtlbnN0ZWluLXRlc3QudHh0
```

`binwalk` carves the embedded archive into `_output.bin.extracted/`, and the file is the same `ZnJhbmtlbnN0ZWluLXRlc3QudHh0` (base64 for `frankenstein-test.txt`). Either path lands you on the same flag.

---

## What Happened Internally (Timeline)

1. The challenge author generated a cyberpunk cover image with an AI art tool (Stable Diffusion / Midjourney / DALL-E). AI-generated cover art is important because it means there is no "known cover" you can compare against — you cannot tell what bytes were changed.
2. The author took the novel *Frankenstein* by Mary Shelley (Project Gutenberg text) and **embedded it as a ZIP archive** in the **red and alpha channel bytes** of the bottom rows of the image. They did this by taking each ZIP byte and writing it into the 3rd and 4th byte of consecutive 4-byte pixels in the BGRA pixel array.
3. The blue and green bytes (positions 0, 1 of each pixel) of the same rows were left as normal image data so the cover still looks like a cyberpunk character. The stego lives only in R and A.
4. The author packaged the file as a 32-bit BMP (uncompressed, raw pixels) and uploaded it to `artifacts.picoctf.net/c/403/output.bmp`.
5. We downloaded the BMP, opened it in an image tool, and noticed the obvious noise pattern in the bottom of the red and alpha channels (hint 1: "something doesn't quite add up with this image") and the degraded image quality (hint 2: "how's the image quality?" — the noisy R/A bytes mess up the color fidelity).
6. We extracted the R and A channel bytes (skipping B and G) with the YouTube walkthrough script.
7. We ran `unzip` (or `binwalk -e`) on the extracted blob — both gave us a text file `ZnJhbmtlbnN0ZWluLXRlc3QudHh0` containing all 448,642 bytes of Mary Shelley's *Frankenstein*.
8. We `grep`'d for `picoCTF{` and found the flag on one of the lines (it appears mid-text, embedded in a Frankenstein sentence).

The challenge title "Invisible WORDs" is two clues at once: (1) the *words* (text of Frankenstein) are hidden in the *invisible* channel bytes, and (2) the hidden content is a "WORDs" (Microsoft Word / document-style) text — i.e., a plain `.txt` file. The "trafficking in classics" line in the description is the hint that the hidden content is a public-domain classic novel.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Confirm the BMP is a 32-bit BGRA image of the expected dimensions |
| Python 3 (read + write) | Read the BMP, walk the pixel array from offset `0x8a` (138), skip 2 of every 4 bytes (B, G), keep the last 2 (R, A), write straight to `output_ra.zip` |
| `unzip` | Extract the embedded ZIP (it contains `ZnJhbmtlbnN0ZWluLXRlc3QudHh0`, base64 for `frankenstein-test.txt`) |
| `base64 -d` | Decode the embedded filename to learn the real name |
| `grep` | Find the `picoCTF{` line in the 448,642-byte Frankenstein text |
| `binwalk` (alt) | Scan the extracted blob for the ZIP signature at offset 0 if you want a non-loop extraction path |

---

## Key Takeaways

- **The hidden channel trick is the core of this challenge.** With a 32-bit BMP, 3 of the 4 bytes per pixel are visible (B, G, R) and 1 is throwaway (A). The author used the last 2 bytes of each 4-byte pixel group (R and A in the BGRA layout) as the hidden carrier and the first 2 bytes (B and G) as the visual cover. This works because the human eye is much more sensitive to *luminance* than to *individual channel* values, so writing the ZIP into the R and A bytes does not visibly change the image.
- **binwalk is the swiss-army knife of CTF forensics.** Whenever you have a binary blob and suspect a hidden file inside, `binwalk` first, then `binwalk -e` to extract. It handles PNG, ZIP, PDF, JPEG, gzip, tar, and dozens more formats automatically.
- **"Something doesn't quite add up" + "image quality" hints** are the clue that pixel-level inspection will reveal the answer. Open the image in CyberChef / GIMP / stegsolve and look at the individual channels. The stego is almost always in the channel that does not match the rest of the image.
- **Skip R and A vs skip B and G in BGRA is the standard trick.** For any 32-bit BMP challenge where the author wants to hide data in 2 of 4 channels, the pattern is "skip the first 2, keep the last 2" (or its inverse). This is the inverse of "take the first 3, skip the last 1" which would be an RGB(A) extraction.
- **Different writeups can disagree on which channels.** picoctfsolutions.com and the YouTube walkthrough give opposite answers here (B+G vs R+A). When in doubt, try both — write to two different output files, then try to unzip each. The one that yields a valid ZIP is the right extraction.
- **The embedded file's name is base64.** The `ZnJhbmtlbnN0ZWluLXRlc3QudHh0` you see in the binwalk output looks random but is just the file's actual name (frankenstein-test.txt) base64-encoded. ZIP stores filenames as bytes — they are not required to be readable UTF-8.
- **The picoCTF artifact server can be hard to reach** (Cloudflare false-positive malware report — see picoCTF status 2024-03-25). When `wget` and `curl` both return 403, use the in-browser **Webshell** from the challenge page sidebar — that terminal has direct CDN access because the picoCTF platform allowlists its own egress IPs. `wget` the file there, then download it via the webshell's file browser.
- **Try alternative artifact paths.** The writeup might say `c/371/Invisible WORDs.bmp` but the actual "here" link on the challenge page can point to a different path (e.g., `c/403/output.bmp`). The 403 is path-based, not challenge-wide. Right-click the "here" link and copy the actual URL.
- **Wordplay decode of the challenge name**: *Invisible* = the bytes are there but you cannot see them; *WORDs* = the hidden content is text (the novel Frankenstein). The author's "trafficking in classics" line confirms the hidden text is a public-domain classic novel.
- **Wordplay decode of the flag**: `w0rd_d4wg_y0u_f0und_5h3113ys_m4573rp13c3` reads in leet as `WORD DAWG YOU FOUND SHELLEY'S MASTERPIECE`. "Word dawg" is a pun on "Word doc" + "dawg" (slang for friend); "you found Shelley's masterpiece" is the explicit reference to *Frankenstein; or, The Modern Prometheus* by Mary Shelley, which is the file hidden in the BMP. The trailing 8-character hex suffix `e4f8c8f0` is the per-instance nonce.

### What the actual flag looks like in my run

The flag printed by `grep picoCTF ZnJhbmtlbnN0ZWluLXRlc3QudHh0` in step 6 for my instance was on a line in the middle of the Frankenstein text:

```
At that age I became acquainted with the celebrated picoCTF{w0rd_d4wg_y0u_f0und_5h3113ys_m4573rp13c3_e4f8c8f0}
```

The trailing 8-character hex suffix `e4f8c8f0` is the per-instance nonce — it will be different for your download. The rest of the flag (`w0rd_d4wg_y0u_f0und_5h3113ys_m4573rp13c3`) is the same across all instances.

You will get your own unique value when you run the steps above on your downloaded `output.bmp`.
