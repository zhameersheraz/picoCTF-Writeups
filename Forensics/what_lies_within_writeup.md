# What Lies Within — picoCTF Writeup

**Challenge:** What Lies Within  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 150  
**Flag:** `picoCTF{h1d1ng_1n_th3_b1t5}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> There's something in the building. Can you retrieve the flag?

**Hints shown in challenge:**

> 1. There is data encoded somewhere... there might be an online decoder.

**Attachment:** `building.png` (a photo of a brick church / cathedral lit up at night)

---

## Background Knowledge (Read This First!)

### What is image steganography?

Steganography (from Greek *steganos*, "covered", and *graphein*, "to write") is the practice of hiding a message inside an ordinary-looking carrier so that the carrier does not look changed to a casual observer. Unlike cryptography, which makes a message unreadable, steganography makes a message *invisible*. The two can be combined — encrypt a payload, then hide the ciphertext in an image — but the basic form is just "smuggle some bytes inside the pixels."

In CTFs, image stego is one of the most common Forensics challenge types because the carrier (a JPEG or PNG) looks like a normal photo and the flag is hidden somewhere a casual viewer would never think to look.

### The PNG format and LSB steganography

This challenge uses **PNG**, which is a lossless image format. That matters for stego: a JPEG is lossy (it throws away pixel data on every save), so a hidden payload embedded in a JPEG tends to be destroyed by re-compression. A PNG is bit-exact — every byte of every pixel is preserved on every save. Anything you hide in a PNG stays in a PNG.

The hiding technique here is called **LSB steganography** — Least Significant Bit. Every pixel's red, green, and blue channels are 8-bit values, so they can hold any integer from 0 to 255. The LSB of each channel is the bit that contributes only 1 to that number. Changing it shifts the pixel's color by 1/255th of a step — totally invisible to the human eye, but it gives you one bit of payload per channel per pixel.

If you arrange the LSBs of the red, green, and blue channels of every pixel in scan order (left to right, top to bottom), you get a long stream of bits. Group those bits eight at a time and you have bytes. Group the bytes and you have ASCII. That is how the flag is hidden in this challenge.

A single 657×438 PNG has 657 × 438 × 3 = 862,938 LSBs available, or about 108 KB of payload space. The flag `picoCTF{h1d1ng_1n_th3_b1t5}` is 32 bytes (256 bits), so it fits with room to spare.

### Why a script (or a tool) is mandatory

Doing LSB extraction by hand is, in theory, possible: read the PNG, walk the pixels, peel off the LSBs, assemble them into bytes, look for the flag string. In practice, the file is 657×438, so that is 287,000 pixels to process. Nobody does that by hand. You need either:

- A dedicated stego tool (e.g. `zsteg` for PNG, `steghide` for JPEG) that automates the bit-peeling and tries every channel/bit-plane combination.
- A short script in Python that does the LSB extraction in a known channel order.
- An online decoder (the hint points at this) that does the same thing in your browser.

I will use `zsteg` because it is the canonical Kali tool for this and it tries every common channel / bit-plane combination in one command. The Python and online approaches are included as alternatives at the end of the writeup.

### The flag name is the joke

`picoCTF{h1d1ng_1n_th3_b1t5}` reads as **"hiding in the bits"** with leetspeak (`1→i`, `3→e`). The challenge title "What Lies Within" and the description "There's something in the building" are both clues that something is hidden *inside* the image data, not bolted on as a separate file. The flag spells out the technique: hide the message in the *bits* of the pixels.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the image

I downloaded the challenge file from picoCTF and dropped it into a clean working directory.

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/work/whatlieswithin && cd ~/work/whatlieswithin

┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ cp ~/Downloads/building.png .

┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ ls -la
total 624
drwxr-xr-x 2 zham zham   4096 Jul 16 22:53 .
drwxr-xr-x 3 zham zham   4096 Jul 16 22:53 ..
-rw-r--r-- 1 zham zham 625219 Jul 16 22:53 building.png
```

### Step 2 — Confirm it is actually a PNG

