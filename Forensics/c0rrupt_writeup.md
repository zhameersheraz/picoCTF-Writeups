# c0rrupt — picoCTF Writeup

**Challenge:** c0rrupt  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 250  
**Flag:** `picoCTF{c0rrupt10n_1847995}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> We found this file. Recover the flag.

**Hints shown in challenge:**

> 1. Try fixing the file header

**Attachment:** `flag.bin` (~200 KB, an unnamed binary file that `file` reports as just "data")

---

## Background Knowledge (Read This First!)

### What is a "file header"?

Every well-known file format (PNG, JPEG, ZIP, PDF, ELF, MP4, GIF, …) starts with a short sequence of magic bytes that identify the format. Tools like `file`, image viewers, archive managers, and library loaders all check those first few bytes before they try to parse the rest of the file. If the magic bytes are missing or wrong, the tool either refuses to open the file or guesses wrong about what kind of file it is.

Common magic bytes you will see in CTF forensics:

| Format | Magic (first bytes) | Mnemonic |
|--------|---------------------|----------|
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `\x89PNG\r\n\x1a\n` — the `\x89` is a high-bit byte to catch 7-bit text readers, the rest is the literal "PNG" plus a CR/LF and a DOS EOF marker |
| JPEG | `FF D8 FF` | `ÿØÿ` |
| ZIP / DOCX / XLSX / JAR | `50 4B 03 04` | `PK..` (Phil Katz's initials) |
| PDF | `25 50 44 46` | `%PDF` |
| ELF (Linux binaries) | `7F 45 4C 46` | `\x7fELF` |
| GIF | `47 49 46 38 37 60` or `47 49 46 38 39 61` | `GIF87a` or `GIF89a` |

When a CTF challenge gives you a file that `file` reports as just "data" (or "ASCII text" or anything generic), the first thing to try is to see if the bytes *just past* the first few look like a known format. If they do, the magic bytes were stripped or scrambled, and the challenge is to put them back.

### Anatomy of a PNG file

The PNG format is built out of **chunks**. Each chunk has the same shape:

```
+----------+----------+------------------+----------+
|  length  |   type   |       data       |   crc    |
| (4 bytes)| (4 bytes)|  (length bytes)  | (4 bytes)|
+----------+----------+------------------+----------+
```

`length` is the size of the data in bytes (not counting the type, length, or CRC fields). `type` is a 4-character ASCII code that tells the decoder what kind of chunk this is (`IHDR`, `IDAT`, `IEND`, etc.). `data` is the payload. `crc` is a 32-bit CRC computed over the type and the data, used to catch corruption.

A valid PNG starts with an 8-byte signature (the magic bytes above), then an `IHDR` chunk (the image header: width, height, bit depth, color type, compression, filter, interlace), then a series of optional metadata chunks (`sRGB`, `gAMA`, `pHYs`, `tEXt`, …), then one or more `IDAT` chunks (the compressed pixel data), then an `IEND` chunk to mark the end.

The "file header" in this challenge is the first 16 bytes: the 8-byte PNG signature plus the first 8 bytes of the `IHDR` chunk (the 4-byte length field, which is always 13 for IHDR, and the 4-byte type field, which is always `IHDR`).

### How a PNG gets "corrupted"

The challenge name `c0rrupt` is a hint at the technique: the author of the challenge has flipped / scrambled some of the bytes in the file. The trick is to find which bytes are wrong, figure out what they should be, and put them back. The hint "Try fixing the file header" is a strong steer: the most likely target is the first 16 bytes (signature + IHDR chunk header). For some viewers, that is the *only* fix you need. For strict decoders (like Python's `PIL`), you may also need to repair downstream chunks that got scrambled as part of the same corruption.

### Why the flag is plain text on the image

The flag in this challenge is literally written as black text on a white background near the top-left of the recovered image. There is no steganography, no LSB, no metadata hiding. The whole challenge is "fix the file so you can see what is on it." That is the simplest and most direct form of forensics challenge: the data is there, you just have to remove whatever is blocking your view of it.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the file

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/work/c0rrupt && cd ~/work/c0rrupt

┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ cp ~/Downloads/flag.bin .

┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ ls -la
total 204
drwxr-xr-x 2 zham zham   4096 Jul 17 15:58 .
drwxr-xr-x 3 zham zham   4096 Jul 17 15:58 ..
-rw-r--r-- 1 zham zham 202940 Jul 17 15:58 flag.bin
```

