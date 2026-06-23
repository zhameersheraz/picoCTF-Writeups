# shift registers — picoCTF Writeup

**Challenge:** shift registers  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{l1n3ar_f33dback_sh1ft_r3g}`  
**Platform:** picoCTF (2026)  
**Author (challenge):** Philip Thayer  
**Writeup by:** zham  

---

## Description

> I learned about lfsr today in school so i decided to implement it in my program. It must be safe right? chall.py output.txt

## Hints

> (no hint given)

Two files came with the challenge:

- `chall.py` — the encryption program (an LFSR-based stream cipher)
- `output.txt` — the hex-encoded ciphertext

`chall.py`:

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes
from Crypto.Random import get_random_bytes

key = bytes_to_long(get_random_bytes(126))

def steplfsr(lfsr):
    b7 = (lfsr >> 7) & 1
    b5 = (lfsr >> 5) & 1
    b4 = (lfsr >> 4) & 1
    b3 = (lfsr >> 3) & 1

    feedback = b7 ^ b5 ^ b4 ^ b3
    lfsr = (feedback << 7) | (lfsr >> 1)
    return lfsr

def encrypt_lfsr(pt_bytes):
    output = bytearray()
    lfsr = key & 0xFF
    for p in pt_bytes:
        lfsr = steplfsr(lfsr)
        ks = lfsr
        output.append(p ^ ks)
    return bytes_to_long(bytes(output))

pt = b"[redacted]"
ct = encrypt_lfsr(pt)

print(long_to_bytes(ct).hex())
```

`output.txt`:

```
21c1b705764e4bfdafd01e0bfdbc38d5eadf92991cdd347064e37444e517d661cea9
```

The author generates a 126-byte random key, but only the lowest 8 bits (`key & 0xFF`) ever become the initial LFSR state. So the actual cryptographic state is 8 bits — there are only **256** candidate keystreams, and we can brute force all of them.

---

## Background Knowledge (Read This First!)

If you have never seen an LFSR before, read this section — it is the only theory you need for this challenge.

### 1. Stream Cipher Refresher

A stream cipher encrypts one byte at a time by XORing the plaintext with a pseudo-random keystream:

```
ciphertext[i] = plaintext[i] XOR keystream[i]
```

If you can recover the keystream, you can recover the plaintext (since XOR is its own inverse). A stream cipher is only as secure as its keystream — if the keystream comes from a small state space, the cipher is brute-forceable.

### 2. LFSR (Linear Feedback Shift Register)

An LFSR is a tiny state machine that produces a long pseudo-random bit sequence from a small fixed state. For an 8-bit LFSR:

- The **state** is 8 bits, e.g. `0b10110001`.
- The **feedback** is the XOR of a few chosen bit positions of the current state.
- At each step, every bit shifts one position toward the LSB, the old LSB is discarded, and the feedback bit becomes the new MSB.

In this challenge the taps are at bit positions 7, 5, 4, 3:

```python
b7 = (lfsr >> 7) & 1   # tap
b5 = (lfsr >> 5) & 1   # tap
b4 = (lfsr >> 4) & 1   # tap
b3 = (lfsr >> 3) & 1   # tap
feedback = b7 ^ b5 ^ b4 ^ b3
lfsr = (feedback << 7) | (lfsr >> 1)
```

The characteristic polynomial is `x^8 + x^6 + x^5 + x^4 + 1`, but you do not need to check whether it is primitive — a brute force over all 256 states does not care.

### 3. The Vulnerability in One Sentence

The author generated 126 random bytes, then used **only the lowest 8 bits** as the LFSR state. The other 119 bytes are pure waste — they have zero cryptographic effect. The actual keystream state space is 2^8 = 256, which is trivially brute-forceable.

### 4. Why the Author's "Safety" Reasoning Fails

It is tempting to think "I used 126 bytes of randomness, so my key has 1008 bits of entropy." That is true for the **key generation**, but entropy is only useful if it is actually **consumed** by the cipher. Here `key & 0xFF` collapses 1008 bits of randomness down to 8 bits at the very first line of `encrypt_lfsr`. The remaining 1000 bits are never read again. The cipher is therefore equivalent to an 8-bit key cipher regardless of how big the original key was.

---

## Solution — Step by Step

I worked out of `~/ctf/shift-registers` on my Kali VM.

### Step 1 — Make a Working Directory and Drop the Files In

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/shift-registers && cd ~/ctf/shift-registers

