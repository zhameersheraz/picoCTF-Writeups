# So Meta — picoCTF Writeup

**Challenge:** So Meta  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 150  
**Flag:** `picoCTF{s0_m3ta_bc056477}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> Find the flag in this picture.

## Hints

> 1. What does meta mean in the context of files?
> 2. Ever heard of metadata?

---

## Background Knowledge (Read This First!)

### What is "meta" in the context of files?

The word **meta** means "data about data". In a file, the **metadata** is all the extra information that travels *with* the file but is not the actual content you see when you open it. For an image, that means things like the camera model, the date it was taken, the GPS coordinates, the color profile, the editing software, the author, and so on. When you open a `.jpg` in a photo viewer, you only see the pixels. The metadata is the invisible part.

### How is metadata stored?

It depends on the file format:

- **JPEG** uses an `EXIF` (Exchangeable Image File Format) block, plus optional `IPTC` and `XMP` blocks. You read it with `exiftool` or `identify -verbose`.
- **PNG** uses **text chunks** with names like `tEXt`, `iTXt`, and `zTXt`, plus an optional `eXIf` chunk. You read it with `exiftool`, `pngcheck -v`, or any PNG-aware tool.
- **PDF** uses an info dictionary (`Title`, `Author`, `Subject`, `Keywords`).
- **MP3 / MP4** uses ID3 tags or atom boxes.

PNG is technically a "container" with named chunks. Each chunk has a 4-byte type code, a 4-byte length, the data, and a 4-byte CRC. Most of the chunks are `IHDR` (header), `IDAT` (compressed pixel data), and `IEND` (end marker), but you can also add `tEXt` chunks for plain text metadata, `iTXt` for international text, and so on. Editors like Photoshop, GIMP, and Adobe ImageReady happily write `tEXt` chunks when you save a PNG.

### Why the hints literally say "meta" and "metadata"

Hint 1: "What does meta mean in the context of files?" — that is asking you to look at the file's metadata, not the picture itself.
Hint 2: "Ever heard of metadata?" — same hint, a second time, in case you missed the first one.

When two hints both point at the same thing, that is the answer. The flag is in the metadata of the picture.

### The tool to use

The standard tool is **`exiftool`** by Phil Harvey. It handles EXIF, IPTC, XMP, PNG text chunks, PDF info, ID3, basically every metadata format you will ever meet. On Kali it is one `apt` away:

```bash
sudo apt install libimage-exiftool-perl
```

If you do not want to install anything, you can read PNG text chunks by hand (it is just a few lines of Python) or use `pngcheck -v` to dump all chunks. The writeup below shows both paths.

---

## Solution — Step by Step

I solved this on Kali Linux in VirtualBox. The challenge file was a PNG named `pico_img.png`, sitting in my shared Downloads folder.

### Step 1 — Copy the file out and confirm what it is

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls -la pico_img.png
-rwxrwxrwx 1 root root 108795 Jul 21 23:05 pico_img.png

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file pico_img.png
pico_img.png: PNG image data, 600 x 600, 8-bit/color RGB, non-interlaced
```

A plain RGB PNG, 600 by 600, no alpha, no interlacing. No special channel layout that would suggest stego.

### Step 2 — Install exiftool (if you do not already have it)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt update

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sudo apt install -y libimage-exiftool-perl

┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool -ver
12.38
```

### Step 3 — Run exiftool on the picture

The defaults print every metadata field it can find. That is what you want here.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool pico_img.png
ExifTool Version Number         : 12.38
File Name                       : pico_img.png
Directory                       : .
File Size                       : 108 kB
File Modification Date/Time     : 2024:01:01 00:00:00+00:00
File Access Date/Time           : 2024:01:01 00:00:00+00:00
File Inode Change Date/Time     : 2024:01:01 00:00:00+00:00
File Permissions                : rwxrwxrwx
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 600
Image Height                    : 600
Bit Depth                       : 8
Color Type                      : RGB
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Software                        : Adobe ImageReady
Artist                          : picoCTF{s0_m3ta_bc056477}
Image Size                      : 600x600
Megapixels                      : 0.360
```

There it is, in the `Artist` field:

```
Artist: picoCTF{s0_m3ta_bc056477}
```

The flag is `picoCTF{s0_m3ta_bc056477}`.

The other interesting field is `Software: Adobe ImageReady` — that is the editor the challenge author used to export the PNG and fill in the Author/Artist tag.

### Step 4 — Submit

Paste the flag into the picoCTF challenge page and submit.

---

## Alternative Solve (No exiftool, just Python)

If you cannot install exiftool (or you are on a system that does not have it), you can read PNG text chunks directly. PNG files are just a sequence of named chunks, so a tiny Python script is enough:

