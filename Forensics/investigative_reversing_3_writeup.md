# Investigative Reversing 3 — picoCTF Writeup

**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 400  
**Platform:** picoCTF 2019  
**Author:** Santiago C / Danny T  
**Flag:** `picoCTF{4n0th3r_L5b_pr0bl3m_00000000000002e8c5e47}`  
**Writeup by:** zham

---

> We have recovered a binary and an image. See what you can make of it. There should be a flag somewhere.

> Hints
> 1. You will want to reverse how the LSB encoding works on this problem

---

## Background Knowledge

This is the third challenge in the picoCTF "Investigative Reversing" series. The first two use very simple LSB encoding (every byte of the cover image gets one bit of the flag, no padding). This one is more interesting: it interleaves the encoded bytes with unmodified "passthrough" bytes, so a 9-byte block carries 8 bits of flag and 1 byte of clean cover data.

The hint tells us the binary is an LSB encoder. So our job is to:
1. Reverse the binary to figure out exactly where the flag bits are stored in the file.
2. Read the LSBs back out in the right order to reconstruct the flag.

Tools I used:
- `capstone` (Python) to disassemble the ELF.
- `Pillow` to open the image and check its structure.
- A small Python extractor script (shown below) to read the LSBs.

---

## Step 1: Look at the files

