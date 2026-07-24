# B1g_Mac — picoCTF Writeup

**Challenge:** B1g_Mac  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 500  
**Flag:** `picoCTF{M4cTim35!}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This is a forensics challenge that requires knowledge of file timestamp analysis. Some file types are commonly used to hide messages in them, and your job is to find out what type of file is hidden inside this one.

**Attachments:** `b1g_mac.zip` → `test/` folder (18 BMP files + `main.exe`)

---

## Hints

> 1. Forensics is about noticing things that others might miss. What could be hidden in the file timestamps of a Microsoft document?
> 2. The file extension is misleading. What is the actual file type?
> 3. What kind of data is stored in the file system about a file? The most common way to store this data is in something called "metadata".

---

## Background Knowledge (Read This First!)

### What does "MAC" mean in this challenge?

The title **"B1g_Mac"** is a play on three ideas at once:

1. **Apple Mac** (the obvious one, but it is a red herring — nothing here is Mac-specific).
2. **MAC address / Message Authentication Code** (also misleading).
3. **NTFS MAC timestamps** — this is the real one. On Windows NTFS, every file has three timestamps commonly called the **MAC times**:
   - **M**odification time (when the file's *data* was last changed)
   - **A**ccess time (when the file was last *read*)
   - **C**reation time (when the file was *born* on the volume)

These timestamps are stored in two places:
- Inside the NTFS **MFT** (Master File Table) record for each file.
- Inside the **NTFS extra field** of any ZIP archive that was created on an NTFS volume using a tool that preserves these fields (Windows Explorer, 7-Zip, etc.).

### What is the NTFS Extra Field in ZIP?

The ZIP file format allows every central directory entry to have an **extra field** — a list of variable-length records tagged with a 16-bit ID. One of those IDs, **`0x000a`**, is reserved for **NTFS metadata**. Inside it lives a copy of the `$STANDARD_INFORMATION` attribute, which is exactly where the M/A/C timestamps live.

The record layout (32 bytes total) is:

```
+0:  tag (0x000a)               <- 2 bytes
+2:  record size (0x0020)        <- 2 bytes
+4:  reserved (0x00000000)       <- 4 bytes
+8:  attribute tag (0x0001)      <- 2 bytes  ($STANDARD_INFORMATION)
+10: attribute size (0x0018)     <- 2 bytes  (24 bytes follow)
+12: LastWriteTime (FILETIME)    <- 8 bytes  <-- the stego lives here
+20: ChangeTime     (FILETIME)   <- 8 bytes
+28: LastAccessTime (FILETIME)   <- 8 bytes
```

**FILETIME** is a 64-bit Windows timestamp counting 100-nanosecond intervals since January 1, 1601. The **lowest 2 bytes** of LastWriteTime can hold any byte value — which makes it a perfect place to smuggle two characters of a flag.

### Why the "Copy" files exist

When you `cp foo.bmp foo - Copy.bmp` on NTFS, the new file's M/A/C times are set to the copy moment, while the data is identical. The "Copy" files in this challenge are byte-for-byte identical to the originals — but the **NTFS extra field in the ZIP remembers the original NTFS times**, and those are the ones that encode the flag.

---

## Solution — Step by Step

### Step 1 — Unzip the archive and look around

Unzip into a working folder and list the contents. The folder `test/` contains 18 BMPs (9 original + 9 "Copy") and a Windows PE called `main.exe`.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip -o b1g_mac.zip -d b1g_mac/
Archive:  b1g_mac.zip
   creating: b1g_mac/test/
  inflating: b1g_mac/test/Item01.bmp
  inflating: b1g_mac/test/Item01 - Copy.bmp
  inflating: b1g_mac/test/Item02.bmp
  inflating: b1g_mac/test/Item02 - Copy.bmp
  ...
  inflating: b1g_mac/test/main.exe
```

The BMPs are each 127,654 bytes. The interesting bit is the `Copy` variants — those are the stego carriers.

### Step 2 — Confirm the BMPs are byte-identical to their originals

Compute hashes on the unzipped files first. On Linux the unzip step may have rewritten the timestamps, so this check is just to see that the **data itself** is unmodified.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cd b1g_mac/test && md5sum Item*.bmp
65191f4a1ba5d8e02bf8628c20b704b6  Item01.bmp
65191f4a1ba5d8e02bf8628c20b704b6  Item01 - Copy.bmp
65191f4a1ba5d8e02bf8628c20b704b6  Item02.bmp
65191f4a1ba5d8e02bf8628c20b704b6  Item02 - Copy.bmp
...
```

Every pair matches. So the stego is **not** in the BMP bytes. It must be in the metadata the unzip step just discarded.

### Step 3 — Read the NTFS extra fields straight from the original ZIP

The standard `unzip` on Linux rewrites timestamps and **drops** the NTFS extra field, so the on-disk files no longer carry the stego. We have to read the central directory of the **original ZIP** directly.

I keep a small Python script for this. Save the following as `extract_b1g.py` in `C:\Users\ASUS\Downloads\` (it syncs to `/media/sf_downloads/` on the Kali share).

```python
"""Extract the B1g_Mac flag from NTFS LastWriteTime in ZIP central directory."""
import struct
import sys
import zipfile


