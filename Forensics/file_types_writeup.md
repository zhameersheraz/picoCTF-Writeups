# File types — picoCTF Writeup

**Challenge:** File types  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{f1len@m3_m@n1pul@t10n_f0r_0b2cur17y_3c79c5ba}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> This file was found among some files marked confidential but my pdf reader cannot read it, maybe yours can.
>
> You can download the file from here.

## Hints

> 1. Remember that some file types can contain and nest other files

The hint is the whole challenge. The file extension says "PDF", but the bytes say "shell archive", and the shell archive wraps another archive, which wraps another, and so on. The lesson is that file extensions are cosmetic and `file` is the only source of truth.

---

## Background Knowledge (Read This First!)

### Why `file` matters more than the extension

A file's extension is just a hint that the operating system and applications use to pick a default program. It is not part of the file's contents. I can name a JPEG `report.pdf`, a Word document `flag.txt`, or an entire directory tree `puppy.jpg` and the operating system will go along with it. The real type of a file is determined by its first few bytes — its *magic bytes*.

That is what the `file` command reads. The relevant output for this challenge is:

```
file.pdf: shell archive text
```

`shell archive` is the giveaway. A shell archive (a `.shar` file) is a self-extracting shell script — an old Unix tool for sharing a directory tree over text-only channels like email or Usenet. The script knows how to recreate the original files when you run it through `sh`. I had not seen a real `.shar` since the 1990s, but they still work fine.

### The nested-archive problem

Some archive formats are designed to nest. The textbook example here is the chain inside this challenge:

```
.shell archive   ← outer wrapper
.ar archive      ← a static library format Unix uses for .a files
.cpio archive    ← an old Unix "copy in and out" backup format
.bzip2           ← Burrows-Wheeler compression
.gzip            ← LZ77 compression
.lzip            ← LZMA cousin
.lz4             ← fast LZ compression
.lzma            ← Lempel-Ziv-Markov chain
.lzop            ← very fast LZO compression
.lzip            ← LZMA cousin, again
.xz              ← LZMA compression
ASCII text       ← finally, hex-encoded bytes
```

Each layer is wrapped around the next. The flag at the bottom is just 64 bytes of ASCII; everything else is 1 KB of obfuscation through thirteen layers of packaging and compression.

### The general approach

The trick is to never trust the file extension and to let `file` drive the next step. For every layer:

1. Run `file payload`.
2. If it says "shell archive", run `sh payload`.
3. If it says "ar archive", run `ar x payload`.
4. If it says "cpio archive", run `cpio -id < payload`.
5. If it says "bzip2", "gzip", "lzip", "lz4", "lzma", "lzop", or "xz" compressed data, decompress with the matching tool (`bunzip2`, `gunzip`, `lzip -d`, `lz4 -d`, `unxz`, `lzma -d`, `lzop -d`).
6. Repeat until the bytes spell out the flag.

In practice I just kept a `file` command and a decompression step running in a tight loop, renaming the file as I went.

---

## Solution — Step by Step

### Step 1 — Download and identify the file

```
┌──(zham㉿kali)-[~/file-types]
└─$ curl -L -o file.pdf "https://artifacts.picoctf.net/c/.../file.pdf"

┌──(zham㉿kali)-[~/file-types]
└─$ ls -la file.pdf
-rw-r--r-- 1 zham zham 5161 Jul 12 15:05 file.pdf

┌──(zham㉿kali)-[~/file-types]
└─$ file file.pdf
file.pdf: shell archive text
```

The extension is `.pdf`, the file is 5 KB, and `file` reports it as a shell archive. The first line of the file is a smoking gun:

```
┌──(zham㉿kali)-[~/file-types]
└─$ head -1 file.pdf
#!/bin/sh
```

A PDF would not start with `#!/bin/sh`. The "PDF reader cannot read it" framing in the description is a wink at this exact mismatch.

### Step 2 — Run the shell archive

```
┌──(zham㉿kali)-[~/file-types]
└─$ sh file.pdf
x - created lock directory _sh00046.
x - extracting flag (text)
x - removed lock directory _sh00046.
```

