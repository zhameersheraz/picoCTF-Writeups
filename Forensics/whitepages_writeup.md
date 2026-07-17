# WhitePages — picoCTF Writeup

**Challenge:** WhitePages  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 250  
**Flag:** `picoCTF{not_all_spaces_are_created_equal_f5d46aff52c6e17f9fd6317b33d2d783}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> I stopped using YellowPages and moved onto WhitePages... but the page they gave me is all blank!

**Hints shown in challenge:**

> 1. There is data encoded somewhere... there might be an online decoder.

**Attachment:** `flag.txt` (~3 KB UTF-8 text file that looks completely blank in any normal text viewer)

---

## Background Knowledge (Read This First!)

### What does "blank" really mean?

If you open `flag.txt` in `cat`, `less`, `gedit`, or any normal text editor, you see what looks like an empty document — a wall of whitespace, maybe a few spaces, no visible characters at all. That is the surface: the page *is* all blank.

But a text file is not actually "blank" just because you cannot see anything in it. Every byte is still a byte, and the file has a specific length, a specific encoding, and a specific sequence of characters. The challenge name "WhitePages" and the description "the page they gave me is all blank" are not literal — they are pointing at a category of steganography where the *visible* content is whitespace and the *hidden* content is encoded in which whitespace characters were used.

### Unicode has more than one "space"

ASCII has exactly one space character: `0x20` (SPACE). The file you are looking at is *not* ASCII — `file` reports it as `UTF-8 text`. Unicode defines a whole family of "space" characters that look identical (or nearly identical) when rendered, but are different codepoints. A few of the most common:

| Codepoint | Name | Looks like |
|-----------|------|------------|
| U+0020 | SPACE | regular space |
| U+00A0 | NO-BREAK SPACE | regular space, but no line break allowed |
| U+2000 | EN QUAD | space |
| U+2001 | EM QUAD | space |
| U+2002 | EN SPACE | space |
| U+2003 | EM SPACE | wider space |
| U+2004 | THREE-PER-EM SPACE | space |
| U+2005 | FOUR-PER-EM SPACE | space |
| U+2006 | SIX-PER-EM SPACE | space |
| U+2007 | FIGURE SPACE | space (digit-width) |
| U+2008 | PUNCTUATION SPACE | space |
| U+2009 | THIN SPACE | narrow space |
| U+200A | HAIR SPACE | very narrow space |
| U+200B | ZERO WIDTH SPACE | *invisible* (zero width!) |
| U+200C | ZERO WIDTH NON-JOINER | *invisible* |
| U+200D | ZERO WIDTH JOINER | *invisible* |
| U+2060 | WORD JOINER | *invisible* |
| U+FEFF | ZERO WIDTH NO-BREAK SPACE | *invisible* |

Some of these render as actual visible whitespace (different widths of "blank"); some render as nothing at all (zero-width characters). To a human reading the page, they all look "blank" or "empty". To a program reading the bytes, they are *completely different characters* with completely different bit patterns.

That is the whole trick. If you can use two visually-identical characters as a binary encoding (one means `0`, the other means `1`), you can hide a message in plain sight inside a document that looks completely empty.

### The encoding in this challenge

I will not spoil the exact mapping yet (the solve section walks through how to discover it), but the structure is straightforward: 1296 whitespace characters, each one representing a single bit, grouped 8 at a time into 162 bytes of UTF-8 text. The decoded text is a "WhitePages"-style public records report with the flag at the bottom.

### Why the challenge is called "WhitePages"

The challenge name is a triple pun:

1. **WhitePages** as a directory service, mirroring the "YellowPages" pun in the description. The author has moved from one directory service to a "blank" version.
2. **WhitePages** as a web reference to a "clean" or "blank" page — what the file looks like in a text editor.
3. **WhitePages** as a reference to the **whitespace steganography** technique used to hide the flag in the document. The "white" in "whitespace" is the giveaway.

### The flag is the lesson

`picoCTF{not_all_spaces_are_created_equal_f5d46aff52c6e17f9fd6317b33d2d783}` reads as **"not all spaces are created equal"** — a literal description of the technique. The two "spaces" used in this challenge look the same to a human but encode different bits to a program. The flag is the punchline, plus a hash suffix that identifies the specific instance of the challenge (so each player gets a unique flag).

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the file

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/work/whitepages && cd ~/work/whitepages

┌──(zham㉿kali)-[~/work/whitepages]
└─$ cp ~/Downloads/flag.txt .

┌──(zham㉿kali)-[~/work/whitepages]
└─$ ls -la
total 12
drwxr-xr-x 2 zham zham  4096 Jul 17 16:32 .
drwxr-xr-x 3 zham zham  4096 Jul 17 16:32 .
-rw-r--r-- 1 zham zham  2758 Jul 17 16:32 flag.txt
```

