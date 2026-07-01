# endianness-v2 — picoCTF Writeup

**Challenge:** endianness-v2  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{cert!f1Ed_iNd!4n_s0rrY_3nDian_76e05f49}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> Here's a file that was recovered from a 32-bits system that organized the bytes a weird way. We're not even sure what type of file it is.
>
> Download it here and see what you can get out of it.

**Hints shown in challenge:** *None — this challenge has no hints.*

---

## Background Knowledge (Read This First!)

### What is Endianness?

Endianness is the order in which bytes are arranged inside a multi-byte value when it is stored in memory or written to disk.

- **Big Endian (BE)** — most significant byte first. This matches the way we read numbers left to right. Network protocols like IPv4 and IPv6 use big endian ("network byte order").
- **Little Endian (LE)** — least significant byte first. The byte order is reversed. Most desktop CPUs (Intel, AMD) read and write integers in little endian by default.

Example with the 32-bit value `0x12345678`:

| Byte order on disk | Style |
|---|---|
| `12 34 56 78` | Big Endian |
| `78 56 34 12` | Little Endian |

### Why does this matter for files?

Every file format starts with a "magic number" — the first few bytes that identify the file type. JPEG always begins with `FF D8 FF E0` (or `FF D8 FF E1` for Exif). PNG begins with `89 50 4E 47 0D 0A 1A 0A`. ZIP begins with `50 4B 03 04`. If those bytes get stored in the wrong order, the file looks like garbage to every viewer and `file` reports it as plain `data`.

### What this challenge does

The challenge file is a JPEG image, but each **32-bit (4-byte) chunk** has had its bytes reversed. So instead of starting with `FF D8 FF E0`, it starts with `E0 FF D8 FF`. The fix is simply to swap the bytes back inside every 4-byte group, then save the result as `.jpg`.

### Common File Magic Bytes (for reference)

| Format | Magic bytes |
|---|---|
| JPEG / JFIF | `FF D8 FF E0` |
| PNG | `89 50 4E 47` |
| GIF | `47 49 46 38` |
| ZIP / PKZip | `50 4B 03 04` |
| PDF | `25 50 44 46` |

If you ever see a file with no extension or `file` says `data`, the first thing to do is `hexdump` (or `xxd`) and compare the first 4 bytes against this table. That alone solves a huge number of forensics challenges.

---

## Solution — Step by Step

### Step 1 — Download the challenge file

I make a working directory and pull the file from the picoCTF artifact server. The challenge gives a generic link with no filename, so I save it as `challengefile`.

```
┌──(zham㉿kali)-[~/Downloads]
└─$ mkdir -p /work/endianness-v2 && cd /work/endianness-v2

┌──(zham㉿kali)-[/work/endianness-v2]
└─$ wget https://artifacts.picoctf.net/c_titan/85/challengefile
--2026-07-01 20:06:00--  https://artifacts.picoctf.net/c_titan/85/challengefile
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 18.155.173.16
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|18.155.173.16|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3421 (3.3K)
Saving to: 'challengefile'

challengefile                100%[==================================>]   3.34K  --.-KB/s    in 0s

┌──(zham㉿kali)-[/work/endianness-v2]
└─$ ls -la
-rw-r--r-- 1 zham zham 3421 Jul  1 20:06 challengefile
```

### Step 2 — Try `file` first

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ file challengefile
challengefile: data
```

`file` has no idea what this is. That alone tells me the magic bytes are corrupted — time to look at them.

### Step 3 — Inspect the hex of the first bytes

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ hexdump -C challengefile | head -1
00000000  e0 ff d8 ff 46 4a 10 00  01 00 46 49 01 00 00 01  |....FJ....FI....|
```

The first 16 bytes are `e0 ff d8 ff 46 4a 10 00 01 00 46 49 01 00 00 01`. Comparing to the JPEG/JFIF magic:

| Offset | Bytes in file | What they should be | Meaning |
|---|---|---|---|
| 0 | `E0 FF D8 FF` | `FF D8 FF E0` | JPEG/JFIF magic |
| 4 | `46 4A 10 00` | `46 4A 10 00` | `"FJ\x10\x00"` (length 16) — same! |
| 8 | `01 00 46 49` | `01 00 46 49` | JFIF version 1.01 — same! |
| 12 | `01 00 00 01` | `01 00 00 01` | density / thumbnail — same! |

The first four bytes are simply the JPEG magic with the byte order reversed within the 32-bit word. Everything else inside each 4-byte group is identical to what a real JPEG has. That is the entire pattern: **every 4-byte group is byte-swapped.**

