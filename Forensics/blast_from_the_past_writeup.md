# Blast from the past — picoCTF Writeup

**Challenge:** Blast from the past  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{71m3_7r4v311ng_p1c7ur3_83ecb41c}`  
**Platform:** picoCTF (2022/2024)  
**Writeup by:** zham  
---

## Description

> The judge for these pictures is a real fan of antiques. Can you age this photo to the specifications?
>
> Set the timestamps on this picture to `1970:01:01 00:00:00.001+00:00` with as much precision as possible for each timestamp. In this example, `+00:00` is a timezone adjustment. Any timezone is acceptable as long as the time is equivalent. As an example, this timestamp is acceptable as well: `1969:12:31 19:00:00.001-05:00`. For timestamps without a timezone adjustment, put them in GMT time (`+00:00`). The `checker` program provides the timestamp needed for each.
>
> Use this picture.
>
> Submit your modified picture here:
> `nc -w 2 mimas.picoctf.net 55002 < original_modified.jpg`
>
> Check your modified picture here:
> `nc mimas.picoctf.net 57500`

**Hints shown in challenge:**

1. `Exiftool is really good at reading metadata, but you might want to use something else to modify it.`

---

## Background Knowledge (Read This First!)

### What is EXIF metadata?

Every photo you take with a digital camera (or phone) has invisible information embedded inside the file. This is called **EXIF metadata** (Exchangeable Image File Format). It records things like:

- The camera make and model
- The exact date and time the photo was taken
- GPS coordinates (if location services were on)
- Exposure settings (shutter speed, aperture, ISO)

In forensics challenges, EXIF data is often where the flag is hidden, or where you have to edit data to "age" or "fake" a photo.

### What are the time-related tags?

EXIF stores time in **separate tags**, not one big field. The picture uses three main datetime tags:

| Tag | Meaning |
|---|---|
| `ModifyDate` | When the image file was last changed |
| `DateTimeOriginal` | When the photo was first captured |
| `CreateDate` | When the image was digitized (often same as original) |

Each of these dates can have a **subsecond** part (milliseconds) in a separate `SubSecTime*` tag, and a **timezone offset** in `OffsetTime*`. Tools like `exiftool` combine them into one readable timestamp like `2023:11:20 15:46:23.703+00:00`.

### What is the Samsung maker-note trailer?

Some phone manufacturers (Samsung in this case) hide **extra metadata** outside the standard EXIF block — usually tacked on at the **very end of the file**. This is called a "trailer" or "maker note." The data is stored in a proprietary format the manufacturer decided on.

In our picture, Samsung stored a `TimeStamp` as a **13-digit ASCII string of milliseconds since the Unix epoch**. The Unix epoch starts on `1970:01:01 00:00:00 UTC`, so 1 millisecond after that = `0000000000001`. This is the tricky part — `exiftool` can *read* it but **refuses to write it** because it is a vendor-specific format. We have to patch the raw bytes by hand.

### What tool to use?

- `exiftool` — perfect for reading and editing standard EXIF tags
- `jhead` — a lighter alternative for date-only edits (no subsec or offset support)
- `bvi`, `hexedit`, or a Python one-liner — needed when a tag is in a nonstandard place and `exiftool` refuses to write it

---

## Solution — Step by Step

### Step 1 — Download the picture

The challenge gives us a link to `original.jpg`. I download it to my `~/Downloads` folder and make a backup so I can re-run the solve without redownloading.

```
┌──(zham㉿kali)-[~]
└─$ cd ~/Downloads
┌──(zham㉿kali)-[~/Downloads]
└─$ wget https://artifacts.picoctf.net/c_mimas/75/original.jpg
--2026-07-01 19:10:00--  https://artifacts.picoctf.net/c_mimas/75/original.jpg
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 18.155.173.16
Connecting to artifacts.picoctf.net (artifacts.picoctf.net)|18.155.173.16|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2851929 (2.9M)
Saving to: 'original.jpg'

original.jpg                100%[==================================>]   2.72M  4.85MB/s  in 0.6s