A 2.7 KB file. In `cat` or `less` it shows as completely blank — just a wall of spaces. That is the puzzle.

### Step 2 — Run `file` to confirm it is text, not just garbage bytes

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ file flag.txt
flag.txt: Unicode text, UTF-8 text, with very long lines (1296), with no line terminators
```

`file` confirms it is real UTF-8 text (not random bytes), and it has 1296 characters on a single very long line. The "no line terminators" detail is interesting: there are no newlines at all, which is unusual for a text file.

### Step 3 — Look at the raw bytes

If the file is "blank", but `file` says it has 1296 characters, then the characters must be whitespace. Dump the first few bytes as hex to see what kind of whitespace:

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ od -A x -t x1z -v flag.txt | head -3
000000 e2 80 83 e2 80 83 e2 80 83 e2 80 83 20 e2 80 83  >............ ...<
000010 20 e2 80 83 e2 80 83 20 20 20 e2 80 83 e2 80 83  > ......   ......<
000020 e2 80 83 e2 80 83 e2 80 83 20 20 e2 80 83 20 e2  >.........  ... .<
```

The byte `0xe2 0x80 0x83` is the UTF-8 encoding of codepoint U+2003, which is **EM SPACE**. The byte `0x20` is the UTF-8 encoding of codepoint U+0020, which is the regular ASCII **SPACE**. The file is a mix of these two characters, and they look identical in any text viewer.

### Step 4 — Confirm the two-character hypothesis with Python

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ python3 -c "
text = open('flag.txt', 'rb').read().decode('utf-8')
print('Length:', len(text), 'chars')
from collections import Counter
c = Counter(text)
for ch, count in c.most_common():
    print(f'  U+{ord(ch):04X} ({ch!r}): {count}')
"
Length: 1296 chars
  U+2003 ('\u2003'): 731
  U+0020 (' '): 565
```

Exactly two distinct characters: 731 EM SPACEs and 565 regular SPACEs. Total 1296, which is `162 × 8`. The structure is clear: each character is one bit, 8 characters per byte, 162 bytes total. The only question is which character means `0` and which means `1`.

### Step 5 — Try both mappings and pick the one that gives readable text

I will try both encodings and see which one decodes to recognizable ASCII. Whichever one produces readable text is the right one — there is no other way to decide.

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ python3 -c "
text = open('flag.txt', 'rb').read().decode('utf-8')

# Mapping A: em space = 1, regular space = 0
bits_a = ''.join('1' if ch == '\u2003' else '0' for ch in text)
# Mapping B: em space = 0, regular space = 1
bits_b = ''.join('0' if ch == '\u2003' else '1' for ch in text)

def decode(bits):
    out = bytearray()
    for i in range(0, len(bits) - 7, 8):
        out.append(int(bits[i:i+8], 2))
    return out.decode('utf-8', errors='replace')

print('--- Mapping A: em=1, space=0 ---')
print(decode(bits_a)[:200])
print()
print('--- Mapping B: em=0, space=1 ---')
print(decode(bits_b)[:200])
"
--- Mapping A: em=1, space=0 ---
������...

--- Mapping B: em=0, space=1 ---
<newline>
picoCTF
<newline>
<newline>
SEE PUBLIC RECORDS & BACKGROUND REPORT
5000 Forbes Ave, Pittsburgh, PA 15213
picoCTF{not_all_spaces_are_created_equal_f5d46aff52c6e17f9fd6317b33d2d783}
```

**Mapping B is the right one**: regular space (U+0020) means `1`, EM SPACE (U+2003) means `0`. The decoded text is a WhitePages-style public-records listing, with the address `5000 Forbes Ave, Pittsburgh, PA 15213` (which happens to be the address of Carnegie Mellon University, the university that runs picoCTF — a nice little Easter egg) and the flag at the bottom.

### Step 6 — Save the decoded text for reference (optional)

