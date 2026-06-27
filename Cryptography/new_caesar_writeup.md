# New Caesar — picoCTF Writeup

**Challenge:** New Caesar  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 60  
**Flag:** `picoCTF{et_tu?_77866c61}`  
**Platform:** picoCTF (2021)  
**Writeup by:** zham  

---

## Description

> We found a brand new type of encryption, can you break the secret code? (Wrap with picoCTF{})
>
> Encrypted message: `fegdeogdgecoeocgcgchcfcffccfca`
>
> Source file: `new_caesar.py`

The challenge ships the encryption script itself, which is almost always the most useful artifact in a custom-cipher challenge — once you can see the encryption, decryption is usually just running the operations in reverse.

> **Note on copy-paste:** picoCTF regenerates this challenge per instance, so the ciphertext and the hex suffix inside the flag change every time you open it. The cipher mechanics stay the same, but always re-copy the ciphertext straight from the challenge page on the run you are solving.

## Hints

> 1. How does the cipher work if the alphabet isn't 26 letters?
> 2. Even though the letters are split up, the same paradigms still apply

Hint 1 tells us the alphabet is shorter than 26 letters (in fact, exactly 16). Hint 2 tells us the structure of classical Caesar still applies — we just need to shift over a smaller ring.

---

## Background Knowledge (Read This First!)

### 1. The Classic Caesar Cipher

In the standard Caesar cipher, every letter is shifted by a fixed number of positions in the 26-letter English alphabet:

```
plaintext:  h e l l o
shift +3:   k h o o r
```

Decryption is shifting by `-k` (or equivalently `26 - k`). The "key" is the shift amount, and there are only 26 candidates — so you brute-force.

### 2. What This Challenge Changes

The author swapped the 26-letter alphabet for a **16-letter alphabet** `a..p`. That means:

- The Caesar shift now happens in **mod 16** instead of mod 26.
- There are only **16 possible keys**, not 26 — even easier to brute-force.

But there is a second trick: before the Caesar shift happens, every plaintext character is first **encoded into two letters of the 16-letter alphabet**. Why? Because the alphabet only has 16 letters, so it can only represent 16 distinct values — enough to encode 4 bits, not a full byte. So each plaintext byte (8 bits) is split into **two 4-bit nibbles**, and each nibble is mapped to one letter of the 16-letter alphabet.

This is exactly what the variable name `b16_encode` suggests: **base-16 (hex) encoding**, but using the letters `a..p` instead of the usual `0..9a..f`.

Putting it all together:

```
plaintext byte (8 bits)
  -> split into two 4-bit nibbles
  -> nibble 0..15 mapped to ALPHABET[0..15]  =  a..p   (this is b16_encode)
  -> each letter shifted by key[i % len(key)] in ALPHABET (this is the Caesar step)
  -> final ciphertext
```

Because b16 doubles every byte, an N-byte plaintext produces a **2N-character** ciphertext. If your ciphertext has any other length, you almost certainly have a copy-paste error and need to re-check the challenge page.

### 3. The Key Length

The script asserts `len(key) == 1`. So the same single shift is applied to every letter of the ciphertext. That collapses the entire keyspace to exactly 16 possibilities — one for each letter `a` through `p`.

### 4. Reversing the Cipher

Reversing is just running the steps backward:

1. **Undo the Caesar shift** with each candidate key `k` from `a` to `p`. If the original shift was `+k`, the reverse is `-k` (mod 16).
2. **Decode the b16 layer.** Take each pair of letters, look up their indices in `ALPHABET` (so `a=0, b=1, ..., p=15`), combine them into a byte: `hi_nibble * 16 + lo_nibble`, then turn that byte into a character with `chr()`.
3. **Pick the candidate whose decoded plaintext is readable ASCII** (and, ideally, looks like part of a picoCTF flag).

---

## Solution — Step by Step

### Step 1 — Read the Source

I opened the attached `new_caesar.py` and read through it carefully. The interesting pieces:

```python
ALPHABET = string.ascii_lowercase[:16]      # "abcdefghijklmnop"
LOWERCASE_OFFSET = ord("a")

def b16_encode(plain):
    enc = ""
    for c in plain:
        binary = "{0:08b}".format(ord(c))
        enc += ALPHABET[int(binary[:4], 2)]
        enc += ALPHABET[int(binary[4:], 2)]
    return enc

def shift(c, k):
    t1 = ord(c) - LOWERCASE_OFFSET
    t2 = ord(k) - LOWERCASE_OFFSET
    return ALPHABET[(t1 + t2) % len(ALPHABET)]

assert len(key) == 1
```

Confirmed: 16-letter alphabet, single-character key, hex-then-Caesar structure.

### Step 2 — Write the Solver

I wrote a Python solver that brute-forces all 16 keys. The full file is `solve.py`:

```
┌──(zham㉿kali)-[~/picoctf/new-caesar]
└─$ nano solve.py
```