```python
import struct

with open("pico_img.png", "rb") as f:
    sig = f.read(8)  # PNG magic 89 50 4E 47 0D 0A 1A 0A
    while True:
        header = f.read(8)
        if len(header) < 8:
            break
        length, ctype = struct.unpack(">I4s", header)
        ctype_str = ctype.decode("ascii", errors="replace")
        data = f.read(length)
        crc = f.read(4)

        if ctype_str == "tEXt":
            null = data.find(b"\x00")
            keyword = data[:null].decode("latin-1")
            value = data[null + 1:].decode("latin-1", errors="replace")
            print(f"{keyword}: {value}")
        elif ctype_str == "iTXt":
            null = data.find(b"\x00")
            keyword = data[:null].decode("latin-1")
            print(f"{keyword}: <iTXt data, {len(data)} bytes>")
        elif ctype_str == "IEND":
            break
```

Save the script as `read_png_meta.py` and run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 read_png_meta.py
Software: Adobe ImageReady
XML:com.adobe.xmp: <iTXt data, 802 bytes>
Artist: picoCTF{s0_m3ta_bc056477}
```

Same answer, no extra tools required.

### Other CLI alternatives

If you happen to have these installed, they all work too:

- `pngcheck -v pico_img.png` — lists every chunk including `tEXt` content.
- `identify -verbose pico_img.png` — ImageMagick, prints `Properties:` including the Artist tag.
- `strings pico_img.png | grep -i pico` — brute force, just dump printable strings and grep. Lazy, but it works on this challenge.

---

## What Happened Internally (Timeline)

1. The challenge author opened an image (the circular maze you can see in the picture) in an editor — in this case, Adobe ImageReady, an old Adobe product for web graphics.
2. Before exporting, the author filled in the **Artist** metadata field with the flag string `picoCTF{s0_m3ta_bc056477}`. (You can also see a big XMP block with editing history, but the flag is not in there — just in the simple `tEXt` chunk.)
3. The editor saved the PNG. As part of saving, the editor wrote the metadata as PNG `tEXt` chunks: one for `Software` (the editor name) and one for `Artist` (the flag). The 802-byte `iTXt` chunk is the standard Adobe XMP packet, also generated automatically.
4. The PNG was uploaded to the picoCTF challenge server.
5. We downloaded the PNG and ran `exiftool` on it. `exiftool` understands PNG `tEXt` chunks, so it printed every keyword/value pair, including the `Artist` field with the flag.
6. We pasted the flag into the challenge page and got the points.

A fun detail: the `Software: Adobe ImageReady` field is a small hint in itself. ImageReady is the old companion app to Photoshop that Adobe killed off in 2007. It is still useful for one thing — it has a very simple "File Info" dialog where you can type into the Author/Artist field without dealing with Photoshop's larger metadata panel. The challenge author clearly used that.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Confirm the file is a real PNG |
| `exiftool` | Read every metadata field (Software, Artist, XMP, EXIF, etc.) |
| `apt` | Install `libimage-exiftool-perl` (the Debian package for exiftool) |
| Python 3 (`struct` stdlib) | Fallback to read PNG text chunks by hand if exiftool is not installed |
| `pngcheck` (alt) | Dump every PNG chunk including `tEXt` content |
| `identify` (alt) | ImageMagick's verbose mode also prints metadata |
| `strings` + `grep` (alt) | Quick and dirty — just look for printable ASCII in the binary |

---

## Key Takeaways

- **"So Meta" is a literal title.** The challenge is teaching you that the picture itself is not the answer — the data *about* the picture is. Always check metadata first on any "find the flag in this image" challenge, especially when the hints mention the word "meta".
- **`exiftool` is the single most useful forensics tool for image challenges.** It supports JPEG, PNG, TIFF, WebP, HEIC, PDF, MP4, and a hundred other formats. If a flag is in metadata, `exiftool <file>` is almost always the first command to try.
- **PNG text chunks are the PNG equivalent of JPEG EXIF.** They come in three flavors: `tEXt` (Latin-1, uncompressed), `zTXt` (compressed Latin-1), and `iTXt` (UTF-8, optional compression). Editors usually write `tEXt` for simple fields like Author/Artist/Software/Description.
- **The XMP block is usually noise, but it can hide a flag.** In this challenge the XMP packet was 802 bytes of standard Adobe editing metadata. On other challenges, the flag is hidden inside the XMP XML. When you see a big `iTXt` chunk, skim it with `exiftool -xmp:all` or open the raw bytes.
- **No steganography this time.** No LSB tricks, no hidden layers, no password. The flag is right there in plaintext, in a metadata field that any text editor can find. Read the hints. They are usually literal.
- **Wordplay decode of the flag:** `picoCTF{s0_m3ta_bc056477}` → read the part inside the braces: `s0_m3ta` is leet-speak for **"so meta"** (the challenge name), and `bc056477` is just a per-instance nonce (8 hex chars) so the flag cannot be reused across regenerations of the artifact. The whole flag is "so meta, plus a unique id". Meta on meta.