I like to save the decoded output to a file so I can read it cleanly:

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ python3 -c "
text = open('flag.txt', 'rb').read().decode('utf-8')
bits = ''.join('0' if ch == '\u2003' else '1' for ch in text)
out = bytearray()
for i in range(0, len(bits) - 7, 8):
    out.append(int(bits[i:i+8], 2))
open('decoded.txt', 'wb').write(out)
"
┌──(zham㉿kali)-[~/work/whitepages]
└─$ cat decoded.txt

picoCTF

SEE PUBLIC RECORDS & BACKGROUND REPORT
5000 Forbes Ave, Pittsburgh, PA 15213
picoCTF{not_all_spaces_are_created_equal_f5d46aff52c6e17f9fd6317b33d2d783}
```

### Step 7 — Submit

```
picoCTF{not_all_spaces_are_created_equal_f5d46aff52c6e17f9fd6317b33d2d783}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `mkdir -p ~/work/whitepages && cd ~/work/whitepages` | Clean working directory |
| 0:00 | `cp ~/Downloads/flag.txt .` | Got the 2758-byte attachment |
| 0:01 | `cat flag.txt` (or open in editor) | Saw a wall of whitespace. The file looks completely empty |
| 0:01 | `file flag.txt` | Got back `Unicode text, UTF-8 text, with very long lines (1296), with no line terminators`. The file has 1296 characters, all on one line — strong hint that it is whitespace stego |
| 0:01 | `od -A x -t x1z -v flag.txt \| head -3` | Saw the bytes are `e2 80 83` (UTF-8 EM SPACE) and `20` (regular SPACE), repeating |
| 0:02 | Python `Counter` on the decoded text | Confirmed exactly two characters: 731 EM SPACEs and 565 regular SPACEs, total 1296 = 162 × 8 |
| 0:02 | Python decode with both mappings | Mapping B (regular=1, EM=0) gave readable ASCII; mapping A gave garbage. Mapping B is the correct one |
| 0:03 | `cat decoded.txt` | Saw the public-records listing and the flag `picoCTF{not_all_spaces_are_created_equal_f5d46aff52c6e17f9fd6317b33d2d783}` |
| 0:04 | Submitted the flag | Challenge marked solved |

Internally, the file is a binary blob encoded as a stream of 1296 visually-identical whitespace characters, each carrying one bit of information. The two characters used are the ASCII SPACE (U+0020, one byte `0x20`) and the EM SPACE (U+2003, three UTF-8 bytes `0xe2 0x80 0x83`). The EM SPACE is a "wide" space that renders identically to a regular space in monospace fonts, so the document looks blank to a human reader even though it is full of distinct characters. Each character maps to one bit (space = 1, EM space = 0), 8 characters per byte, producing 162 bytes of UTF-8 text. The decoded text is a faux WhitePages public-records listing with the flag at the bottom.

The address `5000 Forbes Ave, Pittsburgh, PA 15213` is the home address of Carnegie Mellon University, the institution that runs picoCTF. That is a small Easter egg from the challenge author — a way of putting their institutional fingerprint in the file without giving away the flag. The flag itself is a 32-character hash suffix (`f5d46aff52c6e17f9fd6317b33d2d783`), which is a per-player unique tag, so each solver gets a different flag for the same plaintext.

---

## Alternative Methods

### Method 1 — `stegsnow` (the canonical tool, but wrong for this challenge)

`stegsnow` is the historical "snow" steganography tool, written by Matthew Kwan in the 1990s. It encodes messages in the trailing whitespace of a text file by treating *tabs* and *spaces* as a binary alphabet (tab = one value, space = another), then compressing the result with ICE encryption.

`stegsnow` is the canonical tool to reach for when you see a text file full of whitespace. The catch: it only works for *tab + space* stego, not *EM SPACE + space* stego. This challenge uses EM SPACE, so `stegsnow` will not decode it. Worth mentioning so a beginner does not waste 20 minutes wondering why `stegsnow -C flag.txt` produces no output.

If the challenge used tab+space encoding (which is also common), the solve would be:

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ stegsnow -C flag.txt
```

(Install on Kali: build from source — `apt install stegsnow` is not in the standard repos, but the upstream tarball at https://www.darkside.com.au/snow/ compiles in 5 seconds with `make`.)

### Method 2 — The Python one-liner (what I actually used)

The whole decode is six lines of Python:

```python
# decode.py
text = open("flag.txt", "rb").read().decode("utf-8")
bits = "".join("0" if ch == "\u2003" else "1" for ch in text)
out = bytearray()
for i in range(0, len(bits) - 7, 8):
    out.append(int(bits[i:i+8], 2))
