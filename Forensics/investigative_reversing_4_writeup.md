# Investigative Reversing 4 — picoCTF Writeup

**Challenge:** Investigative Reversing 4  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 400  
**Flag:** `picoCTF{N1c3_R3ver51ng_5k1115_00000000000e17ba9e6}`  
**Platform:** picoCTF 2019  
**Authors:** Santiago C / Danny T  
**Writeup by:** zham  

---

## Description

> We have recovered a binary and 5 images: image01, image02, image03, image04, image05. See what you can make of it. There should be a flag somewhere.

**Attachments:** `mystery` (the encoder binary) + `Item01_cp.bmp` … `Item05_cp.bmp` (the encoded outputs)

---

## Hints (from the writeup community)

> 1. The binary is a C program that compiled to a shared library — it was run on the server to encode the flag into the images.
> 2. Open the binary in a reverse-engineering tool (Ghidra is the recommended one) and look at `main()`. The `flag` and `flag_index` globals are set up here.
> 3. The encoding happens in `codedChar()` and `encodeDataInFile()`. The 5 images are processed in reverse order (Item05 down to Item01), and the flag is split across all five.
> 4. The flag is read 50 bytes at a time from `flag.txt`. Each image gets 10 flag bytes embedded, one bit per file byte, in classic LSB steganography.

---

## Background Knowledge (Read This First!)

### What is LSB Steganography?

**LSB (Least Significant Bit) steganography** is the classic way to hide data in an image (or any binary file). The idea: the lowest bit of each byte is usually noise — flipping it changes the value by at most 1, so the visual change is imperceptible. You replace each LSB with one bit of your secret message, and the image still looks the same.

In this challenge, the "image" bytes are not actually pixels — they're just bytes in a PNG file (the `.bmp` extension is a misdirection). The encoder treats the file as a raw byte stream and hides flag bits in the LSBs of specific bytes.

### Why the binary matters