┌──(zham㉿kali)-[~/ctf/shift-registers]
└─$ cp /media/sf_downloads/chall.py .
┌──(zham㉿kali)-[~/ctf/shift-registers]
└─$ cp /media/sf_downloads/output.txt .
```

### Step 2 — Spot the Vulnerability

```
┌──(zham㉿kali)-[~/ctf/shift-registers]
└─$ grep -n "key" chall.py
key = bytes_to_long(get_random_bytes(126))
...
lfsr = key & 0xFF
```

There it is. `key` is 126 bytes (1008 bits), but only `key & 0xFF` ever feeds the cipher. Initial LFSR state has only **256** possible values. We brute force all of them.

### Step 3 — Write the Brute-Force Solver

```
┌──(zham㉿kali)-[~/ctf/shift-registers]
└─$ nano solve_lfsr.py
```

Paste the following inside `nano`:

```python
#!/usr/bin/env python3
"""
shift_registers - picoCTF solver.

The author generated 126 random bytes but only used `key & 0xFF` as the
LFSR initial state. So the actual state space is 2^8 = 256 -- brute
force all of them, generate the keystream, XOR with the ciphertext,
and look for the picoCTF{ prefix.
"""

ct_hex = "21c1b705764e4bfdafd01e0bfdbc38d5eadf92991cdd347064e37444e517d661cea9"
ct = bytes.fromhex(ct_hex)
ct_len = len(ct)


def steplfsr(lfsr):
    b7 = (lfsr >> 7) & 1
    b5 = (lfsr >> 5) & 1
    b4 = (lfsr >> 4) & 1
    b3 = (lfsr >> 3) & 1
    feedback = b7 ^ b5 ^ b4 ^ b3
    return (feedback << 7) | (lfsr >> 1)


def keystream_from(init, n):
    lfsr = init
    out = bytearray()
    for _ in range(n):
        lfsr = steplfsr(lfsr)
        out.append(lfsr & 0xFF)   # ks = lfsr (already 8-bit)
    return bytes(out)


if __name__ == "__main__":
    print(f"[*] ciphertext is {ct_len} bytes; brute-forcing 256 LFSR init states")
    for init in range(256):
        if init == 0:
            continue   # all-zero state is absorbing (always emits 0x00)
        ks = keystream_from(init, ct_len)
        pt = bytes(c ^ k for c, k in zip(ct, ks))
        if pt.startswith(b"picoCTF{") and pt.rstrip(b"\x00").endswith(b"}"):
            try:
                decoded = pt.decode()
            except UnicodeDecodeError:
                continue
            print(f"[+] init = 0x{init:02x}  flag = {decoded}")
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

A few notes on the script:

- I skip `init = 0` because the all-zero state is absorbing: once the LFSR reaches `0b00000000`, the feedback is always 0 and the keystream is permanently `0x00`. The decryption would be identical to the ciphertext, which never starts with `picoCTF{`.
- The two-line `startswith(...)` + `endswith(...)` check is a cheap sanity filter that catches both the prefix and the trailing `}`.
- I `decode()` before printing to make sure the flag is real ASCII. If the candidate fails `decode()`, I just keep looking.

### Step 4 — Run the Solver

```
┌──(zham㉿kali)-[~/ctf/shift-registers]
└─$ python3 solve_lfsr.py

[*] ciphertext is 34 bytes; brute-forcing 256 LFSR init states
[+] init = 0xa2  flag = picoCTF{l1n3ar_f33dback_sh1ft_r3g}
[+] init = 0xa3  flag = picoCTF{l1n3ar_f33dback_sh1ft_r3g}
```

Both `init = 0xa2` and `init = 0xa3` produce the same flag — they generate identical keystream bytes over the 34-byte plaintext window (just one shifted), so both decryptions look identical up to where the ciphertext ends. The flag is unambiguous either way:

```
picoCTF{l1n3ar_f33dback_sh1ft_r3g}
```

### Step 5 — Verify by Hand

```
┌──(zham㉿kali)-[~/ctf/shift-registers]
└─$ python3 -c "
def steplfsr(lfsr):
    b7 = (lfsr >> 7) & 1
    b5 = (lfsr >> 5) & 1
    b4 = (lfsr >> 4) & 1
    b3 = (lfsr >> 3) & 1
    feedback = b7 ^ b5 ^ b4 ^ b3
    return (feedback << 7) | (lfsr >> 1)

ct = bytes.fromhex('21c1b705764e4bfdafd01e0bfdbc38d5eadf92991cdd347064e37444e517d661cea9')
lfsr = 0xa2
pt = bytearray()
for c in ct:
    lfsr = steplfsr(lfsr)
    pt.append(c ^ lfsr)
print(pt.decode())
"
picoCTF{l1n3ar_f33dback_sh1ft_r3g}
```

Done.

---

## What Happened Internally (Timeline)