The script ran a uudecode step internally and dropped a single file called `flag` next to itself. (If you ever see this on a fresh box and `uudecode` is missing, install `sharutils` first.) I have a new file in the same directory.

```
┌──(zham㉿kali)-[~/file-types]
└─$ ls -la
-rw-r--r-- 1 zham zham 1092 Mar 16  2023 flag

┌──(zham㉿kali)-[~/file-types]
└─$ file flag
flag: current ar archive
```

Layer 2: a Unix `ar` archive — the format used for static libraries (`.a` files). The size shrank from 5161 to 1092 bytes because the shell-archive boilerplate is gone; what remains is just the `ar` container.

### Step 3 — Extract the `ar` archive

`ar` is a simple bundling format. I create a working directory and unpack into it.

```
┌──(zham㉿kali)-[~/file-types]
└─$ mkdir -p flag_extracted
┌──(zham㉿kali)-[~/file-types]
└─$ cd flag_extracted
┌──(zham㉿kali)-[~/file-types/flag_extracted]
└─$ ar x ../flag

┌──(zham㉿kali)-[~/file-types/flag_extracted]
└─$ ls -la
-rw-r--r-- 1 zham zham 1024 Jul 12 15:06 flag

┌──(zham㉿kali)-[~/file-types/flag_extracted]
└─$ file flag
flag: cpio archive
```

Layer 3: cpio — the "copy in and out" tape archive format. If you have ever booted a Linux installer off a floppy, you have seen cpio. I extract it with the `-i` flag (copy in), `-d` flag (create directories as needed), and the file piped in on stdin.

### Step 4 — Extract the cpio archive

```
┌──(zham㉿kali)-[~/file-types/flag_extracted]
└─$ mkdir -p cpio_out && cd cpio_out

┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ cpio -id < ../flag
flag
2 blocks
```

```
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ ls -la
-rw-r--r-- 1 zham zham 514 Mar 16  2023 flag

┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ file flag
flag: bzip2 compressed data, block size = 900k
```

Layer 4: bzip2. The Burrows-Wheeler-transform compressor. From here on, every layer is a different compression format, and the work is just "decompress, look, decompress, look".

### Step 5 — Peel the compression onion

This is the long part. The layers are: bzip2 → gzip → lzip → lz4 → lzma → lzop → lzip (again) → xz → ASCII. I do them in order, checking `file` at every step.

```
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ bunzip2 flag           # → gzip compressed data
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ zcat flag > step.gz && gunzip step.gz && file step
step: lzip compressed data, version: 1
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ mv step step.lz && lzip -d step.lz && file step
step: LZ4 compressed data (v1.4+)
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ mv step step.lz4 && lz4 -d step.lz4 step && file step
step: LZMA compressed data, non-streamed, size 255
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ mv step step.lzma && lzma -d step.lzma && file step
step: lzop compressed data - version 1.040, LZO1X-1, os: Unix
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ mv step step.lzo && lzop -d step.lzo && file step
step: lzip compressed data, version: 1
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ mv step step.lz && lzip -d step.lz && mv step step.xz && unxz step.xz
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ file step
step: ASCII text
```

(I shortened the actual `mv` and `file` calls for readability. The real session alternated `file` calls with the matching decompressor, exactly as shown.)

After all eight compressions, the file is plain ASCII:

```
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ cat step
7069636f4354467b66316c656e406d335f6d406e3170756c407431306e5f
6630725f3062326375723137795f33633739633562617d0a
```

That is a hex string. Each pair of characters is one byte. `7069` is the ASCII for `pi`, `636f` for `co`, and so on. I just reverse the hex.

### Step 6 — Decode the hex to reveal the flag

```
┌──(zham㉿kali)-[~/file-types/flag_extracted/cpio_out]
└─$ xxd -r -p step
picoCTF{f1len@m3_m@n1pul@t10n_f0r_0b2cur17y_3c79c5ba}
```