def extract_lastwritetime(zf, filename):
    info = zf.getinfo(filename)
    extra = info.extra
    tag, _ = struct.unpack_from('<HH', extra, 0)
    if tag != 0x000a:
        return None
    # tag(2) + size(2) + reserved(4) + attr_tag(2) + attr_size(2) = 12
    return struct.unpack_from('<Q', extra, 12)[0]


def main(zip_path):
    flag = ''
    with zipfile.ZipFile(zip_path) as zf:
        copy_files = sorted(n for n in zf.namelist() if 'Copy.bmp' in n)
        for f in copy_files:
            mtime = extract_lastwritetime(zf, f)
            if mtime is None:
                continue
            c0 = mtime & 0xff
            c1 = (mtime >> 8) & 0xff
            pair = chr(c1) + chr(c0)
            print(f'  {f}:  ->  {pair!r}')
            flag += pair
    print(f'\nRecovered flag: {flag}')


if __name__ == '__main__':
    main(sys.argv[1] if len(sys.argv) > 1 else 'b1g_mac.zip')
```

Run it from the share:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 extract_b1g.py b1g_mac.zip
  test/Item01 - Copy.bmp:  ->  'pi'
  test/Item02 - Copy.bmp:  ->  'co'
  test/Item03 - Copy.bmp:  ->  'CT'
  test/Item04 - Copy.bmp:  ->  'F{'
  test/Item05 - Copy.bmp:  ->  'M4'
  test/Item06 - Copy.bmp:  ->  'cT'
  test/Item07 - Copy.bmp:  ->  'im'
  test/Item08 - Copy.bmp:  ->  '35'
  test/ItemTest - Copy.bmp:  ->  '!}'

Recovered flag: picoCTF{M4cTim35!}
```

The flag is `picoCTF{M4cTim35!}`.

### Step 4 — Why byte-swap?

Looking at the first Copy's raw `LastWriteTime`:

```
extra[12:20] = 69 70 33 49 61 e3 d4 01
```

As a little-endian 64-bit integer the **low 2 bytes** are `0x69 0x70` (`i p`). But the expected first two flag characters are `p i`, not `i p`. So the stego swaps the two low bytes: the **second** byte holds the first flag character, the **first** byte holds the second. That is why the script reads `c1 = (mtime >> 8) & 0xff` (second byte) before `c0 = mtime & 0xff` (first byte).

---

## What Happened Internally

1. The author built a 9-image BMP dataset on a Windows NTFS volume.
2. They copied each BMP once to make a "Copy" sibling, generating 9 stego carriers.
3. A small Windows program (`main.exe`) ran after each copy, opened the new file, and **set its `LastWriteTime` FILETIME so that the lowest 2 bytes spell out 2 characters of the flag** (with the byte swap shown above).
4. They zipped the folder using a tool that preserves NTFS extra fields (the default Windows ZIP handler does this).
5. The Linux `unzip` we used dropped the extra field, so we had to parse the original archive directly.

If you want to confirm step 3 without the script, the `main.exe` binary contains a hard-coded password string `PASSw0rd` near offset `0xA1B` (used by the encode routine) and a `DECODE` keyword that matches the script naming. A `strings` pass shows the password is the same one used by the investigative-reversing challenges — it is a fixed key embedded in the source.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `unzip` | Extract the archive to inspect the layout | Easy |
| `md5sum` | Verify that the Copy BMPs are byte-identical to the originals | Easy |
| `python3` + `zipfile` + `struct` | Parse the NTFS extra field (tag `0x000a`) directly from the ZIP central directory | Medium |
| `strings` (optional) | Spot the `PASSw0rd` key inside `main.exe` and confirm the encode routine's identity | Easy |

---

## Key Takeaways

- **The file content was not the stego carrier.** Every Copy BMP was a byte-for-byte clone of the original. Always hash the suspicious files first — if the hashes match, the message lives in metadata, not data.
- **Linux `unzip` strips NTFS extra fields.** Working on the unzipped files gave us zero trace of the stego. Whenever a Windows-generated ZIP is involved in a forensics challenge, parse the **original archive** with `python3 -c "import zipfile; ..."` instead of relying on extracted files.
- **ZIP extra field ID `0x000a` is the NTFS record.** Its 32-byte payload is a `$STANDARD_INFORMATION` attribute, and the first 8 bytes are the FILETIME `LastWriteTime`. The lowest 2 bytes are arbitrary and a natural stego target.
- **Mind the byte order.** FILETIME is little-endian, but the stego in this challenge swapped the two low bytes — `(mtime >> 8) & 0xff` is the *first* character, `mtime & 0xff` is the *second*. Print the raw bytes first, then match against the expected flag prefix (`pi`) to confirm the ordering before you commit.
- **The flag is a pun.** `M4cTim35!` = "MacTimes!" in leet (`4`→`a`, `3`→`e`, `5`→`s`). The challenge title `B1g_Mac` is "Big Mac" in leet (`1`→`i`) — the author was always telling you the answer.