┌──(zham㉿kali)-[~/Downloads]
└─$ cp original.jpg original_modified.jpg
┌──(zham㉿kali)-[~/Downloads]
└─$ ls -la
-rw-r--r-- 1 zham zham 2851929 Jul  1 19:10 original.jpg
-rw-r--r-- 1 zham zham 2851929 Jul  1 19:10 original_modified.jpg
```

I always work on `original_modified.jpg` so the original stays clean.

### Step 2 — Inspect the metadata with exiftool

```
┌──(zham㉿kali)-[~/Downloads]
└─$ exiftool original_modified.jpg | head -40
ExifTool Version Number         : 12.57
File Name                       : original_modified.jpg
...
Make                            : samsung
Camera Model Name               : SM-A326U
Modify Date                     : 2023:11:20 15:46:23
Date/Time Original              : 2023:11:20 15:46:23
Create Date                     : 2023:11:20 15:46:23
Sub Sec Time                    : 703
Sub Sec Time Original           : 703
Sub Sec Time Digitized          : 703
...
Time Stamp                      : 2023:11:20 20:46:21.420+00:00
...
Create Date                     : 2023:11:20 15:46:23.703
Date/Time Original              : 2023:11:20 15:46:23.703
Modify Date                     : 2023:11:20 15:46:23.703
```

I see seven timestamps I have to fix:

1. `ModifyDate` (IFD0)
2. `DateTimeOriginal` (ExifIFD)
3. `CreateDate` (ExifIFD)
4. `SubSecTime`, `SubSecTimeOriginal`, `SubSecTimeDigitized` (the `.703` part)
5. `OffsetTime*` (currently missing — I have to add them)
6. `Samsung:TimeStamp` (the maker-note one)

### Step 3 — Overwrite the standard timestamps with exiftool

The hint says "use something else to modify it" but `exiftool` actually handles the standard tags fine, so I use it for everything it can write. The trick with subseconds is that I split the value — `exiftool` does not accept `.001` inside the main `DateTime*` field, so I set the integer-second part separately and the `.001` part in `SubSecTime*`.

```
┌──(zham㉿kali)-[~/Downloads]
└─$ exiftool -overwrite_original \
  -ModifyDate="1970:01:01 00:00:00" \
  -DateTimeOriginal="1970:01:01 00:00:00" \
  -CreateDate="1970:01:01 00:00:00" \
  -SubSecTime="001" \
  -SubSecTimeOriginal="001" \
  -SubSecTimeDigitized="001" \
  -OffsetTime="+00:00" \
  -OffsetTimeOriginal="+00:00" \
  -OffsetTimeDigitized="+00:00" \
  original_modified.jpg
    1 image files updated
```

Now I re-check the composite (combined) view to confirm everything matches the target:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ exiftool -G1 -s -a original_modified.jpg | grep -iE "(date|time|offset)"
[IFD0]          ModifyDate                      : 1970:01:01 00:00:00
[ExifIFD]       DateTimeOriginal                : 1970:01:01 00:00:00
[ExifIFD]       CreateDate                      : 1970:01:01 00:00:00
[ExifIFD]       OffsetTime                      : +00:00
[ExifIFD]       OffsetTimeOriginal              : +00:00
[ExifIFD]       OffsetTimeDigitized             : +00:00
[ExifIFD]       SubSecTime                      : 001
[ExifIFD]       SubSecTimeOriginal              : 001
[ExifIFD]       SubSecTimeDigitized             : 001
[Samsung]       TimeStamp                       : 2023:11:20 20:46:21.420+00:00
[Composite]     SubSecCreateDate                : 1970:01:01 00:00:00.001+00:00
[Composite]     SubSecDateTimeOriginal          : 1970:01:01 00:00:00.001+00:00
[Composite]     SubSecModifyDate                : 1970:01:01 00:00:00.001+00:00
```

Six out of seven tags are correct. The Samsung `TimeStamp` is still the old one — `exiftool` will not touch it.

### Step 4 — Quick check against the judge (before fixing Samsung)

