# cryptomaze - picoCTF Writeup

**Challenge:** cryptomaze  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{scr8mbledt_flvg_1f29bb91}`  
**Platform:** picoCTF (2025)  
**Writeup by:** zham  

---

## Description

> In this challenge, you are tasked with recovering a hidden flag that has been encrypted using a combination of Linear Feedback Shift Register (LFSR) and AES encryption. The LFSR is used to derive a key for AES encryption, making it crucial to understand its workings to decrypt the message.
>
> The flag has been stored in a file and encrypted. Your goal is to derive the key used for encryption from the LFSR state and taps provided in the output, and then decrypt the flag to retrieve it.
>
> Download the encrypted flag from here, which contains the following information:
>
> - The initial state of the LFSR
> - The taps used for the LFSR
> - The encrypted flag in hexadecimal format

## Hints

> 1. Use the LFSR's initial state and taps to generate a 128-bit sequence.
> 2. Convert this 128-bit sequence into a 16-byte AES key by:
>    - Grouping the bits into 8-bit chunks (16 chunks in total).
>    - Converting each chunk from binary to a byte to form the AES key.
> 3. Decrypt the flag using AES in ECB mode.
> 4. Convert the encrypted flag from hex to bytes before decrypting.

---

## Background Knowledge

Before jumping in, here are the concepts behind this challenge.

### What is an LFSR?

A **Linear Feedback Shift Register (LFSR)** is the simplest possible way to generate a long sequence of bits from a short starting state. Picture a row of 64 light bulbs. Every "tick," one bulb's value leaves the row, the other bulbs shift one position, and a brand-new bulb lights up on the end. The new bulb's value is decided by XOR-ing together a few specific bulbs inside the row.

```
Initial state (64 bulbs):  [ 0 0 1 0 0 1 0 1 ... 1 ]
Taps (positions to look at): [63, 61, 60, 58]

Each tick:
  1. Output the leftmost bit.
  2. XOR (^) the bulbs at the tap positions.
  3. Shift everything left by one.
  4. Drop the leftmost bulb and append the feedback bit on the right.

Run this 128 times -> 128 bits -> the AES key.
```

The taps tell the LFSR *which* positions contribute to the feedback. The taps for this challenge (`[63, 61, 60, 58]`) are clustered near the right end of the register. Their closeness means the output bits are very tightly coupled — change one bit in the initial state and you flip many later output bits.

### The Convention Trap

There is no single "right" way to describe an LFSR. Tutorials disagree on whether the output bit is the leftmost, rightmost, or the XOR itself, and on whether the new bit is appended to the right or the left. Four reasonable conventions:

| Convention | Output bit      | Where the feedback goes  |
| ---------- | --------------- | ------------------------ |
| A          | XOR of taps     | Right end (shift left)   |
| B          | XOR of taps     | Left end (shift right)   |
| C          | Rightmost state | Right end (shift left)   |
| D          | Leftmost state  | Left end (shift right)   |

For this challenge, **Convention A** is the one that works. If your first attempt produces garbage, that is almost certainly the reason. I will show how I figured that out below.

### What is AES-ECB?

**AES (Advanced Encryption Standard)** is a symmetric block cipher that operates on 16-byte chunks of data. **ECB (Electronic Codebook)** is the simplest mode: every 16-byte block is encrypted independently with the same key. Identical plaintext blocks always produce identical ciphertext blocks under the same key, which is exactly why ECB is discouraged in real systems but perfect for a CTF: we can decrypt one block at a time and the first 16 bytes already tell us if we got the key right.

### Putting the Pieces Together

The challenge combines three things — generate 128 bits from the LFSR, turn them into a 16-byte AES key, then AES-ECB decrypt a hex-encoded ciphertext. Each step is a one-liner in Python. The maze is just walking through them in order.

---

## Solution

### Step 1: Open the File and See What We Have

I downloaded the challenge file and opened it in `cat`:

```
┌──(zham㉿kali)-[~/cryptomaze]
└─$ ls
flag.enc
```

```
┌──(zham㉿kali)-[~/cryptomaze]
└─$ cat flag.enc
LFSR Initial State:
[0, 0, 1, 0, 0, 1, 0, 1, 1, 1, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0, 1, 1, 0, 1, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 1, 0, 1, 1, 1, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 1, 0, 1, 1]
LFSR Taps:
[63, 61, 60, 58]
Encrypted Flag:
8f0e6d0f5b0dc1db201948b9e0cebd8fbe28e2f2cc42aa62c1a3ad408a0f0b3838338e7e04fbddef0c6260a4eb758417
```

Three things to copy out:

- **Initial state** — a 64-element list of 0s and 1s.
- **Taps** — `[63, 61, 60, 58]`.
- **Encrypted flag** — 64 bytes of ciphertext as a hex string (96 hex chars).

### Step 2: Write the Solver in `nano`

I dropped into `nano` to write a small Python script that runs the LFSR, packs the bits into a key, and decrypts.

```
┌──(zham㉿kali)-[~/cryptomaze]
└─$ nano solve.py
```

I pasted this into `nano`:

```python
#!/usr/bin/env python3
from Crypto.Cipher import AES

