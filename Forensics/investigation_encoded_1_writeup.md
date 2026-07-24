# investigation_encoded_1 — picoCTF Writeup

**Challenge:** investigation_encoded_1  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 450  
**Flag:** `encodedoaczznvnuo`  
**Platform:** picoCTF 2019  
**Author:** Santiago C  
**Writeup by:** zham

---

## Description

> We have recovered a binary and 1 file: image01. See what you can make of it. NOTE: The flag is not in the normal picoCTF{XXX} format.

**Attachments:** `mystery` (encoder binary, `chal1.c`) + `image01` (the encoded output, 23 bytes)

---

## Hints

> 1. The binary is not stripped — strings like `chal1.c`, `secret`, `matrix`, `encode`, `getValue`, `isValid`, `save`, `remain`, `buffChar` are all visible.
> 2. `isValid` only accepts letters and spaces. Spaces are translated to `{` internally — that is the only way the brace character enters the alphabet.
> 3. The output is a bit-packed variable-length prefix code, not a byte-level cipher. Each character maps to a different number of bits.
> 4. The codebook is built from two globals: a 27-entry `matrix` of (bit_length, offset) pairs and a 37-byte `secret` byte array.

---

## Background Knowledge (Read This First!)

### Variable-length prefix codes

A **prefix code** is a way to map symbols to bit strings such that no code is a prefix of another code. Huffman coding is the most famous example. The trick is that more common characters get shorter codes, so common text compresses well.

To **decode** a prefix-coded bitstream you do a **greedy match**: at each step, scan the codebook for the entry whose bit string is a prefix of the remaining bits, emit that character, then advance past those bits. Because no code is a prefix of another, the match is always unique (or fails on padding).

### What `chal1.c` does

1. Reads `flag.txt` into a buffer.
2. For each character, checks it is a-z / A-Z / space. Lowercases it, swaps space → `{`.
3. Looks up that character in a 27-entry `matrix` (one entry per letter a-z plus space/curly-brace). Each entry is `(bit_length, offset_into_secret)`.
4. Calls `getValue(offset + k)` for `k = 0..bit_length-1` to read the next bit of this character's code from a 37-byte `secret` array.
5. Calls `save(bit)` for each bit. `save` accumulates bits MSB-first into `buffChar` and `fputc`s a byte every 8 bits. Between characters, padding zeros get appended so the final bit position lands on a byte boundary.

### Why a 23-byte input can hold a 17-character flag

Each character is encoded into a different number of bits (e is 4 bits, j is 16 bits). The total flag content encodes to 174 bits = 22 bytes + 2 bits, padded to 23 bytes with 6 trailing zeros (which decode as one trailing space, which we strip).

---

## Solution — Step by Step

### Step 1 — Quick recon

```
┌──(zham㉿kali)-[/media/sf_downloads/inv_enc]
└─$ file mystery image01
mystery:  ELF 64-bit LSB pie executable, x86-64, not stripped
image01:  data
```

```
┌──(zham㉿kali)-[/media/sf_downloads/inv_enc]
└─$ strings mystery | grep -E 'encode|matrix|secret|flag|chal1'
flag.txt
./flag.txt not found
output
I'm Done, check ./output
Error, file bigger that 65535
Error, I don't know why I crashed
encode
matrix
secret
buffChar
flag_index
flag_size
isValid
lower
remain
save
getValue
chal1.c
```

The binary is `chal1.c` compiled, the source is in the symbol table, and the relevant globals/functions are all visible. No need for a decompiler yet — `strings` and the symbol list tell us the whole story.

### Step 2 — Decompile the four key functions

A quick pass through `main` / `encode` / `getValue` / `save` (the decompiler output is in many public writeups, e.g. Dvd848's CTFs repo, so I won't reproduce all 100 lines here). The shape is:

```c
// main: reads flag.txt, calls encode(), writes "output"
// encode(): per character:
//   idx = matrix[(c - 'a') * 8 + 4]   // offset (uint32)
//   end = idx + matrix[(c - 'a') * 8] // length (uint32)
//   for k in 0..end-idx-1:
//     save(getValue(idx + k))

// getValue(idx): returns the single bit at secret[idx/8] position
//                (7 - (idx mod 8))  -- MSB-first

// save(bit): accumulates bits into buffChar MSB-first, fputc every 8 bits
```

So each character is mapped to a bit string whose length and content are derived from two globals: `matrix` (a 54-uint32 table of `(length, offset)` pairs) and `secret` (a 37-byte bit pool).

### Step 3 — Extract `matrix` and `secret` from the binary

I used `objdump` / `radare2` to dump the two globals as C arrays:

```c
const uint8_t secret[] = {
  0xb8, 0xea, 0x8e, 0xba, 0x3a, 0x88, 0xae, 0x8e, 0xe8, 0xaa,
  0x28, 0xbb, 0xb8, 0xeb, 0x8b, 0xa8, 0xee, 0x3a, 0x3b, 0xb8,
  0xbb, 0xa3, 0xba, 0xe2, 0xe8, 0xa8, 0xe2, 0xb8, 0xab, 0x8b,
  0xb8, 0xea, 0xe3, 0xae, 0xe3, 0xba, 0x80
};

const uint32_t matrix[] = {
  0x08,0x00, 0x0c,0x08, 0x0e,0x14, 0x0a,0x22, 0x04,0x2c,
  0x0c,0x30, 0x0c,0x3c, 0x0a,0x48, 0x06,0x52, 0x10,0x58,
  0x0c,0x68, 0x0c,0x74, 0x0a,0x80, 0x08,0x8a, 0x0e,0x92,
  0x0e,0xa0, 0x10,0xae, 0x0a,0xbe, 0x08,0xc8, 0x06,0xd0,
  0x0a,0xd6, 0x0c,0xe0, 0x0c,0xec, 0x0e,0xf8, 0x10,0x106,
  0x0e,0x116, 0x04,0x124
};
```