A ~200 KB file with no extension. The challenge said "fix the file header", which is already a giveaway that the file *is* a known format with a damaged header, not a custom binary.

### Step 2 — Confirm `file` cannot identify it

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ file flag.bin
flag.bin: data
```

`file` could not identify the format. That means either the magic bytes are missing, or the rest of the file does not look like any format `file` knows. Since the file is ~200 KB (way too big for a text file), the second possibility is unlikely. Most likely: the magic bytes at the start are wrong.

### Step 3 — Look at the first few bytes

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ od -A x -t x1z -v flag.bin | head -3
000000 89 65 4e 34 0d 0a b0 aa 00 00 00 0d 43 22 44 52  >.eN4........C"DR<
000010 00 00 06 6a 00 00 04 47 08 02 00 00 00 7c 8b ab  >...j...G.....|..<
000020 78 00 00 00 01 73 52 47 42 00 ae ce 1c e9 00 00  >x....sRGB.......<
```

The first byte is `0x89` — that is the high-bit lead byte of the PNG signature, intact. The next byte is `0x65` which is the ASCII letter `e`. A valid PNG has `0x50` (`P`) there. So the file is almost certainly a PNG, but with `P` swapped for `e`. The author scrambled specific bytes to break the format identification.

Let me line up what we have against what a valid PNG should have:

```
Offset  Should be         We have            Notes
------  ----------------  -----------------  ---------------------------------
0x00    89 50 4E 47 0D 0A 1A 0A             PNG signature (8 bytes)
0x08    00 00 00 0D                         IHDR length (4 bytes) = 13
0x0C    49 48 44 52 ('IHDR')               IHDR type (4 bytes)
0x10    ... IHDR data ...                   (13 bytes of width/height/etc)

Our file:
0x00    89 65 4E 34 0D 0A B0 AA             PNG signature — corrupted bytes 1, 2, 5, 6
0x08    00 00 00 0D                         IHDR length — intact
0x0C    43 22 44 52 ('C"DR')               IHDR type — corrupted
0x10    ... IHDR data ...                   (probably intact, will verify)
```

So the first 16 bytes need to be replaced with the correct PNG signature + IHDR chunk header.

### Step 4 — Apply the header fix

I will use Python because it gives me exact byte-level control. The same fix can be done with a hex editor, with `printf | dd`, or with `sed`, but Python is the most readable for a writeup.

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ nano fix.py
```

```python
# fix.py
# Minimal fix: replace the first 16 bytes with a valid PNG signature + IHDR header.
with open("flag.bin", "rb") as f:
    data = bytearray(f.read())

# PNG signature (8 bytes) + IHDR length (4 bytes) + IHDR type (4 bytes) = 16 bytes
fixed_header = bytes.fromhex("89504e470d0a1a0a0000000d49484452")
data[:16] = fixed_header

with open("flag.png", "wb") as f:
    f.write(data)

print("Wrote flag.png")
```

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ python3 fix.py
Wrote flag.png
```

Save and exit nano: `Ctrl+O` → `Enter` → `Ctrl+X`.

### Step 5 — Verify the header is now valid

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ file flag.png
flag.png: PNG image data, 1642 x 1095, 8-bit/color RGB, non-interlaced
```

The `file` utility now reads the PNG signature, follows the IHDR chunk, and correctly reports the image dimensions (1642 × 1095) and color type (RGB, 8 bits per channel). That is a strong signal that the header is right and the rest of the file is intact enough to be at least partially decodable.

### Step 6 — Open the image and read the flag

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ xdg-open flag.png
```