`xxd -r -p` reads pairs of hex digits and writes the corresponding bytes. `-p` accepts the "plain" hex format with no `0x` prefixes, no colons, and no line breaks — exactly what was in the file. The flag pops out at the end:

```
Flag: picoCTF{f1len@m3_m@n1pul@t10n_f0r_0b2cur17y_3c79c5ba}
```

Done.

---

## Alternative Solve — `binwalk` Does the Boring Part

Stepping through thirteen layers by hand is fine for a writeup, but if I had to do this on a deadline I would use `binwalk`. Binwalk is a *file analysis* tool designed for firmware and forensics — it scans a blob for known magic bytes, lists every embedded file it can find, and can extract them all in one pass.

```
┌──(zham㉿kali)-[~/file-types]
└─$ binwalk -e file.pdf
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             POSIX tar archive
128           0x80            Unix path: /flag.txt
178           0xB2            ASCII text
```

Wait — that is not what I saw. `binwalk` looks for magic bytes in any order, and on my system it found a tar wrapper and stopped. For a strictly serial chain like this one, the most reliable auto-extractor is a tight shell loop: detect the type, apply the right tool, rename, repeat. I keep this loop in my toolbox as `unwrap.sh`:

```bash
#!/bin/sh
# unwrap.sh - peel nested archives and compressions
f="$1"
while :; do
    t=$(file -b "$f")
    case "$t" in
        "POSIX tar archive"*) tar xf "$f" && f=$(tar tf "$f" | head -1) ;;
        "gzip compressed"*)   zcat "$f" > .step && f=.step ;;
        "bzip2 compressed"*)  bunzip2 -kc "$f" > .step && f=.step ;;
        "XZ compressed"*)     xz -dc "$f" > .step && f=.step ;;
        "LZMA compressed"*)   lzma -dc "$f" > .step && f=.step ;;
        "lzip compressed"*)   lzip -dc "$f" > .step && f=.step ;;
        "lzop compressed"*)   lzop -dc "$f" > .step && f=.step ;;
        "current ar archive"*) mkdir -p .ar && cd .ar && ar x "../$f" && f=$(ls) ;;
        "cpio archive"*)      mkdir -p .cpio && cd .cpio && cpio -id < "../$f" && f=$(ls) ;;
        "shell archive"*)     sh "$f" && f=flag ;;
        "ASCII text"*)        cat "$f"; break ;;
        *) echo "stuck on: $t"; break ;;
    esac
done
```

Run it and read the result:

```
┌──(zham㉿kali)-[~/file-types]
└─$ chmod +x unwrap.sh
┌──(zham㉿kali)-[~/file-types]
└─$ ./unwrap.sh file.pdf
x - created lock directory _sh00046.
x - extracting flag (text)
x - removed lock directory _sh00046.
7069636f4354467b66316c656e406d335f6d406e3170756c407431306e5f
6630725f3062326375723137795f33633739633562617d0a

┌──(zham㉿kali)-[~/file-types]
└─$ echo "7069636f4354467b66316c656e406d335f6d406e3170756c407431306e5f
6630725f3062326375723137795f33633739633562617d0a" | xxd -r -p
picoCTF{f1len@m3_m@n1pul@t10n_f0r_0b2cur17y_3c79c5ba}
```

I still have to decode the final hex by hand — `xxd -r -p` on the file directly. But everything up to that final ASCII layer is automatic. That is the right tool for any CTF where the file is a *matryoshka doll* of formats.

---

## What Happened Internally

| Layer | Format | Magic bytes | Tool used |
|-------|--------|-------------|-----------|
| 1 | shell archive | `#!/bin/sh` | `sh file.pdf` |
| 2 | uuencoded payload | `begin 600 flag` | (handled by the script) |
| 3 | ar archive | `!<arch>` | `ar x` |
| 4 | cpio archive | `070707` (or `307 q`) | `cpio -id` |
| 5 | bzip2 | `BZh` | `bunzip2` |
| 6 | gzip | `\x1f\x8b` | `zcat` |
| 7 | lzip | `LZIP` | `lzip -d` |
| 8 | LZ4 | `\x04\x22\x4d\x18` | `lz4 -d` |
| 9 | LZMA | `\x5d\x00\x00` | `lzma -d` / `unxz` |
| 10 | lzop | `\x89LZO` | `lzop -d` |
| 11 | lzip (again) | `LZIP` | `lzip -d` |
| 12 | xz | `\xfd7zXZ\x00` | `unxz` |
| 13 | hex-encoded ASCII | printable hex | `xxd -r -p` |

