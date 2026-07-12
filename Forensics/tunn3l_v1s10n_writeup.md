# tunn3l v1s10n — picoCTF Writeup

**Challenge:** tunn3l v1s10n  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 40  
**Flag:** `picoCTF{qu1t3_a_v13w_2020}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> We found this file. Recover the flag.

**Hints shown in challenge:**

> 1. Weird that it won't display right...

---

## Background Knowledge (Read This First!)

### The BMP file format

BMP (Bitmap Image File) is one of the oldest image formats on Windows. Every BMP file has **two headers** stacked at the start, followed by the raw pixel data:

1. **BMP file header** — 14 bytes, always present. Starts with the two-byte magic number `"BM"` (ASCII `0x42 0x4D`). Contains the total file size, two reserved fields (always zero), and the byte offset where pixel data starts.
2. **DIB header** (Device Independent Bitmap) — variable size, almost always 40 bytes (`BITMAPINFOHEADER`). The first 4 bytes of the DIB header declare its own size (a self-describing meta-header). After the size field come width, height, color planes, bits per pixel, compression method, image size, horizontal/vertical resolution, colors used, and "important" colors.
3. **Pixel data** — raw bytes, one row at a time, **bottom-up** in BMP (row 0 in the file is the *bottom* row of the image). Each row is padded to a 4-byte boundary.

The DIB header is what tells image readers "this is a real BMP, here is how to interpret the pixel data". If any of its critical fields are corrupted, the file becomes unopenable or renders as a stripe of static. The hint "weird that it won't display right..." is the challenge author's nudge: open the file in a hex editor, look at the header, and figure out which fields are wrong.

### The three BMP header fields that matter for this challenge

| Offset | Size | Field | What it controls | What it should be here |
|---|---|---|---|---|
| 10 | 4 bytes | Pixel data offset | Where the pixel data starts in the file | `0x36` (54) — right after the 14-byte file header + 40-byte DIB header |
| 14 | 4 bytes | DIB header size | The size of the DIB header that follows | `0x28` (40) — `BITMAPINFOHEADER` |
| 22 | 4 bytes (signed) | Height | How many rows of pixels to read | `0x352` (850) — needed to fit the actual pixel data |

If any of these three are wrong, the image either does not open, opens as garbage, or shows only a slice of the picture (a "tunnel" through the original). The challenge title `tunn3l v1s10n` is a wink at the third failure mode: you can only see a narrow band of the image because the height is too small — the rest is "below" the visible region that the header is willing to admit exists.

### Why the file opens at all after the fix

Once the three fields are corrected, any standard image viewer (Wireshark for BMPs is not a thing — Wireshark is for packets — but `eog`, `feh`, `xdg-open`, `convert`, or PIL all work) will read the corrected header, seek to the pixel offset, and start decoding rows. Since the pixel data itself was never touched, the picture that emerges is bit-for-bit identical to whatever the original creator made.

### The standard toolchain for this kind of challenge

- **`file`** — quick sanity check. `file` reads the magic bytes of a file and prints what kind of file it thinks it is. A valid BMP starts with `"BM"`; if `file` says `data` instead of `PC bitmap`, the magic is wrong or the header is so mangled that `file` cannot parse it. This is usually the first hint that you have a header-corruption challenge on your hands.
- **`head -c N | od -An -tx1 -w16`** — view the first N bytes of the file as hex. The 14-byte BMP file header and the 40-byte DIB header are small enough to read in one screen.
- **Python with the `struct` module** — `struct.unpack_from('<I', data, offset)` reads a 4-byte little-endian unsigned int from a given offset. This is the easiest way to parse and re-pack a BMP header field by field.
- **`convert` (ImageMagick) or Python PIL** — render the fixed BMP to PNG so you can view it in a screenshot.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the file

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/tunn3l-v1s10n && cd /work/tunn3l-v1s10n

┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ cp ~/Downloads/tunn3l_v1s10n .
```

### Step 2 — Confirm what we have

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ file tunn3l_v1s10n
tunn3l_v1s10n: data

┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ ls -la tunn3l_v1s10n
-rw-r--r-- 1 zham zham 2893454 Jul 12 22:30 tunn3l_v1s10n
```

`file` says `data` — it does not recognize the file. That is a strong hint that the magic bytes are wrong, the file header is missing, or the DIB header is so mangled that `file` cannot identify the format. Time to look at the bytes.

### Step 3 — Hex dump of the first 64 bytes

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ head -c 64 tunn3l_v1s10n | od -An -tx1 -w16
 42 4d 8e 26 2c 00 00 00 00 00 ba d0 00 00 ba d0
 00 00 6e 04 00 00 32 01 00 00 01 00 18 00 00 00
 00 00 58 26 2c 00 25 16 00 00 25 16 00 00 00 00
 00 00 00 00 00 00
```

The first two bytes are `42 4D` = `"BM"`, so the file **is** a BMP — the magic is fine, but `file` was confused by the mangled DIB header. The DIB header is what tells `file` "yes, this is a real BMP and here is its size", and the DIB header size field is wrong. Let me parse the bytes by hand:

| Offset | Bytes | Field | Value | Status |
|---|---|---|---|---|
| 0 | `42 4d` | BMP signature "BM" | OK |
| 2 | `8e 26 2c 00` | File size (LE) | `0x002C268E` = 2,893,454 | OK — matches `ls -la` |
| 6 | `00 00` | Reserved1 | 0 | OK |
| 8 | `00 00` | Reserved2 | 0 | OK |
| 10 | `ba d0 00 00` | Pixel data offset | `0x0000D0BA` = 53,434 | **WRONG** — should be 54 (0x36) |
| 14 | `ba d0 00 00` | DIB header size | `0x0000D0BA` = 53,434 | **WRONG** — should be 40 (0x28) |
| 18 | `6e 04 00 00` | Width | 0x46E = 1134 | OK |
| 22 | `32 01 00 00` | Height | 0x132 = 306 | **WRONG** — should be 850 (0x352) |
| 26 | `01 00` | Planes | 1 | OK |
| 28 | `18 00` | Bits per pixel | 24 | OK |
| 30 | `00 00 00 00` | Compression | 0 (BI_RGB, none) | OK |
| 34 | `58 26 2c 00` | Image size | `0x002C2658` = 2,893,400 | OK — this is the actual pixel data size |
| 38 | `25 16 00 00` | X pixels-per-meter | 5669 | OK |
| 42 | `25 16 00 00` | Y pixels-per-meter | 5669 | OK |
| 46 | `00 00 00 00` | Colors used | 0 | OK |
| 50 | `00 00 00 00` | Important colors | 0 | OK |

Three wrong fields. Time to figure out the correct values.

### Step 4 — Confirm the correct DIB header size and pixel data offset

The DIB header for a 24-bit BMP with no color table is the standard `BITMAPINFOHEADER` — 40 bytes (`0x28`). The pixel data starts immediately after the 14-byte BMP file header + 40-byte DIB header = 54 bytes (`0x36`).

