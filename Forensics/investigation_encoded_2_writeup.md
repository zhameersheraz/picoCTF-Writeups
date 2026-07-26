# investigation_encoded_2 — picoCTF Writeup

**Challenge:** investigation_encoded_2  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 500  
**Flag:** `t1m3f1i35000000000009322d8fc`  
**Platform:** picoCTF 2019  
**Author:** Santiago C  
**Writeup by:** zham

---

## Description

> What can you make of this? We have recovered a binary and 1 file: image01. See what you can make of it. NOTE: The flag is not in the normal picoCTF{XXX} format.

**Attachments:** `no_auth_mistery` (encoder binary, `chal1.c` derivative) + `image01` (encoded output, 57 bytes)

---

## Hints

> 1. Only use lower case letters and numbers

---

## Background Knowledge (Read This First!)

This challenge is the harder sequel to `investigation_encoded_1`. The encoding is the same (variable-length prefix code packed into bytes MSB-first), but two extra layers were bolted on:

1. **A login gate** — the binary connects to `fakeauthsite.com:80`, sends a base64-encoded "Auth:" header, and waits for the server to send back "Authorized to execute..." before it will actually run the encoder. Without network access (or a patched binary), the encode function never runs. The fix: reverse engineer the encoding algorithm statically and decode the output directly, or patch the binary to skip the auth check.
2. **A different alphabet + per-character permutation** — the alphabet is no longer a-z + space. It's a-z + 0-9 (36 characters), and each character's index in the codebook is permuted by adding 18 mod 36 before lookup. So `'a'` does not encode to the bits at table index 0 — it encodes to the bits at table index 18.

### The encoding algorithm (reversed from `no_auth_mistery`)

For each character of the flag:

1. **Normalize**: lowercase if A-Z. If space, map to `0x85` (which becomes index 36 in the array after `-0x61`). If digit `'0'`-`'9'`, add `0x4B` (so `'0' = 0x30 + 0x4B = 0x7B`).
2. **Subtract 0x61**: now every valid character is in `[0, 36)`. Space is exactly 36 (`0x24`).
3. **Permute** (only for non-space): `idx = (char + 0x12) mod 36` then `abs()`. This shuffles the alphabet by adding 18.
4. **Look up the bit string**: read `length = table[idx+1] - table[idx]` bits from the `secret` byte array starting at bit offset `table[idx]`. The bits are read MSB-first within each byte (`bit_pos = 7 - (bit_idx mod 8)`).
5. **Pack the bits MSB-first** into the output buffer. When 8 bits accumulate, write a byte and reset.
6. **Pad with zeros** to the next byte boundary when the input is exhausted.

### The decode algorithm (what we actually do)

Since the encoding is a prefix code, the decode is just:

1. **Build the codebook** by extracting the `secret` array and the `table` of bit offsets from the binary, then computing each character's bit string the same way the encoder does.
2. **Convert** `image01` to a bit stream.
3. **Greedy match** the bit stream against the codebook (no code is a prefix of another, so the match is always unique).
4. **Stop** when the remaining bits are all zero padding.

---

## Solution — Step by Step

### Step 1 — Inspect the binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file no_auth_mistery image01
no_auth_mistery: ELF 64-bit LSB pie executable, x86-64, not stripped
image01:        data (57 bytes)
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings no_auth_mistery | grep -E 'fakeauth|Authorize|encoded|output|flag'
fakeauthsite.com
QXV0aG9yaXplZCB0byBleGVjdXRlLi4u
Encoded to ./output
flag.txt
output
```

The string `QXV0aG9yaXplZCB0byBleGVjdXRlLi4u` is base64 for `Authorized to execute...` — the response the binary expects from the auth server. The hostname `fakeauthsite.com` decodes to `ZmFrZWF1dGhzaXRl.com` in base64 (which is just the domain name). So the binary is checking in with a server that doesn't exist, and the encode function never runs.

### Step 2 — Locate the two globals

The `secret` byte array and the `table` of bit offsets are easy to find because the binary isn't stripped. Using capstone to disassemble the encoding function at `0xd27`, we see two `lea rax, [rip + ...]` instructions pointing to the two tables:

- `secret` array: `lea rax, [rip + 0x679]` at `0xc80` — vaddr `0x1300`, **96 bytes long** (the bit offsets go up to 560, so the array has to be at least 70 bytes; the slot before the table is 96 bytes of zeroes-tipped secret).
- `table` of bit offsets: `lea rax, [rip + 0x56f]` at `0xdea` — vaddr `0x1360`, **37 uint32 entries** (one for each prefix + a sentinel).

Reading these out:

```python
import struct
data = open('no_auth_mistery','rb').read()
secret = data[0x1300:0x1360]                       # 96 bytes
table  = [struct.unpack_from('<I', data, 0x1360 + i*4)[0] for i in range(37)]
```

### Step 3 — Build the codebook

The codebook maps each printable char (a-z, 0-9) to its variable-length bit string:

```python
chars = [chr(ord('a')+i) for i in range(26)] + [str(i) for i in range(10)]