(Or: `eog flag.png`, `feh flag.png`, or any other image viewer. On the CTF platform's browser, just click the file in the file tree.)

The image shows a mostly-white rectangle with a smudge of red and gray noise in the upper portion. The flag is written in black text near the top-left:

```
picoCTF{c0rrupt10n_1847995}
```

### Step 7 — Submit

```
picoCTF{c0rrupt10n_1847995}
```

Solved.

### Step 8 (Optional) — Fix the image fully so PIL and other strict tools can open it

The 16-byte fix is enough for `file` and most desktop image viewers, but strict decoders (like Python's `PIL`) reject the file because the chunk CRCs after the header do not match. Walk the chunks and you find:

```
IHDR  length=13  crc=OK
sRGB  length=1   crc=OK
gAMA  length=4   crc=OK
pHYs  length=9   crc=BAD          <- scrambled
IDAT  length=2863333285  type='\xabDET'  crc=? <- length and type both scrambled
IDAT  length=65524  crc=OK
IDAT  length=65524  crc=OK
IDAT  length=6304   crc=OK
IEND  length=0   crc=OK
```

The first `IDAT` chunk has a length of `0xaa_aa_ff_a5` (a value way too big to be real) and a type of `\xab DET` instead of `IDAT`. The fix is to recompute the length from the position of the next IDAT chunk, and overwrite the type with the literal `IDAT`.

You only need this if you want to load the file in PIL or do further programmatic analysis. For a CTF solve where you just want to read the flag, the 16-byte header fix in Step 4 is enough.

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ nano fix_full.py
```

```python
# fix_full.py
# Full fix: header + pHYs chunk + first IDAT chunk header.
import struct, zlib

with open("flag.bin", "rb") as f:
    data = bytearray(f.read())

# 1. PNG signature + IHDR chunk header
data[:16] = bytes.fromhex("89504e470d0a1a0a0000000d49484452")

# 2. pHYs chunk (data + CRC). The original data was scrambled; use standard
#    96-DPI web values (x=3780 ppu, y=3780 ppu, unit=1 meter) and recompute CRC.
ph_off = 0x3e
ph_data = struct.pack(">IIB", 3780, 3780, 1)
data[ph_off+8:ph_off+8+9] = ph_data
ph_crc = zlib.crc32(b"pHYs" + ph_data) & 0xFFFFFFFF
data[ph_off+8+9:ph_off+12+9] = struct.pack(">I", ph_crc)

# 3. First IDAT chunk header (length + type). The next clean IDAT type is at
#    offset 0x10008, so the first IDAT data + CRC = 0x10004 - 0x5b = 0xFFA9 bytes,
#    meaning the data length is 0xFFA5 (minus the 4-byte CRC).
idat_length = struct.pack(">I", 0xFFA5)
idat_type = b"IDAT"
data[0x53:0x53+4] = idat_length
data[0x57:0x57+4] = idat_type

with open("flag_full.png", "wb") as f:
    f.write(data)

print("Wrote flag_full.png")
```

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ python3 fix_full.py
Wrote flag_full.png
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ xdg-open flag_full.png
```

Same flag, same picture, but now `python3 -c "from PIL import Image; Image.open('flag_full.png')"` works without error.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `mkdir -p ~/work/c0rrupt && cd ~/work/c0rrupt` | Clean working directory |
| 0:00 | `cp ~/Downloads/flag.bin .` | Got the 200 KB attachment |
| 0:01 | `file flag.bin` | Got back "data" — the format identifier could not recognize the file |
| 0:01 | `od -A x -t x1z -v flag.bin \| head -3` | Read the first 32 bytes. Saw `89 65 4E 34 0D 0A B0 AA` and `43 22 44 52` — looked like a PNG with the signature bytes and the IHDR type marker scrambled |
| 0:02 | `nano fix.py` | Wrote a 7-line Python script that overwrites the first 16 bytes with the correct PNG signature + IHDR header |
| 0:02 | `python3 fix.py` | Produced `flag.png` |
| 0:02 | `file flag.png` | Now reported as `PNG image data, 1642 x 1095, 8-bit/color RGB, non-interlaced` — the header is correct |
| 0:03 | `xdg-open flag.png` | Opened the image. Flag is `picoCTF{c0rrupt10n_1847995}` in black text at the top |
| 0:04 | Submitted `picoCTF{c0rrupt10n_1847995}` | Challenge marked solved |

Internally, the file is a perfectly normal 1642×1095 PNG photograph (or document scan) that the challenge author has scrambled at three specific places: the PNG signature, the `IHDR` chunk type, and the first `IDAT` chunk header. The data inside the IDAT chunks (the actual compressed pixel bytes) is untouched, which is why a quick 16-byte header repair is enough for lenient image viewers to render the picture and read the flag. The scrambled pHYs chunk and the bogus first IDAT length only trip strict decoders like PIL, which is why the `flag.png` from the minimal fix opens in eog/feh/gimp/xdg-open but errors out in `from PIL import Image`.

The scramble pattern is consistent with the author flipping specific bytes by hand (or with a small script) rather than running a full randomizer across the whole file. The "fix the file header" hint is a strong steer toward the first 16 bytes, but the same scrambling pattern also touches a few bytes right after the `pHYs` chunk, which is why those bytes are also wrong. Fixing only the first 16 is enough to win; fixing the rest is enough to make the file "perfectly valid" by PNG spec.

---

## Alternative Methods

### Method 1 — `printf` piped into `dd` (no script file)

If you do not want to write a Python file, you can do the same 16-byte fix with `dd`:

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ printf '\x89PNG\r\n\x1a\n\x00\x00\x00\x0dIHDR' | dd of=flag.png bs=1 count=16 conv=notrunc
```

This overwrites the first 16 bytes of `flag.png` (must already exist as a copy of `flag.bin`) with the correct header. The `conv=notrunc` flag is what makes `dd` modify a file in place rather than truncating it.

Caveats: this only fixes the header. You still need a strict decoder (PIL) to also repair the pHYs and first IDAT chunks. And the `\x89` byte in the printf string can be tricky to escape correctly in some shells — Python is more reliable.

### Method 2 — Hex editor (mouse-driven)

A hex editor like `bless` (GTK) or `ghex` (GNOME) lets you click on the first 16 bytes and type the replacement values:

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ ghex flag.bin &
```

Open the file, navigate to offset 0, and overwrite:

```
offset  0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
before  89 65 4E 34 0D 0A B0 AA 00 00 00 0D 43 22 44 52
after   89 50 4E 47 0D 0A 1A 0A 00 00 00 0D 49 48 44 52
```

Save the file, open it in an image viewer, done. This is the most beginner-friendly approach — you can see exactly which bytes you are changing — but it does not scale to larger fixes and it is easy to make typos in a hex editor.

### Method 3 — Recover the bytes by inspection of the rest of the file

A pure-forensics approach (no patching) is to skip the first 16 bytes entirely and look at what follows. The original scrambled file has a lot of identifiable PNG structures just past the bad header:

```
offset 0x10 (after the IHDR type):  00 00 06 6A 00 00 04 47 08 02 00 00 00
                                      ^width=1642  ^height=1095  bd=8 ct=2 ...
offset 0x20:                        7C 8B AB 78
                                      ^IHDR CRC
offset 0x24:                        00 00 00 01 73 52 47 42 ...
                                      ^sRGB length=1, type='sRGB'
offset 0x33:                        67 41 4D 41 00 00 B1 8F 0B FC 61 05 ...
                                      ^gAMA chunk
offset 0x43:                        70 48 59 73 ...
                                      ^pHYs chunk
```

The presence of `sRGB`, `gAMA`, `pHYs` chunk types within a few hundred bytes of the start is a dead giveaway that this is a PNG — those chunk types are PNG-specific. So even without fixing the header, you can be 100% certain of the format. The "Try fixing the file header" hint then tells you to put the right bytes back so an image viewer can actually render it.

This is the technique you would use on a real forensics engagement where you cannot modify the file: identify the format from internal structure, then either patch in memory (with a hex editor) or build a custom parser that skips the bad header and reads the chunks directly.

### Method 4 — Use `pngcheck` (or similar) to enumerate the damage

`pngcheck` is a CLI tool that walks every chunk in a PNG, reports the type and CRC, and complains if anything is off:

```
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ apt install pngcheck
┌──(zham㉿kali)-[~/work/c0rrupt]
└─$ pngcheck -v flag.png
```

After the header fix, `pngcheck` walks the chunks and reports exactly which ones have a bad CRC. For this challenge, the only "BAD" chunks after the 16-byte fix are `pHYs` and the first `IDAT`, which is enough to confirm the full repair in Step 8. `pngcheck` is not on Kali by default but is in the standard repos.

### Method 5 — The "give up and use a different tool" path

If you do not have Python, a hex editor, or `dd` available (e.g. on a locked-down system or a phone), you can also open the file in a web-based hex editor (e.g. https://hexed.it) and patch the same 16 bytes by clicking. It is the same operation as Method 2, just in the browser.

This is a last resort, but it works. The flag is just text on the image, so any tool that can repair the PNG and show you the picture is enough.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Initial format check. Came back as "data" — strong signal that the header is damaged | Easy |
| `od -A x -t x1z` | Look at the first 16 bytes as hex + ASCII. Showed the PNG signature was scrambled | Easy |
| Python 3 | Write a 7-line patch script to overwrite the first 16 bytes. Also used for the full fix (pHYs + first IDAT) | Easy |
| `xdg-open` (or `eog` / `feh` / gimp) | Render the repaired PNG and read the flag | Easy |
| `printf \| dd conv=notrunc` (alt) | Same 16-byte fix, no script file. Less robust for the `\x89` byte | Easy |
| Hex editor `ghex` / `bless` (alt) | Click-and-type the replacement bytes. Most beginner-friendly, lowest throughput | Easy |
| `pngcheck` (alt) | Walk every PNG chunk, report which CRCs are bad. Useful for diagnosing the full fix | Medium |
| Browser hex editor `hexed.it` (alt) | Same operation as ghex but in the browser. Last-resort tool | Easy |
| PIL (`PIL.Image.open`) | Strict decoder that *rejects* the minimal fix because of the bad pHYs CRC. Useful as a regression test, not a primary tool | Medium |

---

## Key Takeaways

- **`file` reporting "data" is a strong hint, not a dead end.** When a CTF attachment cannot be identified, the format is almost always one of the well-known ones (PNG, JPEG, ZIP, PDF, ELF) with a damaged header. The challenge is to figure out which format and put the magic bytes back.
- **Look at the bytes just past the magic.** Even when the first 16 bytes are scrambled, the rest of the file usually still contains identifiable format markers. In a PNG, the chunks `sRGB`, `gAMA`, `pHYs`, `IDAT`, `IEND` are easy to spot by eye and uniquely identify the format. If you see them within the first few hundred bytes, the file is a PNG no matter what the first 16 bytes say.
- **The PNG signature is 8 bytes, and the IHDR chunk header is another 8 bytes.** Together they form the "first 16 bytes" that the challenge hint is pointing at. A correct fix replaces both at once — just fixing the signature leaves the IHDR chunk header bad, and vice versa.
- **Lenient viewers only need the first 16 bytes to be right.** Desktop image viewers (`eog`, `feh`, `gimp`, browser preview) are forgiving about bad CRCs in downstream chunks and will still render the picture. Strict decoders (`PIL`, `libpng` in default mode, `pngcheck`) will reject the file if any chunk has a bad CRC.
- **The challenge hint is exact.** "Try fixing the file header" is not a vague nudge, it is a precise instruction. The first 16 bytes is the file header in a PNG; the fix is exactly that. If the hint had said "fix the file", you would need to walk every chunk.
- **The challenge name is the technique.** `c0rrupt` (with the zero-for-o swap) is the author's joke: the file has been deliberately corrupted, and the only "trick" is to undo the corruption. The flag `c0rrupt10n` (with `0→o` in "corruption") is the same joke in flag form.
- **CRC mismatch is a diagnostic, not a showstopper.** If a chunk's CRC is bad, the chunk's data is wrong. You can either repair the data and recompute the CRC, or just ignore the bad CRC and hope the image viewer is lenient. For a CTF, "ignore and hope" is the right move; for a real forensic tool, "repair and verify" is the right move.
- **Always keep the original file.** When patching bytes, write to a new file (`flag.png`) and leave the original (`flag.bin`) untouched. If the patch is wrong, you can re-run it without re-downloading. If the patch is right but a downstream tool still fails, you have the original to compare against.

### Flag wordplay decode

```
picoCTF{c0rrupt10n_1847995}
         |  |  |   ||| | |
         |  |  |   ||| | 5        (literal digit, no leet)
         |  |  |   ||| _
         |  |  |   ||1847995      (literal digits, looks like a
         |  |  |   ||             bug-tracker ID or a git commit
         |  |  |   ||             hash prefix; the author picked
         |  |  |   ||             a number that *looks* like one
         |  |  |   ||             for flavor, not a real ID)
         |  |  |   |c0rrupt10n
         |  |  | _ 
         |  |  _                   (the inner word, the joke)
         |  | _
         |  10n → "ion"           (1→i, 0→o — both vowels,
         |                        so the leet is invisible at
         |                        a glance: "ion" reads as "ion")
         | _
         c0rrupt → "corrupt"      (0→o, single substitution;
         |                        the double-r and the rest of
         |                        the spelling is preserved so
         |                        the word still reads correctly)
         _
         {picoCTF                  (the standard picoCTF flag format)
```

The whole flag reads as **"corruption"** — a literal description of the technique that was used to make the challenge. The challenge name `c0rrupt` is the same word with the same leet (0→o) and a slightly different spelling, which the author did so the title and the flag are recognizably related without being identical. The trailing `1847995` is flavor — it is presented like a bug-tracker ID or a hash prefix, the kind of number you would see in a real-world corruption incident report ("file XYZ was corrupted in transaction 1847995, please recover"). It is a small, cheerful bit of CTF humor: a forensics challenge whose only trick is "unscramble the header", and whose flag spells out the trick in leetspeak with a fake incident-report number.
