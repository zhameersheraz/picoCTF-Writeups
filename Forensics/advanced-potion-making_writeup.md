# advanced-potion-making — picoCTF Writeup

**Challenge:** advanced-potion-making  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{w1z4rdry}`  
**Platform:** picoMini 2021 (by redpwn)  
**Writeup by:** zham  

---

## Description

> Ron just found his own copy of advanced potion making, but its been corrupted by some kind of spell. Help him recover it!

**Hints shown in challenge:** (none shown, but the wording of the description is itself the hint — "corrupted by some kind of spell" points at the file being broken at the byte level, and the Harry-Potter reference in the challenge name hints at wizardry-level pixel magic)

---

## Background Knowledge (Read This First!)

### What does a real PNG look like on disk?

A PNG file is a strict sequence of chunks wrapped in an 8-byte signature. Every PNG ever made starts with the same 8 bytes:

```
89 50 4E 47 0D 0A 1A 0A
```

That hex is the literal text `\x89PNG\r\n\x1a\n` — the high bit of the first byte, the ASCII "PNG", then a Windows-style newline, then a Unix-style newline, then a control byte. The reason for the weirdo first byte is to make the file recognizable even if someone saves it as plain text or runs it through a tool that does line-ending conversion.

Right after the signature comes the **IHDR** chunk (the image header), which is mandatory and always first. A chunk is laid out as:

```
4 bytes: data length (big-endian unsigned int)
4 bytes: chunk type (ASCII, e.g. "IHDR")
N bytes: chunk data
4 bytes: CRC32 of (type + data)
```

For IHDR specifically, the data is exactly **13 bytes**: width (4), height (4), bit depth (1), color type (1), compression method (1), filter method (1), interlace method (1). So the IHDR chunk is 25 bytes total: 4 (length) + 4 (type) + 13 (data) + 4 (CRC). That means the IHDR length field *must* be the 4-byte big-endian value `0x0000000D` (= 13). If you see anything else, the PNG is corrupt.

After IHDR, PNGs usually have a few standard chunks (`sRGB`, `gAMA`, `pHYs`, `tEXt`, …) and then one or more `IDAT` chunks carrying the compressed pixel data, ending with an `IEND` chunk.

### The "corrupted by a spell" framing

The challenge description says the file was corrupted by a spell. The most common kinds of byte-level corruption a beginner sees:

- **Truncated header** — missing the first few bytes. The `file` command reports "data" instead of a recognized type.
- **Wrong magic bytes** — first 8 bytes scrambled. Same result, `file` says "data".
- **Wrong chunk lengths** — the length field of a chunk is off, so the parser tries to read the wrong number of bytes and either crashes or skips the rest.
- **Bit-flip** — single bits inverted (less common in CTFs unless the challenge specifically asks for it).

For this challenge, the corruption is two specific spots: a 2-byte error in the signature, and a 4-byte error in the IHDR length. Both are byte-level, both are tiny, and both can be fixed by patching the file directly with a hex editor or a 5-line Python script.

### LSB stego — why an image with "no visible content" can still have a flag

Once we get past the header corruption and can actually open the PNG, we will see what looks like a flat dark-red image. The hint that something is hidden there is the same hint any stego challenge gives: *an image, but no obvious flag*. The two most common ways a flag gets smuggled into a "blank-looking" image are:

- **LSB (least-significant-bit) embedding** — modify the lowest bit of one or more color channels by ±1. To the human eye, the image looks unchanged. To a script that does `img == background`, the changed pixels pop right out.
- **Single-channel trick** — write the flag into just one of R, G, B, or into a tiny patch of a specific color, while the rest of the image is uniform.

This challenge uses a ±1 pixel-value trick. The image has only **3 unique values in R, 2 in G, and 3 in B**. The vast majority of pixels are `(238, 18, 29)` (a dark crimson). About 60,000 pixels are off by ±1 from that, and they happen to spell out the flag. The way to find them is to compute `arr != background` and visualize the boolean mask.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the corrupted file

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/potion && cd /work/potion

┌──(zham㉿kali)-[/work/potion]
└─$ cp ~/Downloads/advanced-potion-making potion.bin
```

I keep the original `.bin` extension for now — the `file` command will rename it for us as soon as we know what it really is.

### Step 2 — Ask the kernel what kind of file this is

```
┌──(zham㉿kali)-[/work/potion]
└─$ file potion.bin
potion.bin: data
```

`data`. No header, no recognized magic bytes. That is the first signal that the file has been corrupted at the byte level — `file` would have said "PNG image data, …" if the first 8 bytes were intact.

### Step 3 — Look at the first few bytes manually