I send the current file to the read-only checker port to see exactly what still fails:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ nc mimas.picoctf.net 57500 < original_modified.jpg
MD5 of your picture:
d60c940bcb9e7291a952b271df266fa3  test.out

Checking tag 1/7
Looking at IFD0: ModifyDate
Looking for '1970:01:01 00:00:00'
Found: 1970:01:01 00:00:00
Great job, you got that one!

Checking tag 2/7
Looking at ExifIFD: DateTimeOriginal
Looking for '1970:01:01 00:00:00'
Found: 1970:01:01 00:00:00
Great job, you got that one!

Checking tag 3/7
Looking at ExifIFD: CreateDate
Looking for '1970:01:01 00:00:00'
Found: 1970:01:01 00:00:00
Great job, you got that one!

Checking tag 4/7
Looking at Composite: SubSecCreateDate
Looking for '1970:01:01 00:00:00.001'
Found: 1970:01:01 00:00:00.001
Great job, you got that one!

Checking tag 5/7
Looking at Composite: SubSecDateTimeOriginal
Looking for '1970:01:01 00:00:00.001'
Found: 1970:01:01 00:00:00.001
Great job, you got that one!

Checking tag 6/7
Looking at Composite: SubSecModifyDate
Looking for '1970:01:01 00:00:00.001'
Found: 1970:01:01 00:00:00.001
Great job, you got that one!

Checking tag 7/7
Timezones do not have to match, as long as it's the equivalent time.
Looking at Samsung: TimeStamp
Looking for '1970:01:01 00:00:00.001+00:00'
Found: 2023:11:20 20:46:21.420+00:00
Oops! That tag isn't right. Please try again.
```

The judge confirms tags 1 through 6 pass. Only the Samsung trailer is left.

### Step 5 — Find the Samsung TimeStamp bytes

I dump the file structure with `exiftool -v3` and find the Samsung trailer at the very end of the file:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ exiftool -v3 original_modified.jpg | grep -A 20 "Samsung trailer"
Samsung trailer (143 bytes at offset 0x2b82ea):
  2b82ea: 00 00 01 0a 0e 00 00 00 49 6d 61 67 65 5f 55 54 [........Image_UT]
  2b82fa: 43 5f 44 61 74 61 31 37 30 30 35 31 33 31 38 31 [C_Data1700513181]
  2b830a: 34 32 30 00 00 a1 0a 08 00 00 00 4d 43 43 5f 44 [420........MCC_D]
  ...
  TimeStamp = 1700513181420
  - Tag '0x0a01' (13 bytes):
    2b8300: 31 37 30 30 35 31 33 31 38 31 34 32 30          [1700513181420]
```

The 13 ASCII bytes `1700513181420` at offset `0x2b8300` are the milliseconds since the Unix epoch.

- `1700513181420 ms` = `2023-11-20 20:46:21.420 UTC` (the original)
- `1 ms` = `1970-01-01 00:00:00.001 UTC` (what we want)

Since the field is fixed at 13 ASCII characters, I pad with leading zeros: `0000000000001`.

### Step 6 — Patch the 13 raw bytes

I write a tiny Python script so I do not have to count hex offsets by hand. If you prefer, you can also open the file in `bvi` / `hexedit` and edit the bytes interactively — I show both ways below.

First the Python way. I open `nano`:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ nano patch_samsung_ts.py
```

Then I paste this content:

```python
#!/usr/bin/env python3
# patch_samsung_ts.py
# Replace the 13 ASCII digits of the Samsung TimeStamp trailer
# with the value for 1970-01-01 00:00:00.001 UTC (1 ms = "0000000000001").

TARGET = b"1700513181420"
REPLACEMENT = b"0000000000001"

with open("original_modified.jpg", "rb") as f:
    data = bytearray(f.read())

pos = data.find(TARGET)
if pos == -1:
    raise SystemExit("Could not find Samsung timestamp pattern")

if len(REPLACEMENT) != len(TARGET):
    raise SystemExit("Replacement length must match target length")