I pasted this:

```python
#!/usr/bin/env python3
"""
Brute-force the New Caesar cipher.

The key is a single character from ALPHABET (a..p), so we only have
16 candidates. For each candidate, undo the Caesar shift, then decode
the b16 layer (pair of letters -> byte -> character).
"""
import string

ALPHABET = string.ascii_lowercase[:16]
LOWERCASE_OFFSET = ord("a")

def b16_decode(enc):
    """Pair of letters from ALPHABET -> one byte -> one character."""
    out = ""
    for i in range(0, len(enc), 2):
        hi = ALPHABET.index(enc[i])
        lo = ALPHABET.index(enc[i + 1])
        out += chr(hi * 16 + lo)
    return out

def unshift(c, k):
    t1 = ord(c) - LOWERCASE_OFFSET
    t2 = ord(k) - LOWERCASE_OFFSET
    return ALPHABET[(t1 - t2) % 16]

cipher = "fegdeogdgecoeocgcgchcfcffccfca"

print(f"Ciphertext ({len(cipher)} chars -> {len(cipher) // 2} byte plaintext):\n")
print(f"{'key':<5} {'printable':<11} plaintext")
print("-" * 60)
for key in ALPHABET:
    unshifted = "".join(unshift(c, key) for c in cipher)
    plain = b16_decode(unshifted)
    printable = all(32 <= ord(ch) <= 126 for ch in plain)
    print(f"{key:<5} {str(printable):<11} {plain!r}")
```

Save and exit:

```
^O   (WriteOut)
File Name to Write: solve.py -> press Enter
^X   (Exit)
```

### Step 3 — Run the Solver

```
┌──(zham㉿kali)-[~/picoctf/new-caesar]
└─$ python3 solve.py
Ciphertext (30 chars -> 15 byte plaintext):

key   printable   plaintext
------------------------------------------------------------
a     True        "TcNcd.N&&'%%R% "
b     False       'CR=RS\x1d=\x15\x15\x16\x14\x14A\x14\x1f'
c     False       '2A,AB\x0c,\x04\x04\x05\x03\x030\x03\x0e'
d     False       '!0\x1b01\xfb\x1b\xf3\xf3\xf4\xf2\xf2/\xf2\xfd'
e     False       '\x10/\n/ \xea\n\xe2\xe2\xe3\xe1\xe1\x1e\xe1\xec'
f     False       '\x0f\x1e\xf9\x1e\x1f\xd9\xf9\xd1\xd1\xd2\xd0\xd0\r\xd0\xdb'
g     False       '\xfe\r\xe8\r\x0e\xc8\xe8\xc0\xc0\xc1\xcf\xcf\xfc\xcf\xca'
h     False       '\xed\xfc\xd7\xfc\xfd\xb7\xd7\xbf\xbf\xb0\xbe\xbe\xeb\xbe\xb1'
i     False       '\xdc\xeb\xc6\xeb\xec\xa6\xc6\xae\xae\xaf\xad\xad\xda\xad\xa8'
j     False       '\xcb\xda\xb5\xda\xdb\x95\xb5\x9d\x9d\x9e\x9c\x9c\xc9\x9c\x97'
k     False       '\xba\xc9\xa4\xc9\xca\x84\xa4\x8c\x8c\x8d\x8b\x8b\xb8\x8b\x86'
l     False       '\xa9\xb8\x93\xb8\xb9s\x93{{|zz\xa7zu'
m     False       '\x98\xa7\x82\xa7\xa8b\x82jjkii\x96id'
n     False       '\x87\x96q\x96\x97QqYYZXX\x85XS'
o     False       'v\x85`\x85\x86@`HHIGGtGB'
p     True        'et_tu?_77866c61'
```

Two candidates produced printable ASCII: keys `a` and `p`. Key `a` gave `TcNcd.N&&'%%R% ` (gibberish). Key `p` gave `et_tu?_77866c61`, which is a phrase with English words — and as a sanity check, **"et tu?"** is famously the Latin phrase Julius Caesar supposedly said when he was assassinated ("and you?"). A perfect thematic pun for a Caesar cipher challenge.

### Step 4 — Verify by Re-encrypting

I confirmed the flag by running the original encryption forward on `et_tu?_77866c61` with key `p` and confirming I got the original ciphertext back. The verify script:

```
┌──(zham㉿kali)-[~/picoctf/new-caesar]
└─$ nano verify.py
```