| Step | What the program did | What I did |
|------|----------------------|------------|
| 1 | Author called `get_random_bytes(126)` to make a 1008-bit key. | — |
| 2 | `encrypt_lfsr()` set `lfsr = key & 0xFF`, dropping 1000 of the 1008 key bits on the floor. | Read `chall.py`, spotted the `& 0xFF` mask — the actual state space is 8 bits. |
| 3 | For each plaintext byte, the LFSR stepped and the new 8-bit state was XORed into the ciphertext. The final byte array was `bytes_to_long`'d and hexlified to `output.txt`. | Parsed the hex back into 34 bytes. |
| 4 | — | Wrote a 256-iteration brute force that re-implements `steplfsr` and XORs each candidate keystream against the ciphertext. |
| 5 | — | Filtered for plaintexts starting with `picoCTF{` and ending with `}`. |
| 6 | — | Two adjacent init states (`0xa2`, `0xa3`) both decrypted to `picoCTF{l1n3ar_f33dback_sh1ft_r3g}`. |

The whole attack is "guess the 8-bit LFSR state by trying all 256 values" — well under a second on any hardware.

---

## Alternative Method 1 — Algebraic LFSR Solve via Berlekamp–Massey

If the plaintext were long enough (≥ 2 × register size = 16 bytes) we could skip the 256-iteration brute force entirely and recover the LFSR feedback polynomial from any 16 known plaintext/ciphertext pairs. With a 16+ byte keystream in hand, **Berlekamp–Massey** recovers the minimal LFSR in linear time:

```
┌──(zham㉿kali)-[~/ctf/shift-registers]
└─$ python3 -c "
# Pseudo: assuming we knew 16+ bytes of plaintext (e.g. via crib 'picoCTF{')
# we could rebuild the LFSR polynomial in O(n^2) and generate the rest.
"
```

We have the prefix `picoCTF{` (8 bytes), which is not quite enough for Berlekamp–Massey on an 8-bit LFSR (we would need 16 keystream bits). With the brute force above we do not need it — but for larger LFSR sizes (16, 32, 64-bit registers) Berlekamp–Massey on a known-plaintext prefix becomes the right hammer.

---

## Alternative Method 2 — Sage `lfsr` Berlekamp–Massey Helper

Sage ships with a built-in `lfsr` function that does the Berlekamp–Massey step for you over GF(2):

```python
# sage
from sage.crypto.lfsr import lfsr_sequence
# Suppose you have 16 keystream bits k[0..15]:
# (connection_polynomial, feedback_sequence) = lfsr_sequence(k, return_polynomial=True)
# connection_polynomial.reverse() gives the characteristic polynomial.
```

For an 8-bit LFSR this is overkill, but if you ever face a 64-bit LFSR challenge the same Berlekamp–Massey approach scales without rewriting the math.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` / `grep` | Inspect `chall.py` and confirm `key & 0xFF` |
| `nano` | Write the brute-force solver script |
| Python 3 | Iterated over 256 candidates and decoded the flag |

---

## Key Takeaways

- **Generating entropy is not the same as consuming it.** The author used 126 random bytes but only read 8 of them. The other 119 bytes contributed zero cryptographic strength. Always trace every bit of your key from generation to its final use.
- **Mask off the unused bits, mentally.** When you see `key & 0xFF` in a stream cipher, the state space is at most 256. When you see `key & 0xFFFFFFFF`, the state space is at most 2^32. Brute force ranges apply even when the surrounding code looks "big and scary."
- **LFSRs are bit-efficient, not byte-efficient.** A properly designed LFSR stream cipher uses a long register (≥ 128 bits) AND combines several LFSRs together to break the linearity (e.g. the GSM A5/1 cipher uses three LFSRs). A single short LFSR with raw XOR is trivially broken the moment you know any 2L bits of keystream.
- **Filter brute-force output, do not eyeball it.** The two-line `startswith("picoCTF{") and endswith("}")` check catches false positives without human inspection and scales if you ever face a larger (but still brute-forceable) state space.
- **If the cipher state is ≤ 2^32, your laptop wins.** Brute-forcing 256 candidates is instant. Brute-forcing 2^32 candidates is minutes. Anything ≤ 2^40 is at worst overnight on commodity hardware. Treat every "small state" stream cipher as broken.

### Flag Wordplay Decode

The flag is `picoCTF{l1n3ar_f33dback_sh1ft_r3g}`. Read the inner word in four parts:

- `l1n3ar` → "linear" with `1` for `i` and `3` for `e`.
- `f33dback` → "feedback" with `3` for both `e`'s.
- `sh1ft` → "shift" with `1` for `i`.
- `r3g` → "reg" with `3` for `e` (short for register).

So the body decodes to **"linear feedback shift reg"** — literally describing the LFSR (Linear Feedback Shift Register) that the challenge implements. The flag is a wink at the cipher you just broke.
