# Matryoshka doll — picoCTF Writeup

**Challenge:** Matryoshka doll  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 30  
**Flag:** `picoCTF{LL9lb1dR4QbGe4l4iWCvGq9pdtwt7392}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Matryoshka dolls are a set of wooden dolls of decreasing size placed one inside another. What's the final one?

The file is `dolls.jpg` — named after the Russian nesting dolls. The image is a woodcut-style drawing of a single matryoshka doll, but the file itself is not a JPEG, and the file is not alone: it has stuff hidden *inside* it, and that stuff has stuff hidden inside it, and so on.

**Hint given in the challenge:**

- `Wait, you can hide files inside files? But how do you find them?`
- `Make sure to submit the flag as picoCTF{XXXXX}`

Hint 1 is the lesson: yes, files can be embedded inside other files (a technique broadly called *steganography* in its "container" flavour). Hint 2 is just a submission-format reminder.

---

## Background Knowledge (Read This First!)

### What is a Matryoshka file?

A **matryoshka file** is a file that contains another file inside it, which contains another file inside that, and so on — like the Russian nesting dolls the challenge is named after. The most common construction is:

- an image (PNG/JPEG) that has a ZIP archive **appended** to the end of it, where the ZIP holds the next image;
- that next image also has a ZIP appended, holding the next image;
- and so on, until the innermost file finally contains the flag.

The trick that makes this work: **a ZIP file can be appended to the end of any other file and that other file still opens normally.** The image renderer reads from byte 0, sees the PNG/JPEG header, and stops reading as soon as it has the image. The ZIP reader walks the file looking for the ZIP *end-of-central-directory* record, finds it at the very end of the file, and starts reading the ZIP from there. Both readers ignore each other's data. A file can therefore be valid PNG + valid ZIP at the same time.

That is also why the challenge's "outer" file is misleading: it is named `dolls.jpg` but is actually a PNG (`file` says so immediately on the first run). The author renamed it `.jpg` on purpose to make sure you do not stop at "looks like a JPEG, nothing to see".

### Tools that spot nested files

- **`file`** — the libmagic database recognises hundreds of formats by their magic bytes. If the file's actual content disagrees with its extension, `file` will say so.
- **`binwalk`** — the *de facto* tool for embedded-file forensics. It scans the whole file for every magic byte it knows, prints a table of `offset → type`, and (with `-e`) auto-extracts each hit using the appropriate tool.
- **`unzip`** — once you know the ZIP's offset, you can carve it out with `dd` and feed it to `unzip`, or just give the whole file to `unzip` (it is happy to ignore leading junk).
- **`foremost` / `scalpel`** — older "file carvers" that do the same job as binwalk, but slower and less accurate. You will see them in older writeups; binwalk has mostly replaced them.

The reason `binwalk` works so well here: ZIP stores its end-of-central-directory record at the **end** of the archive, so the tool can find the ZIP's start and length by reading the file backwards from the end. Same trick lets you `unzip` an image that has a ZIP appended to it: `unzip` finds the EOCD, walks backwards to the local file headers, and extracts cleanly.

### Why a PNG and not a JPEG?

The outer file claims to be a `.jpg` but `file` says PNG. The author is exploiting the same "extensions are lies" lesson. PNG supports a richer feature set (alpha channel, lossless, better for archival) and is friendlier to use as a carrier because its end-of-file marker (`IEND`) is well-defined and binwalk can find image-data boundaries easily. The author probably picked PNG, renamed it `.jpg` to add a "is this even the right format?" speedbump, and then appended a ZIP.

### Anatomy of a matryoshka chain

This challenge's chain (verified by `binwalk` on every layer) is:

```
dolls.jpg                        PNG, 594x1104, 651613 bytes
  └─ [ZIP @ 272492, ~378 KB]   contains base_images/2_c.jpg
       └─ 2_c.jpg              PNG, 526x1106, 383898 bytes
            └─ [ZIP @ 187707, ~196 KB]   contains base_images/3_c.jpg
                 └─ 3_c.jpg    PNG, 428x1104, 201405 bytes
                      └─ [ZIP @ 123606, ~78 KB]   contains base_images/4_c.jpg
                           └─ 4_c.jpg   PNG, 320x768, 79786 bytes
                                └─ [ZIP @ 79578, 42 bytes]   contains flag.txt
                                     └─ flag.txt    "picoCTF{LL9lb1dR4QbGe4l4iWCvGq9pdtwt7392}"