(Stored in the binary as little-endian bytes; the values above are already extracted as `(length, offset)` pairs for clarity.)

### Step 4 — Build the codebook

For each of the 27 characters (a..z, space), look up its `(length, offset)` in the matrix, then read `length` bits from `secret` starting at `offset` using the same `getValue` logic. Result:

| Char | Bits   | | Char | Bits |
|------|--------|-|------|------|
| a    | 10111000   || n    | 11101000   |
| b    | 111010101000 || o    | 11101110111000 |
| c    | 11101011101000 || p    | 10111011101000 |
| d    | 1110101000  || q    | 1110111010111000 |
| e    | 1000        || r    | 1011101000 |
| f    | 101011101000 || s    | 10101000 |
| g    | 111011101000 || t    | 111000 |
| h    | 1010101000  || u    | 1010111000 |
| i    | 101000      || v    | 101010111000 |
| j    | 1011101110111000 || w    | 101110111000 |
| k    | 111010111000 || x    | 11101010111000 |
| l    | 101110101000 || y    | 1110101110111000 |
| m    | 1110111000  || z    | 11101110101000 |
|      |             || (space) | 0000 |

Notice that some codes are 4 bits (e, space), some 16 (j, q, y). This is why the encoded output is so much shorter than the plaintext flag.

### Step 5 — Decode the bitstream

Convert `image01` (23 bytes) to a bitstream and greedy prefix-match:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 decode_investigation_encoded_1.py image01
File: image01  (23 bytes)
Hex:  8e8eba3bb8ea23a8eee2e3ae8eea3ba8e8ab8e8ae3bb80

Decoded: 'encodedoaczznvnuo'
Flag:    encodedoaczznvnuo
```

The 23-byte input expands to 184 bits, which decode to 17 characters + 1 trailing space (padding). Strip the trailing space and that is the flag — **no `picoCTF{...}` wrapper**, despite the standard format. The challenge description says "the flag is not in the normal picoCTF{XXX} format" — that is literal, not a hint to look beyond the format.

```
encodedoaczznvnuo
```

---

## What Happened Internally (Timeline)

1. The challenge author wrote a C program (`chal1.c`) that implements a hand-rolled variable-length prefix code: 27 symbols (a-z + space) mapped to bit strings of length 4..16.
2. The bit strings are stored in two globals — a 37-byte `secret` bit pool and a 27-entry `matrix` of `(bit_length, offset_into_secret)` pairs. The `getValue` function reads a single bit at a time from `secret` (MSB-first within each byte).
3. To encode, the binary walks the flag character by character. For each character, it looks up the code in the matrix and appends those bits to the output stream (MSB-first packing into `buffChar`, flushed every 8 bits). When the flag is exhausted, it pads with zero bits until the next byte boundary, then writes the final byte.
4. You, the solver, get the packed output (`image01`) and the encoder binary (`mystery`). Run `strings`, see the `chal1.c` symbol and the function names, decompile the four key functions in Ghidra/r2, extract `matrix` and `secret`, build the codebook, and decode the bitstream greedily.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` / `strings` | Identify the binary, pull out the function/global names | Easy |
| `objdump` / `radare2` | Dump the `matrix` and `secret` arrays as C initializers | Medium |
| `python3` | Reimplement `getValue` and do greedy prefix-decoding | Easy |
| Ghidra (alternative) | Decompile `main` / `encode` / `getValue` / `save` to confirm the algorithm | Medium |

---

## Key Takeaways

- **`strings` and the symbol table give you the whole shape.** The binary is unstripped and the function names (`encode`, `matrix`, `secret`, `getValue`, `save`, `remain`, `buffChar`) are a complete outline of the algorithm before you even open a decompiler.
- **Variable-length codes are why output < input.** A naive byte cipher preserves length; a prefix code lets common characters take fewer bits. The fact that `image01` (23 bytes) is much shorter than a typical picoCTF flag immediately suggests compression, not encryption.
- **Greedy prefix-match is the universal decoder.** For any codebook where no code is a prefix of another, walking the bitstream and emitting the first matching code at each step is correct. The padding bits (zeros) at the end may decode to "no match" — that's normal, just trim the trailing space.
- **Spaces become `{` in the alphabet.** `isValid` rejects everything except a-z / A-Z / space; `lower` lowercases; if the result is space, it gets swapped to `{` before the matrix lookup. So the alphabet that the codebook actually covers is a-z + `{` (27 entries), with `{` standing in for the space character.
- **The codebook size is the giveaway.** 27 entries = 26 letters + space. The matrix has 27 (length, offset) pairs, the secret pool has 37 bytes (~296 bits total capacity, well over the ~180 bits the flag needs).
- **Skip the Ghidra re-decompile if the data is published.** The matrix and secret arrays are not secret — they are constants baked into the binary, and any public writeup of this challenge will have the exact bytes. The decoding is the interesting part, not the RE.
- **Read the description literally.** "The flag is not in the normal picoCTF{XXX} format" means submit the raw decoded text, not `picoCTF{decoded_text}`. Most picoCTF writeups wrap the result anyway (out of habit), and the server will reject it. If the description says "not in the normal format", trust it.