The source file is `chal2.c` (visible in the binary's debug strings). The `mystery` binary is the compiled C code. To reverse the encoding, you need to read the binary and understand exactly how the bits are placed.

### How the encoder works (high level)

1. Open `flag.txt` and read 50 bytes (the flag, padded with nulls if shorter).
2. For each of the 5 input BMPs (Item05.bmp, Item04.bmp, …, Item01.bmp, in that order):
   - Open the input and output (Item01_cp.bmp, etc.) files.
   - Copy the first **2019 bytes** (the BMP header) unchanged.
   - For a 50-position loop, on every **5th position** (j=0, 5, 10, …, 45), embed 8 bits of one flag byte into the next 8 file bytes, one bit per byte in the LSB.
   - On the other positions, just copy the input byte unchanged.
   - Copy the rest of the file unchanged.
3. Each image gets 10 flag bytes (10 embed rounds × 8 bits). Total across 5 images = 50 flag bytes.

### The bit order

Within each flag byte, the 8 bits are placed in **LSB-first order**: bit 0 of the flag byte goes to the LSB of the first cover byte, bit 1 to the LSB of the second, and so on. To decode, read the LSB of each of the 8 cover bytes and assemble them as bits 0..7 of the flag byte.

### File order matters

The encoder processes images from Item05 down to Item01 (in that order). So when reassembling the flag from the encoded outputs (Item01_cp.bmp through Item05_cp.bmp), you read them in **reverse** — Item05_cp.bmp has flag bytes 0–9, Item04_cp.bmp has 10–19, …, Item01_cp.bmp has 40–49.

---

## Solution — Step by Step

### Step 1 — Inspect the binary

The `mystery` file is a 64-bit ELF. Quick recon:

```
┌──(zham㉿kali)-[/media/sf_downloads/ir4]
└─$ file mystery
mystery: ELF 64-bit LSB shared object, x86-64, ...
```

```
$ strings mystery
GLIBC_2.2.5
fopen
fputc
fread
fclose
__libc_start_main
gfff
Item01_cH
p.bmp
Item01.b
No output found, please run this on the server
flag.txt
No flag found, please make sure this is run on the server
Invalid Flag
codedChar
buff_index
encodeAll
flag_index
encodeDataInFile
flag_size
buff
buff_size
flag
fileName
…
```

Key strings: `flag.txt` (the flag source), `Item01.bmp`/`Item01_cp.bmp` (the input/output filename pattern), `codedChar`/`encodeDataInFile`/`encodeAll` (the function names).

### Step 2 — Disassemble to find the encoding

Using `capstone` (or Ghidra, IDA, etc.), the three key functions are:

**`codedChar` at 0x7aa** — the embed helper, takes `(index, flag_byte, input_byte)`:

```c
int codedChar(int index, char flag_byte, char input_byte) {
    if (index != 0) flag_byte >>= index;        // shift right to bring target bit to position 0
    int flag_bit = flag_byte & 1;               // mask off just that one bit
    return (input_byte & 0xfe) | flag_bit;      // clear LSB of input, OR in the flag bit
}
```

For `index=k`, this puts bit `k` of the flag byte into bit 0 of the input byte.

**`encodeDataInFile` at 0x7f6** — opens the input and output BMPs, copies the first 0x7e3 = **2019 bytes** unchanged, then runs a 50-position loop where every 5th position embeds 8 flag bits and the other positions just copy a byte. Flag_index increments after each embed round (10 rounds per file).

**`encodeAll` at 0xa1a** — loops from `'5'` down to `'1'`, calling `encodeDataInFile` for each.

**`main` at 0xa9d** — sets up `flag` and `flag_index` globals (pointers into a local buffer), reads 0x32 = **50 bytes** from `flag.txt`, then calls `encodeAll()`.

### Step 3 — Identify the modified bytes in each output

For each encoded BMP, the 80 modified bytes (10 flag bytes × 8 bits each) live at these offsets:

```
Round 0  (flag bytes  0..9)  -> offsets 2019 + 0*12 = 2019..2026
Round 1  (flag bytes 10..19) -> offsets 2019 + 1*12 = 2031..2038
Round 2  (flag bytes 20..29) -> offsets 2019 + 2*12 = 2043..2050
Round 3  (flag bytes 30..39) -> offsets 2019 + 3*12 = 2055..2062
Round 4  (flag bytes 40..49) -> offsets 2019 + 4*12 = 2067..2074
...
Round 9  (flag bytes 90..99) -> offsets 2019 + 9*12 = 2127..2134
```

Each "round" of the j loop consumes 12 output bytes: 8 embed bytes + 4 plain copy bytes.

### Step 4 — Reassemble the flag

For each encoded BMP (read in **reverse** order — Item05 first, Item01 last), for each of the 10 embed rounds, read the LSB of the 8 cover bytes and assemble them into a flag byte:

```python
def extract_flag_bytes(path):
    with open(path, "rb") as f:
        data = f.read()
    flag_bytes = []
    for round_idx in range(10):
        flag_byte = 0
        for k in range(8):
            off = 2019 + round_idx * 12 + k
            flag_byte |= ((data[off] & 1) << k)   # bit k = LSB of cover byte
        flag_bytes.append(flag_byte)
    return bytes(flag_bytes)
```

Concatenating the 10-byte chunks from Item05_cp.bmp down to Item01_cp.bmp gives the full 50-byte flag.

### Step 5 — The recovered flag

Run the decoder against the five 1.5 MB `Item0?_cp.bmp` files:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 decode_ir4.py
Item05_cp.bmp: b'picoCTF{N1'
Item04_cp.bmp: b'c3_R3ver51'
Item03_cp.bmp: b'ng_5k1115_'
Item02_cp.bmp: b'0000000000'
Item01_cp.bmp: b'0e17ba9e6}'

Full flag: picoCTF{N1c3_R3ver51ng_5k1115_00000000000e17ba9e6}
```

The flag for this instance is:

```
picoCTF{N1c3_R3ver51ng_5k1115_00000000000e17ba9e6}
```

The first 30 bytes (`picoCTF{N1c3_R3ver51ng_5k1115_`) are constant across every instance — the trailing 20 hex chars (`00000000000e17ba9e6`) are the per-instance hash and will be different for every deployment. The structure is 11 zero hex digits + 9 nonzero hex digits, which is the random instance ID formatted as a 20-hex string.

> If the per-instance hash in your output doesn't render as printable text, double-check that you used the right source files (the binary's `mystery` is the encoder; the 5 BMPs are the *encoded* outputs `Item01_cp.bmp`–`Item05_cp.bmp`, not the originals `Item01.bmp`–`Item05.bmp`). Also check the file size — the encoded outputs are ~1.5 MB each, the original BMPs are ~88 KB.

---

## What Happened Internally (Timeline)

1. The challenge author wrote a small C program (`chal2.c`) that does classic LSB steganography — 1 bit per file byte, 8 file bytes per flag byte, embedded in 5 different cover files.
2. At deployment time, the server ran this binary. The instance's `flag.txt` (containing the per-instance flag) was read into a 50-byte buffer. The 5 original BMPs (Item01.bmp…Item05.bmp) were read in reverse order (05→01), and the corresponding 5 encoded copies (Item01_cp.bmp…Item05_cp.bmp) were written.
3. For each cover file, the binary: (a) copied the first 2019 bytes verbatim, (b) for 50 byte-positions, either copied the byte (j%5≠0) or embedded 8 flag bits into 8 consecutive bytes' LSBs (j%5=0), and (c) copied the rest of the file. The global `flag_index` advanced by 1 after each embed round, so the flag bytes were consumed in order across the 5 files.
4. You, the solver, have the 5 encoded outputs. To recover the flag: disassemble `mystery` to confirm the embedding scheme, identify the 80 modified byte offsets per file, extract the LSBs, and reassemble in reverse file order.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` / `strings` | Identify the binary and pull out the meaningful strings (`flag.txt`, `codedChar`, `Item01.bmp`, etc.) | Easy |
| `capstone` (Python) | Disassemble the 64-bit ELF and read the embed loop | Medium |
| `python3` | Decode the flag bits back into bytes | Easy |
| `Ghidra` (alternative) | Decompile `main`/`codedChar`/`encodeDataInFile` if you prefer a GUI RE tool | Medium |

---

## Key Takeaways

- **The hint is in the strings.** Before reaching for a decompiler, `strings mystery` already tells you the encoder reads `flag.txt`, writes `Item01_cp.bmp`/`Item02_cp.bmp`/etc., and the relevant function is `codedChar`. That alone narrows the search to ~3 functions.
- **The 2019-byte offset is the magic number.** The first 2019 bytes of every output are the BMP header, untouched. All 80 embedded bytes live in the next 120 bytes (10 rounds × 12 bytes per round, 8 of which are embed bytes).
- **The j%5 dance is just for the "skips 4 bytes, embed 8 bits" pattern.** It looks clever in the disassembly, but it's equivalent to: skip 4 cover bytes, embed 8 flag bits, skip 4, embed 8, … for 10 cycles.
- **Read the files in reverse.** The encoder processes Item05 → Item04 → … → Item01, so the first 10 flag bytes go into Item05_cp.bmp's modified region, the next 10 into Item04_cp.bmp's, etc. Reassemble the same way.
- **LSB = bit 0 of the cover byte, bit `index` of the flag byte.** Within each 8-byte embed round, the encoder stores bit `k` of the flag byte at the LSB of cover byte `k` (k=0..7). No bit-reversal, no XOR — just a straight one-to-one LSB splice.
- **The challenge is rated Hard for a reason.** The binary is small, the algorithm is simple, but the disassembly-to-decoder path requires careful static analysis. If you can confidently read the C-equivalent of an x86-64 `codedChar` function, this one falls fast.
- **The flag's wordplay is the leet-speak classic.** `N1c3_R3ver51ng_5k1115` reads as "Nice Reversing Skills" in leet (`1`→`i`, `3`→`e`, `5`→`s`). The challenge title is "Investigative Reversing 4" — the flag is essentially a compliment.
- **The 20-hex suffix is per-instance.** `00000000000e17ba9e6` in this run, but every picoCTF instance has its own random ID. The pattern is 11 zero hex digits + 9 nonzero hex digits (so the instance ID fits in ~36 bits). If a public writeup gives you a different flag, that's normal — re-decode your own `Item0?_cp.bmp` files.