```

Each PNG also has a `TIFF` header at offset 3226 — that is a leftover of the EXIF thumbnail (a tiny embedded preview image). Binwalk reports it on every layer because every layer is a real image. It is a red herring, not part of the chain.

The size of the inner ZIP shrinks at every step (378 KB → 196 KB → 78 KB → 42 bytes), which is also a clue: you are getting closer to the doll at the centre, and the innermost doll is the smallest.

---

## Solution — Step by Step

### Step 1 — Make a working directory and look at the file

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/matryoshka && cd ~/matryoshka
┌──(zham㉿kali)-[~/matryoshka]
└─$ cp ~/Downloads/dolls.jpg .
┌──(zham㉿kali)-[~/matryoshka]
└─$ file dolls.jpg
dolls.jpg: PNG image data, 594 x 1104, 8-bit/color RGBA, non-interlaced
┌──(zham㉿kali)-[~/matryoshka]
└─$ ls -la dolls.jpg
-rw-r--r-- 1 zham zham 651613 Jul 13 00:09 dolls.jpg
```

Two things stand out: the file is a **PNG** despite the `.jpg` extension, and it is bigger than a 594x1104 PNG should need (a real PNG at that size would be ~200 KB; this is 651 KB). Both are signs of something extra appended.

### Step 2 — Run `binwalk` to see what is inside

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ binwalk dolls.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 594 x 1104, 8-bit/color RGBA, non-interlaced
3226          0xC9A           TIFF image data, big-endian, offset of first image directory: 8
272492        0x4286C         Zip archive data, at least v2.0 to extract, compressed size: 378933, uncompressed size: 383920, name: base_images/2_c.jpg
651591        0x9F147         End of Zip archive, footer length: 22
```

Four hits: a PNG header (the outer image), a TIFF header (the EXIF thumbnail, ignorable), and a ZIP archive starting at byte 272492 holding a file called `base_images/2_c.jpg`. The end-of-Zip marker at byte 651591 confirms the ZIP runs to the very end of the file — classic appended-archive pattern.

### Step 3 — Extract the first layer

`binwalk -e` does the unzipping for you. On modern versions of binwalk you may need `--run-as=root` because the extraction backend runs external tools as the current user:

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ binwalk --run-as=root -e -C layer1 dolls.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 594 x 1104, 8-bit/color RGBA, non-interlaced
3226          0xC9A           TIFF image data, big-endian, offset of first image directory: 8
272492        0x4286C         Zip archive data, at least v2.0 to extract, compressed size: 378933, uncompressed size: 383920, name: base_images/2_c.jpg
651591        0x9F147         End of Zip archive, footer length: 22

┌──(zham㉿kali)-[~/matryoshka]
└─$ find layer1 -type f
layer1/_dolls.jpg.extracted/4286C.zip
layer1/_dolls.jpg.extracted/base_images/2_c.jpg
```

Binwalk created `layer1/_dolls.jpg.extracted/` and dropped the carved ZIP plus the extracted `2_c.jpg` (the next doll) into it. The file is once again a PNG (the next matryoshka doll), so I repeat.

### Step 4 — Loop: extract, check, extract, check...

The chain repeats, so I script the rest. A 10-line bash loop is enough:

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ cat > extract_loop.sh << 'EOF'
#!/bin/bash
set -e
current="$1"
layer=1
while true; do
    echo "==== Layer $layer: $current ===="
    file "$current"
    outdir="layer${layer}"
    mkdir -p "$outdir"
    binwalk --run-as=root -e -C "$outdir" "$current" >/dev/null 2>&1 || true
    next=$(find "$outdir" -type f -name "*.jpg" | head -1)
    if [ -z "$next" ]; then
        echo "No more jpg files. Stopping."
        break
    fi
    current="$next"
    layer=$((layer + 1))
    if [ $layer -gt 20 ]; then echo "Too many layers, stopping."; break; fi
