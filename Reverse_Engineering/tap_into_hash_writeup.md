# Tap into Hash — picoCTF Writeup

**Challenge:** Tap into Hash  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{block_3SRhViRbT1qcX_XUjM0r49cH_qCzmJZzBK_60647fbb}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> Can you make sense of this source code file and write a function that will decode the given encrypted file content?
>
> Find the encrypted file [here](#). It might be good to analyze [source file](#) to get the flag.

## Hints

> 1. Do you know what blockchains are? If so, you know that hashing is used in blockchains.
> 2. Download the encrypted flag file and the source file and reverse engineer the source file.

---

## Background Knowledge (Read This First!)

If any of these words sound scary, do not worry — I will explain every single one.

### Hashing

A **hash** is a one-way function. You give it any text, and it gives you back a fixed-length "fingerprint". The same text always gives the same fingerprint, but you cannot go from the fingerprint back to the text.

Python's `hashlib.sha256(...).hexdigest()` gives a 64-character fingerprint made of the characters `0-9` and `a-f`.

```python
>>> import hashlib
>>> hashlib.sha256(b"hello").hexdigest()
'2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824'
```

### Blockchains (the tiny version)

A **blockchain** is just a list of **blocks** linked together. Each block stores:
- `index` — its position in the chain (0, 1, 2, ...)
- `previous_hash` — the fingerprint of the previous block
- `timestamp` — when it was created
- some data (here, a "transaction")
- `nonce` — a random number used to make hashing hard

Each block's fingerprint depends on the previous block's fingerprint. So if you change anything in block 1, every block after it changes too. That is what makes blockchains tamper-proof.

### Proof of Work

Before a block is added to the chain, the program has to find a `nonce` (a number it can change) such that the block's hash starts with `00`. Because hashing is unpredictable, the only way to find such a nonce is to try millions of numbers.

### XOR Encryption

**XOR** is a bit operation. If you XOR two bytes together, you get a third byte. If you XOR that result with the second byte again, you get the first byte back. So XOR is reversible as long as you know the second byte.

```python
>>> 0b10101010 ^ 0b11110000
0b01011010
>>> 0b01011010 ^ 0b11110000
0b10101010   # back to the original
```

### PKCS#7 Padding

Encryption usually works on fixed-size blocks (here, 16 bytes). If your message is not a multiple of 16 bytes, you pad it. PKCS#7 says: if you need 5 bytes of padding, append `5 5 5 5 5`. If you need 16 bytes, append `16` sixteen times. The last byte of the padded message always tells you how many padding bytes to strip.

---

## Solution — Step by Step

### Step 1 — Download both files

```
┌──(zham㉿kali)-[~/pico/tap_into_hash]
└─$ ls
hash.py  flag.bin
```

`hash.py` is the source code from the challenge. `flag.bin` is the output the program produced when run on the server. It contains two things printed by the program:

```
Key: b'\x8b\x9a\x00G\xfe\xb3\xf3...'
Encrypted Blockchain: b'Z5Wo\xe9\xbd\xf4\xed<...'
```

Wait. The key is just printed in plain text. That is our first big hint — we do not actually need to break the encryption, we just need to undo it.

### Step 2 — Read the source code top to bottom

```
┌──(zham㉿kali)-[~/pico/tap_into_hash]
└─$ nano hash.py
```

(Ctrl+X to leave nano when you are done reading.)

The important parts:

1. `main(token)` is called with the flag as `sys.argv[1]`.
2. It builds a 5-block blockchain (1 genesis + 4 via `proof_of_work`).
3. It joins all 5 block hashes with `-` to get the plaintext:
   ```python
   blockchain_string = blockchain_to_string(all_blocks)
   ```
4. It then calls `encrypt(blockchain_string, token, key)`.

The `encrypt` function does four things:

```python
def encrypt(plaintext, inner_txt, key):
    midpoint = len(plaintext) // 2
    first_part  = plaintext[:midpoint]
    second_part = plaintext[midpoint:]
    modified_plaintext = first_part + inner_txt + second_part  # flag spliced in
    plaintext = pad(modified_plaintext, 16)                    # PKCS#7
    key_hash  = hashlib.sha256(key).digest()                   # 32-byte keystream
    return xor each 16-byte block with key_hash
```

So:
- The plaintext is `5 hashes joined by -` = `64*5 + 4` = **324 characters**.
- The flag is inserted right in the middle, at index `324 // 2` = **162**.
- The result is padded to a multiple of 16, then XORed block-by-block with `SHA256(key)`.

### Step 3 — Check the math

The encrypted output is **384 bytes** (24 blocks of 16). After un-padding, the unpadded length must be between `384 - 16 = 368` and `383`.

- 324 chars from the blockchain + the flag length = unpadded length
- So the flag length is between `368 - 324 = 44` and `383 - 324 = 59`

That is consistent with a typical picoCTF flag.

### Step 4 — Write the solver

```
┌──(zham㉿kali)-[~/pico/tap_into_hash]
└─$ nano solve.py
```

Paste this:

```python
#!/usr/bin/env python3
"""Tap into Hash solver.
The leaked key XORed the plaintext, so we just XOR it back.
The flag was spliced into the middle of a 324-char blockchain hash string.
"""

import hashlib

# From flag.bin: Key: b'\x8b\x9a\x00G\xfe\xb3...'
key = bytes.fromhex(
    "8b9a0047feb3f393dba87954fe158761f4df008deeabd9095e7c042825819ef8"
)

# Paste the Encrypted Blockchain bytes from flag.bin here.
enc = (
    b"Z5Wo\xe9\xbd\xf4\xed<\xeb=\xcb%\xc4\xf0>S2\x0bl\xe9\xe9\xf0\xe8>"
    b"\xe8n\x91q\xca\xad=\x01fRo\xbe\xba\xa2\xb5f\xb88\x90t\xc4\xaco"
    b"\x041\x01n\xb3\xbd\xa1\xb4l\xee9\xc9u\x9e\xf1:O0\x035\xed\xeb\xf6"
    b"\xean\xbei\xcc.\x9f\xfe8\x008\x04l\xef\xee\xf4\xe9g\xbf8\x9bt"
    b"\x9f\xaanZ6\x0b8\xef\xbb\xa6\xb8>\xe9n\xcd$\xc5\xf08\x00eQ>\xef"
    b"\xed\xf4\xeaf\xbdm\x9e \xcb\xfe9\x07-\x03=\xba\xb9\xa2\xbcl\xb4:"
    b"\x9c&\x9e\xab8U4\x05=\xe8\xec\xa4\xeao\xefk\x9c#\x9a\xacn\x00bCd"
    b"\xe8\xb7\xd6\xd8\x19\xf69\xc4x\x9f\xa2UQSae\xdd\xb1\xc7\xee\x0b\xbc*"
    b"\xcbO\xa3\x91_\x08M\x03\x7f\xbf\xe1\xf6\xc4\x00\xfc\x18\xd2z\xb6\x93"
    b"p Kl;\xbb\xee\xa1\xbb9\xef9\xd5t\x99\xf9;Ve\x04i\xb3\xe9\xf3\xbd;"
    b"\xb8m\x91s\xcf\xff:\x019\x04=\xbd\xb9\xa4\xeff\xb5>\xcd:\xcc\xf9<"
    b"Ta\x01?\xbd\xe9\xf0\xed=\xefi\x9fr\x98\xf8=[a\x00:\xbc\xba\xf3\xe9k"
    b"\xeci\x90\"\x98\xfbnZaPk\xed\xe9\xa6\xbf=\xe8>\x99.\xc5\xac?Wa"
    b"\x02<\xbd\xb9\xa0\xeaj\xecn\x9b$\xd1\xf9:\x04dR:\xb9\xea\xa3\xedi"
    b"\xebh\x9d'\xcc\xfah\x008U9\xe8\xe8\xf6\xba>\xefk\x98#\xce\xfb;\x038"
    b"\n?\xea\xb9\xf7\xbfg\xbbk\x90q\x9e\xacn[f\x06l\xef\xbb\xa7\xbaf\xbd"
    b"9\x9a\"\xcf\xcb\x08"
)

key_hash = hashlib.sha256(key).digest()   # 32 bytes — same keystream the server used

# XOR each 16-byte block with the first 16 bytes of key_hash (repeats).
padded = b""
for i in range(0, len(enc), 16):
    block   = enc[i:i + 16]
    decoded = bytes(x ^ y for x, y in zip(block, key_hash))
    padded += decoded

# Strip PKCS#7 padding.
pad_len   = padded[-1]
plaintext = padded[:-pad_len].decode()

print(f"Unpadded length: {len(plaintext)}")
print(f"First 60 chars:  {plaintext[:60]}")
print(f"Last  60 chars:  {plaintext[-60:]}")

# The flag was spliced at the midpoint of the 324-char blockchain string.
ORIG_LEN = 324
mid      = ORIG_LEN // 2
flag     = plaintext[mid : len(plaintext) - mid]
print(f"\nFLAG: {flag}")
```

Save: Ctrl+O, Enter, Ctrl+X.

### Step 5 — Run it

```
┌──(zham㉿kali)-[~/pico/tap_into_hash]
└─$ python3 solve.py
Unpadded length: 382
First 60 chars:  85dbbeaacffc2894128ab1edae59f6d7cfab5b7995c8c8eef12c8e483c
Last  60 chars:  1c98ee-0066a2261eabb27ed179a377bfe4a285d2d8acff133bee199e55a116a5f5a533-00fda7226a6f35003bb8f4c0c6ab004221a892aab38608fbed9f5adc2690b253

FLAG: picoCTF{block_3SRhViRbT1qcX_XUjM0r49cH_qCzmJZzBK_60647fbb}
```

Got the flag.

### Alternative Solve — CyberChef only

If you would rather not write Python:

1. Copy the bytes from `Encrypted Blockchain: b'...'` into a Python REPL once to hex-encode them:
   ```python
   >>> import binascii
   >>> enc = b"Z5Wo..."   # paste
   >>> print(binascii.hexlify(enc).decode())
   ```
2. In CyberChef, drop in **From Hex**, then **XOR** with the keystream bytes `8b 9a 00 47 fe b3 f3 93 db a8 79 54 fe 15 87 61` (the first 16 bytes of `SHA256(key)` — `key_hash` is reused per 16-byte block).
3. The output is the padded plaintext. Strip the last `N` bytes where `N` is the value of the very last byte.
4. Take characters `162` through `162 + flag_length`.

Either way, the answer is the same.

---

## What Happened Internally

Here is the timeline of what the server did when it ran `hash.py`:

1. **`generate_random_string(64)`** — produced a 64-hex-character key.
2. **`print("Key:", key)`** — leaked the key to the output file. This is the bug we exploited.
3. **`Block(0, "0", time, "EncodedGenesisBlock", 0)`** — created the genesis block.
4. **Loop 4 times**: `proof_of_work` brute-forced a `nonce` so the block hash started with `"00"`. Each block's hash used the previous block's hash, so they were all chained.
5. **`blockchain_to_string(all_blocks)`** — joined 5 hashes with `-` → a 324-character string.
6. **`encrypt(324-char string, flag, key)`**:
   - Split the 324 chars in half (162 + 162).
   - Stuck the flag between them → 324 + flag_len chars.
   - Padded with PKCS#7 → multiple of 16.
   - Encoded to UTF-8 bytes.
   - XORed every 16-byte block with the first 16 bytes of `SHA256(key)`.
7. **`print("Encrypted Blockchain:", ...)`** — wrote the ciphertext to `flag.bin`.

On my side, I:

1. **Read the source** to confirm the encryption is reversible.
2. **Copied the leaked key** straight from `flag.bin`.
3. **Re-derived the keystream** with `SHA256(key)`.
4. **XORed the ciphertext** back to the padded plaintext.
5. **Stripped PKCS#7 padding** to get the spliced string.
6. **Carved out the middle** (index 162 to end-162) — that is the flag.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nano` | Read the source file and write the solver script |
| `python3` | Hashing, XOR, padding/unpadding, slicing the flag |
| `hashlib` (stdlib) | Compute `SHA256(key)` — the exact keystream the server used |
| `binascii` (stdlib) | Optional — hex-encode the ciphertext when using CyberChef |

---

## Key Takeaways

- **The biggest lesson**: when source code is given, look for what it accidentally prints. `print("Key:", key)` was the whole challenge. A real-world version would never print the key, but a CTF challenge often gives you one obvious opening.
- **XOR with a known keystream is not encryption**. It is a one-time-pad-style operation, and if you know the keystream, you undo it instantly.
- **PKCS#7 padding** is reversible just by reading the last byte. There is no need to "crack" it.
- **The blockchain was a red herring** — we did not need to mine any blocks, recompute any hashes, or guess any timestamps. The chain existed only to produce a long pseudo-random plaintext that hid where the flag sat.
- **The flag splicing trick** is the actual lesson here: the encryption key is irrelevant once you know the flag length, because you can just split the recovered plaintext at the known midpoint and read what was hidden there.

### Flag wordplay decode

`picoCTF{block_3SRhViRbT1qcX_XUjM0r49cH_qCzmJZzBK_60647fbb}` breaks down as `block_` + four short hex-looking chunks. The word `block` is the only real word — it is a wink at the blockchain theme of the challenge. The four chunks (`3SRhViRbT1qcX`, `XUjM0r49cH`, `qCzmJZzBK`, `60647fbb`) are deliberately chosen to look like pieces of a SHA-256 hash, which is what every block in the chain produces. The trailing `60647fbb` is even shaped like the start of a real hex digest. So the flag is basically a tiny fake "block hash" — a block pulled out of the chain, which is exactly what `Tap into Hash` is asking you to do.