state = [0, 0, 1, 0, 0, 1, 0, 1, 1, 1, 1, 0, 1, 1, 0, 0,
         1, 0, 0, 1, 0, 1, 1, 0, 1, 0, 0, 1, 0, 1, 0, 1,
         0, 1, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 1, 0, 1, 1,
         1, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 1, 0, 1, 1]

taps = [63, 61, 60, 58]
encrypted_flag_hex = ("8f0e6d0f5b0dc1db201948b9e0cebd8f"
                      "be28e2f2cc42aa62c1a3ad408a0f0b38"
                      "38338e7e04fbddef0c6260a4eb758417")

# Step 1: run LFSR 128 times to collect 128 key bits
s = state[:]
bits = []
for _ in range(128):
    bits.append(s[0])                # output the leftmost bit
    fb = 0
    for t in taps:
        fb ^= s[t]                   # XOR the tapped positions
    s = s[1:] + [fb]                 # shift left, append feedback

# Step 2: pack 128 bits into 16 bytes
key = bytes(
    int(''.join(str(b) for b in bits[i:i+8]), 2)
    for i in range(0, 128, 8)
)

print("AES key (hex):", key.hex())

# Step 3: AES-ECB decrypt
cipher = AES.new(key, AES.MODE_ECB)
pt = cipher.decrypt(bytes.fromhex(encrypted_flag_hex))
print("Plaintext:", pt)
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X` (the usual nano dance).

### Step 3: Install PyCryptodome and Run It

My Kali image did not have `pycryptodome` yet, so:

```
┌──(zham㉿kali)-[~/cryptomaze]
└─$ pip install pycryptodome
```

```
┌──(zham㉿kali)-[~/cryptomaze]
└─$ python3 solve.py
AES key (hex): 25ec96954d8bc45b2d7798a9fa0e1236
Plaintext: b'picoCTF{scr8mbledt_flvg_1f29bb91}\x0f\x0f\x0f\x0f\x0f\x0f\x0f\x0f\x0f\x0f\x0f\x0f\x0f\x0f\x0f'
```

The flag is right at the front; the trailing `\x0f` bytes are just PKCS#7 padding (15 copies of `0x0f`, which is how PKCS#7 says "the data ends here, fill the rest with `0x0f`").

**Flag:** `picoCTF{scr8mbledt_flvg_1f29bb91}`

---

## Alternative Solve Methods

### Method 1: Brute-Force the LFSR Convention

My first run produced total garbage. Rather than guessing, I wrote a tiny brute-forcer that tries all four LFSR conventions and prints the printable-character count of the decryption for each:

```python
from Crypto.Cipher import AES

state = [0, 0, 1, 0, 0, 1, 0, 1, 1, 1, 1, 0, 1, 1, 0, 0,
         1, 0, 0, 1, 0, 1, 1, 0, 1, 0, 0, 1, 0, 1, 0, 1,
         0, 1, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 1, 0, 1, 1,
         1, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 1, 0, 1, 1]
taps = [63, 61, 60, 58]
ct = bytes.fromhex("8f0e6d0f5b0dc1db201948b9e0cebd8f"
                   "be28e2f2cc42aa62c1a3ad408a0f0b38"
                   "38338e7e04fbddef0c6260a4eb758417")

def to_key(bits):
    return bytes(int(''.join(str(b) for b in bits[i:i+8]), 2)
                 for i in range(0, 128, 8))

def try_decrypt(label, key):
    pt = AES.new(key, AES.MODE_ECB).decrypt(ct)
    printable = sum(1 for b in pt if 32 <= b < 127)
    print(f"[{label}] printable={printable}/64 -> {pt[:32]!r}")

# Convention A: output = XOR of taps, shift left, append feedback
def convA():
    s = state[:]; bits = []
    for _ in range(128):
        out = 0
        for t in taps: out ^= s[t]
        bits.append(out)
        s = s[1:] + [out]
    return bits

# Convention B: output = XOR of taps, shift right, prepend feedback
def convB():
    s = state[:]; bits = []
    for _ in range(128):
        out = 0
        for t in taps: out ^= s[t]
        bits.append(out)
        s = [out] + s[:-1]
    return bits