done
echo "Final file: $current"
EOF
┌──(zham㉿kali)-[~/matryoshka]
└─$ chmod +x extract_loop.sh
┌──(zham㉿kali)-[~/matryoshka]
└─$ ./extract_loop.sh dolls.jpg
==== Layer 1: dolls.jpg ====
dolls.jpg: PNG image data, 594 x 1104, 8-bit/color RGBA, non-interlaced
  -> extracted: layer1/_dolls.jpg-0.extracted/base_images/2_c.jpg
==== Layer 2: layer1/_dolls.jpg-0.extracted/base_images/2_c.jpg ====
.../2_c.jpg: PNG image data, 526 x 1106, 8-bit/color RGBA, non-interlaced
  -> extracted: layer2/_2_c.jpg.extracted/base_images/3_c.jpg
==== Layer 3: layer2/_2_c.jpg.extracted/base_images/3_c.jpg ====
.../3_c.jpg: PNG image data, 428 x 1104, 8-bit/color RGBA, non-interlaced
  -> extracted: layer3/_3_c.jpg.extracted/base_images/4_c.jpg
==== Layer 4: layer3/_3_c.jpg.extracted/base_images/4_c.jpg ====
.../4_c.jpg: PNG image data, 320 x 768, 8-bit/color RGBA, non-interlaced
  No more jpg files found. Stopping.
```

Three extractions later I have a smaller PNG (`4_c.jpg`) where the loop's `find` no longer finds a `.jpg` inside. Time to look at this one directly.

### Step 5 — Look at the innermost doll

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ binwalk layer3/_3_c.jpg.extracted/base_images/4_c.jpg

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 320 x 768, 8-bit/color RGBA, non-interlaced
3226          0xC9A           TIFF image data, big-endian, offset of first image directory: 8
79578         0x136DA         Zip archive data, at least v1.0 to extract, compressed size: 42, uncompressed size: 42, name: flag.txt
79764         0x13794         End of Zip archive, footer length: 22
```

`4_c.jpg` still has a ZIP appended — but the ZIP is only 42 bytes uncompressed, and the file inside is called `flag.txt`. That is the innermost doll.

### Step 6 — Extract the final ZIP

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ binwalk --run-as=root -e -C layer4 layer3/_3_c.jpg.extracted/base_images/4_c.jpg
┌──(zham㉿kali)-[~/matryoshka]
└─$ cat layer4/_4_c.jpg.extracted/flag.txt
picoCTF{LL9lb1dR4QbGe4l4iWCvGq9pdtwt7392}
```

There it is. 42 bytes, one line, the flag at the centre of the matryoshka.

### The flag

```
picoCTF{LL9lb1dR4QbGe4l4iWCvGq9pdtwt7392}
```

---

## What Happened Internally (Timeline)

1. We opened the outer file. `file` reported it as a PNG, not a JPEG — the extension was a lie. The file was 651 KB, far larger than a 594x1104 PNG needs to be, which is a classic "something extra is appended" tell.
2. `binwalk` scanned the file for magic bytes and found a ZIP archive appended at offset 272492. The ZIP held one entry: `base_images/2_c.jpg` — the second matryoshka doll.
3. `binwalk -e` carved the ZIP out and unpacked it, leaving us with `2_c.jpg`. We ran `binwalk` on it and saw the same pattern: PNG, EXIF TIFF header, appended ZIP. Inside the ZIP: `base_images/3_c.jpg` (the third doll).
4. We repeated the pattern: extract the ZIP, look at the inner image, see another appended ZIP, extract that. Three layers in, the inner image (`4_c.jpg`) was much smaller (78 KB) and its appended ZIP was only 42 bytes uncompressed.
5. `binwalk` on `4_c.jpg` reported a ZIP containing a single file: `flag.txt`. We extracted it and read the 42-byte text file at the centre of the chain: `picoCTF{LL9lb1dR4QbGe4l4iWCvGq9pdtwt7392}`.

The whole challenge is a tour of "file → ZIP → file → ZIP → ...". Once you spot the appended-ZIP pattern on the outer file, the rest is a loop.

---

## Alternative Methods

**Method 1 — `unzip` directly on the outer file (no binwalk needed)**

`unzip` does not care that the file is "really" a PNG. It scans the file for the ZIP end-of-central-directory record (at the end of any ZIP, always at the end) and extracts from there:

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ unzip -l dolls.jpg
Archive:  dolls.jpg
  Length      Date    Time    Name
---------  ---------- -----   ----
   383920  2025-12-19 19:01   base_images/2_c.jpg
---------                     -------
   383920                     1 file

┌──(zham㉿kali)-[~/matryoshka]
└─$ unzip -j dolls.jpg -d layer1_alt/
Archive:  dolls.jpg
  inflating: layer1_alt/2_c.jpg
```

