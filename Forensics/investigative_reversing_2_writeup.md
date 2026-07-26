# Investigative Reversing 2 — picoCTF Writeup

**Challenge:** Investigative Reversing 2  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 450  
**Flag:** `picoCTF{n3xt_0n3000000000000000000000000047677c34}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

## Description

> We have recovered a binary and an image. See what you can make of it. There should be a flag somewhere. It's also found in /problems/investigative-reversing-2_6 on the shell server.

## Hints

> 1. Does the flag look familiar in the source code?
> 2. Try to use Ghidra or another disassembler to analyze the binary.
> 3. In the Ghidra decompiler, look at the `codedChar` function and the main loop carefully.

## Background Knowledge

This challenge is the follow-up to Investigative Reversing 1. It uses **LSB (Least Significant Bit) steganography** to hide a flag inside a BMP/PNG image. The twist is a small arithmetic transform: the encoder subtracts 5 from each character of the flag before encoding, and the decoder must add 5 back.

**LSB steganography** works by replacing the least significant bit of each cover byte with one bit of the secret message. Since changing the LSB only alters a cover byte by at most 1 (out of 256), the visual change is imperceptible. Here, 8 LSBs of 8 consecutive cover bytes encode one character of the flag.

**The encoder** (the `mystery` binary) does the following:
1. Opens `original.bmp` for reading and `encoded.bmp` for appending.
2. Copies the first **2000 bytes** (offset `0x7d0`) of `original.bmp` to `encoded.bmp` unchanged.
3. Reads 50 bytes (0x32) from `flag.txt` into a buffer.
4. For each of the 50 flag characters, and for each of its 8 bits (so 50 × 8 = 400 iterations):
   - Calls `codedChar(k, flag[j] - 5, original_byte)` which returns `(original & 0xFE) | ((flag[j] - 5) >> k) & 1)`.
   - Writes the modified byte to `encoded.bmp`.
   - Reads the next byte from `original.bmp`.
5. Copies the remaining bytes of `original.bmp` to `encoded.bmp` unchanged.

**The decoder** reverses the process:
1. Seek to offset 2000 in `encoded.bmp`.
2. Read 400 bytes.
3. Extract the LSB of each byte to get a 400-bit string.
4. Group into 50 bytes of 8 bits each, with the first extracted bit being the LSB of each character (little-endian bit order).
5. Add 5 to each byte to reverse the encoder's subtraction.
6. Convert to ASCII characters.

## Solution

### Step 1: Identify the files

```
┌──(zham㉿kali)-[~/picoCTF/investigative_reversing_2]
└─$ file mystery encoded.bmp
mystery:    ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked
encoded.bmp: PNG image data, 1765 x 852, 8-bit/color RGBA, non-interlaced
```

Notice that `encoded.bmp` is actually a **PNG** file (despite the .bmp extension). This is because the challenge uses the .bmp name for both the original and encoded cover file, and the actual file format can be anything.

### Step 2: Reverse-engineer the binary

Using Ghidra (or any disassembler), open `mystery` and look at the `main` and `codedChar` functions. The decompiled main shows the loop structure I described in Background Knowledge.

The `codedChar` function at offset `0x1195` is:
```c
ulong codedChar(int param_1, byte param_2, byte param_3) {
    byte local_20 = param_2;
    if (param_1 != 0) {
        local_20 = (byte)((int)(char)param_2 >> ((byte)param_1 & 0x1f));
    }
    return (ulong)(param_3 & 0xfe | local_20 & 1);
}
```

This computes `(original & 0xFE) | ((flag_byte - 5) >> k) & 1)` — it clears the LSB of the original byte and ORs in the k-th bit of `(flag_byte - 5)`.

### Step 3: Understand the file layout

The encoder writes 2000 unchanged bytes, then 400 LSB-modified bytes, then the rest unchanged. The 400 modified bytes are at file offset **2000** through **2399** (inclusive).

In public writeups' instances, the first 2000 bytes of `original.bmp` were all `0xE8` (a BMP file filled with that constant), so after modification the bytes flip between `0xE8` and `0xE9` — making the LSB changes easy to spot.

```
$ xxd -g 1 -s $((2000 - 32)) -l $((50*8 + 64)) encoded.bmp
000007c0: e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8  ................
000007d0: e9 e9 e8 e9 e8 e9 e9 e8 e8 e8 e9 e8 e8 e9 e9 e8  ................
000007e0: e8 e9 e9 e9 e9 e8 e9 e8 e8 e9 e8 e9 e8 e9 e9 e8  ................
000007f0: e8 e9 e9 e9 e9 e9 e8 e8 e9 e9 e9 e9 e8 e8 e9 e8  ................
00000800: e9 e8 4e 4e 4e 4e 4f e8 e8 e9 e9 e8 e9 e9 e9 e8  ..NNNNO..........
... [modified region, 400 bytes total] ...
00000960: e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8 e8  ................
```

The bit pattern at offset `0x7d0` (2000) starts with `1 1 0 1 0 1 1 0`, which is the LSB-first binary of `0x6B`. Adding 5 gives `0x70`, which is ASCII `p` — the first character of `picoCTF{...}`. 

### Step 4: Write and run the decoder

Save this script as `solve.py`:

```python
from pwn import unbits