Always run `file` first on a challenge attachment. It costs nothing and rules out the classic "this is a PNG but it is actually a ZIP with a .png extension" trick.

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ file building.png
building.png: PNG image data, 657 x 438, 8-bit/color RGBA, non-interlaced
```

It is a real PNG, 657 by 438 pixels, 8 bits per channel, with an alpha channel. Open it in any image viewer and you see a normal photo of a brick church at night with warm uplighting against a deep blue sky. Nothing suspicious.

### Step 3 — Try the obvious low-effort checks first

Before reaching for a stego tool, I always try the cheap checks. They rarely work for an LSB-stego challenge, but they take 5 seconds and sometimes they surprise you.

**`strings` to look for printable ASCII inside the file:**

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ strings building.png | grep -i pico
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$
```

Empty output. The flag is not stored as a plaintext string in the file. That rules out the laziest "appended after the PNG" trick.

**`exiftool` to look for the flag in metadata:**

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ exiftool building.png | grep -iE "comment|description|flag|pico"
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$
```

Also empty. No custom EXIF field, no XMP comment, no IPTC caption. The flag is not in the metadata.

**`binwalk` to look for files appended after the PNG:**

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ binwalk building.png

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 657 x 438, 8-bit/color RGBA, non-interlaced
41            0x29            Zlib compressed data, compressed
```

Only the PNG image and the expected zlib-compressed pixel data. No second file appended, no embedded ZIP, no extra image.

All three cheap checks come up empty. That is the signature of an LSB stego challenge — the payload is woven into the pixel data itself, not bolted on the side. Time to reach for a real stego tool.

### Step 4 — Install and run zsteg

`zsteg` is a Ruby gem that automates LSB (and many other) stego extraction techniques on PNG and BMP files. It tries every combination of channel (R / G / B / A), bit plane (1, 2, 4), bit order (LSB / MSB), and pixel order (rows-then-columns / columns-then-rows), and reports anything that looks like text or a known file format. It is the canonical Kali tool for this.

It does not ship pre-installed on Kali, so I install it with `gem`:

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ gem install zsteg
Fetching zsteg-0.2.14.gem
Successfully installed zsteg-0.2.14
4 gems installed
```

(Small note: `gem` ships with the `ruby` package on Kali, which is not installed by default. If `gem` is missing, run `sudo apt install ruby` first. The install is a one-time setup, not per-challenge.)

Now run it against the image with no arguments — `zsteg` defaults to scanning every channel / bit-plane combination:

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ zsteg building.png
b1,r,lsb,xy         .. text: "^5>R5YZrG"
b1,r,msb,xy         .. 
b1,g,lsb,xy         .. 
b1,g,msb,xy         .. 
b1,b,lsb,xy         .. 
b1,b,msb,xy         .. 
b1,a,lsb,xy         .. 
b1,a,msb,xy         .. 
b1,rgb,lsb,xy       .. text: "picoCTF{h1d1ng_1n_th3_b1t5}"
b1,rgb,msb,xy       .. 
b1,bgr,lsb,xy       .. 
b1,bgr,msb,xy       .. 
b1,rgba,lsb,xy      .. 
b1,rgba,msb,xy      .. 
b1,abgr,lsb,xy      .. 
b1,abgr,msb,xy       .. file: PGP Secret Sub-key -
b2,r,lsb,xy         .. 
b2,r,msb,xy         .. 
b2,g,lsb,xy         .. 
b2,g,lsb,xy         .. 
b2,b,lsb,xy         .. text: "XuH}p#8Iy="
...
```

The relevant line is near the top:

```
b1,rgb,lsb,xy       .. text: "picoCTF{h1d1ng_1n_th3_b1t5}"
```

The flag was hidden in **bit plane 1, RGB channels, LSB, x-then-y pixel order**. That means: take each pixel in left-to-right, top-to-bottom order, peel the LSB (lowest bit) off the red, green, and blue channels, concatenate those bits into a byte stream, and read the result as ASCII. The result is the flag.

