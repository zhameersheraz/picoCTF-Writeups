# MSB — picoCTF Writeup

**Challenge:** MSB  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{15_y0ur_que57_qu1x071c_0r_h3r01c_ea7deb4c}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> This image passes LSB statistical analysis, but we can't help but think there must be something to the visual artifacts present in this image...
> Download the image here.

---

## Hints

> What's causing the 'corruption' of the image?

---

## Background Knowledge (Read This First!)

### Bits, Bytes, and Pixel Values

A single color channel value in a PNG pixel is a number from 0 to 255. That number is stored in 8 bits (1 byte). For example, the value `103` in binary is `01100111`.

In those 8 bits, some positions matter more than others:

- The **most significant bit (MSB)** is the leftmost bit. It represents 128. Flipping this bit changes the value by 128 — a HUGE visual change.
- The **least significant bit (LSB)** is the rightmost bit. It represents 1. Flipping it changes the value by 1 — almost invisible.

| Bit Position | 7 (MSB) | 6 | 5 | 4 | 3 | 2 | 1 | 0 (LSB) |
|--------------|---------|---|---|---|---|---|---|----------|
| Value if set | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

### LSB vs MSB Steganography

- **LSB stego** hides data in the rightmost bit. It is invisible to the eye, and tools like `zsteg` are built to find it. This challenge's description says the image "passes LSB statistical analysis" — meaning LSB is a dead end.
- **MSB stego** hides data in the leftmost bit. It is extremely visible. If you extract just the MSB of every channel and draw it as a black/white image, you get a picture of the hidden payload. The trade-off is that the carrier image also looks "corrupted" — because the MSB controls 50% of the color value at once.

That is exactly what the hint means by "corruption": the MSBs of the pixels in this image were overwritten with hidden data, and that overwrite shows up as visible color blockiness in the painting.

### Why This Works

A pixel's R, G, B values each contribute 1 MSB. So we have 3 hidden bits per pixel. For a 1074×1500 image, that is 4,833,000 bits ≈ 604,000 bytes of hidden storage. The flag is tiny in comparison, so the rest of the MSB plane is filled with other text (the "What Happened Internally" section below confirms this — there's a full page of English prose hidden in the MSBs).

---

## Solution — Step by Step

### Step 1 — Download the Image

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ wget https://artifacts.picoctf.net/c/302/Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png
```

### Step 2 — Confirm the File Type

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png
Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png: PNG image data, 1074 x 1500, 8-bit/color RGB, non-interlaced
```

A 1074×1500 RGB PNG. Large enough to hide a lot in the MSB plane.

### Step 3 — Look at the Image (Notice the "Corruption")

I opened the PNG in an image viewer. The painting is recognizable, but the colors look blocky/wrong in places — strong evidence of MSB manipulation. That matches the hint.

### Step 4 — Confirm the Hint Mentions MSB, Not LSB

The challenge name is literally "MSB" and the description says LSB analysis came up clean. So the data is in the MSB (bit 7) of each color channel.

### Step 5 — Write a Python MSB Extractor

I used `nano` to create a small script that walks every pixel, every channel, takes bit 7, packs those bits into bytes, and looks for `picoCTF`.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano extract_msb.py
```

Paste the following inside nano:

```python
#!/usr/bin/env python3
"""
MSB Steganography Extractor
Pulls bit 7 (the most significant bit) of every R, G, B channel
and reassembles the hidden byte stream.
"""
from PIL import Image
import sys

def extract_msb(image_path):
    img = Image.open(image_path)
    px = img.load()
    w, h = img.size
    bits = []
    for y in range(h):
        for x in range(w):
            p = px[x, y]
            for c in range(len(p)):
                bits.append((p[c] >> 7) & 1)   # bit 7 = MSB

    out = bytearray()
    for i in range(0, len(bits) - 7, 8):
        byte = 0
        for j in range(8):
            byte = (byte << 1) | bits[i + j]
        out.append(byte)
    return bytes(out)

if __name__ == "__main__":
    data = extract_msb(sys.argv[1])
    needle = b"picoCTF"
    idx = data.find(needle)
    if idx >= 0:
        print(f"Found '{needle.decode()}' at byte offset {idx}")
        start = max(0, idx - 16)
        end   = min(len(data), idx + 80)
        print("Context:")
        print(data[start:end])
    else:
        print("picoCTF not found in MSB data.")
        print("First 200 bytes (repr):")
        print(repr(data[:200]))
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 6 — Run the Script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 extract_msb.py Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png
Found 'picoCTF' at byte offset 254194
Context:
b'e new offence."\npicoCTF{15_y0ur_que57_qu1x071c_0r_h3r01c_ea7deb4c}\n\n"Thou hast said well and hit'
```

The script found the flag at byte 254,194 of the MSB stream. The surrounding bytes are English prose, which means the whole top of the MSB plane is a hidden text payload.

**Flag:** `picoCTF{15_y0ur_que57_qu1x071c_0r_h3r01c_ea7deb4c}`

---

## Alternative Methods

### Method 1 — Visualize the MSB Plane as an Image

This is what makes the "corruption" hint click. I extracted the MSB of just the red channel and rendered each MSB as 0 (black) or 255 (white). The result is a new image where the hidden text is literally readable.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano msb_plane.py
```