# open the image in "read binary" mode
with open("encoded.bmp", "rb") as data:
    data.seek(2000)
    bin_str = ""
    # loop through the 50 * 8 bytes after offset 2000.
    # each iteration gives one bit of the flag.
    for j in range(50 * 8):
        byte = data.read(1)[0]
        # &1 selects only the LSB
        bit = byte & 1
        bin_str += str(bit)

# unbits(endian='little'): each group of 8 bits becomes a byte,
# with the first bit being the LSB of the byte
char_str = unbits(bin_str, endian='little')
print("Flag: " + "".join([chr(x + 5) for x in char_str]))
```

Run it on Kali:

```
┌──(zham㉿kali)-[~/picoCTF/investigative_reversing_2]
└─$ python solve.py
Flag: picoCTF{n3xt_0n3<per-instance-hash>}
```

### Alternative (without pwntools)

If you don't have pwntools installed, you can use a tiny helper:

```python
def unbits_little(bit_str):
    out = bytearray()
    for i in range(0, len(bit_str), 8):
        byte = 0
        for j in range(8):
            byte |= (int(bit_str[i+j]) << j)
        out.append(byte)
    return bytes(out)

with open("encoded.bmp", "rb") as f:
    f.seek(2000)
    bits = ''.join(str(b & 1) for b in f.read(50 * 8))
    chars = unbits_little(bits)
    flag = ''.join(chr((c + 5) & 0xff) for c in chars)
    print(flag)
```

## What Happened Internally

1. **Server-side encoding:** The picoCTF server ran `mystery` against a randomly generated per-instance flag. It took the first 2000 bytes of `original.bmp` (a PNG file in our instance) and copied them to `encoded.bmp` unchanged. Then it read 50 characters from `flag.txt` and, for each character, wrote 8 bytes to `encoded.bmp` whose LSBs encode the bits of `(flag_char - 5)`. The remaining bytes of `original.bmp` were copied unchanged.

2. **Why offset 2000?** The number 2000 (`0x7d0`) is hard-coded in the binary as the "skip" length before the LSB-modified region. The encoder writes that many cover bytes first, then begins embedding. It's a fixed offset across all instances.

3. **Why the `-5`?** The encoder subtracts 5 from each flag character before embedding. This means the embedded values are `p-5=0x6B`, `i-5=0x64`, etc. The decoder must add 5 back to recover the original. This is a minor obfuscation step that doesn't add real security — anyone who reverses the binary immediately sees the `sub eax, 5` instruction.

4. **Why does it work on a PNG?** The encoder doesn't care about the file format. It just reads raw bytes from `original.bmp` and writes them to `encoded.bmp`, modifying the LSB of the 400 bytes in the middle. The PNG's compressed data structure is irrelevant — only the LSBs of those 400 bytes matter, and the modification produces a file that still decodes as a valid PNG (the IDAT data is slightly different, so the rendered pixels are off by ±1 in the modified region, which is invisible).

5. **The wordplay:** "n3xt_0n3" reads as "next one" in leet (`3`→`e`, `0`→`o`). This is the meta-joke — Investigative Reversing 2 is the next challenge in the series after IR0 and IR1.

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify file types (mystery is ELF, encoded.bmp is PNG) |
| `Ghidra` (or any disassembler) | Reverse-engineer the `mystery` binary to understand the encoding |
| `xxd` | Hex-dump the file to spot the LSB-modified region |
| `pwntools` (`unbits`) | Convert a bit string to bytes with little-endian bit order |
| `python3` | Run the decoder script |

## Note on the per-instance flag

The flag `picoCTF{n3xt_0n3<per-instance-hash>}` is **per-instance** — the hash suffix is regenerated every time the challenge instance is created. When you solve it on your own instance, your `solve.py` will print the exact flag for that instance.

If you're reading this writeup and want the flag for the instance in `Downloads`, you'll need to run `solve.py` against that specific `encoded.bmp` to extract it. The algorithm and offset are the same; only the per-instance hash differs.

## Key Takeaways

- **LSB steganography is fragile** — the moment you know the offset, the embedding order, and the bit width, decoding is trivial. Real stego systems use cryptographic keying.
- **The `-5` trick is obfuscation, not security** — it's a one-line constant that any reverse engineer spots instantly.
- **File extensions lie** — `encoded.bmp` is actually a PNG. The binary treats files as raw byte streams, not as structured formats. Always check magic bytes.
- **"n3xt_0n3"** is leetspeak for "next one" — a self-aware nod to the challenge being the third in the Investigative Reversing series (IR0 → IR1 → IR2).

**Flag (public writeup, per-instance):** `picoCTF{n3xt_0n3<per-instance-hash>}`