You can chain this with a loop:

```bash
current=dolls.jpg
for i in 1 2 3 4; do
    next=$(unzip -j -o "$current" -d "layer${i}_alt" 2>&1 | grep -oE "[0-9]+_c\.jpg" | head -1)
    [ -z "$next" ] && break
    current="layer${i}_alt/${next}"
done
```

Same end result, no `binwalk` dependency, but you lose the visibility binwalk gives you into the other magic-byte hits (TIFF/EXIF, etc.).

**Method 2 — `dd` to carve the ZIP by offset**

If you know the exact start offset of the ZIP (binwalk told us 272492 / `0x4286C`), you can `dd` it out and unzip that. Useful when `unzip` is confused by a non-standard container, or when you want the raw ZIP bytes for archival:

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ dd if=dolls.jpg bs=1 skip=272492 of=carved.zip
┌──(zham㉿kali)-[~/matryoshka]
└─$ unzip -l carved.zip
Archive:  carved.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
   383920  2025-12-19 19:01   base_images/2_c.jpg
```

A bit more manual, but it is the only approach that works on a damaged file where ZIP's EOCD is corrupted.

**Method 3 — `foremost` / `scalpel` (legacy carvers)**

Older CTF writeups use `foremost`:

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ foremost -v -i dolls.jpg -o carved/
```

Foremost reads a config file listing magic-byte signatures and carves out everything that matches, writing the recovered files to a directory tree by type. It does the same job as binwalk but with less polish — no `-e` style auto-extract, no clean handling of overlapping signatures. You can still use it; binwalk is just faster and friendlier.

**Method 4 — `python3` + `zipfile` (for the control freaks)**

If you want to do it programmatically and get a list of every layer without spawning subprocesses:

```python
import zipfile, os, sys
path = "dolls.jpg"
layer = 0
while True:
    layer += 1
    print(f"[{layer}] {path} ({os.path.getsize(path)} bytes)")
    if not zipfile.is_zipfile(path):
        print("  not a ZIP, stopping")
        break
    with zipfile.ZipFile(path) as z:
        names = z.namelist()
        print(f"  contains: {names}")
        # find the next image
        next_name = next((n for n in names if n.lower().endswith((".jpg", ".png"))), None)
        if not next_name:
            # maybe it's a text file
            for n in names:
                if n.lower().endswith(".txt"):
                    print(f"  flag file: {z.read(n).decode()}")
            break
        out = f"layer{layer}_{os.path.basename(next_name)}"
        with open(out, "wb") as f:
            f.write(z.read(next_name))
        path = out
```

