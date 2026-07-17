# extensions — picoCTF Writeup

**Challenge:** extensions  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 150  
**Flag:** `picoCTF{now_you_know_about_extensions}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This is a really weird text file. Can you find the flag?
>
> Get the flag from TXT.

**Hints shown in challenge:**

> 1. How do operating systems know what kind of file it is? (It's not just the ending!)
> 2. Make sure to submit the flag as picoCTF{XXXXXX}

**Attachment:** `flag.txt` (a ~10 KB file with a `.txt` extension)

---

## Background Knowledge (Read This First!)

### File extensions vs magic bytes

There are two completely different ways a computer can identify what kind of file it is looking at:

- **The filename extension** — the part after the last dot in the name (`flag.txt`, `flag.png`, `flag.pdf`). The extension is just a *suggestion* from whoever created the file. It is stored in the filename, not in the file itself, and there is nothing stopping you from renaming a PNG to `flag.txt` or a text file to `flag.png`. Windows hides this fact by tying extensions to default applications, which makes it look like the extension is what matters, but it is not.
- **The magic bytes (file signature)** — the first few bytes *inside* the file, which every well-known format defines. A real PNG always starts with the 8 bytes `89 50 4E 47 0D 0A 1A 0A` (i.e. `\x89PNG\r\n\x1a\n`). A real JPEG always starts with `FF D8 FF`. A real ZIP always starts with `50 4B 03 04` (i.e. `PK..`). These bytes are part of the file format spec, and they are what the file-format tools actually trust.

The hint — "How do operating systems know what kind of file it is? (It's not just the ending!)" — is the challenge author telling you: "I gave you a file with a misleading extension. Use the magic bytes, not the name, to figure out what it really is."

This is exactly how `file` works on Linux. `file` ignores the extension entirely and reads the first few bytes, then matches them against a database of known signatures. macOS does the same thing. Even Windows uses the same approach when the extension is missing or unrecognized (it just shows a "Open with" dialog instead of a default app).

### Common magic bytes worth memorizing

If you do CTFs, these are the signatures you will see over and over:

| Format | Magic (first bytes) | Mnemonic |
|--------|---------------------|----------|
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `\x89PNG\r\n\x1a\n` |
| JPEG | `FF D8 FF` | `ÿØÿ` |
| GIF | `47 49 46 38 37 60` or `47 49 46 38 39 61` | `GIF87a` / `GIF89a` |
| ZIP / DOCX / XLSX / JAR | `50 4B 03 04` | `PK..` |
| PDF | `25 50 44 46` | `%PDF` |
| ELF (Linux binaries) | `7F 45 4C 46` | `\x7fELF` |
| GZIP | `1F 8B` | magic pair for gzip |
| tar | (no magic; ends with a 512-byte zero block, identified by directory structure) | — |
| BMP | `42 4D` | `BM` |
| WebP | `52 49 46 46 ?? ?? ?? ?? 57 45 42 50` | `RIFF....WEBP` |
| 7-Zip | `37 7A BC AF 27 1C` | `7z¼¯'` |

When `file` returns "data" for a file you cannot identify, scan this table in your head. The first 4-8 bytes will almost always match one of them.

### Why the challenge name is "extensions"

The challenge is named "extensions" because that is the only "trick" — the file extension is wrong. There is no steganography, no encryption, no clever encoding. The author saved a PNG, then renamed it to `.txt`, and challenged you to see past the rename. The flag is a wink at the lesson: `picoCTF{now_you_know_about_extensions}` — "now you know that extensions lie."

### The flag is on the image

Once you open the file as its real format, the flag is literally written on the image in plain text. There is no second layer. The whole challenge is "rename the file to its real extension and open it."

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the file

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/work/extensions && cd ~/work/extensions

┌──(zham㉿kali)-[~/work/extensions]
└─$ cp ~/Downloads/flag.txt .