### Step 4 — Write the un-swap script in `nano`

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ nano fix.py
```

Paste this inside nano:

```python
#!/usr/bin/env python3
# fix.py - reverse the byte order of every 4-byte word in challengefile
# so that the corrupted JPEG magic bytes become a valid JFIF header.

with open("challengefile", "rb") as f:
    data = f.read()

out = bytearray()
for i in range(0, len(data), 4):
    chunk = data[i:i+4]
    out += chunk[::-1]   # reverse this 4-byte group

with open("recovered.jpg", "wb") as f:
    f.write(out)

print(f"Wrote {len(out)} bytes to recovered.jpg")
```

Save and exit:

- `Ctrl + O` → `Enter` (write the file)
- `Ctrl + X` (quit nano)

### Step 5 — Run the script

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ python3 fix.py
Wrote 3421 bytes to recovered.jpg
```

### Step 6 — Verify the recovered file is now a real JPEG

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ file recovered.jpg
recovered.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1,
segment length 16, baseline, precision 8, 300x150, components 3
```

The first 4 bytes are now `FF D8 FF E0` (proper JFIF magic), and `file` happily reports it as a real JPEG image, 300×150 pixels.

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ hexdump -C recovered.jpg | head -1
00000000  ff d8 ff e0 00 10 4a 46  49 46 00 01 01 00 00 01  |......JFIF......|
```

`FF D8 FF E0` followed by length `00 10` and the ASCII text `JFIF` — perfect JPEG header.

### Step 7 — Open the recovered JPEG and read the flag

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ xdg-open recovered.jpg
```

The image is a tiny 300×150 white canvas with one line of black text printed on it:

```
picoCTF{cert!f1Ed_iNd!4n_s0rrY_3nDian_76e05f49}
```

I paste it into the picoCTF submission box and the challenge is solved.

```
picoCTF{cert!f1Ed_iNd!4n_s0rrY_3nDian_76e05f49}
```

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `wget` the file as `challengefile` | 3.3 KB file on disk, no extension |
| 0:00 | `file challengefile` | Returned `data` — magic bytes don't match anything |
| 0:01 | `hexdump -C challengefile \| head -1` | Saw `E0 FF D8 FF` as the first 4 bytes |
| 0:01 | Compared to JPEG magic `FF D8 FF E0` | Realized the 32-bit word is byte-swapped |
| 0:01 | Compared remaining 4-byte groups | All other chunks were also internally reversed but in pairs that happened to look fine on their own — confirmed the pattern is uniform across the file |
| 0:02 | Wrote `fix.py` that reverses each 4-byte group | File ready to run |
| 0:02 | `python3 fix.py` | Produced `recovered.jpg`, 3421 bytes |
| 0:03 | `file recovered.jpg` | Confirmed it is now a real JFIF JPEG (300×150) |
| 0:03 | `xdg-open recovered.jpg` | Image showed the flag as printed text |
| 0:03 | Submitted `picoCTF{cert!f1Ed_iNd!4n_s0rrY_3nDian_76e05f49}` | Challenge marked solved |

The "internally" detail that matters: every byte of the file is still in the correct *position*; only the **order inside each 4-byte word** is reversed. That is why a simple `chunk[::-1]` over chunks of 4 recovers everything — headers, Huffman tables, image data, and the EOI marker `FF D9` at the very end.

---

## Alternative Methods

### Method 1 — One-liner with `hexdump` + `xxd -r`

If you don't want to write a Python script, you can pipe the file through `hexdump` formatted as 4-byte big-endian hex words and use `xxd -r -p` to turn it back into raw bytes. `hexdump` reads in 4-byte groups and prints them with the most significant byte first (`%08X`), which reverses the byte order that was inside each little-endian word.

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ hexdump -v -e '1/4 "%08X"' -e '"\n"' challengefile | xxd -r -p > recovered.jpg

┌──(zham㉿kali)-[/work/endianness-v2]
└─$ file recovered.jpg
recovered.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1,
segment length 16, baseline, precision 8, 300x150, components 3
```

Open the resulting `recovered.jpg` and read the flag. No code, no editor — just two classic Unix tools chained with a pipe.

### Method 2 — CyberChef (GUI in the browser)

If you'd rather not use the terminal:

1. Open https://gchq.github.io/CyberChef/
2. Drag `challengefile` into the **Input** box
3. Add the **Swap Endianness** operation; set Word Length to `4` and Endianness to `Little → Big`
4. Set Output to `Raw` and click **Save output to file** — name it `recovered.jpg`
5. Open the saved file with any image viewer

CyberChef also has a **Render Image** operation you can slap after Swap Endianness — it will display the recovered JPEG inline so you don't even need to save it.

### Method 3 — Manual hex editor (`ghex` / `hexedit`)