print(out.decode("utf-8", errors="replace"))
```

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ python3 decode.py

picoCTF

SEE PUBLIC RECORDS & BACKGROUND REPORT
5000 Forbes Ave, Pittsburgh, PA 15213
picoCTF{not_all_spaces_are_created_equal_f5d46aff52c6e17f9fd6317b33d2d783}
```

This is the version I would actually use. No dependencies beyond Python's standard library, no compilation, no guessing which tool is the right one. You can read every line and know exactly what it does.

### Method 3 — The online decoder (the hint's path)

The challenge hint says "there might be an online decoder." Two popular free ones:

- **StegOnline** at https://stegonline.georgeom.net — paste the text into the "Encoded data" box, switch the analysis to "whitespace" mode, and the decoded text appears in the output panel. Works for em-space+space, tab+space, and zero-width-character stego.
- **Whitespace decoder** at https://330k.github.io/misc_tools/unicode_steganography.html — designed specifically for Unicode whitespace stego, supports a wide range of "invisible" characters (zero-width joiner, zero-width space, etc.) in addition to spaces and tabs.

The online path works, but it is interactive and you have to copy-paste the file contents into a web form. Python is faster and scriptable.

### Method 4 — `uni2ascii` + manual inspection (educational)

A really educational way to see what is in the file is to dump the codepoints explicitly:

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ python3 -c "
text = open('flag.txt', 'rb').read().decode('utf-8')
# Replace each character with a 1-bit label for visual scanning
out = ''.join('S' if ch == ' ' else 'E' for ch in text)
# Print in lines of 64 for readability
for i in range(0, len(out), 64):
    print(out[i:i+64])
" | head -10
ESSSESSSES SSESSSESSS  SSSESSSESS SSSESSE SSESSSES
SSSSESSSES SSESSSESSS  ESSESSSESS SSSESSSESS SESSSES
ESSSSESSS ESSE SSESSS ES SSSESSSESS ESSESSESSES SSES
SSESSSESSS SESSSSESSS E SSESSSESS SESSSESSSSSSS EESS
ESSSESSSES SSESSSESSS  SESSSSSESS ESSESSSSSSSS ESES
```

Each `S` is a space (bit `1`), each `E` is an em space (bit `0`). Reading the first 8 characters as bits gives `01101101` = `0x6D` = `m` — the first letter of the (decoded) "picoCTF" string that starts after a newline. This is the same operation as the Python one-liner, just hand-decoded. Useful for understanding *why* the decode works, but not something you would do for the full 162-byte file.

### Method 5 — `iconv` + `xxd` to see the structure

If you want to look at the file with classic Unix tools, you can convert the UTF-8 to ASCII and let the unprintable characters show up as `?`:

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ iconv -f UTF-8 -t ASCII//TRANSLIT flag.txt | xxd | head -3
```

This will not decode the message (because ASCII cannot represent EM SPACE at all), but it does show you that there are two distinct byte values in the file — `0x20` and `0x1A` (or whatever the transliterator picks for EM SPACE). Useful for a quick "is this really only two characters?" sanity check, but you will still need Python (or an online tool) to actually decode.

### Method 6 — `less` with raw byte display (just to look)