I also peek at byte 54 to confirm the pixel data is already sitting there (i.e., the data was placed at the right offset, and only the *header's claim* of the offset is wrong):

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ dd if=tunn3l_v1s10n bs=1 skip=54 count=64 2>/dev/null | od -An -tx1 -w16
 23 1a 17 27 1e 1b 29 20 1d 2a 21 1e 26 1d 1a 31
 28 25 35 2c 29 33 2a 27 38 2f 2c 2f 26 23 33 2a
 26 2d 24 20 3b 32 2e 32 29 25 30 27 23 33 2a 26
 38 2c 28 36 2b 27 39 2d 2b 2f 26 23 1d 12 0e 23
```

These bytes are not all zero and not all the same — they look like real pixel data (varying bytes in the 0x1A-0x3B range, which is mid-tone). So the data starts at offset 54, and the header's claim of `0xD0BA` is wrong.

### Step 5 — Figure out the correct height

The "image size" field at offset 34 is `0x002C2658` = 2,893,400. For a 24-bit BMP with no compression:

```
row_bytes = ceil(width * 3 / 4) * 4
          = ceil(1134 * 3 / 4) * 4
          = ceil(3402 / 4) * 4
          = 851 * 4
          = 3404
```

So if `height × 3404 = 2,893,400`, then `height = 850`. That is the original height. The current value of 306 was an attempt to "crop" or "shrink" the image so only a narrow band of the picture was visible — the "tunnel" in the title.

I verify with a one-liner:

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ python3 -c "print(2893400 // 3404, 2893400 % 3404)"
850 0
```

Clean. `2893400 / 3404 = 850` exactly. So the original image is `1134 × 850` × 24bpp.

### Step 6 — Patch the three corrupted fields

Python with the `struct` module is the cleanest way to do this:

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ nano fix_bmp.py
```

I paste the following in nano, then `Ctrl+O`, `Enter`, `Ctrl+X`:

```python
import struct

with open('tunn3l_v1s10n', 'rb') as f:
    data = bytearray(f.read())

# Fix 1: DIB header size at offset 14
struct.pack_into('<I', data, 14, 40)        # 0x28

# Fix 2: Pixel data offset at offset 10
struct.pack_into('<I', data, 10, 54)        # 0x36

# Fix 3: Height at offset 22
struct.pack_into('<i', data, 22, 850)       # 0x352, signed int

with open('tunn3l_v1s10n_fixed.bmp', 'wb') as f:
    f.write(data)
```

Then run it:

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ python3 fix_bmp.py
```

### Step 7 — Verify the patched file is a valid BMP

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ file tunn3l_v1s10n_fixed.bmp
tunn3l_v1s10n_fixed.bmp: PC bitmap, Windows 3.x format, 1134 x 850 x 24, image size 2893400, resolution 5669 x 5669 px/m, cbSize 2893454, bits offset 54
```

`file` now reports it as a proper 1134 × 850 × 24 BMP. The `bits offset 54` at the end confirms the pixel data offset was patched correctly.

### Step 8 — Render to PNG and view

I convert the BMP to PNG so any image viewer will open it:

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ convert tunn3l_v1s10n_fixed.bmp tunn3l_v1s10n_fixed.png
```

I open the PNG in my image viewer (feh, eog, xdg-open, or just reading the file directly):

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ xdg-open tunn3l_v1s10n_fixed.png
```

The image renders as a wide landscape: a winter scene of a Yosemite-style valley with snow-dusted granite cliffs, a pine forest, and a frozen creek in the foreground. Across the top of the image, in white text, is the flag:

```
picoCTF{qu1t3_a_v13w_2020}
```

(There is also a yellow `notaflag{sorry}` decoy baked into the middle of the picture. Ignore it.)

### Step 9 — Submit

```
picoCTF{qu1t3_a_v13w_2020}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Copied `tunn3l_v1s10n` into `/work/tunn3l-v1s10n` | Clean working directory |
| 0:00 | `file tunn3l_v1s10n` | `file` said `data` instead of `PC bitmap` — first sign the header is broken |
| 0:01 | `head -c 64 ... \| od -An -tx1 -w16` | Saw `"BM"` magic at the start (so it is a BMP) and three suspicious values: pixel offset `0xD0BA`, DIB header size `0xD0BA`, height `306` |
| 0:02 | `dd if=tunn3l_v1s10n bs=1 skip=54 count=64 ...` | Confirmed real pixel data starts at offset 54 |
| 0:02 | `python3 -c "print(2893400 // 3404, 2893400 % 3404)"` | `image_size / 3404 = 850` exactly, so the original height is 850 |
| 0:03 | Wrote `fix_bmp.py` and ran it | Patched offset 10 → 54, offset 14 → 40, offset 22 → 850 |
| 0:03 | `file tunn3l_v1s10n_fixed.bmp` | `file` now reports `PC bitmap, 1134 x 850 x 24` — valid BMP |
| 0:04 | `convert tunn3l_v1s10n_fixed.bmp ... .png` + view | Saw the winter landscape with `picoCTF{qu1t3_a_v13w_2020}` at the top |
| 0:04 | Submitted the flag | Challenge marked solved |

Internally, the file was a real BMP whose creator deliberately overwrote three header fields to make it unopenable. The actual pixel data was never touched — it was sitting at the natural offset (54 bytes into the file) and was large enough to fill 850 rows. The "tunnel" effect in the original 306-row render was a band of pixels near the bottom of the original picture (BMPs render bottom-up, so a low height hides the top of the image). The fix was just to declare the right header values so the image viewer would know to read all 850 rows.

---

## Alternative Methods

### Method 1 — Patch with a hex editor

If you do not want to write a Python script, any hex editor will do. The bytes to change are:

- Offset `10-13`: `ba d0 00 00` → `36 00 00 00`
- Offset `14-17`: `ba d0 00 00` → `28 00 00 00`
- Offset `22-25`: `32 01 00 00` → `52 03 00 00`

In `ghex` (Kali's default GUI hex editor), open the file, navigate to each offset, type the new bytes, save, and open the resulting `.bmp` in any image viewer.

### Method 2 — Use Python with `binascii` instead of `struct`

The `struct.pack_into` calls in my fix script can be replaced with raw byte writes:

```python
with open('tunn3l_v1s10n', 'rb') as f:
    data = bytearray(f.read())

# Patch using direct byte writes
data[10:14] = b'\x36\x00\x00\x00'   # pixel offset = 54
data[14:18] = b'\x28\x00\x00\x00'   # DIB header size = 40
data[22:26] = b'\x52\x03\x00\x00'   # height = 850

with open('tunn3l_v1s10n_fixed.bmp', 'wb') as f:
    f.write(data)
```

Same result, no `struct` import needed. Useful if you want a single-file one-liner that does not depend on a standard library.

### Method 3 — Use `convert` (ImageMagick) directly

ImageMagick's `convert` is surprisingly forgiving about a corrupted BMP and will sometimes render a damaged file even with a bad header, as long as the width, height, and bpp fields are sensible. If you cannot get the file to open as a BMP, try forcing the format:

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ convert -size 1134x850 -depth 8 bgr:tunn3l_v1s10n tunn3l_v1s10n.png 2>&1 | head
```

The `-size 1134x850` overrides the corrupted height, and `bgr:` tells ImageMagick to interpret the bytes as raw BGR pixel data (the on-disk order for 24-bit BMPs). This sidesteps the broken header entirely. The downside is that the image will be 850 rows tall but the pixel data starts at the wrong offset (0xD0BA instead of 54), so you will get the bottom 850 rows of the original file rather than the top 850 rows. For this challenge, that happens to still show the flag, but it is not the cleanest solve.

### Method 4 — Pure `xxd` patch from the terminal

If you prefer to live in the terminal and do not want to open a text editor, you can patch the three 4-byte values in place with `printf` and `dd`:

```
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ cp tunn3l_v1s10n tunn3l_v1s10n_fixed.bmp

# Patch offset 10-13 (pixel data offset) to 54
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ printf '\x36\x00\x00\x00' | dd of=tunn3l_v1s10n_fixed.bmp bs=1 seek=10 conv=notrunc

# Patch offset 14-17 (DIB header size) to 40
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ printf '\x28\x00\x00\x00' | dd of=tunn3l_v1s10n_fixed.bmp bs=1 seek=14 conv=notrunc

# Patch offset 22-25 (height) to 850
┌──(zham㉿kali)-[/work/tunn3l-v1s10n]
└─$ printf '\x52\x03\x00\x00' | dd of=tunn3l_v1s10n_fixed.bmp bs=1 seek=22 conv=notrunc
```

`conv=notrunc` is the critical flag — without it, `dd` would truncate the file to the new write size and lose the pixel data. With it, `dd` patches just the requested bytes and leaves the rest of the file untouched.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | First sanity check — `data` instead of `PC bitmap` told me the header was corrupt | Easy |
| `head -c N \| od -An -tx1 -w16` | Hex-dump the first 64 bytes of the file to read the BMP file header and DIB header by hand | Easy |
| `dd if=... bs=1 skip=... count=...` | Sample 64 bytes starting at offset 54 to confirm the pixel data is sitting right after the headers | Easy |
| Python + `struct` | Parse the corrupted fields, calculate the correct height from the `image_size` field, and patch the three 4-byte values in place | Medium |
| `convert` (ImageMagick) | Render the patched BMP to PNG so any image viewer can open it | Easy |
| `xdg-open` / `eog` / `feh` | Open the PNG and read the flag off the picture | Easy |
| Hex editor (`ghex`, `bless`) (alt) | Click-and-type the three 4-byte fixes in a GUI hex editor | Easy |
| `printf \| dd ... conv=notrunc` (alt) | Patch the three values in place from the shell, no Python needed | Medium |
| `convert -size 1134x850 bgr:...` (alt) | Force ImageMagick to interpret the raw bytes as 1134×850 BGR data, sidestepping the broken header | Medium |

---

## Key Takeaways

- **`file` is your first line of defense for format challenges.** A clean `data` response means `file` cannot recognize the file at all, which is almost always a sign of header corruption. A `PC bitmap` response would have meant the file is fine; the `data` response told me to crack open a hex editor and start reading bytes.
- **The BMP header is small enough to read in one screen.** 14 bytes of file header + 40 bytes of DIB header = 54 bytes total. A single `head -c 64 ... | od -An -tx1 -w16` fits both on one terminal page. Every field's location is documented in the BMP spec, so once you have the hex dump, you can check each field against the expected value one by one.
- **BMPs are bottom-up.** This is the most common newbie gotcha. When a 24-bit BMP says "height = 306", the 306 rows that get rendered are the *bottom* 306 rows of the picture. The top of the picture is at the *end* of the file. That is why the "tunnel" in the challenge title was at the *bottom* of the 306-row render — the top of the original was being chopped off, not the bottom.
- **The `image_size` field is your secret decoder ring for height problems.** The BMP format specifies that the `image_size` field must be set to `width × row_bytes` for BI_RGB, even though some generators leave it as 0. When it is non-zero, you can recover the original height by dividing it by the row size. The challenge creator accidentally left this field intact, which is the only reason we can solve it without guessing.
- **Always patch the file, not the original.** When the fix is small and reversible, copy the file first (`cp tunn3l_v1s10n tunn3l_v1s10n_fixed.bmp`) and patch the copy. This way, if you make a mistake, you can recopy and try again without re-downloading.
- **`dd conv=notrunc` is the most dangerous "safe-looking" command in Linux.** It patches bytes in place. Forgetting `conv=notrunc` when writing 4 bytes to the middle of a 2.8 MB file will silently truncate the file to 4 bytes and lose everything past the write point. Always triple-check the `bs=1 seek=N` arguments and the `conv=notrunc` flag.
- **The challenge title is a hint, not just a name.** "tunn3l v1s10n" (tunnel vision) is a literal description of what the broken header does — it shows you only a thin band of the picture, like looking through a cardboard tube. Once you fix the header, the "tunnel" widens to the full view, and the flag's wordplay (`qu1t3_a_v13w` = "quite a view") rewards you for noticing.

### Flag wordplay decode

```
picoCTF{qu1t3_a_v13w_2020}
        ||||||  ||||| ||||
        ||||||  ||||| 2020 = the year of the challenge / the year the photo was taken
        ||||||  v13w  = "view" (1 → I, 3 → E, classic leet)
        ||||||  _ = literal underscore
        qu1t3 = "quite" (1 → I, 3 → E)
        a = literal "a"
```

The whole chunk reads as **"quite a view, 2020"** — a perfect one-liner for a challenge where the only thing standing between you and the full picture was a deliberately-mangled BMP header. The wordplay works on two levels:

1. **Surface reading**: once you fix the header, the image is literally a wide scenic view of a Yosemite-style valley in winter, and the flag comments on how nice that view is.
2. **Meta reading**: the whole challenge is about getting "tunnel vision" — only seeing a small band of the picture — and the act of solving the challenge is what gives you the "full view". The flag's "quite a view" is the challenge author winking at the player: "congrats, you widened your tunnel — now you can see the whole picture".

The `2020` suffix is the year, doubling as a timestamp: this view is from 2020, the year the challenge was made. The author of the challenge (Danny, per the challenge page) is a fan of the wordplay, which is also why the challenge name is itself a leet-speak pun on "tunnel vision".