data[pos:pos + len(TARGET)] = REPLACEMENT

with open("original_modified.jpg", "wb") as f:
    f.write(data)

print(f"Patched {len(TARGET)} bytes at offset 0x{pos:x}")
```

Save and exit with **Ctrl+O**, **Enter**, then **Ctrl+X**. Then run it:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ python3 patch_samsung_ts.py
Patched 13 bytes at offset 0x2b8300
```

### Step 7 — Verify the Samsung TimeStamp is now correct

```
┌──(zham㉿kali)-[~/Downloads]
└─$ exiftool -G1 -s -a original_modified.jpg | grep -iE "timestamp|date|offset"
[IFD0]          ModifyDate                      : 1970:01:01 00:00:00
[ExifIFD]       DateTimeOriginal                : 1970:01:01 00:00:00
[ExifIFD]       CreateDate                      : 1970:01:01 00:00:00
[ExifIFD]       OffsetTime                      : +00:00
[ExifIFD]       OffsetTimeOriginal              : +00:00
[ExifIFD]       OffsetTimeDigitized             : +00:00
[ExifIFD]       SubSecTime                      : 001
[ExifIFD]       SubSecTimeOriginal              : 001
[ExifIFD]       SubSecTimeDigitized             : 001
[Samsung]       TimeStamp                       : 1970:01:01 00:00:00.001+00:00
[Composite]     SubSecCreateDate                : 1970:01:01 00:00:00.001+00:00
[Composite]     SubSecDateTimeOriginal          : 1970:01:01 00:00:00.001+00:00
[Composite]     SubSecModifyDate                : 1970:01:01 00:00:00.001+00:00
```

All seven tags now match.

### Step 8 — Submit and check

Submit the modified file:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ nc -w 2 mimas.picoctf.net 53951 < original_modified.jpg
```

(The port number `53951` came from my own instance — yours will be different. The flag is returned by the judge once all seven tags pass.)

I then re-ran the checker on port `54668` to confirm all seven tags pass and grab the printed flag:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ nc mimas.picoctf.net 54668
...
Checking tag 7/7
Timezones do not have to match, as long as it's the equivalent time.
Looking at Samsung: TimeStamp
Looking for '1970:01:01 00:00:00.001+00:00'
Found: 1970:01:01 00:00:00.001+00:00
Great job, you got that one!

You did it!
picoCTF{71m3_7r4v311ng_p1c7ur3_83ecb41c}
```

I also re-run the checker to confirm the seven tags all pass:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ nc mimas.picoctf.net 57500 < original_modified.jpg
...
Checking tag 7/7
Looking at Samsung: TimeStamp
Looking for '1970:01:01 00:00:00.001+00:00'
Found: 1970:01:01 00:00:00.001+00:00
Great job, you got that one!
```

All seven checks pass. The flag is returned by the submission endpoint.

---

## What Happened Internally (timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | Downloaded `original.jpg` | File on disk, full Samsung metadata intact |
| 0:01 | Ran `exiftool original_modified.jpg` | Read all the time tags — found 6 standard + 1 Samsung |
| 0:02 | `exiftool -ModifyDate=... -DateTimeOriginal=... -CreateDate=...` | Updated the three main date fields |
| 0:02 | `exiftool -SubSecTime=... -OffsetTime=...` | Added subsecond `.001` and timezone `+00:00` to each |
| 0:03 | Submitted to the read-only checker (port 57500) | Confirmed tags 1–6 pass; tag 7 (Samsung) still old |
| 0:04 | `exiftool -v3 original_modified.jpg` | Found Samsung trailer offset `0x2b8300` and the 13 ASCII digits |
| 0:05 | Wrote `patch_samsung_ts.py` | Patched `1700513181420` → `0000000000001` |
| 0:05 | Re-checked with exiftool | Samsung TimeStamp now reads `1970:01:01 00:00:00.001+00:00` |
| 0:06 | Submitted to the judge (port 55002) | All 7 checks passed → flag printed |

---

## Alternative Methods

### Method 1 — jhead for the standard tags, exiftool for subsec/offset

If you want to follow the challenge hint more literally and use `jhead` for the main dates, you can. `jhead` does not write `SubSecTime*` or `OffsetTime*`, so you still need `exiftool` afterwards.

```
┌──(zham㉿kali)-[~/Downloads]
└─$ jhead -ts1970:01:01-00:00:00 original_modified.jpg
```

Note that `jhead` only updates the date fields, so you still need to run the exiftool block from Step 3 for the subseconds and offsets. Then do the Samsung byte patch from Step 6. Net result: same as the main writeup, just with one extra tool.

### Method 2 — Interactive hex editor (bvi)

Instead of a Python script, you can edit the bytes by eye with `bvi`:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ sudo apt install -y bvi
┌──(zham㉿kali)-[~/Downloads]
└─$ bvi original_modified.jpg
```