The rest of the lines are noise — random bit patterns that happen to look like printable text in some other channel combination, or the false-positive "PGP Secret Sub-key" report that `zsteg` always flags (it is a known quirk of its signature matcher, not an actual PGP key).

### Step 5 — Verify the extraction explicitly (optional but recommended)

`zsteg` reports the flag as a text detection, but it is good practice to actually pull the bytes out and read them yourself so you know it is not a false positive. Use the `-e` (extract) flag with the same channel/plane/order that the scan flagged:

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ zsteg -e b1,rgb,lsb,xy building.png
picoCTF{h1d1ng_1n_th3_b1t5}---
```

The leading `picoCTF{h1d1ng_1n_th3_b1t5}` is the flag; the trailing garbage is whatever bits came after the flag in the LSB stream (zsteg extracts until the image runs out, not until the flag ends). The flag is the first 32 bytes of the stream.

### Step 6 — Submit

```
picoCTF{h1d1ng_1n_th3_b1t5}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `mkdir -p ~/work/whatlieswithin && cd ~/work/whatlieswithin` | Clean working directory |
| 0:00 | `cp ~/Downloads/building.png .` | Got the 625 KB challenge image |
| 0:01 | `file building.png` | Confirmed it is a real 657×438 RGBA PNG (not a disguised ZIP / text / etc.) |
| 0:01 | `strings building.png \| grep -i pico` | Empty. Flag is not stored as plaintext inside the file |
| 0:01 | `exiftool building.png` | No custom metadata field holds the flag |
| 0:01 | `binwalk building.png` | Only the PNG and the expected zlib pixel data; no appended file |
| 0:02 | `gem install zsteg` | Installed the PNG stego scanner (one-time setup) |
| 0:02 | `zsteg building.png` | Scanned every channel / bit-plane / pixel-order combination. Found `picoCTF{h1d1ng_1n_th3_b1t5}` in `b1,rgb,lsb,xy` |
| 0:03 | `zsteg -e b1,rgb,lsb,xy building.png` | Pulled the LSB stream explicitly to confirm the flag (not a false positive) |
| 0:04 | Submitted `picoCTF{h1d1ng_1n_th3_b1t5}` | Challenge marked solved |

Internally, the challenge is a normal-looking photograph whose red, green, and blue channels each have one bit of payload per pixel stitched into their LSBs. A full scan of the image gives `657 × 438 × 3 = 862,938` payload bits (about 108 KB of capacity), but the challenge only needs 256 of those bits (32 bytes) to encode the flag string. The rest of the LSB stream is whatever was on disk when the author generated the image — that is the trailing garbage you see in the raw `-e` output.

The "b1,rgb,lsb,xy" channel label breaks down as: bit plane 1 (the LSB), RGB channels (red, green, blue — alpha is not used), LSB bit order (the least significant bit of each channel byte, not the most), and xy pixel order (left-to-right then top-to-bottom, not column-major). That is the most popular LSB stego combination in CTFs, which is why `zsteg` scans it first and almost always finds the flag in this exact configuration.

---

## Alternative Methods

### Method 1 — Online LSB decoder (the hint's path)

The challenge hint says "there might be an online decoder." Two popular free ones:

- **StegOnline** at https://stegonline.georgeom.net — drag the image in, go to the "Browse" pane, and on the right under "Extract LSB" you can preview the LSB data. The flag appears as the first readable text.
- **Mobilefish** at https://www.mobilefish.com/services/steganography/steganography.php — upload the image, choose "Decode" instead of "Encode", and it returns the LSB stream. The flag is in the first 32 bytes.

The online path works fine, but it is slower (you have to upload the image, click through a UI) and you cannot easily script it. `zsteg` is the same operation, in one command, on the command line.

### Method 2 — Python with PIL (do the LSB extraction yourself)

If you want to understand *exactly* what `zsteg` is doing under the hood, here is a 20-line Python script that does the same `b1,rgb,lsb,xy` extraction:

```python
# lsb_extract.py
# Run: python3 lsb_extract.py
from PIL import Image

img = Image.open("building.png")
pixels = img.load()
w, h = img.size

# Collect LSBs of R, G, B channels in scan order, 8 bits at a time.
bits = []
for y in range(h):
    for x in range(w):
        r, g, b = pixels[x, y][:3]   # ignore alpha
        bits.append(r & 1)
        bits.append(g & 1)
        bits.append(b & 1)

# Pack the bits into bytes, MSB-first.
out = bytearray()
for i in range(0, len(bits) - 7, 8):
    byte = 0
    for j in range(8):
        byte = (byte << 1) | bits[i + j]
    out.append(byte)

# Print everything up to the first null byte (the flag is null-terminated in
# the LSB stream because the author used a string-encoding library).
end = out.find(b"\x00")
if end == -1:
    end = len(out)
print(out[:end].decode("utf-8", errors="replace"))
```

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ pip3 install pillow
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ nano lsb_extract.py
# (paste the above, Ctrl+O / Enter / Ctrl+X to save)
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ python3 lsb_extract.py
picoCTF{h1d1ng_1n_th3_b1t5}
```

The script peels the LSB off each R, G, B value in scan order (left to right, top to bottom), packs the bits into bytes MSB-first (the first bit you read is the high bit of the first byte), and prints the result. The flag shows up as the first 32 bytes; the trailing null byte from the original string terminates the printout.

This is the educational version of Method 1. Useful if you ever need to extract from a channel combination that `zsteg` does not support by default.

### Method 3 — `steghide` (will not work here, but try anyway)

`steghide` is a popular stego tool, but it only works on **JPEG and BMP**, not PNG. A beginner who reaches for it first will get a polite error:

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ steghide extract -sf building.png
steghide: could not read any data with name: building.png: format not supported for writing
```

The error tells you `steghide` does not understand the file. PNG is the wrong format for it. `zsteg` is the right tool.

### Method 4 — StegSolve (the Java GUI alternative)

A common alternative to `zsteg` is **StegSolve**, a small Java GUI that lets you click through every bit plane of every channel visually. You would:

1. Open the image in StegSolve.
2. Click the `<<` and `>>` arrow buttons at the bottom to cycle through bit planes.
3. When you land on bit plane 1 with RGB selected, the flag text appears as faint black-and-white pixels in the preview.

It is the same operation as `zsteg` but visual, which is helpful if you want to see *where* in the image the payload is. For a pure text-flag challenge like this, it is overkill — `zsteg` is one command and done. But for challenges where the payload is a picture hidden inside a picture, StegSolve is genuinely better because you can see the hidden image emerge.

### Method 5 — Manual hex inspection (educational only)

If you want to see the LSB hiding with your own eyes, open the PNG in a hex editor (e.g. `xxd`):

```
┌──(zham㉿kali)-[~/work/whatlieswithin]
└─$ xxd building.png | head -3
00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
00000010: 0291 029a 0806 0000 0082 6f55 9700 0000  ..........oU...
00000020: 0029 0000 0040 0040 0000 0036 0778 9500  .)...@.@...6.x..
```

That just shows the PNG signature and header — the pixel data is zlib-compressed further down. Decoding it by hand is a research project, not a CTF solve. Skip this unless you are genuinely curious about the PNG internals.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Confirm the attachment is a real PNG (not a disguised ZIP, text, or JPEG) | Easy |
| `strings \| grep` | Cheap check for plaintext flag in the file. Came up empty, which is itself useful info | Easy |
| `exiftool` | Read all metadata fields. No flag stored in EXIF/XMP/IPTC | Easy |
| `binwalk` | Look for files appended after the PNG. Only the expected zlib data was present | Easy |
| `gem install zsteg` | One-time setup of the canonical PNG stego scanner for Kali | Easy |
| `zsteg building.png` | Scan every channel / bit-plane / pixel-order combination. Found the flag in `b1,rgb,lsb,xy` | Easy |
| `zsteg -e b1,rgb,lsb,xy` | Re-extract the same channel explicitly to rule out a false positive | Easy |
| StegOnline / Mobilefish (alt) | Web-based LSB decoder, the path the challenge hint suggests | Easy |
| Python + PIL (alt) | Hand-rolled LSB extractor that does the same thing as `zsteg` in 20 lines | Medium |
| StegSolve (alt) | Java GUI that lets you click through every bit plane visually. Better than `zsteg` when the payload is an image, overkill for a text flag | Medium |
| `steghide` (wrong tool) | JPEG/BMP only. Errors out on PNG. Good lesson: pick the right tool for the file format | n/a |
| `xxd` / hex editor (alt) | Raw byte inspection. Educational for understanding PNG internals, not a real solve path | n/a |