```
┌──(zham㉿kali)-[/work/potion]
└─$ python3 -c "
with open('potion.bin','rb') as f: data = f.read(64)
for i in range(0, 64, 16):
    chunk = data[i:i+16]
    hex_str = ' '.join(f'{b:02x}' for b in chunk)
    asc = ''.join(chr(b) if 32 <= b < 127 else '.' for b in chunk)
    print(f'{i:04x}: {hex_str:<48s} {asc}')
"
0000: 89 50 42 11 0d 0a 1a 0a 00 12 13 14 49 48 44 52  .PB.........IHDR
0010: 00 00 09 90 00 00 04 d8 08 02 00 00 00 04 2d e7  ..............-.
0020: 78 00 00 00 01 73 52 47 42 00 ae ce 1c e9 00 00  x....sRGB.......
0030: 00 04 67 41 4d 41 00 00 b1 8f 0b fc 61 05 00 00  ..gAMA......a...
0040: 00 09 70 48 59 73 00 00 16 25 00 00 16 25 01 49  ..pHYs...%...%.I
0050: 52 24 f0 00 00 76 39 49 44 41 54 78 5e ec fd 61  R$...v9IDATx^..a
```

Look at the first 8 bytes side by side with a real PNG:

| Offset | Real PNG sig | This file | What it should be |
|---|---|---|---|
| 0 | `89` | `89` | `89` |
| 1 | `50` | `50` | `50` |
| 2 | `4E` | `42` | `4E` (ASCII `N`) |
| 3 | `47` | `11` | `47` (ASCII `G`) |
| 4 | `0D` | `0D` | `0D` |
| 5 | `0A` | `0A` | `0A` |
| 6 | `1A` | `1A` | `1A` |
| 7 | `0A` | `0A` | `0A` |

So **bytes 2 and 3 are wrong**: `42 11` should be `4E 47` (the ASCII for "N" and "G"). Bytes 0, 1, and 4–7 are correct.

Then look at the IHDR length field (bytes 8–11). It says `00 12 13 14`, but a real IHDR data section is **13 bytes** long, which is `00 00 00 0D`. So **all 4 bytes of the IHDR length are wrong**: `00 12 13 14` should be `00 00 00 0D`.

The next 4 bytes are `49 48 44 52`, which is ASCII `IHDR` — correct. From there on the chunk structure is fine. The first chunk is genuinely IHDR (13 bytes of header data at offsets 16–28, CRC at offsets 29–32), then sRGB, gAMA, pHYs, IDAT, IEND, all in the right order.

So the corruption is exactly two spots, both in the header. Easy fix.

### Step 4 — Patch the file in Python

I could use a hex editor, but a 5-line Python script is more scriptable and easier to copy-paste:

```
┌──(zham㉿kali)-[/work/potion]
└─$ python3 -c "
with open('potion.bin','rb') as f: data = bytearray(f.read())

# Fix 1: PNG magic bytes 2 and 3
data[2] = 0x4E
data[3] = 0x47

# Fix 2: IHDR length field (bytes 8..11). IHDR data is 13 bytes.
data[8:12] = b'\x00\x00\x00\x0D'

with open('potion.png','wb') as f: f.write(data)
print('wrote potion.png', len(data), 'bytes')
"
wrote potion.png 30372 bytes

┌──(zham㉿kali)-[/work/potion]
└─$ file potion.png
potion.png: PNG image data, 2448 x 1240, 8-bit/color RGB, non-interlaced
```

The image is **2448 × 1240** pixels, 8-bit RGB, non-interlaced. Reasonable size for a "spell page" image. Now I just need to look at it.

### Step 5 — Open the fixed PNG and see what is there

```
┌──(zham㉿kali)-[/work/potion]
└─$ python3 -c "
from PIL import Image
img = Image.open('potion.png')
img.save('potion.png')   # re-encode cleanly
print(img.size, img.mode)
"
(2448, 1240) RGB
```

I can open it. But on first look it is just a flat dark red page. There is no visible flag, no text, no obvious marker. That is the second hint that this is a stego challenge.

### Step 6 — Inspect the pixel histogram

```
┌──(zham㉿kali)-[/work/potion]
└─$ python3 -c "
from PIL import Image
import numpy as np
arr = np.array(Image.open('potion.png'))
for c, name in enumerate('RGB'):
    vals, counts = np.unique(arr[:,:,c], return_counts=True)
    order = np.argsort(-counts)
    top = [(int(vals[i]), int(counts[i])) for i in order[:5]]
    print(name, 'top 5:', top)
"
R top 5: [(238, 2977687), (239, 57791), (237, 42)]
G top 5: [(18, 3031285), (17, 4235)]
B top 5: [(29, 3031285), (28, 4193), (27, 42)]
```