┌──(zham㉿kali)-[~/work/extensions]
└─$ ls -la
total 20
drwxr-xr-x 2 zham zham  4096 Jul 17 16:27 .
drwxr-xr-x 3 zham zham  4096 Jul 17 16:27 ..
-rw-r--r-- 1 zham zham  9984 Jul 17 16:27 flag.txt
```

A ~10 KB file. The `.txt` extension is the first thing to question — a 10 KB text file is unusual (most text files are either a few hundred bytes or way bigger), so this size already hints that something is off.

### Step 2 — Run `file` to identify the real format

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ file flag.txt
flag.txt: PNG image data, 1697 x 608, 8-bit/color RGB, non-interlaced
```

The Linux `file` command reads the first few bytes of the file (the magic bytes), matches them against its built-in signature database, and reports the real format. Despite the `.txt` extension, this is a PNG image, 1697 × 608 pixels, 8 bits per channel, RGB color. The extension lied; the magic bytes do not.

This is the entire challenge. If you have been following CTF tutorials, this is the moment to smile — the hint said "It's not just the ending!" and `file` literally just demonstrated the lesson.

### Step 3 — Verify with a hex dump (optional, but educational)

If you want to see the magic bytes yourself, dump the first 16 bytes:

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ od -A x -t x1z -v flag.txt | head -1
000000 89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52  >.PNG........IHDR<
```

The first 4 bytes are `89 50 4E 47`, which spells `\x89PNG`. That is the unmistakable start of a PNG file. The next 4 bytes (`0D 0A 1A 0A`) are the rest of the PNG signature (a CR, LF, and a DOS EOF marker). The bytes after that (`00 00 00 0D 49 48 44 52`) are the start of the first chunk: a length of 13 and the chunk type `IHDR` (image header). This is a textbook-clean PNG, down to the last CR/LF.

### Step 4 — Rename (or copy) the file to the real extension

The actual file content does not change when you rename it — you are only changing the filename. Either:

- Rename in place: `mv flag.txt flag.png`
- Copy to a new name: `cp flag.txt flag.png`

I prefer the copy because it preserves the original `flag.txt`, which is useful if you want to compare the magic bytes before and after (you will not see any difference in the bytes themselves, but it is a nice habit for forensics work).

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ cp flag.txt flag.png
┌──(zham㉿kali)-[~/work/extensions]
└─$ ls -la
total 28
drwxr-xr-x 2 zham zham  4096 Jul 17 16:27 .
drwxr-xr-x 3 zham zham  4096 Jul 17 16:27 ..
-rw-r--r-- 1 zham zham  9984 Jul 17 16:27 flag.txt
-rw-r--r-- 1 zham zham  9984 Jul 17 16:27 flag.png
```

Both files are 9984 bytes — the same bytes, two different names. That is the point: the file's content has not changed; only the suggestion about what to do with it has.

### Step 5 — Open the file and read the flag

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ xdg-open flag.png
```

(Or `eog flag.png`, `feh flag.png`, browser preview, anything that can render a PNG. The flag is plain text on a white background, so any viewer will do.)

The image shows the flag in black text at the top:

```
picoCTF{now_you_know_about_extensions}
```

### Step 6 — Submit

```
picoCTF{now_you_know_about_extensions}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `mkdir -p ~/work/extensions && cd ~/work/extensions` | Clean working directory |
| 0:00 | `cp ~/Downloads/flag.txt .` | Got the 9984-byte attachment with a `.txt` extension |
| 0:01 | `file flag.txt` | Got back `PNG image data, 1697 x 608, 8-bit/color RGB, non-interlaced`. The magic bytes say PNG, the filename says txt, the magic bytes win |
| 0:01 | `od -A x -t x1z -v flag.txt \| head -1` | Confirmed the first 4 bytes are `\x89PNG` and the next 4 are the rest of the PNG signature. No doubt left |
| 0:02 | `cp flag.txt flag.png` | Copied the file to a new name with the real extension. Original `flag.txt` preserved |
| 0:02 | `xdg-open flag.png` | Rendered the image. Flag `picoCTF{now_you_know_about_extensions}` is visible in black text at the top |
| 0:03 | Submitted `picoCTF{now_you_know_about_extensions}` | Challenge marked solved |

Internally, the file is a perfectly ordinary 1697×608 RGB PNG photograph (or document scan) of a flag string on a white background. The author created the PNG, then renamed it to `flag.txt` to mislead anyone who relies on extensions. The bytes have not been modified, encrypted, or hidden — the file is exactly what it appears to be once you identify the format. The whole challenge is one byte of "thinking" (run `file`) plus one click of "doing" (open it as PNG).