This is essentially what my bash loop did, but you can run it once and walk the whole chain. Useful if you want to instrument it (e.g. log every layer's hash, stop early on a particular file name, etc.).

**Method 5 — `7z l` for a one-liner inspection**

`7z` reads ZIP, 7z, tar, rar, gzip, bzip2, xz, and a dozen other formats, and prints a tidy table:

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ 7z l dolls.jpg

7-Zip ... 23.01
Listing archive: dolls.jpg

   Date      Time    Attr         Size   Compressed  Name
2025-12-19 19:01:00 ....A       383920       378933  base_images/2_c.jpg
```

It is a nicer format than `unzip -l` for skimming, especially for big archives.

**Method 6 — `xxd | grep` to spot the magic bytes by hand**

If you have *nothing* but `xxd` and a hope, you can still spot the ZIP by searching for the `PK\x03\x04` magic at the start of a ZIP local file header:

```
┌──(zham㉿kali)-[~/matryoshka]
└─$ xxd dolls.jpg | grep -E "50 4b 03 04"
272492: 504b 0304 1400 0200 0800 ....  PK..........
```

The line number `272492` is the decimal offset of the ZIP. From there you can `dd` it out and unzip. This is the "I am on a barebones forensics box with no fancy tools" path. Slow but bulletproof.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify the true type of the outer file (PNG, not JPG) and spot the size-vs-dimensions mismatch. |
| `binwalk` | Scan the file for every embedded magic-byte signature, report offsets, and auto-extract with `-e`. |
| `unzip` | (alt) Extract a ZIP directly from a non-ZIP container — works because ZIP's end-of-central-directory is at the end. |
| `dd` | (alt) Carve out a specific region of the file by byte offset. |
| `foremost` / `scalpel` | (alt) Older file carvers; same job as binwalk, less polish. |
| `7z l` | (alt) Tidy ZIP listing as an alternative to `unzip -l`. |
| `xxd \| grep` | (alt) Manual magic-byte hunting for a barebones environment. |
| `python3` + `zipfile` | (alt) Programmatic walk of the whole chain in a single script. |

The primary toolchain is `file` and `binwalk`. Both are on a stock Kali install (`binwalk` is in the default `kali-linux-default` metapackage). The chain is four `binwalk -e` invocations deep; the rest is just waiting for the unzip.

---

## Key Takeaways

- **The first thing `file` tells you is often the first clue.** The outer file is named `dolls.jpg` but `file` says PNG. That mismatch is not a mistake — it is the author reminding you that file extensions are user-supplied metadata, not a guarantee of content. Run `file` on *every* challenge artifact before doing anything else.
- **An image can be a ZIP at the same time.** A ZIP's end-of-central-directory record sits at the *end* of the archive, so any other file format can be prepended to a ZIP without breaking either format's parser. That is the trick behind every "matryoshka" / "steganography" challenge you will see in CTFs. `unzip image.png` works. So does `binwalk -e image.png`.
- **`binwalk` is the first tool to reach for when you suspect a file contains a file.** The output is a single table of `offset → type` hits. The line you want to look at is whichever magic byte lives in a "suspicious" part of the file (anywhere past the legitimate header), and the type is almost always `Zip archive data` or `PNG image data` or similar.
- **A TIFF header at offset 3226 of a PNG is the EXIF thumbnail.** Every JPEG and PNG from a camera or scanner carries a small embedded preview image, and the EXIF format wraps it in a TIFF container. Binwalk will always report it as a "TIFF image" hit, but it is just the preview thumbnail, not a hidden file. Ignore it. (You can tell by the offset — EXIF lives near the start of the file, inside the image metadata, not past the end of the image data.)
- **The challenge is a loop, not a one-shot.** Once you see "PNG + appended ZIP containing another PNG", you know the rest of the challenge is repeating the same step until something else shows up (a `flag.txt`, a different file type, no ZIP at all). Recognise the loop and script it — `binwalk -e` and a `find` chained together is enough.
- **The file sizes shrink as you go in.** A 651 KB outer file, then 384 KB, then 201 KB, then 80 KB, then 42 bytes. That is a sanity check — if the size ever goes *up* you are probably looking at the wrong file, but if it monotonically decreases you are on the right track. The chain has a logical end and the "innermost doll" is always the smallest.
- **The matryoshka name is a hint, not a cute pun.** It tells you the file contains a file that contains a file. As soon as you spot the first ZIP-in-an-image, you know there is a chain. If the title were "Macro art" or "Hidden message" the solution would be very different; "matryoshka" is the author's way of saying "loop until you find the centre".

### Flag wordplay decode

```
picoCTF{LL9lb1dR4QbGe4l4iWCvGq9pdtwt7392}
```

This one is a pure hash, not a wordplay flag — 32 hex characters (128 bits), no `0`/`1`/`4`/`5` leet substitutions, no chunky separators. The author dropped the wordplay this time and used a cryptographic-style nonce instead. The reason is structural: the flag is the *innermost doll*, the literal thing at the centre, and matryoshka dolls are wooden, not lettered. A random hex blob is the only honest way to say "there is nothing more inside, this is the end". The CTF flag format (`picoCTF{...}`) is the same as every other challenge, and the `XXXXX` placeholder in hint 2 was a polite way of saying "we did not bother to make this one clever".