I downloaded the binary and the image. The binary is a 64-bit ELF (PIE, dynamically linked, stripped of debugging info but with symbols). The image is a PNG (despite the `.bmp` extension on the binary's filename hints) that shows three colored scribble lines (red, green, yellow) on a cyan background.

```
$ file mystery
mystery: ELF 64-bit LSB pie executable, x86-64, ...

$ xxd encoded.bmp | head -1
00000000: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
```

So the image is a PNG. The binary's strings tell us the names it expects:

```
$ strings mystery | head
/lib64/ld-linux-x86-64.so.2
...
flag.txt
original.bmp
a
encoded.bmp
No flag found, please make sure this is run on the server
No output found, please run this on the server
Invalid Flag
```

So the binary reads `flag.txt` and `original.bmp`, then writes to `encoded.bmp` in append mode (`"a"`).

---

## Step 2: Disassemble the binary

I used Python's `capstone` to disassemble both functions. The binary has only two interesting functions: `codedChar` and `main`.

### `codedChar` (at 0x1195, 74 bytes)

```c
char codedChar(int bit_index, char input_byte, char mask_byte) {
    if (bit_index != 0)
        input_byte >>= bit_index;
    input_byte &= 1;
    mask_byte &= 0xfe;
    mask_byte |= input_byte;
    return mask_byte;
}
```

In plain English: it takes bit `bit_index` of `input_byte` and writes it into the LSB of `mask_byte`, leaving the other 7 bits of `mask_byte` unchanged.

### `main` (at 0x11df, 648 bytes)

The C-equivalent is:

```c
int main(int argc, char **argv) {
    FILE *flag = fopen("flag.txt", "r");
    FILE *orig = fopen("original.bmp", "r");
    FILE *out  = fopen("encoded.bmp", "a");
    if (!flag) { puts("No flag found, ..."); }
    if (!orig) { puts("No output found, ..."); }

    char buf;
    fread(&buf, 1, 1, orig);              // read 1 byte

    for (int i = 0; i < 0x2d3; i++) {     // 723 iterations
        fputc(buf, out);
        fread(&buf, 1, 1, orig);
    }
    // After this loop: 723 bytes copied, buf holds byte #724 (offset 0x2d3)

    char flagbuf[50];
    if (fread(flagbuf, 50, 1, flag) <= 0) { puts("Invalid Flag"); exit(0); }

    for (int i = 0; i < 100; i++) {       // 100 outer iterations
        if ((i & 1) == 0) {                // even i: 8 bytes encode 1 flag char
            for (int j = 0; j < 8; j++) {
                char c = codedChar(j, flagbuf[i/2], buf);
                fputc(c, out);
                fread(&buf, 1, 1, orig);
            }
        } else {                            // odd i: 1 unmodified passthrough byte
            fputc(buf, out);
            fread(&buf, 1, 1, orig);
        }
    }

    while (fread(&buf, 1, 1, orig) == 1)   // copy the rest verbatim
        fputc(buf, out);

    fclose(out); fclose(orig); fclose(flag);
    return 0;
}
```

---

## Step 3: Understand the encoding pattern

So the encoder's plan is:

1. Copy the first **0x2d3 = 723** bytes of `original.bmp` to `encoded.bmp` unchanged.
2. Read 50 bytes of the flag into memory.
3. Loop 100 times (`i = 0..99`):
   - Even `i` (50 times): take 8 bytes from `original.bmp`, replace each one's LSB with bit `j` (for `j = 0..7`) of `flagbuf[i/2]`, write the 8 modified bytes to `encoded.bmp`.
   - Odd `i` (50 times): take 1 byte from `original.bmp` and copy it unchanged to `encoded.bmp` (passthrough).
4. Copy the rest of `original.bmp` to `encoded.bmp` unchanged.

So for each flag character, the binary uses **9 bytes** of the cover file: 8 modified + 1 passthrough. The 8 modified bytes carry the 8 bits of the character, in order: byte 0's LSB = bit 0, byte 1's LSB = bit 1, …, byte 7's LSB = bit 7.

The total flag length is **50 bytes** (binary reads exactly 50), and the encoded region runs from byte **0x2d3** to byte **0x2d3 + 50*9 - 1 = 0x6a4** of `encoded.bmp` (1172).

---

## Step 4: Write the extractor

The extractor just reads 50 cycles of 9 bytes, pulls the LSB of the first 8 in each cycle, and reassembles the flag byte in LSB-first order.

```python
with open('encoded.bmp', 'rb') as f:
    f.seek(0x2d3)
    data = f.read(50 * 9)

flag = ''
for i in range(50):
    bits = ''
    for j in range(8):
        byte = data[i * 9 + j]
        bits += str(byte & 1)        # LSB-first
    char_val = int(bits[::-1], 2)     # reverse for big-endian byte value
    if char_val == 0:
        break
    flag += chr(char_val)
print(flag)
```

In Kali, the same script as `extract.py`:

```python
$ nano extract.py
```

```
┌──(zham㉿kali)-[~/picoCTF/inv3]
└─$ python extract.py
picoCTF{4n0th3r_L5b_pr0bl3m_00000000000002e8c5e47}
```

The flag is `picoCTF{4n0th3r_L5b_pr0bl3m_00000000000002e8c5e47}`. The `00000000000002e8c5e47` suffix is per-instance, so a different challenge run will give a different 22-character tail but the same `picoCTF{4n0th3r_L5b_pr0bl3m_` prefix.

---

## What Happened Internally

A short timeline of the encoder:

1. **Bytes 0..0x2d2 of `original.bmp` are copied verbatim to `encoded.bmp`.** This is the cover file's "header" + a small amount of cover data. For a 1765x852 RGBA PNG, the first 723 bytes include the 8-byte PNG signature, the 25-byte IHDR chunk, and 690 bytes of the first IDAT chunk.
2. **`flag.txt` is read into a 50-byte buffer.** If the file has fewer than 50 bytes, the binary prints "Invalid Flag" and exits — that is why picoCTF instance flags are always exactly 50 bytes long.
3. **For each even outer index `i = 0, 2, 4, …, 98`**, the encoder grabs the next 8 bytes of `original.bmp` and writes a modified version to `encoded.bmp`. In each byte, only the LSB is touched — it becomes bit `j` of `flagbuf[i/2]`. The other 7 bits are the original byte with its LSB cleared (`& 0xfe`) then OR'd with the flag bit.
4. **For each odd outer index `i = 1, 3, 5, …, 99`**, the encoder grabs 1 byte of `original.bmp` and copies it unchanged. This is the "passthrough" byte that gives the file a more natural look after the modification (and gives the encoder a fixed stride so it can always find byte `9*k + 0x2d3` again for the next flag character).
5. **The rest of `original.bmp` is copied verbatim** so the encoded file stays the same length as the original.

The decoder reverses step 3 only. It opens the encoded file, seeks to `0x2d3`, reads 50 * 9 = 450 bytes, and for each 9-byte block it pulls the LSBs of the first 8 bytes into a bit string `b0 b1 … b7`, then reverses that to get the big-endian byte value of one flag character.

Why this works as a stego scheme: the LSB of a byte in compressed (PNG IDAT) data is roughly random, so flipping one bit per byte tends to leave the decompressed image looking nearly the same. The image changes by at most ±1 in the color of a few pixels — invisible to the eye.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `capstone` (Python) | Disassemble the ELF and find `codedChar` + `main` |
| `file`, `xxd` | Confirm the binary is ELF and the image is a PNG |
| `strings` | Pull the file-mode and filename hints out of the binary |
| `Pillow` (`PIL`) | Open the image, count unique colors, sanity-check the format |
| Custom Python script | Read the LSBs at offset 0x2d3 with 9-byte stride to extract the flag |

---

## Key Takeaways

- **Strip the binary before you trust a writeup.** A static analysis of the encoder is more reliable than guessing an offset, because it tells you the exact stride, the exact start, and the exact bit order.
- **The LSBs are stored LSB-first per character**, so you have to reverse the bit string before converting to a byte. This is the same bit-flip pattern used in many other LSB stego challenges.
- **The 8+1 stride is a red herring** — it doesn't make the LSBs harder to find, it just makes the encoder produce a more natural-looking output by leaving a passthrough byte every 9. The 9-byte stride falls out of the loop structure in `main`.
- **The flag format is `picoCTF{4n0th3r_L5b_pr0bl3m_<22-char-hash>}`** (e.g. `00000000000002e8c5e47` for this instance), where the last 22 characters are per-instance. The leet-speak decode: **"ANOTHER LSB PROBLEM"** — `4n0th3r` = another, `L5b` = LSB, `pr0bl3m` = problem. The name is meta — the first two Investigative Reversing challenges used LSB encoding too, just without the 8+1 stride.