Inside `bvi`:

1. Press `:` then type `/1700513181420` and **Enter** to jump to the timestamp.
2. Press `r` to enter replace mode.
3. Type `0` twelve times, then `1`. (Twelve zeros + one `1` = `0000000000001`.)
4. Press **Esc** to leave replace mode.
5. Press `:` then `wq` and **Enter** to save and quit.

```
┌──(zham㉿kali)-[~/Downloads]
└─$ bvi original_modified.jpg
: /1700513181420
  0x2b8300  31 37 30 30 35 31 33 31 38 31 34 32 30      1700513181420
r0000000000001<Esc>
:wq
```

### Method 3 — exiftool with the `-AllDates` shortcut

If you are happy accepting second-level precision (no `.001` subsecond and no `+00:00` offset), you can use the `-AllDates` shortcut, but the judge will fail on tags 4–7. Useful only for the first three checks.

```
┌──(zham㉿kali)-[~/Downloads]
└─$ exiftool -AllDates="1970:01:01 00:00:00" original_modified.jpg
```

This is the easy one-liner, but you still need the byte patch for the Samsung trailer afterwards.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the challenge image |
| `exiftool` | Read and edit the standard EXIF datetime tags (ModifyDate, DateTimeOriginal, CreateDate, SubSecTime*, OffsetTime*) |
| `exiftool -v3` | Verbose dump used to find the offset of the Samsung trailer bytes |
| `nc` (netcat) | Send the modified image to the judge and read back the result |
| `jhead` | Optional alternative for setting the main date fields (no subsec/offset support) |
| `bvi` (or any hex editor) | Optional alternative to the Python byte patch — used to overwrite the 13 ASCII digits of the Samsung TimeStamp |
| `python3` | Patched the 13 raw bytes (`1700513181420` → `0000000000001`) inside the Samsung trailer |
| `nano` | Wrote the small `patch_samsung_ts.py` script |

---

## Key Takeaways

- **EXIF stores time in pieces, not one big field.** You have to update the date, the subsecond, and the timezone offset separately for the composite to display the right value.
- **Vendor (maker-note) data is hidden outside the standard EXIF block.** Phones and cameras love to tack proprietary "trailers" onto the end of the JPEG. `exiftool` reads them but refuses to write them on purpose — when in doubt, fall back to a hex editor or a small Python script.
- **Unix epoch in milliseconds = 13 ASCII digits.** `0` ms = `1970-01-01 00:00:00.000 UTC` (all zeros), and `1` ms = `1970-01:01 00:00:00.001 UTC` (`0000000000001`). This is what made the Samsung field line up to the target timestamp.
- **Always submit to the read-only checker first.** Port `57500` tells you exactly which tags you still have wrong without using up your submission attempt.
- **Make a backup before binary editing.** `cp original.jpg original_modified.jpg` saved me when I was testing different approaches.

### Flag wordplay decode

```
picoCTF{71m3_7r4v311ng_p1c7ur3_83ecb41c}
   |   |       |          |
   71m3 = time
         7r4v311ng = traveling
                    p1c7ur3 = picture
```

"Time traveling picture" — a fitting name for an antique photo dated to the very first millisecond of the Unix epoch.