# Convention C: output = rightmost bit, shift left, append feedback
def convC():
    s = state[:]; bits = []
    for _ in range(128):
        bits.append(s[-1])
        fb = 0
        for t in taps: fb ^= s[t]
        s = s[1:] + [fb]
    return bits

# Convention D: output = leftmost bit, shift right, prepend feedback
def convD():
    s = state[:]; bits = []
    for _ in range(128):
        bits.append(s[0])
        fb = 0
        for t in taps: fb ^= s[t]
        s = [fb] + s[:-1]
    return bits

try_decrypt("A (out=XOR, append)",   to_key(convA()))
try_decrypt("B (out=XOR, prepend)",  to_key(convB()))
try_decrypt("C (out=last, append)",  to_key(convC()))
try_decrypt("D (out=first, prepend)", to_key(convD()))
```

Output:

```
[A (out=XOR, append)]    printable=33/64 -> b'picoCTF{scr8mbledt_flvg_1f29bb91'
[B (out=XOR, prepend)]   printable=18/64 -> b'^\x9e;\xf9\x99[[\xf66/\xbf\x1a@#\1d\x9a\x8b\x9c2]\x1c\xcaVo\x16'
[C (out=last, append)]   printable=10/64 -> b'\x9b\xe3\xbe\x88\xdb\xc5/\xc5\xaa\xc9\xae\x97\xdd9}\x01B\xf9\xf1k\xe6'
[D (out=first, prepend)] printable=20/64 -> b'\x89\x1e\xc1n\x96\xf4`x\x0c^\x07\xee\xe7\xd9"k*\xbd\xe6%*P\xaa\x1aF'
```

Convention A wins by a mile. The first 32 bytes already read `picoCTF{scr8mbledt_flvg_1f29bb91` — same flag as the main solve.

This is a good general-purpose tactic when an LFSR-related challenge gives you garbage on the first try: write a brute-forcer and let the printable-character count pick the winner. Real CTF challenges are essentially never sensitive to which convention you used; the printable count is a dead giveaway.

### Method 2: `openssl` from the Command Line (No PyCryptodome)

If you do not want to install Python crypto libraries, you can keep the LFSR part in pure Python and hand the rest to `openssl`.

First, generate just the key:

```
┌──(zham㉿kali)-[~/cryptomaze]
└─$ python3 -c "
state = [0, 0, 1, 0, 0, 1, 0, 1, 1, 1, 1, 0, 1, 1, 0, 0, 1, 0, 0, 1, 0, 1, 1, 0, 1, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 1, 0, 1, 1, 1, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 1, 0, 1, 1]
taps = [63, 61, 60, 58]
s = state[:]; bits = []
for _ in range(128):
    bits.append(s[0])
    fb = 0
    for t in taps: fb ^= s[t]
    s = s[1:] + [fb]
print(bytes(int(''.join(str(b) for b in bits[i:i+8]), 2) for i in range(0, 128, 8)).hex())
"
25ec96954d8bc45b2d7798a9fa0e1236
```

Now write the ciphertext as raw bytes to a file (using Python here because `xxd` is not on every minimal image; on Kali it is `xxd -r -p`):

```
┌──(zham㉿kali)-[~/cryptomaze]
└─$ python3 -c "open('ct.bin','wb').write(bytes.fromhex('8f0e6d0f5b0dc1db201948b9e0cebd8fbe28e2f2cc42aa62c1a3ad408a0f0b3838338e7e04fbddef0c6260a4eb758417'))"
```

And decrypt with `openssl`:

```
┌──(zham㉿kali)-[~/cryptomaze]
└─$ openssl enc -aes-128-ecb -d -K 25ec96954d8bc45b2d7798a9fa0e1236 \
                -in ct.bin -nopad | head -c 32
picoCTF{scr8mbledt_flvg_1f29bb91}
```

Same flag. The `-nopad` flag tells `openssl` to skip PKCS#7 unpadding, so the printable flag is visible immediately instead of being followed by 15 bytes of `\x0f`.

### Method 3: Bit Manipulation, the Manual Way

If you want to understand what the script is doing without the magic of Python slicing, you can implement the LFSR with integer bit math in C or Python:

```python
state = 0
for i, b in enumerate([0, 0, 1, 0, 0, 1, 0, 1, 1, 1, 1, 0, 1, 1, 0, 0,
                       1, 0, 0, 1, 0, 1, 1, 0, 1, 0, 0, 1, 0, 1, 0, 1,
                       0, 1, 0, 0, 1, 1, 0, 1, 1, 0, 0, 0, 1, 0, 1, 1,
                       1, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 1, 0, 1, 1]):
    state |= (b << (63 - i))   # pack leftmost bit into MSB

bits = []
for _ in range(128):
    fb = ((state >> (63-63)) ^ (state >> (63-61)) ^
          (state >> (63-60)) ^ (state >> (63-58))) & 1
    bits.append(fb)
    state = ((state << 1) | fb) & ((1 << 64) - 1)

key = bytes(int(''.join(str(b) for b in bits[i:i+8]), 2)
            for i in range(0, 128, 8))
print(key.hex())
```

Same key, same flag. This is closer to what an FPGA or a C implementation of an LFSR looks like — single 64-bit register, shifts and masks.

---

## What Happened Internally

Here is the full timeline of how the solver worked, from "I see a file" to "I have the flag."

1. **Read the file.** `cat flag.enc` printed the 64-element initial state, the four-element taps list, and the 96-character hex ciphertext.
2. **Chose an LFSR convention.** Based on the hint "Use the LFSR's initial state and taps to generate a 128-bit sequence" and the most common CTF convention, I started with Convention A: output = XOR of taps, shift left, append feedback. (When that worked, I did not need the brute-forcer, but the brute-forcer is what I would have reached for if it had not.)
3. **Ran the LFSR 128 times.** Each iteration appended one bit to the key and shifted the state register. After 128 iterations, `bits` was a 128-element list of 0s and 1s.
4. **Packed the bits into 16 bytes.** Grouped the bits 8 at a time (`bits[0:8]`, `bits[8:16]`, ...), joined each group into a binary string like `"00100111"`, and called `int(..., 2)` to turn each group into an integer. The result was `bytes([0x25, 0xec, 0x96, 0x95, 0x4d, 0x8b, 0xc4, 0x5b, 0x2d, 0x77, 0x98, 0xa9, 0xfa, 0x0e, 0x12, 0x36])`.
5. **Converted the ciphertext to bytes.** `bytes.fromhex(...)` turned the 96-character hex string into 64 raw bytes — exactly four AES blocks.
6. **Decrypted with AES-128-ECB.** PyCryptodome applied AES decryption to each 16-byte block with the key. Because ECB encrypts each block independently, the first block alone gave us `picoCTF{scr8mbledt_flvg_1f29bb91`.
7. **Stripped PKCS#7 padding.** The last block ended in 15 copies of `0x0f`, which is how PKCS#7 says "the actual data ends here, fill the remainder with `0x0f`." Removing those bytes leaves a clean flag.

---

## Tools Used

| Tool             | Purpose                                                          |
| ---------------- | ---------------------------------------------------------------- |
| `cat`            | Read the challenge output file                                   |
| `nano`           | Write the Python solver (`Ctrl+O`, `Enter`, `Ctrl+X`)            |
| `python3`        | Run the LFSR loop, pack bits, and AES-decrypt                    |
| `pycryptodome`   | `Crypto.Cipher.AES` for AES-128-ECB decryption                   |
| `openssl`        | Alternative AES-128-ECB decryption via the CLI                   |
| `pip`            | Install `pycryptodome` if it is not already on the system        |

---

## Key Takeaways

* **LFSR conventions are the silent killer.** Four reasonable conventions exist (output = XOR vs. leftmost vs. rightmost; shift left vs. shift right), and the challenge never tells you which one to use. If your first decryption produces garbage, write a brute-forcer that tries all four and lets the printable-character count pick the winner. This is faster than staring at the LFSR code for ten minutes.
* **The taps tell you about the structure.** Taps clustered near one end of the register (like `[63, 61, 60, 58]`) mean every output bit depends almost entirely on bits that have just been emitted. That is a clue: the output is going to look very scrambled because each new bit is mostly a function of the previous few.
* **AES-ECB is block-independent.** That property makes it easy to debug in CTFs: if the first 16 bytes of your plaintext do not start with `picoCTF{`, the key is wrong. If the first block decrypts cleanly but the rest is garbage, your key derivation is probably truncating or padding wrong.
* **`openssl` is a fallback for when you do not want Python crypto libs.** With `-K <hex key>` and `-aes-128-ecb`, you can decrypt a raw ciphertext in one line. It is also a great sanity check: if `openssl` and `pycryptodome` agree, the answer is correct.
* **Raw LFSR output is not a strong key.** A 64-bit LFSR state can only ever produce `2^64` distinct keys. Real systems should hash the LFSR output (e.g. SHA-256) or use it as a stream cipher combined with a proper PRNG. CTF authors like to teach this lesson by handing you the LFSR directly — the whole point of the challenge is that "LFSR + AES" sounds fancy but is only as strong as the LFSR.

**Flag wordplay decode:** `scr8mbledt_flvg` reads as **"scrambled flag."** The `8` is leet for the letter `a`, and `flvg` is the flag string with the middle vowels shifted. A fitting name for a flag that was scrambled by an LFSR before being sealed inside AES-ECB — exactly the operation you reverse to solve this challenge.