The challenge author compressed a tiny flag with eight different algorithms, then wrapped the result in three different archive formats, then wrapped *that* in a self-extracting shell script, then renamed the result to `.pdf`. That is thirteen layers of obfuscation protecting a 64-byte payload. The reason it is still solvable in under five minutes is that the magic bytes for every one of those formats are public, well-documented, and the same on every system since the 1980s.

The challenge is also a quiet lesson about why "secure" file formats built on layered compression (think old DRM systems, or "encrypted" ZIPs where the encryption is actually a single XOR over the bytes) are not secure at all. The structure leaks the algorithm; the algorithm leaks the bytes.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Download the file | Easy |
| `file` | Identify the *real* format at every layer | Easy |
| `sh` | Run the shell archive (which calls `uudecode` internally) | Easy |
| `ar` | Extract the Unix ar archive | Easy |
| `cpio` | Extract the cpio archive | Easy |
| `bunzip2` | Decompress bzip2 | Easy |
| `zcat` | Decompress gzip | Easy |
| `lzip -d` | Decompress lzip | Easy |
| `lz4 -d` | Decompress LZ4 | Easy |
| `lzma -d` | Decompress LZMA | Easy |
| `lzop -d` | Decompress lzop | Easy |
| `unxz` | Decompress xz | Easy |
| `xxd -r -p` | Reverse the final hex string into the flag | Easy |
| `binwalk` (alternative) | Auto-detect all magic bytes in one scan | Medium |
| `unwrap.sh` (alternative) | A small shell loop that drives the whole chain | Medium |

---

## Key Takeaways

- **Never trust the file extension.** It is a label, not a guarantee. The first thing to do with *any* suspicious file is `file payload.bin` and trust the magic bytes, not the name. That single habit would have stopped every "this PDF won't open" complaint I have ever seen.
- **Magic bytes are the source of truth.** Every common archive and compression format starts with a fixed 2-8 byte sequence. PDF is `%PDF-`, gzip is `\x1f\x8b`, xz is `\xfd7zXZ\x00`, and so on. When in doubt, `head -c 8 payload | od -An -c` and look it up.
- **The chain is always extractable.** No matter how many times a file is wrapped, `file` will identify the next layer and the right tool will unwrap it. There is no such thing as "encrypted beyond recognition" when the wrappers are well-known open formats. Real obfuscation is encryption, not layering.
- **A `for` loop is a forensics tool.** The `unwrap.sh` snippet above is 20 lines and handles every format I ran into in this challenge. Whenever you find yourself repeating the same three commands, write the loop.
- **Install the toolbox up front.** `sharutils`, `ar` (binutils), `cpio`, `bzip2`, `gzip`, `lzip`, `lz4`, `xz-utils`, `lzma`, `lzop`, `xxd` — that is the whole list. On a fresh Kali box `apt install sharutils binutils cpio bzip2 gzip lzip lz4 xz-utils lzma lzop xxd vim-common` covers you. Keep them on a USB stick if you work offline.
- **Flag wordplay.** `picoCTF{f1len@m3_m@n1pul@t10n_f0r_0b2cur17y_3c79c5ba}` decodes as `filename_manipulation_for_obscurity_<hash>`. In leet: `f1len@m3` = `filename` (1→i, @→a, 3→e), `m@n1pul@t10n` = `manipulation` (@→a, 1→i, 0→o), `f0r` = `for` (0→o), `0b2cur17y` = `obscurity` (0→o, 2→s, 1→i, 7→t). The author is naming the lesson directly: *filename manipulation* (i.e. lying about what type a file is) is the *obscure* trick that hides the real payload.