```python
#!/usr/bin/env python3
"""Verify the flag by re-encrypting the recovered plaintext."""
import string

ALPHABET = string.ascii_lowercase[:16]
LOWERCASE_OFFSET = ord("a")

def b16_encode(plain):
    enc = ""
    for c in plain:
        b = "{0:08b}".format(ord(c))
        enc += ALPHABET[int(b[:4], 2)]
        enc += ALPHABET[int(b[4:], 2)]
    return enc

def shift(c, k):
    t1 = ord(c) - LOWERCASE_OFFSET
    t2 = ord(k) - LOWERCASE_OFFSET
    return ALPHABET[(t1 + t2) % len(ALPHABET)]

plain = "et_tu?_77866c61"
key = "p"
b16 = b16_encode(plain)
enc = "".join(shift(b16[i], key) for i in range(len(b16)))
print(f"Re-encrypted: {enc!r}")
print(f"Expected:     {'fegdeogdgecoeocgcgchcfcffccfca'!r}")
print(f"Match: {enc == 'fegdeogdgecoeocgcgchcfcffccfca'}")
print(f"\nFlag: picoCTF{{{plain}}}")
```

```
┌──(zham㉿kali)-[~/picoctf/new-caesar]
└─$ python3 verify.py
Re-encrypted: 'fegdeogdgecoeocgcgchcfcffccfca'
Expected:     'fegdeogdgecoeocgcgchcfcffccfca'
Match: True

Flag: picoCTF{et_tu?_77866c61}
```

The match is exact, so the flag is correct.

### Step 5 — Submit

Wrap the inner content in `picoCTF{}` (the challenge description tells you the cipher only protects the inner content — you add the wrapper yourself):

```
picoCTF{et_tu?_77866c61}
```

Submitted, accepted.

---

## What Happened Internally (Timeline)

1. **Read `new_caesar.py`.** Identified `ALPHABET = "a..p"` (16 chars) and `len(key) == 1`.
2. **Recognized the structure.** It is **b16 encode -> Caesar shift**. Each plaintext byte becomes two `a..p` letters, then each letter is shifted by the same key character.
3. **Wrote reverse functions.** `unshift(c, k)` does `(idx_c - idx_k) mod 16`, and `b16_decode()` joins nibble pairs into bytes.
4. **Brute-forced all 16 candidates** for the key.
5. **Filtered for printable ASCII.** Two candidates survived; `p` produced a sensible English phrase.
6. **Verified by forward encryption** — re-encrypting the recovered plaintext with key `p` produced the exact original ciphertext.
7. **Wrapped with `picoCTF{}`** per the challenge description and submitted.

---

## Alternative Method — One-Liner with Python `-c`

If you do not want to create a script file, you can solve it inline. Here is the entire attack in a single Python invocation:

```
┌──(zham㉿kali)-[~/picoctf/new-caesar]
└─$ python3 -c "
import string
A = string.ascii_lowercase[:16]; L = ord('a')
c = 'fegdeogdgecoeocgcgchcfcffccfca'
for k in A:
    s = ''.join(A[(ord(x) - L - (ord(k) - L)) % 16] for x in c)
    p = ''.join(chr(A.index(s[i])*16 + A.index(s[i+1])) for i in range(0, len(s), 2))
    if all(32 <= ord(ch) <= 126 for ch in p):
        print(repr(k), repr(p))
"
'p' 'et_tu?_77866c61'
```

Same answer, no script file.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `new_caesar.py` | The challenge-provided encryption script |
| Python 3 | Reverse the cipher and brute-force the 16 possible keys |
| `nano` | Edit the solver script in the terminal |
| Manual inspection | Pick the candidate whose plaintext looks like English |

---

## Key Takeaways

- The flag wordplay: **"et tu?"** is a Latin phrase — supposedly Julius Caesar's last words to his friend Brutus during his assassination ("and you?"). Combined with the challenge title **New Caesar**, the pun is unmistakable. The hex suffix `77866c61` is just a unique challenge tag that picoCTF regenerates per instance.
- picoCTF regenerates this challenge every time you open it, so the ciphertext and the hex suffix inside the flag change between sessions. The cipher mechanics and the key (`p`) are stable, but always re-copy the ciphertext straight from the live challenge page before running the solver.
- Any custom cipher that uses a small keyspace is a brute-force target. Here the key was 1 character from a 16-character alphabet — exactly 16 candidates — so no math cleverness is required, just `for k in alphabet`.
- Even when a cipher looks exotic, look for the underlying structure. This one is "hex-then-Caesar" — both well-known primitives stacked. The hints literally tell you this: Hint 1 points at the small alphabet (16, not 26), Hint 2 says the same old paradigms apply.
- When the encryption script is provided, your first move is always to write the decryption as the inverse of each step in order. If each step is invertible (here: Caesar is invertible, b16 is invertible), the whole cipher is invertible.
- A b16-then-Caesar cipher must produce ciphertext of even length (exactly `2 * len(plaintext)`). If your copied ciphertext has a different length, you almost certainly have a stray character from the screenshot — re-copy from the challenge page before debugging the math.
- The challenge description tells you the cipher only protects the inner content and you must add the `picoCTF{}` wrapper yourself. Always re-read the description before submitting — the format hint is the easiest thing to miss.

**Flag:** `picoCTF{et_tu?_77866c61}`