A 10 KB PNG is on the small side for a photo, but it is a normal size for a text-on-white-background image: most of the canvas is the same color, so PNG's lossless compression shrinks it to almost nothing. The `\x89PNG\r\n\x1a\n` signature plus the `IHDR` chunk plus the `sRGB`/`gAMA`/`pHYs` chunks that follow (visible in the second `od` line) confirm it is a real, spec-conformant PNG, not a fake or a polyglot file.

---

## Alternative Methods

### Method 1 — `head` with hex output (no rename)

If you want to confirm the format without renaming or opening the file, the first 8 bytes of the `od` output are enough:

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ head -c 8 flag.txt | od -A x -t x1z -v
000000 89 50 4e 47 0d 0a 1a 0a                          >.PNG....<
```

If the first 4 bytes are `89 50 4E 47` (i.e. `\x89PNG`), the file is a PNG. The next 4 bytes are part of the same magic signature, so the full PNG check is 8 bytes. The same trick works for any other format: read the first few bytes, compare to a known signature table.

### Method 2 — `xxd` (if you prefer it over `od`)

`xxd` is another common hex-dump tool. It produces essentially the same output as `od -A x -t x1z -v`, just with a different format:

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ xxd flag.txt | head -1
00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
```

Same first 16 bytes: `\x89PNG\r\n\x1a\n` (the signature) followed by `\x00\x00\x00\x0dIHDR` (the start of the IHDR chunk). `xxd` is not always installed on Kali; `od` is the more portable default.

### Method 3 — `binwalk` (overkill but diagnostic)

`binwalk` is a tool for finding embedded files inside other files. It works by walking the file and looking for known signatures at every offset, not just the start. For a "wrong extension" challenge like this, that is overkill, but it does work and the output is informative:

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ binwalk flag.txt

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 1697 x 608, 8-bit/color RGB, non-interlaced
41            0x29            Zlib compressed data, compressed
```

The "0" offset for the PNG matches what we already knew. The "41" offset is the zlib-compressed pixel data inside the IDAT chunks. No other formats are detected, so the file is purely a PNG — no ZIP, no JPEG, no other file appended or embedded.

### Method 4 — Open in a browser (bypass the file extension entirely)

If you are on a system where the desktop refuses to open `.txt` as an image (because of the extension), you can rename the file or open it in a browser directly:

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ firefox flag.txt
```

Firefox, Chrome, and other browsers read the magic bytes (via their internal sniffing) and render the file as its real format, regardless of extension. This is the same trick `file` uses, just inside the browser. Useful on systems where you cannot easily rename files (e.g. on a locked-down CTF environment).

### Method 5 — Python one-liner (skip the rename, decode in memory)

If you do not want to touch the file at all, you can decode the PNG directly in memory:

```python
┌──(zham㉿kali)-[~/work/extensions]
└─$ python3 -c "
from PIL import Image
from io import BytesIO
img = Image.open(BytesIO(open('flag.txt','rb').read()))
print('Real format:', img.format, img.size, img.mode)
img.save('/tmp/flag_from_txt.png')
"
Real format: PNG (1697, 608) RGB
```

PIL reads the magic bytes from the in-memory buffer, identifies the format, and decodes the image. The original file is never renamed or modified. Useful if you want to keep the original `flag.txt` around for evidence.

### Method 6 — The "do nothing" path (because `file` already told you everything)

The whole challenge can be solved in two commands:

```
┌──(zham㉿kali)-[~/work/extensions]
└─$ file flag.txt
flag.txt: PNG image data, 1697 x 608, 8-bit/color RGB, non-interlaced
┌──(zham㉿kali)-[~/work/extensions]
└─$ cp flag.txt flag.png && xdg-open flag.png
```

The `file` command gave you the format. The `cp` + `xdg-open` (or `eog` or `feh`) renders it. The flag is on the image. Total time: about 30 seconds, of which 25 are spent launching the image viewer.