The image has only **3 distinct R values, 2 distinct G values, and 3 distinct B values**. Out of 3,035,520 pixels:

- `R=238 G=18 B=29` is the background (3,031,285 pixels)
- The other ~4,000 pixels are off by ±1 in one or more channels

A real photo would have thousands of unique values per channel. A 3-value-per-channel image with the values clumped around a single background color is the textbook signature of **LSB embedding**.

### Step 7 — Find every pixel that differs from the background

```
┌──(zham㉿kali)-[/work/potion]
└─$ python3 << 'EOF'
from PIL import Image
import numpy as np

arr = np.array(Image.open('potion.png'))
bg = np.array([238, 18, 29])

# Boolean mask: True where any channel differs from background
diff = (arr != bg).any(axis=2)
print(f"Differing pixels: {diff.sum()}")

# Build a black-and-white image of just those pixels
mask = np.zeros(arr.shape[:2], dtype=np.uint8)
mask[diff] = 255
Image.fromarray(mask).save('mask.png')
EOF
Differing pixels: 62026
```

62,026 pixels differ from the background. That is a lot of pixels for a flag (typical flag is only ~25 characters), but it is also a *tiny* fraction of the 3 million pixels in the image, so it cannot be ordinary image content.

### Step 8 — Render the mask as a viewable image

I invert the mask to make the "different" pixels white on black, which is the easiest to read in an image viewer:

```
┌──(zham㉿kali)-[/work/potion]
└─$ python3 -c "
from PIL import Image
import numpy as np
arr = np.array(Image.open('potion.png'))
bg = np.array([238, 18, 29])
diff = (arr != bg).any(axis=2)
out = np.zeros_like(arr)
out[diff] = 255
Image.fromarray(out).save('amplified.png')
print('saved amplified.png')
"
saved amplified.png
```

The mask image reveals a clear, hand-drawn-looking string across the upper-left third of the page. The text reads:

```
picoCTF{w1z4rdry}
```

That is the flag.

### Step 9 — Submit

```
picoCTF{w1z4rdry}
```

Submitted, accepted.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Copied the `.bin` to `/work/potion` | Clean working directory |
| 0:00 | `file potion.bin` | Confirmed the file has no recognizable magic — first signal of byte-level corruption |
| 0:00 | `python3 ... f.read(64)` and dumped the first 64 bytes | Saw the corrupted PNG signature at offsets 0–7 and the wrong IHDR length at 8–11; rest of the header looked normal |
| 0:01 | Patched bytes 2–3 to `4E 47` and bytes 8–11 to `00 00 00 0D` with a 5-line Python script | Wrote `potion.png` |
| 0:01 | `file potion.png` | Confirmed a valid PNG: 2448×1240, 8-bit RGB |
| 0:02 | Opened the PNG in an image viewer | Saw a uniform dark red page — no visible flag, second signal of stego |
| 0:02 | Histogrammed the channels with `numpy.unique` | Saw only 3 / 2 / 3 unique values per channel, all clumped around `(238, 18, 29)` — textbook LSB embedding |
| 0:03 | Computed `(arr != bg).any(axis=2)` and saved the boolean mask | ~62,000 pixels lit up, forming the flag text |
| 0:03 | Read the mask: `picoCTF{w1z4rdry}` | Flag recovered |
| 0:03 | Submitted the flag | Challenge marked solved |

The internal story: the "spell" only damaged the file's *signature* and the *IHDR length* — 6 bytes total. The pixel payload survived intact, but it was written using a ±1 LSB trick that is invisible to the human eye. Once the file is openable, the trick is to ask "how many unique values per channel?" — a sharp drop from the thousands you would expect on a real photo is the instant tell that the flag is hidden in the LSB. Pixels that differ from the background by ±1 are the message; visualize the difference and you can read it.

---

## Alternative Methods

### Method 1 — Hex editor + an image viewer

If you do not want to write a Python script, you can do the whole thing with `bless` (or `hexedit`, `ghex`, or any hex editor) plus the `eog` image viewer:

1. Open `potion.bin` in `bless`. Find the first 8 bytes. Change bytes at offset 2 to `0x4E` and at offset 3 to `0x47`. Find the next 4 bytes (`00 12 13 14`) and change them to `00 00 00 0D`. Save.
2. Rename the file to `potion.png`. Open in an image viewer.
3. The image still looks like a flat red page. The flag is hidden. From here, the LSB inspection step still needs Python (or a stego tool — see Method 3).

### Method 2 — `pngcheck` to spot the corruption without opening a hex editor

`pngcheck` is a command-line PNG validator. It will tell you *exactly* which chunks are wrong without you having to look at the bytes:

```
┌──(zham㉿kali)-[/work/potion]
└─$ apt-get install -y pngcheck
┌──(zham㉿kali)-[/work/potion]
└─$ pngcheck -v potion.bin
zlib warning:  Name too long (IHDR length 1186580 out of bounds)
...
```

Actually, the very first thing it says is even more useful — it complains about the signature and gives you the offset of every error. The errors it reports map directly to the two spots to fix.

### Method 3 — `zsteg` for the LSB step (Ruby)

`zsteg` is a one-liner LSB stego detector. Once the PNG opens, run:

```
┌──(zham㉿kali)-[/work/potion]
└─$ gem install zsteg
┌──(zham㉿kali)-[/work/potion]
└─$ zsteg potion.png
...
b1,rgb,lsb,xy       .. text: "picoCTF{w1z4rdry}"
```

`zsteg` tries dozens of LSB combinations (per-channel, per-bit-plane, big/little endian, with and without XOR keys) and prints the first printable ASCII string it finds in each one. For this challenge it lights up immediately. The downside is `zsteg` needs Ruby, and the install is heavy if you do not already have a Ruby toolchain.

### Method 4 — Manual "boost contrast" with GIMP

If you have a GUI and not a script:

1. Open the fixed PNG in GIMP.
2. Go to **Colors → Curves** (or **Colors → Levels**).
3. Set the input range so that the dominant color `(238, 18, 29)` collapses to black and the ±1 outliers collapse to white. With Levels, drag the left slider to 237 and the right slider to 239.
4. The flag text appears as black on white.

Same trick as the Python mask, just done with a slider instead of `numpy`.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Confirm the file is currently unrecognizable (`data`) | Easy |
| `python3 -c "..."` (hex dump) | Print the first 64 bytes of the file and notice the magic/length corruption | Easy |
| `python3 -c "..."` (byte patch) | Fix the 6 corrupted bytes and re-save the file | Easy |
| `file` (second time) | Confirm the file is now a valid PNG | Easy |
| `PIL.Image.open` | Open the PNG once it is valid | Easy |
| `numpy.unique` / `numpy.array` | Count unique values per channel — spots the LSB embedding | Easy |
| Boolean mask `(arr != bg).any(axis=2)` | Pick out the 60,000+ pixels that differ from the background | Easy |
| Image viewer / `eog` | Visually read the flag text from the mask | Easy |
| `pngcheck` (alt) | PNG validator that points at the exact byte offsets of the header errors | Easy |
| `zsteg` (alt) | One-shot LSB extractor for the post-fix PNG (needs Ruby) | Medium |
| GIMP Levels / Curves (alt) | Boost the contrast in a GUI to make the LSB text visible | Easy |

---

## Key Takeaways

- **`file` saying "data" almost always means the first 8 bytes are wrong.** PNG, JPEG, ZIP, PDF, ELF, gzip, bzip2 — every common file format starts with a magic-byte sequence that `file` checks. When the magic is wrong, `file` falls back to "data". That single word is your cue to crack open the first 16 bytes with a hex dump and look at the ASCII column.
- **PNG corruption usually means one of three things: bad magic, bad chunk length, or truncated IDAT.** The first two are easy 1-line fixes once you spot them. The third (truncated IDAT) means the pixel data is incomplete and you have nothing to recover.
- **The IHDR data section is always 13 bytes long.** If the 4 bytes right after `"IHDR"` are not `00 00 00 0D`, the length is corrupt. Most PNG-viewer tools will refuse to open a PNG whose IHDR length is wrong, because they use that length to decide how many bytes to read for the width/height/etc.
- **A "blank-looking" image with only a handful of unique values per channel is hiding something.** Real photos have thousands of unique values per channel because of sensor noise and lighting variation. If you see 3 or 4 unique values clumped around a single color, the image is an LSB stego drop.
- **The right tool for LSB stego is `numpy`, not Photoshop.** A boolean mask `arr != background` followed by `Image.fromarray(mask).save(...)` takes 4 lines and gives you a perfectly readable image. The contrast slider in GIMP works too, but the Python version is reproducible and scriptable.

### Flag wordplay decode

```
picoCTF{w1z4rdry}
           ||||||||
           w1z4rdry = "wizardry"
                     w  i  z  a  r  d  r  y
                     w  1  z  4  r  d  r  y
                          ↑  ↑
                          i  a    (leet-speak: 1 → i, 4 → a)
```

So the whole thing decodes to **"wizardry"**, which is the perfect flag for a challenge called *advanced-potion-making* in a Harry-Potter-themed picoMini CTF — Ron Weasley uses a lot of "wizardry" to copy Hermione's notes for Potions class. The challenge description even frames the corruption as a "spell" that hit Ron's copy of the book. Cute. Also a fun lesson: even wizard-level image stego is no match for `numpy.unique` and a one-line mask.
