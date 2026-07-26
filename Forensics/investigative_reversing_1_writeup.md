# Investigative Reversing 1 — picoCTF Writeup

**Challenge:** Investigative Reversing 1  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 350  
**Flag:** `picoCTF{An0tha_1_68311242}`  
**Platform:** picoCTF 2019  
**Author:** Danny Tunitz  
**Writeup by:** zham

---

## Description

> We have recovered a binary and a few images: image, image2, image3. See what you can make of it. There should be a flag somewhere.

**Attachments:** `mystery` (ELF binary) + `mystery.png`, `mystery2.png`, `mystery3.png` (three PNG images)

---

## Hints

> 1. Try using some forensics skills on the image
> 2. This problem requires both forensics and reversing skills
> 3. A hex editor may be helpful

---

## Background Knowledge (Read This First!)

### Where PNG files can hide data

A PNG file is a sequence of "chunks" — each chunk has a 4-byte length, 4-byte type, payload, and 4-byte CRC. The PNG specification ends with an `IEND` chunk (length 0, type `IEND`, no payload, 4-byte CRC). After that CRC, parsers stop reading — but the file can still have any number of trailing bytes appended, and most image viewers will just ignore them. This is the classic place to hide "extra" data in a PNG.

```
PNG file:
  [PNG signature: 8 bytes]
  [IHDR chunk]
  [IDAT chunk(s) — actual image data]
  [IEND chunk]
  [CRC of IEND: 4 bytes]
  <-- everything past here is appended junk, invisible to image parsers -->
```

### What the binary does

The `mystery` binary reads 26 bytes from `flag.txt` and scatters those bytes across the three PNG files, **appending** them to the file's tail (past the IEND+CRC). It opens each PNG in `"a"` (append) mode, not `"r"`, so the writes add new bytes without touching the image data. Two of the bytes are modified before writing: byte 0 is increased by 0x15, byte 3 is increased by 4.

The byte-to-file mapping (extracted from the disassembly):

| File           | Indices  | Transform               |
|----------------|----------|-------------------------|
| `mystery3.png` | 1, 2, 5, 10..14 | none             |
| `mystery2.png` | 0, 3     | +0x15 on byte 0, +4 on byte 3 |
| `mystery.png`  | 4, 6..9, 15..25 | none              |