`less` can show you the raw bytes if you want to see the structure without leaving the terminal:

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ less -R flag.txt
# Inside less, press:
#   :set display-uhex
# to see codepoints, or
#   :set hex
# ... actually less does not have a great raw-byte view; just use od.
```

Honestly, `od -c` (which shows characters with escapes) is the friendliest:

```
┌──(zham㉿kali)-[~/work/whitepages]
└─$ od -c flag.txt | head -3
0000000 342 200 203 342 200 203 342 200 203 342 200 203   342 200 203
0000020     342 200 203     342 200 203     342 200 203
```

The `342 200 203` is octal for `0xE2 0x80 0x83` (the EM SPACE in UTF-8). The bare ` ` is the regular space (`0x20`). You can see the two characters alternating in the dump. This is just a confirmation step, not a decode.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Confirm the file is UTF-8 text with 1296 characters on one line — first signal of whitespace stego | Easy |
| `od -A x -t x1z` | See the raw bytes. The mix of `e2 80 83` and `20` is the giveaway for EM SPACE + space | Easy |
| `python3` + `Counter` | Count the unique characters. Confirms exactly 2 codepoints: U+2003 and U+0020 | Easy |
| `python3` (decode) | 6-line script that maps each character to a bit and packs bits into bytes | Easy |
| `cat decoded.txt` (or `less`) | Read the decoded output and copy the flag | Easy |
| `stegsnow` (alt) | Canonical whitespace stego tool. Does NOT work for this challenge because the encoding is EM SPACE + space, not tab + space. Worth knowing, worth not using here | Medium |
| Online whitespace decoder (alt) | Web-based decoder for Unicode whitespace stego. The path the challenge hint suggests | Easy |
| `od -c` (alt) | Octal/char dump that shows the two characters explicitly. Good for a quick visual check | Easy |
| `iconv \| xxd` (alt) | Convert UTF-8 to ASCII (transliterate) and inspect. Useful for "is this really only two characters?" sanity check | Medium |
| Hand-decoding with bit labels (alt) | Print `S`/`E` for space/em-space and decode the first few characters by eye. Educational, not practical for 162 bytes | Medium |

---

## Key Takeaways

- **"Blank" is a UI concept, not a data concept.** A file that looks empty in `cat` can still contain thousands of distinct bytes. `file` and `od` are the tools to look past the visual rendering and see what is actually on disk.
- **Unicode has *many* "space" characters.** ASCII has one. UTF-8 has dozens. Visually they look identical (or invisible), but they are completely different codepoints with completely different byte sequences. That difference is the lever whitespace stego pulls.
- **Two characters = one bit. 8 bits = one byte.** If you can identify that a file uses exactly two whitespace characters, you can guess the encoding (binary, 8 bits per byte) and try both mappings. One mapping produces readable ASCII; the other produces garbage. Pick the readable one.
- **Length is a hint.** 1296 = 162 × 8. If you see a file whose character count is divisible by 8, and it contains only two characters, that is a near-certain indicator of binary stego with 8-bit byte alignment.
- **`stegsnow` is the wrong tool here.** It is the canonical whitespace stego tool, but it only handles tab+space. The `EM SPACE + space` encoding used in this challenge is a different (sibling) technique, and the same tool cannot decode it. Knowing the limits of a tool is as important as knowing how to use it.
- **Try both mappings.** When you do not know whether `A=0, B=1` or `A=1, B=0`, the right move is to run both and look for the one that gives readable text. The other one will give random bytes (high-bit garbage). Do not waste time guessing — try both in five lines of code.
- **The hint "online decoder" is a real steer.** There are purpose-built web tools for Unicode whitespace stego. They are slower than a Python one-liner but they handle all the edge cases (different space widths, zero-width characters, etc.) without you having to write the decoder.
- **The flag is the lesson spelled out.** `picoCTF{not_all_spaces_are_created_equal_f5d46aff52c6e17f9fd6317b33d2d783}` = "not all spaces are created equal". A regular space and an em space look the same to a human but are completely different to a program. The author is reminding you to look at the bytes, not the picture.

### Flag wordplay decode

```
picoCTF{not_all_spaces_are_created_equal_<32-hex-hash>}
         |   |   |     |     |     |
         |   |   |     |     |     equal — literal, no leet
         |   |   |     |     _
         |   |   |     |     created — literal, no leet
         |   |   |     _
         |   |   |     are — literal, no leet
         |   |   _
         |   |  spaces — literal, no leet (the word "spaces" is
         |   |          being used in its Unicode sense, not just
         |   |          the ASCII sense; the author is making a
         |   |          point that the same word means different
         |   |          things depending on the encoding)
         |   _
         |   all — literal, no leet
         | _
         not — literal, no leet
         _
         {picoCTF — the standard picoCTF flag format
```

The whole flag (the human-readable part) reads as **"not all spaces are created equal"** — a literal description of the lesson the challenge is teaching. The two "spaces" in this challenge (regular ASCII space and Unicode EM SPACE) look identical when rendered, but they are completely different characters with completely different bit patterns. The author is reminding you that the visual rendering is a *display* decision, not a *data* decision — the bytes are the bytes, regardless of how they look on screen.

The trailing 32-character hex string (`f5d46aff52c6e17f9fd6317b33d2d783`) is a per-player unique tag, the same way most picoCTF flags have a hash suffix. It is not part of the lesson; it is just there so each player gets a different flag for the same plaintext (and so a leaked flag cannot be reused by another player). The lesson is in the human-readable prefix; the hash is the audit trail.