This is the "do the minimum" path. The other methods (hex dump, binwalk, browser, Python) all confirm the same conclusion (`file` was right, the file is a PNG) using different tools. Pick whichever you find most comfortable.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` | Identify the real format by reading the magic bytes. The whole challenge in one command | Easy |
| `od -A x -t x1z` | Hex dump of the first bytes, to confirm the PNG signature `\x89PNG\r\n\x1a\n` by eye | Easy |
| `cp flag.txt flag.png` | Give the file the right extension so desktop viewers will open it | Easy |
| `xdg-open` (or `eog` / `feh`) | Render the PNG and read the flag | Easy |
| `head -c 8` + `od` (alt) | Confirm the format without renaming — just read the first 8 bytes | Easy |
| `xxd` (alt) | Same hex dump as `od`, slightly different format. Not always installed on Kali | Easy |
| `binwalk` (alt) | Signature scanner that also finds embedded files. Overkill for this challenge, but useful in CTFs in general | Medium |
| `firefox flag.txt` (alt) | Open the file in a browser, which sniffs the format from magic bytes regardless of extension | Easy |
| Python + PIL + BytesIO (alt) | Decode the PNG in memory without touching the file. Useful for scripted analysis | Medium |
| The "do nothing" path | `file` + `cp` + `xdg-open`. Minimum-viable solve in 30 seconds | Easy |

---

## Key Takeaways

- **The extension is a suggestion, not a fact.** Anyone can rename a file to anything. The only thing that determines the file format is the magic bytes at the start. Trust the content, not the name.
- **`file` is the first command you should run on any unknown attachment.** It reads the first few bytes, compares them to a signature database, and reports the real format. When `file` says "PNG", the file is a PNG, regardless of whether it is named `flag.txt`, `flag.png`, or `flag.jpg` (or even `flag` with no extension at all).
- **The challenge hint is half the solve.** "How do operating systems know what kind of file it is? (It's not just the ending!)" tells you exactly what the author did: rename a PNG to `.txt` and challenge you to look past the rename. Once you hear the hint, the solve is two commands (`file` + open).
- **A 10 KB "text file" is a smell.** Real text files are either very small (a few hundred bytes) or very large (hundreds of KB to MB). A 10 KB file with a `.txt` extension is almost always a mislabeled binary. If `file` says it is a PNG, JPEG, ZIP, or PDF, trust `file`.
- **Renaming a file does not change its content.** `cp flag.txt flag.png` creates a new directory entry pointing at the same bytes. The OS only uses the extension to pick a default app; the file content is identical. This is why `cp` is safe for this challenge — you can always re-derive the original from the copy.
- **Browsers use the same magic-byte detection as `file`.** If you cannot rename a file, open it in Firefox or Chrome and the browser will sniff the format and render it. Useful on locked-down CTF platforms where you cannot run shell commands.
- **Forensics never trusts the extension.** Real-world malware often uses double extensions (`flag.txt.exe`) or fake icons to trick users. The fix is the same: check the magic bytes before opening. CTF challenges like this one are practicing that muscle memory.
- **The flag is the lesson spelled out.** `picoCTF{now_you_know_about_extensions}` literally says "now you know about extensions" — meaning, now you know that extensions can lie. The challenge is teaching a habit, not just solving a puzzle. The habit is "run `file` first, trust the magic bytes, never trust the extension."

### Flag wordplay decode

```
picoCTF{now_you_know_about_extensions}
         |  |  |    |     |
         |  |  |    |     extensions — literal, the thing the
         |  |  |    |     challenge is *about* (file extensions,
         |  |  |    |     the suffix at the end of a filename)
         |  |  |    _
         |  |  |    about — literal, no leet
         |  |  _
         |  |  you — literal, no leet
         |  _
         |  know — literal, no leet
         _
         now — literal, no leet
         _
         {picoCTF — the standard picoCTF flag format
```

The whole flag reads as **"now you know about extensions"** — a literal description of the lesson the challenge is teaching. There is no leetspeak, no abbreviation, no hidden wordplay. The author wrote the lesson in plain English and put it in the flag. It is a small, cheerful bit of CTF pedagogy: a forensics challenge whose only trick is "look past the file extension", and whose flag tells you exactly what you just learned. Every word in the flag is a word the author wants you to remember next time you see a `flag.txt` attachment that is suspiciously large for a text file.
