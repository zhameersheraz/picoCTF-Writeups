# Investigative Reversing 0 — picoCTF Writeup

**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 300  
**Platform:** picoCTF 2019  
**Author:** Danny Tunitis  
**Flag:** `picoCTF{f0und_1t_dbf6ab4d}`  
**Writeup by:** zham

---

> We have recovered a binary and an image. See what you can make of it. There should be a flag somewhere.

> Hints
> 1. Try using some forensics skills on the image
> 2. This problem requires both forensics and reversing skills
> 3. A hex editor may be helpful

---

## Background Knowledge

This is the **first** challenge in the picoCTF "Investigative Reversing" series. The whole idea: you get a binary and a PNG, you have to figure out what the binary does to the PNG, and recover what got appended/embedded. The same shape repeats in challenges 2, 3, 4 — but challenge 0 is the simplest, and the giveaway is the literal string `"at insert"` inside the binary, which screams "I'm putting data into a file."

Tools I used:
- `capstone` (Python) to disassemble the 64-bit ELF.
- `Pillow` + raw `bytes` to check the PNG structure (chunks, IEND position).
- A small Python extractor that finds the IEND marker, then pulls the trailing bytes and reverses the per-index transform.
- `exiftool` would also work to spot "trailing data after PNG IEND" — that's a classic image-forensics tell.

---

## Step 1: Look at the files

The challenge gives you two files: `mystery` (16,752-byte ELF) and `mystery.png` (a PNG of the challenge page). Nothing exotic on the surface.

```bash
$ file mystery
mystery: ELF 64-bit LSB pie executable, x86-64, ...

$ file mystery.png
mystery.png: PNG image data, 853 x 680, 8-bit/color RGB, non-interlaced
```

The binary is **not stripped** (you can see `main` and `__libc_csu_init` symbols), which makes the reversing step trivial.

`strings mystery` gives the file-mode and filename hints:

```
flag.txt
a
mystery.png
No flag found, please make sure this is run on the server
mystery.png is missing, please run this on the server
at insert
```

Two things jump out:
- `mystery.png` is opened in mode `"a"` → **append**, not overwrite. The binary is going to write the flag *to the end* of the PNG, not replace any pixels.
- `"at insert"` is a debug printf. It's literally the encoder saying "I am about to write a byte."

---

## Step 2: Disassemble the binary

The whole binary is `main` + libc startup. Disassembling `main` at `0x1195` (509 bytes) gives a tiny straight-line program. The pseudocode:

```c
int main(void) {
    FILE *flag = fopen("flag.txt", "r");
    FILE *img  = fopen("mystery.png", "a");
    if (!flag) puts("No flag found, please make sure this is run on the server");
    if (!img)  puts("mystery.png is missing, please run this on the server");

    char flag_buf[26];
    if (fread(flag_buf, 26, 1, flag) <= 0) { exit(0); }

    puts("at insert");

    // bytes 0-5: as-is
    for (int i = 0; i <= 5; i++)
        fputc(flag_buf[i], img);

    // bytes 6-14: +5
    for (int i = 6; i <= 14; i++)
        fputc(flag_buf[i] + 5, img);

    // byte 15: -3
    fputc(flag_buf[15] - 3, img);

    // bytes 16-25: as-is
    for (int i = 16; i <= 25; i++)
        fputc(flag_buf[i], img);

    fclose(img);
    fclose(flag);
    return 0;
}
```

So `mystery` reads **26 bytes** of the flag and writes 26 transformed bytes to the **end** of `mystery.png` (after the PNG `IEND` chunk). The transformations are positional:

| Byte index (0-indexed) | Transform |
|------------------------|-----------|
| 0 – 5                  | as-is     |
| 6 – 14                 | `+ 5`     |
| 15                     | `− 3`     |
| 16 – 25                | as-is     |

This is the encoding. To decode, we read the **last 26 bytes** of `mystery.png` and apply the **inverse** (`-5` for indices 6-14, `+3` for index 15, the rest as-is).

---

## Step 3: Get the encoded mystery.png

The challenge ships the original `mystery.png` (74,794 bytes, no flag). On a normal picoCTF instance you'd SSH into the shell server and run `./mystery` — the binary reads the real `flag.txt` from the server and appends the encoded flag. The result is `mystery.png` grown to **74,820 bytes**, with `exiftool` (or any tool that reads past `IEND`) showing the trailing 26 bytes as the flag.

If you're working offline (like me on this custom platform), you can:
1. Run the binary yourself with a fake 26-byte `flag.txt` to verify the round-trip, **or**
2. Skip the run entirely and just write the extractor.

The first option is what I did, and it works perfectly (see the next section).

```bash
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cp inv0_image.png mystery.png
└─$ echo -n 'picoCTF{verify_inv0_works_ok!!}' > flag.txt   # 31 chars
└─$ wc -c flag.txt
31 flag.txt
└─$ ./mystery
at insert
└─$ ls -la mystery.png
# grew by 26 bytes
└─$ python3 -c "
raw = open('mystery.png', 'rb').read()[-26:]
out = list(raw[:6])
for i in range(6, 15): out.append((raw[i] - 5) & 0xFF)
out.append((raw[15] + 3) & 0xFF)
for i in range(16, 26): out.append(raw[i])
print(bytes(out).decode())
"
picoCTF{verify_inv0_works_
```

The extractor returned `picoCTF{verify_inv0_works_` — the first 26 chars of the fake flag. The last 5 chars (`ok!!}`) didn't get processed because the binary only reads 26 bytes from `flag.txt`. The reverse-transform is correct.

---

## Step 4: Extract the real flag

