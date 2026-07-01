# St3g0 — picoCTF Writeup

**Challenge:** St3g0  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{7h3r3_15_n0_5p00n_a1062667}`  
**Platform:** picoCTF (2022)  
**Writeup by:** zham  

---

## Description

> Download this image and find the flag.

**Hint shown in challenge:** `We know the end sequence of the message will be $t3g0.`

---

## Background Knowledge (Read This First!)

### What is LSB Steganography?

**LSB** stands for **Least Significant Bit**. In a normal 8-bit color channel (Red, Green, or Blue), each pixel value is a number from 0 to 255. The least significant bit is the rightmost bit — the one that contributes only `0` or `1` to the total.

For example, the value `173` in binary is `10101101`. The LSB is the last `1`. Changing it to `0` flips the value to `172`. To the human eye, the difference between `172` and `173` is invisible.

LSB steganography hides a secret message by replacing the LSB of every pixel channel with bits from the message. The image looks identical, but the message is encoded across thousands of pixel channels.

### Common LSB Encodings

The encoder can pick any combination of:

| Bit | Channels | Order |
|-----|----------|-------|
| 1, 2, 4 (bit depth) | R, G, B, A (which channels) | LSB (least sig) or MSB (most sig) |
| RGB / BGR / RGBA / etc. (channel order) | xy / x (rows) / y (columns) |

That gives dozens of possible layouts. A tool like `zsteg` just tries them all and reports which ones produce readable text.

### What is the `$t3g0` Delimiter?

When you hide a message in LSB, you need a way for the decoder to know **where the message ends**. A common trick is to append a known marker string. The hint tells us the marker is `$t3g0` — a leet-speak version of the challenge name `St3g0` (with `$` swapped in for `S`).

### What is zsteg?

`zsteg` is a Ruby gem that automates LSB / MSB detection on PNG and BMP files. It tries hundreds of bit-plane combinations in seconds and prints any that look like readable text. It is the single most useful tool for this category of challenge.

Install on Kali:

```
sudo apt install -y ruby ruby-dev zlib1g-dev
sudo gem install zsteg
```

---

## Solution — Step by Step

### Step 1 — Download the Image

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ wget https://artifacts.picoctf.net/c/216/pico.flag.png
--2026-07-01 13:00:14--  https://artifacts.picoctf.net/c/216/pico.flag.png
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 172.64.155.42
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|172.64.155.42|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 13441 (13K) [image/png]
Saving to: 'pico.flag.png'

pico.flag.png         100%[===================>]  13.13K  --.-K/s    in 0s

2026-07-01 13:00:14 (90.3 MB/s) - 'pico.flag.png' saved [13441/13441]
```

### Step 2 — Check the File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file pico.flag.png
pico.flag.png: PNG image data, 585 x 172, 8-bit/color RGBA, non-interlaced
```

Just a normal-looking PNG. No flag text anywhere in the visible image.

### Step 3 — Open It Anyway (It Looks Innocent)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ eog pico.flag.png
```

It's just the picoCTF logo. No flag text, no suspicious visible artifacts. The challenge name `St3g0` is screaming "steganography" at us, so the flag is hidden in the pixel data.

### Step 4 — Run zsteg (The Magic Tool)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ zsteg pico.flag.png
b1,r,lsb,xy           .. text: "~__B>wV_G@"
b1,rgb,lsb,xy         .. text: "picoCTF{7h3r3_15_n0_5p00n_a1062667}$t3g0"
b1,abgr,lsb,xy        .. text: "E2A5q4E%uSA"
b2,b,lsb,xy           .. text: "AAPAAQTAAA"
b2,b,msb,xy           .. text: "HWUUUUUU"
b2,a,lsb,xy           .. file: Matlab v4 mat-file (little endian) ><?P, numeric, rows 0, columns 0
b2,a,msb,xy           .. file: Matlab v4 mat-file (little endian) | <?  numeric, rows 0, columns 0
b3,r,lsb,xy           .. file: gfxboot compiled html help file
b4,r,lsb,xy           .. file: Targa image data (16-273) 65536 x 4097 x 1 +4352 +4369 - 1-bit alpha - right "\021\020\001\001\021\021\001\001\021\021\001"
b4,g,lsb,xy           .. file: 0420 Alliant virtual executable not stripped
...
```

There it is — second line:

```
b1,rgb,lsb,xy         .. text: "picoCTF{7h3r3_15_n0_5p00n_a1062667}$t3g0"
```

Read that `b1,rgb,lsb,xy` label as:
- `b1` — 1 bit per channel
- `rgb` — across Red, Green, Blue channels
- `lsb` — least significant bit
- `xy` — read in normal row-major (left-to-right, top-to-bottom) order

### Step 5 — Strip the `$t3g0` Suffix

The hint told us `$t3g0` is the end-of-message delimiter — it's not part of the flag itself. The actual flag is everything before it:

```
picoCTF{7h3r3_15_n0_5p00n_a1062667}
```

The flag is: `picoCTF{7h3r3_15_n0_5p00n_a1062667}`

---

## Alternative Solve — Python + Pillow (No zsteg Required)

If you don't have Ruby / zsteg installed, you can reproduce the exact same LSB extraction in a few lines of Python. This is great for understanding what's happening under the hood.

### Step 1 — Install Pillow

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ pip install pillow
```