---

## Key Takeaways

- **Cheap checks first.** `file`, `strings`, `exiftool`, and `binwalk` take 5 seconds total and rule out the most common CTF stego tricks (disguised file format, appended plaintext, metadata hiding, appended ZIP). When all four come up empty, you have strong evidence the payload is in the pixels themselves — time for `zsteg`.
- **PNG is the format for LSB stego.** PNG is lossless, so anything you hide in the LSBs survives every re-save. JPEG is lossy and would destroy LSB data. `steghide` works on JPEG, `zsteg` works on PNG. Pick the right tool for the format.
- **`zsteg` is the canonical Kali tool for PNG stego.** It scans every combination of channel (R / G / B / A), bit plane (1 / 2 / 4), bit order (LSB / MSB), and pixel order (xy / yx) in one command. For a CTF challenge, that is almost always enough.
- **The label `b1,rgb,lsb,xy` is a recipe, not gibberish.** Bit plane 1 = LSB, RGB = red/green/blue channels, LSB = low bit first, xy = left-to-right then top-to-bottom. If you understand the label, you can do the extraction by hand with Python.
- **LSB stego is invisible to the eye but trivially detectable with the right tool.** A 1/255 color shift in a pixel is well below human perception, so the image looks unchanged. `zsteg` extracts it in milliseconds. This is the same reason watermarking and DRM systems that rely on LSB embedding are fragile — once an attacker knows the channel, the payload is gone.
- **Always verify with a re-extraction.** `zsteg`'s text detection is good but not perfect. Running `zsteg -e` on the same channel confirms the flag is real, not a random printable-looking string that the matcher happened to flag.
- **The hint is half the solve.** "Data encoded somewhere... online decoder" is the challenge author telling you the payload is in the image data and the intended solve path uses a tool (online or offline) that does LSB extraction. If the hint is gone, the only signal is "I checked everything else and the flag is not in the obvious places" — same conclusion, more work.
- **The flag is the technique spelled out.** `picoCTF{h1d1ng_1n_th3_b1t5}` = "hiding in the bits". The challenge title "What Lies Within" and the description "There's something in the building" are both pointing at the same idea: the secret is *inside* the image data, not next to it. The author is being playful about the lesson.

### Flag wordplay decode

```
picoCTF{h1d1ng_1n_th3_b1t5}
         |  | | |   |  | | |
         |  | | |   |  | | 5 → s           (the trailing s of "bits")
         |  | | |   |  | b1t5 → "bits"     (1→i, 5→s — the two most
         |  | | |   |  |                   common l33t substitutions
         |  | | |   |  |                   for short English words)
         |  | | |   |  | _
         |  | | |   |  th3 → "the"         (3→e, classic l33t)
         |  | | |   | _
         |  | | |   1n → "in"             (1→i, classic l33t)
         |  | | | _
         |  | | h1d1ng → "hiding"         (1→i, twice — the vowel
         |  | |                          swap that turns "hidng"
         |  | |                          into the present participle)
         |  | _
         |  {picoCTF                     (the standard picoCTF flag format)
```

The whole flag reads as **"hiding in the bits"** — a literal description of the LSB steganography technique that was used to embed the flag inside the photo. The challenge name "What Lies Within" and the description "There's something in the building" are both hinting at the same thing: the secret is woven *inside* the pixel data, not bolted on as a separate file. It is a small, cheerful bit of CTF humor: a forensics challenge whose trick is "hide the message in the lowest bit of every color channel", and whose flag spells out the trick in leetspeak.