Once you have the real `mystery.png` with the encoded flag appended, run the extractor. The script I used (`decode_inv0_blocks.py`) also walks back through every prior 26-byte block in the file (because each `mystery` run appends another 26 bytes — running the encoder N times produces N×26 trailing bytes, all of which can be decoded the same way):

```python
import sys

def reverse_block(b):
    out = list(b[:6])
    for i in range(6, 15): out.append((b[i] - 5) & 0xFF)
    out.append((b[15] + 3) & 0xFF)
    for i in range(16, 26): out.append(b[i])
    return bytes(out)

def find_iend_end(data):
    idx = data.rfind(b'IEND')
    return idx + 4 + 4  # type + CRC

with open(sys.argv[1] if len(sys.argv) > 1 else 'mystery.png', 'rb') as f:
    data = f.read()

end = find_iend_end(data)
trailer = data[end:]
n_blocks = len(trailer) // 26
print(f'IEND ends at byte {end}, trailing {len(trailer)} bytes, {n_blocks} block(s)')

for b in range(n_blocks):
    block = trailer[b*26:(b+1)*26]
    decoded = reverse_block(block)
    print(f'Block {b}: {decoded}')
```

Running this on the instance's `mystery.png` (125,095 bytes — i.e. 1 real-flag run + 1 of my verification runs + a few stragglers):

```
IEND ends at byte 125043, trailing 52 bytes, 2 block(s)
Block 0: b'picoCTF{f0und_1t_dbf6ab4d}'
Block 1: b'picoCTF{verify_inv0_works_'
```

**Block 0 is the real flag: `picoCTF{f0und_1t_dbf6ab4d}`.**

Block 1 is the verification run I just did (the first 26 chars of `picoCTF{verify_inv0_works_ok!!}`).

---

## What Happened Internally

A short timeline of the encoder:

1. **Open files.** `flag.txt` for read, `mystery.png` for append. Both file pointers are checked against `NULL` and a human-readable error is printed if either is missing — but the program continues anyway and `fread`/`fputc` on a `NULL` would just crash, so in practice you need both files present.

2. **Read 26 bytes from `flag.txt` into a stack buffer.** `fread(flag_buf, 26, 1, flag)` — one element of 26 bytes. If the read fails (`<= 0`), the binary calls `exit(0)` silently. That's why a too-short `flag.txt` would print `"Invalid Flag"` and do nothing — wait, no, the binary here just exits without printing. The `"Invalid Flag"` string from the previous (Investigative Reversing 3) challenge isn't in this binary.

3. **Print `"at insert"`.** A debug printf. Useful for confirming the encoder is actually running on the server.

4. **Loop 1: write bytes 0-5 as-is.** Six unmodified characters. For a flag starting with `picoCTF{`, this is `picoCTF`.

5. **Loop 2: write bytes 6-14 with `+5`.** Nine characters shifted up by 5. The shift hides these bytes in `exiftool` and `strings` output (because most of them become non-printable high bytes after the shift).

6. **Write byte 15 with `-3`.** One character shifted down by 3. The `}` in a typical 27-char flag would land at index 26 — so this `-3` operation doesn't usually affect the closing brace. It's just one more byte of obfuscation.

7. **Loop 3: write bytes 16-25 as-is.** The last 10 characters (whatever they are) are written unmodified. If the flag is 27 chars, only the first 26 are processed and the `}` never makes it into the encoded output — the encoder just silently drops the closing brace. (The 26-char flag in this challenge fits exactly.)

8. **Close files and exit.** The encoded 26 bytes are now sitting past the `IEND` marker in `mystery.png`. Any tool that reads beyond `IEND` (like `exiftool`, `binwalk`, or a custom Python script) will see them; any tool that stops at `IEND` (most image viewers) will ignore them.

The decoder reverses this exactly: it locates `IEND`, slices off everything after it, splits that trailer into 26-byte blocks, and applies the per-index inverse transforms. As long as the flag is at least 26 bytes, you'll get the right answer; for longer flags, you'd need to adjust the loop ranges in the disassembly.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file`, `strings` | Confirm ELF type and find file-mode/filename hints |
| `capstone` (Python) | Disassemble `mystery`'s `main` (509 bytes, trivial) |
| `Pillow` + raw `bytes` | Parse the PNG chunks, locate `IEND` |
| `python3` (one-liner) | Inverse-transform the trailing 26 bytes |
| `exiftool` (would also work) | Flag `Warning: There is trailing data after the PNG IEND chunk` as the tell |
| Custom Python script | `decode_inv0_blocks.py` — handles multiple 26-byte blocks if the file has been run on the encoder more than once |

---

## Key Takeaways

- **"Mode a" is the giveaway.** Whenever a CTF binary opens an image in append mode, the flag is almost always at the end of the file, past the `IEND` chunk. `exiftool` will literally warn you about it. The hint "try using some forensics skills on the image" is a nudge to use `exiftool` or `binwalk` on the PNG.
- **Position-aware encoding.** The per-index `+5` / `−3` shifts are a one-byte obfuscation; the byte position is the key, not the value. A 5-line reverse is enough to recover the flag once you know the offsets.
- **26 bytes is the contract.** The binary reads exactly 26 bytes of the flag. picoCTF flags for this challenge are 26 chars including the closing brace. The encoder doesn't write a newline or terminator, so the raw 26 bytes after `IEND` *are* the entire encoded flag.
- **Wordplay decode:** `f0und_1t` = **"found it"** in leet (`0` → `o`). The flag is telling you that you found the flag — appropriate for a challenge where the meat is just "notice the file is 26 bytes longer, reverse the math."
- **Run the encoder locally to verify.** When you can't access the server, you can still prove your extractor works by running `./mystery` on a 26-byte fake `flag.txt`. If your round-trip recovers the fake flag, the real flag will decode the same way.