### Step 2 — Write the Extractor

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano lsb_extract.py
```

Paste this into the file:

```python
from PIL import Image

img = Image.open('pico.flag.png').convert('RGBA')
pixels = list(img.getdata())

# b1,rgb,lsb,xy means: 1 bit each from R, G, B, in row-major order,
# taking the least significant bit of each channel.
bits = []
for r, g, b, a in pixels:
    bits.append(r & 1)
    bits.append(g & 1)
    bits.append(b & 1)

# Group bits into bytes (8 bits at a time)
message = bytearray()
for i in range(0, len(bits) - 7, 8):
    byte = 0
    for j in range(8):
        byte = (byte << 1) | bits[i + j]
    message.append(byte)
    if message.endswith(b'$t3g0'):
        break

print(message.decode('ascii', errors='replace'))
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 3 — Run It

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 lsb_extract.py
picoCTF{7h3r3_15_n0_5p00n_a1062667}$t3g0
```

Same result as `zsteg` — we read every pixel's R, G, B LSBs in order, packed them 8 at a time into bytes, and stopped when we hit the `$t3g0` delimiter.

---

## What Happened Internally (Timeline)

1. **The challenge author took the picoCTF logo** as a cover image (585x172 RGBA, 13KB).
2. **They picked an LSB layout** — `b1,rgb,lsb,xy` — meaning "use the LSB of the R, G, and B channels, reading pixels left-to-right, top-to-bottom."
3. **They encoded the flag** as a bit stream. Each ASCII character is 8 bits; each pixel contributes 3 bits (one from R, one from G, one from B). So roughly 3 pixels of the logo store 1 character of the flag.
4. **They appended the `$t3g0` delimiter** to mark where the message ends. The decoder knows to stop reading at this string.
5. **They wrote the modified pixel data** back into the PNG. Visually, the logo is unchanged — each LSB flip is at most a 1-unit change in a 0-255 channel, which is below human perception.
6. **We downloaded the image** and ran `zsteg` to brute-force every reasonable LSB layout. The `b1,rgb,lsb,xy` layout produced readable text containing `picoCTF{...}` followed by the `$t3g0` delimiter.
7. **We stripped the `$t3g0` suffix** (it was a marker, not part of the flag) and submitted the remaining text.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download `pico.flag.png` from the picoCTF artifacts server |
| `file` | Confirm the file is a PNG (rules out extension-spoofed files) |
| `eog` | Open the image in a GUI to confirm it looks innocent |
| `zsteg` | Brute-force every common LSB / MSB layout and pull out readable text |
| `python3` + `Pillow` (alt) | Manual LSB extraction when zsteg isn't available |
| `nano` | Edit the Python extractor script |

---

## Key Takeaways

- **LSB steganography hides data in the lowest bits** of each color channel. The image looks identical, but thousands of bits of secret data are smuggled in.
- **zsteg is the Swiss-army knife for image stego.** When you see a PNG challenge with "steganography" in the description, run `zsteg <file>` first — it tries hundreds of bit layouts in seconds and usually finds the message immediately.
- **Always read the hint carefully.** The hint told us two important things: (1) the message is in the LSB (challenge name + "steg" tag), and (2) the message ends with `$t3g0` (so we know the delimiter and where the flag ends).
- **Delimiters are essential in stego.** Without `$t3g0` (or a length prefix), the decoder would have no way to know when the hidden message ends and the random LSB noise begins.
- **You can always fall back to Python + Pillow.** The whole `b1,rgb,lsb,xy` extraction is ~15 lines of Python. Writing it yourself once is the best way to understand what zsteg is doing under the hood.
- **Real-world connection:** LSB steganography is rarely used in serious malware today (modern detection tools can spot it), but variants still appear in academic research, watermarking tools, and beginner CTF challenges like this one.

**Flag wordplay decoded:** `7h3r3_15_n0_5p00n` = "there is no spoon" — the famous line from *The Matrix* (1999), written in leet-speak (`7` → `T`, `3` → `E`, `0` → `O`, `5` → `S`). The flag's reference to a spoon ties back to the `$t3g0` delimiter: imagine a flag as a soup, and the spoon stirs up the bits of the image to fish out the message. A fitting bit of wordplay for a steganography challenge.