(`mystery.png` here = the user's `mystery.png`; the 26-byte flag is split as 11 bytes + 2 bytes + 8 bytes + 5 zeros in the image data = 26, but the file mapping above is what the binary does.)

---

## Solution — Step by Step

### Step 1 — Verify the appended data

```
┌──(zham㉿kali)-[/media/sf_downloads/inv_rev1]
└─$ xxd mystery.png | tail -3
0001e800: 0000 0000 4945 4e44  ae42 6082              ....IEND.B`.

$ python3 -c "d=open('mystery.png','rb').read(); idx=d.find(b'IEND'); print('bytes after IEND+CRC:', d[idx+8:].hex())"
bytes after IEND+CRC: 43467b416e315f36383331313234327d
```

Each PNG has 16, 2, and 8 bytes appended after the IEND+CRC. That's the flag data, broken up by the binary.

### Step 2 — Reverse the binary

The `main` function in Ghidra (or `objdump -d mystery` if you don't have Ghidra) does this:

```c
fread(local_38, 0x1a, 1, flag_file);   // read 26 bytes of flag

fputc(local_38[1], mystery3);            // flag[1] -> mystery3[0]
fputc(local_38[0] + 0x15, mystery2);      // flag[0] + 0x15 -> mystery2[0]
fputc(local_38[2], mystery3);            // flag[2] -> mystery3[1]
local_6b = local_38[3];
fputc(local_33, mystery3);               // flag[5] -> mystery3[2]
fputc(local_34, mystery);                // flag[4] -> mystery[0]
// loop: local_6b++; for j = 6..9: fputc(flag[j], mystery)
//   so flag[6..9] -> mystery[1..4]
//   and (flag[3] + 4) -> mystery2[1]
// loop: for j = 10..14: fputc(flag[j], mystery3)
//   so flag[10..14] -> mystery3[3..7]
// loop: for j = 15..25: fputc(flag[j], mystery)
//   so flag[15..25] -> mystery[5..15]
```

Note the use of `local_33` and `local_34`: these are stack-slot aliases for the 4th and 5th characters of `local_38` (the 26-byte flag buffer), which is why the 5th character gets written to `mystery3` before the 4th goes to `mystery` even though logically `flag[5]` and `flag[4]` should be written in order.

### Step 3 — Reverse the writes

The script just walks the three appended tails in the order the binary wrote them and inverts the two arithmetic offsets:

```python
import os, mmap

def get_tail(filename):
    size = os.path.getsize(filename)
    fd = os.open(filename, os.O_RDONLY)
    m = mmap.mmap(fd, size, access=mmap.ACCESS_READ)
    start = m.find(b"IEND") + 4 + 4   # past IEND + CRC
    tail = m[start:]
    m.close(); os.close(fd)
    return tail

flag = [0] * 26
t1, t2, t3 = get_tail("mystery.png"), get_tail("mystery2.png"), get_tail("mystery3.png")

# mystery3: indices 1, 2, 5, 10..14
flag[1]  = t3[0]
flag[2]  = t3[1]
flag[5]  = t3[2]
for i, idx in enumerate(range(10, 15)):
    flag[idx] = t3[3 + i]

# mystery2: indices 0, 3 (with arithmetic reversal)
flag[0] = t2[0] - 0x15
flag[3] = t2[1] - 4

# mystery: indices 4, 6..9, 15..25
flag[4] = t1[0]
for i, idx in enumerate(range(6, 10)):
    flag[idx] = t1[1 + i]
for i, idx in enumerate(range(15, 26)):
    flag[idx] = t1[5 + i]

print("".join(chr(b) for b in flag))
```

```
┌──(zham㉿kali)-[/media/sf_downloads/inv_rev1]
└─$ python3 decode_investigative_reversing_1.py
mystery.png  tail (16 bytes): 43467b416e315f36383331313234327d
mystery2.png tail (2 bytes):  8573
mystery3.png tail (8 bytes):  696354307468615f

Decoded flag: picoCTF{An0tha_1_68311242}
```

The recovered flag is `picoCTF{An0tha_1_68311242}`.

---

## What Happened Internally (Timeline)

1. The challenge author wrote a C program that takes a 26-byte flag and scatters it across 3 PNG files, appending the scattered bytes after each PNG's IEND+CRC marker so the images still display normally.
2. Two of the bytes are offset before writing (byte 0 + 0x15, byte 3 + 4) so a naive "just concatenate the tails" approach would give you the wrong flag.
3. The author packaged the binary and the three already-modified PNGs as the challenge files. The PNGs *are* valid images — the data you want is hidden in the trailing bytes that no image viewer ever shows.
4. You, the solver, either decompile the binary in Ghidra to learn the exact index-to-file mapping and the two offsets, or use the writeup community's mapping, then run a small script to extract, decode, and re-assemble the 26 bytes back into the original flag.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `xxd` / `file` | Inspect the PNG tails and find the appended data | Easy |
| `Ghidra` (or `objdump` / `r2`) | Decompile the `mystery` binary to find the byte-to-file mapping and the +0x15 / +4 offsets | Medium |
| `python3` | Extract the tails, undo the offsets, re-assemble the 26-byte flag | Easy |

---

## Key Takeaways

- **The trailing bytes of a PNG are free real estate.** Anything past the IEND chunk's 4-byte CRC is invisible to image parsers. This is the most common place for "appended" data in PNG forensics challenges. The standard tool to detect it: `xxd <file> | tail` and look for bytes after the IEND+CRC pair.
- **The two offsets (+0x15 and +4) are a deliberate misdirection.** Without Ghidra you'd concatenate the tails and get a garbage flag (the `i` and the `O` get shifted by 0x15 and 4 respectively, munging every byte that follows). The challenge is forcing you to disassemble the binary — you can't solve it by *just* looking at the file tail.
- **`local_33` / `local_34` are Ghidra artefacts.** Ghidra maps stack-allocated buffers to `local_NN` based on their byte offset from the frame pointer, not their logical index in the array. The binary's `local_38[4]` (the 4th flag byte) becomes `local_34`, and `local_38[5]` (the 5th flag byte) becomes `local_33`. The code reads `local_33` before `local_34` for the two `fputc` calls even though they should be written in order — that's a quirk of how the compiler reuses stack slots, not a logical reordering.
- **The flag wordplay is `An0tha 1` in leet ("another one").** Investigative Reversing 0/1/2/3/4 are all the same shape: binary + a few cover files, flag scattered across the cover files. The flag celebrates the fact that this is just one of many similar challenges in the picoCTF 2019 series.