def gv(idx):
    return (secret[idx>>3] >> (7 - (idx & 7))) & 1

for i, c in enumerate(chars):
    perm_idx = (i + 18) % 36                # the +0x12 permutation
    start, end = table[perm_idx], table[perm_idx+1]
    bits = ''.join(str(gv(start + k)) for k in range(end - start))
    codebook[c] = bits
```

Resulting codebook (excerpt):

| Char | Perm idx | Bit string           |
|------|---------:|----------------------|
| a    | 18       | `101011101110111000`  |
| b    | 19       | `1010101110111000`    |
| e    | 22       | `101010101000`       |
| s    | 0        | `1000`               |
| t    | 1        | `10111010101000`     |
| 0    | 8        | `1110101010111000`   |
| 1    | 9        | `1110101110101110111000` |
| 9    | 17       | `10111011101110111000` |

Note that the bit lengths vary from 4 bits (`s`, `m`) up to 22 bits (`1`, `8`, `u`, `w`, `z`). The most-frequent characters in a typical flag prefix get the shortest codes.

### Step 4 — Decode `image01`

```python
enc = open('image01','rb').read()
bitstream = ''.join(f'{b:08b}' for b in enc)   # 456 bits

inv = {v: k for k, v in codebook.items()}
bits, out = bitstream, []
while bits:
    for code, ch in inv.items():
        if bits.startswith(code):
            out.append(ch); bits = bits[len(code):]; break
    else:
        break                              # padding zeros — stop
print(''.join(out))
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 decode_investigation_encoded_2.py
Decoded (28 chars): t1m3f1i35000000000009322d8fc
Flag:              t1m3f1i35000000000009322d8fc
```

The recovered flag is `t1m3f1i35000000000009322d8fc`.

---

## What Happened Internally (Timeline)

1. The challenge author wrote a C program (`chal1.c` family) that uses a variable-length prefix code to compress a 36-character alphabet (a-z + 0-9). They stored the alphabet's bit strings in two globals: a 96-byte `secret` bit pool and a 37-entry `table` of bit offsets. They then bolted on a fake auth-server handshake so that nobody could run the encoder offline.
2. At deployment time, the server ran the binary in a controlled environment where the auth check would pass. The flag (a 28-char string like `t1m3f1i35000000000003d746a40`) was read from `flag.txt`, encoded via the prefix code, and the 57-byte result was saved as `image01`.
3. The flag's wordplay is `t1m3_f1i35` = "time flies" in leet (`1`→`i`, `3`→`e`, `5`→`s`). The suffix is per-instance random hex.
4. You, the solver, have only the binary and the encoded file. You either (a) patch the binary in a hex editor to NOP the auth check and brute-force the codebook by running the patched binary on each character, or (b) reverse the algorithm statically with capstone and rebuild the codebook in Python. We chose (b) because the binary is small and the algorithm is just two `lea`+`mov` lookups.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `file` / `strings` | Identify the binary, find the auth-server strings and the encoder function names | Easy |
| `capstone` (Python) | Disassemble the encoding function to find the `secret` array offset and the `table` offset | Medium |
| `python3` | Read the two globals, build the codebook, do greedy prefix-decoding on `image01` | Easy |
| Optional: hex editor / `pwntools` | Patch the binary to skip the auth check if you prefer the empirical approach | Medium |

---

## Key Takeaways

- **The "login" is fake.** The string `QXV0aG9yaXplZCB0byBleGVjdXRlLi4u` is just `Authorized to execute...` base64-encoded, and the server hostname is hard-coded. The auth check exists only to make the encoder hard to run; it has no real cryptographic role. A determined solver always patches or sidesteps the auth.
- **The encoding is a prefix code, not a cipher.** Unlike investigation_encoded_1, this one adds a permutation step (`(idx + 18) mod 36`) before the prefix-code lookup, but it's still a static lookup table. No secret, no key, no decryption — just figure out the table.
- **The +0x12 / 0x38e38e39 magic is a divide-by-36 constant.** Compiler-generated division by 36: the multiply-by-`0x38e38e39`-then-shift-by-35 pattern. Recognizing this in disassembly lets you see the 36-character alphabet without trial-and-error.
- **The flag is a leet pun.** `t1m3_f1i35` is "time flies" with `1`→`i`, `3`→`e`, `5`→`s`. The challenge title is `investigation_encoded_2` and the year is 2019 — the flag is a tongue-in-cheek "time flies, doesn't it?" aimed at forensics players who spent a weekend on the picoCTF 2019 series.
- **Read the hint about alphabet.** "Only use lower case letters and numbers" tells you up front that the alphabet is a-z + 0-9 (36 symbols), saving you the trouble of figuring out whether space is in the alphabet. It is, sort of (handled as index 36), but the flag itself never contains a space, so the permutation skip doesn't matter.