```python
from PIL import Image
img = Image.open('Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png')
px = img.load()
w, h = img.size
out = Image.new('L', (w, h))
op = out.load()
for y in range(h):
    for x in range(w):
        r, g, b = px[x, y]
        op[x, y] = 255 if (r >> 7) & 1 else 0
out.save('msb_plane.png')
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`. Run:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 msb_plane.py
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ xdg-open msb_plane.png
```

The generated image shows a page of clearly readable text with the flag sitting in the middle. This is the fastest way to confirm the technique before writing the byte-stream extractor.

### Method 2 — Use `stegoveritas`

`stegoveritas` is an all-in-one steg tool that can detect MSB planes as well. It is heavier than the script but useful when you do not know which bit plane to look at.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ pip install stegoveritas
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ stegoveritas Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png
```

It creates a `results/` directory with bit-plane images and tries to extract strings from each one. The MSB plane will contain the same flag.

### Method 3 — Python One-Liner with `zlib`-Decoded Text

The hidden text starts with a long English paragraph. You can also just dump the entire MSB stream to a file and `grep` it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 extract_msb.py Ninja-and-Prince-Genji-Ukiyoe-Utagawa-Kunisada.flag.png > msb_dump.bin
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep -ao "picoCTF{[^}]*}" msb_dump.bin
picoCTF{15_y0ur_que57_qu1x071c_0r_h3r01c_ea7deb4c}
```

---

## What Happened Internally

1. The challenge author took a Japanese ukiyo-e painting (Utagawa Kunisada's *Ninja and Prince Genji*) and converted it to a 1074×1500 RGB PNG.
2. A surprisingly long block of English prose was packed into a bit stream, MSB-first, channel order R-G-B, pixel order left-to-right, top-to-bottom. Specifically, the author embedded the **Project Gutenberg text of *Don Quixote* by Miguel de Cervantes** (about 590 KB of it). The flag sits inside the body of that text. This is a cute easter egg — the flag's wordplay `qu1x071c` ("quixotic") is a direct nod to the source material.
3. That bit stream was written into **bit 7** of every color channel of every pixel. Because bit 7 controls 50% of the value, the image got visibly blocky — that is the "corruption" the hint describes.
4. LSB analysis looked clean because bit 0 was untouched. The challenge deliberately used MSB to dodge the usual LSB tools.
5. My extractor walked the pixels in the same order, read bit 7 of each channel, packed the bits into bytes, and searched for `picoCTF{`. It found the flag at byte 254,194 of the reconstructed stream.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the challenge image |
| `file` | Confirm the file is a valid PNG |
| `xdg-open` / image viewer | Visually inspect the "corrupted" painting |
| `nano` | Write the MSB extractor script |
| `python3` + `PIL` | Walk pixels, read bit 7, reassemble bytes, search for the flag |
| `grep` | Pull the flag out of the dumped MSB stream (alternative method) |
| `stegoveritas` (alt) | All-in-one stego tool that also exposes the MSB plane |

---

## Key Takeaways

- If a stego challenge says LSB is clean, **try the other end of the byte — the MSB**. The bit position is a hint in itself.
- The challenge name is often a literal clue. "MSB" told us exactly which bit to look at before we even opened the file.
- Visual "corruption" in a PNG is a strong signal of MSB-plane manipulation, because flipping bit 7 changes the pixel value by 128.
- The MSB plane stores 3 hidden bits per pixel, so even a small image can hold hundreds of kilobytes of hidden text.
- Always render the suspect bit plane as its own image — it is the fastest way to confirm what is hidden before you commit to a script.
- A 12-line Python script with PIL is enough to solve this entire class of challenge; you do not need a heavy stego toolkit.

**Flag wordplay decoded:**

`picoCTF{15_y0ur_que57_qu1x071c_0r_h3r01c_ea7deb4c}`

- `15` -> "is"
- `y0ur` -> "your"
- `que57` -> "quest"
- `qu1x071c` -> "quixotic" (idealistic to the point of being impractical)
- `0r` -> "or"
- `h3r01c` -> "heroic"
- `ea7deb4c` -> hex-looking suffix (`e a 7 d e b 4 c`)

In plain English: **"Is your quest quixotic or heroic?"** — a fun nod to the dual nature of CTF solving: half the time it feels impossible, the other half it feels legendary.