This is the painful but instructive way. Open the file in a GUI hex editor:

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ sudo apt install -y ghex
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ ghex challengefile
```

Then manually swap the first four bytes from `E0 FF D8 FF` to `FF D8 FF E0`, save the file as `challengefile.jpg`, and confirm with `file`. This only works for the *header* check; the rest of the image data is still corrupted until you reverse **every** 4-byte group in the file — which is why I recommend the script instead.

### Method 4 — `perl` one-liner (no Python needed)

If `python3` isn't around but `perl` is, you can do the same 4-byte reversal with `perl`:

```
┌──(zham㉿kali)-[/work/endianness-v2]
└─$ perl -e 'open(F,"<challengefile"); binmode F; read(F,$d,3421); close F;
            $d =~ s/(.)(.)(.)(.)/$4$3$2$1/gs;
            open(O,">recovered.jpg"); binmode O; print O $d; close O;'

┌──(zham㉿kali)-[/work/endianness-v2]
└─$ file recovered.jpg
recovered.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1,
segment length 16, baseline, precision 8, 300x150, components 3
```

The regex `(.)(.)(.)(.)/$4$3$2$1/gs` matches each 4-byte run and rewrites it in reverse. The `/s` modifier is important — without it, perl's `.` does not match newlines, and JPEG data often contains `0x0A` bytes that would silently fall through the regex and break every offset after them.

> **Note:** the usual "byte-swab with `dd`" trick (`dd conv=swab`) only swaps *pairs* of bytes, so it cannot reverse 4-byte groups on its own. Two passes of `conv=swab` would just undo the first pass and land you back on the original file. That's why `dd` isn't a good fit for this particular challenge — stick with the script, the `hexdump | xxd` pipe, or `perl`.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `wget` | Download the challenge file from picoCTF's artifact server | Easy |
| `file` | Identify (or fail to identify) the original magic bytes | Easy |
| `hexdump -C` | View the first bytes and compare against the JPEG/JFIF magic | Easy |
| `nano` | Write the small Python fix script | Easy |
| `python3` | Reverse the byte order inside every 4-byte word | Easy |
| `xdg-open` | Open the recovered JPEG and read the flag text | Easy |
| `hexdump` + `xxd -r -p` (alt) | One-liner pipe that does the same fix without writing Python | Medium |
| CyberChef "Swap Endianness" (alt) | Browser-based GUI for the same transformation | Easy |
| `ghex` (alt) | Manual hex editor for inspecting and patching the header by eye | Medium |
| `perl` one-liner (alt) | Same byte-swap done with a `perl` regex, no Python needed | Medium |

---

## Key Takeaways

- **Magic bytes are your first forensic move.** When `file` says `data`, hexdump the first 4–16 bytes and compare against the [list of file signatures](https://en.wikipedia.org/wiki/List_of_file_signatures). Even when the bytes look "wrong", they are usually a recognizable signature with a small transformation applied — which is the whole puzzle.
- **Endianness is a per-word property, not per-byte.** Endianness only affects how multi-byte values are laid out. To reverse 32-bit endianness you reverse every **group of 4 bytes**; to reverse 16-bit endianness you reverse every **pair of bytes**. Position of the group does not change.
- **The `hexdump -v -e '1/4 "%08X"' | xxd -r -p` trick is worth memorizing.** It is the fastest non-Python way to do a 4-byte endian swap on a whole file. The format string `'1/4 "%08X"'` makes `hexdump` print every 4 bytes as one big-endian 8-hex-digit number, which is exactly the byte-swapped version of the original little-endian word; `xxd -r -p` then converts those hex words back into raw bytes.
- **Always verify your output with `file`.** If your fix is correct, `file` will go from `data` to a real format description (here, `JPEG image data, JFIF standard 1.01`). If it still says `data`, you got the chunk size wrong.
- **There is no hint in this challenge on purpose.** It is the natural follow-up to the General Skills `endianness` challenge: there you only swapped bytes within a 2-byte word for a text string; here you have to apply the same idea to a whole binary file in 4-byte words. The category and difficulty go up, but the core concept is the same.

### Flag wordplay decode

```
picoCTF{cert!f1Ed_iNd!4n_s0rrY_3nDian_76e05f49}
   |     |     |     |       |     |
   cert  = certificate (the JPEG's "magic" / signature)
         !f1Ed = ified  ("cert-ified")
                iNd!4n = Indian ("in-ian")
                       s0rrY = sorry
                              3nDian = endian ("end-ian")
                                      76e05f49 = random hash suffix
```

Put together: **"certified Indian, sorry endian"** — a tongue-in-cheek nod to the challenge itself. The challenge author is "sorry" they had to swap the endianness on a perfectly good JPEG, but at least the flag is "certified" to be there once you put the bytes back the way they belong.
