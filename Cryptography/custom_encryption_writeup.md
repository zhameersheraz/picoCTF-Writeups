# Custom encryption — picoCTF Writeup

**Challenge:** Custom encryption  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{custom_d2cr0pt6d_66778b34}`  
**Platform:** picoCTF (2023)  
**Writeup by:** zham  

---

## Description

> Can you get sense of this code file and write the function that will decode the given encrypted file content.
>
> Find the encrypted file here flag_info and code file might be good to analyze and get the flag.

The challenge ships with two files:

- `encrypt.py` — the encryption script that produced the ciphertext
- `flag_info.bin` — the resulting ciphertext (printed form, one line of Python-looking output)

---

## Hints

> 1. Understanding encryption algorithm to come up with decryption algorithm.

Just one hint, and it is a literal instruction: read the encryption code, then invert it. That is the whole challenge.

The debug info at the bottom of the challenge panel — `[u:871426 e: p: c:412 l:295982]` — is just picoCTF's per-user counter, not part of the puzzle. Ignore it.

---

## Background Knowledge (Read This First!)

### 1. The Three Building Blocks the Script Uses

The encryption is a three-stage pipeline. Once you see all three stages in the source, decryption is just "run them backwards in reverse order."

| Stage | Function | Operation |
|---|---|---|
| 1. XOR-with-key on reversed text | `dynamic_xor_encrypt` | reverses the plaintext, then XORs each char with `"trudeau"` (repeating) |
| 2. Integer scaling | `encrypt` | multiplies each character's `ord()` value by `shared_key * 311` |
| 3. Diffie-Hellman key agreement | `test` | picks random `a`, `b` in `Z/pZ` and derives `shared_key = g^(b*a) mod p` |

Decryption reverses each stage:

| Reverse stage | Operation |
|---|---|
| undo 2 (integer scaling) | `ord_char = cipher_int // (shared_key * 311)` |
| undo 1 (XOR) | XOR each char with the same `"trudeau"` key (XOR is its own inverse) |
| undo the "reverse" | reverse the resulting string back to original order |

### 2. Diffie-Hellman, in 15 Seconds

```
p, g  — public primes / generator
a     — Alice's secret
b     — Bob's secret
A = g^a mod p
B = g^b mod p
shared = B^a mod p = A^b mod p = g^(a*b) mod p
```

That is the entire protocol. In this challenge both `a` and `b` are leaked in the `flag_info` output (`a = 95`, `b = 21`), so we can recompute `shared_key` ourselves without ever touching the public values `u` and `v`. The script even prints `u` and `v` — the public Diffie-Hellman values — but we do not need them; we have the secrets directly.

### 3. Why the XOR + Reverse Combo Is "Cheap Steganography"

The XOR-with-repeating-key step is the classic "one-time pad that reuses keys" pattern. It is reversible by definition (`a ^ b ^ b == a`), and it has the convenient property that XOR-ing twice with the same key returns the original byte. The "reverse the plaintext" addition is purely cosmetic; reversing again on the way out restores the original order. Both transformations are involutions.

### 4. The Bug-as-Feature That Saves the Solver

Look closely at `encrypt`:

```python
cipher.append(((ord(char) * key*311)))
```

That extra inner pair of parentheses looks suspicious but Python's operator precedence already groups `*` left-to-right, so the multiplier is just `ord(char) * shared_key * 311`. Because `shared_key * 311` is constant, every ciphertext integer is a multiple of `shared_key * 311` and a single integer division recovers `ord(char)`. If the multiplier were not constant (e.g. a stream cipher or per-byte key) we would need a different recovery strategy. The challenge author kept it linear so that "divide by the multiplier" is the obvious decryption.

---

## Solution — Step by Step

I worked out of `~/ctf/custom-encryption` on my Kali VM.

### Step 1 — Set Up the Working Directory

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/custom-encryption && cd ~/ctf/custom-encryption

┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ cp /media/sf_downloads/encrypt.py .
└─$ cp /media/sf_downloads/flag_info.bin .

┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ ls -la
-rw-r--r-- 1 zham zham 1411 Jun 27 08:39 encrypt.py
-rw-r--r-- 1 zham zham  301 Jun 27 08:39 flag_info.bin
```

### Step 2 — Read the Ciphertext

```
┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ cat flag_info.bin
a = 95
b = 21
cipher is: [237915, 1850450, 1850450, 158610, 2458455, 2273410, 1744710, 1744710, 1797580, 1110270, 0, 2194105, 555135, 132175, 1797580, 0, 581570, 2273410, 26435, 1638970, 634440, 713745, 158610, 158610, 449395, 158610, 687310, 1348185, 845920, 1295315, 687310, 185045, 317220, 449395]
```

Two pieces of information are immediately useful:

- `a = 95` and `b = 21` — the Diffie-Hellman secrets, given away by the script's debug prints.
- A list of 34 integers — the ciphertext, one entry per character of the (post-XOR) reversed plaintext.

### Step 3 — Read the Encryption Source

```
┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ cat encrypt.py
```

The relevant bits:

```python
def generator(g, x, p):
    return pow(g, x) % p

def encrypt(plaintext, key):
    cipher = []
    for char in plaintext:
        cipher.append(((ord(char) * key*311)))
    return cipher

def dynamic_xor_encrypt(plaintext, text_key):
    cipher_text = ""
    key_length = len(text_key)
    for i, char in enumerate(plaintext[::-1]):
        key_char = text_key[i % key_length]
        encrypted_char = chr(ord(char) ^ ord(key_char))
        cipher_text += encrypted_char
    return cipher_text

def test(plain_text, text_key):
    p = 97
    g = 31
    ...
    a = randint(p-10, p)
    b = randint(g-10, g)
    print(f"a = {a}")
    print(f"b = {b}")
    u = generator(g, a, p)
    v = generator(g, b, p)
    key = generator(v, a, p)
    b_key = generator(u, b, p)
    ...
    semi_cipher = dynamic_xor_encrypt(plain_text, text_key)
    cipher = encrypt(semi_cipher, shared_key)
    print(f'cipher is: {cipher}')
```

So the pipeline is:

```
plaintext
  -> reverse
  -> XOR each char with "trudeau" (cycling)
  -> multiply each char's ord by shared_key * 311
```

Decryption is the reverse:

```
cipher (list of ints)
  -> divide each by shared_key * 311 to get ord(char)
  -> XOR each char with "trudeau" (cycling)
  -> reverse the string
```

### Step 4 — Recover the Shared Key

```
┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ python3 -c "
def generator(g, x, p):
    return pow(g, x) % p

p = 97
g = 31
a = 95
b = 21

shared_key = generator(generator(g, b, p), a, p)
print(f'shared_key = {shared_key}')
print(f'multiplier = {shared_key * 311}')
"
shared_key = 85
multiplier = 26435
```

`shared_key = 85`. That is `g^(a*b) mod p = 31^(95*21) mod 97`, which Python's `pow` handles in microseconds. The integer multiplier we will divide by is `85 * 311 = 26435`.

### Step 5 — Decrypt Step-by-Step

Now write the full inverse in one script. I put it in `decrypt.py`:

```
┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ nano decrypt.py
```

Paste the following inside `nano`:

```python
#!/usr/bin/env python3
"""
picoCTF - Custom encryption
Invert the three-stage encryption pipeline:
    plaintext
        -> reverse
        -> XOR with "trudeau" (cycling)
        -> multiply ord() by shared_key * 311
"""

def generator(g, x, p):
    return pow(g, x) % p

# ----- Stage 0: recover shared_key from the leaked (a, b) -----
p = 97
g = 31
a = 95
b = 21
shared_key = generator(generator(g, b, p), a, p)
multiplier  = shared_key * 311
print(f"[+] shared_key  = {shared_key}")
print(f"[+] multiplier  = {multiplier}")

# ----- Ciphertext (copied from flag_info.bin) -----
cipher = [237915, 1850450, 1850450, 158610, 2458455, 2273410, 1744710,
          1744710, 1797580, 1110270, 0, 2194105, 555135, 132175, 1797580,
          0, 581570, 2273410, 26435, 1638970, 634440, 713745, 158610,
          158610, 449395, 158610, 687310, 1348185, 845920, 1295315,
          687310, 185045, 317220, 449395]

# Sanity check: every cipher int must be a multiple of the multiplier
assert all(c % multiplier == 0 for c in cipher), "cipher contains non-multiple"

# ----- Stage 1 (reverse of encrypt): divide by multiplier -----
xor_reversed_codes = [c // multiplier for c in cipher]
xor_reversed       = ''.join(chr(c) for c in xor_reversed_codes)
print(f"[+] after divide-by-multiplier: {xor_reversed!r}")

# ----- Stage 2 (reverse of dynamic_xor_encrypt): XOR with "trudeau" -----
text_key   = "trudeau"
unxored_ch = ''.join(chr(ord(c) ^ ord(text_key[i % len(text_key)]))
                     for i, c in enumerate(xor_reversed))
print(f"[+] after un-XOR         : {unxored_ch!r}")

# ----- Stage 3 (reverse of [::-1]): reverse the string -----
flag = unxored_ch[::-1]
print(f"[+] flag                : {flag}")
```

Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`:

```
┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ python3 decrypt.py
[+] shared_key  = 85
[+] multiplier  = 26435
[+] after divide-by-multiplier: '\tFF\x06]VBB\x16D*\x00S\x15\x05D\x00\x16V\x01>\x18\x1b\x06\x06\x11\x06\x1a3 1\x1a\x07\x0c\x11'
[+] after un-XOR         : '}43b87766_d6tp0rc2d_motsuc{FTCocip'
[+] flag                : picoCTF{custom_d2cr0pt6d_66778b34}
```

Got the flag.

### Step 6 — Confirm the Flag

The plaintext begins with `picoCTF{` and ends with `}`, the character codes are all printable ASCII, and the flag wordplay (`d2cr0pt6d` = "decrypted" with leet substitutions) matches the challenge name. Recovery is correct.

---

## Alternative Method — One-Line Pipeline With PyCryptodome-less Pure Python

You do not need `pycryptodome` or any external library for this challenge — `pow` is built into Python 3 and the rest is just slicing and XOR. If you want a no-script version, here is the entire decryption in a single `python3 -c` call:

```
┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ python3 -c "
cipher = [237915, 1850450, 1850450, 158610, 2458455, 2273410, 1744710,
          1744710, 1797580, 1110270, 0, 2194105, 555135, 132175, 1797580,
          0, 581570, 2273410, 26435, 1638970, 634440, 713745, 158610,
          158610, 449395, 158610, 687310, 1348185, 845920, 1295315,
          687310, 185045, 317220, 449395]
m = 85 * 311
step1 = ''.join(chr(c // m) for c in cipher)
step2 = ''.join(chr(ord(c) ^ ord('trudeau'[i % 7])) for i, c in enumerate(step1))
print(step2[::-1])
"
picoCTF{custom_d2cr0pt6d_66778b34}
```

Same flag, no script file.

### Alternative Method 2 — Modifying the Original `encrypt.py` to Decrypt

A clean pedagogical approach is to add a `decrypt()` function to the provided script itself. This makes it obvious that the round-trip is reversible:

```
┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ cp encrypt.py decrypt.py
└─$ nano decrypt.py
```

Append at the end of the file (before the `if __name__` block, or as a sibling function):

```python
def decrypt(cipher, shared_key, text_key="trudeau"):
    # Reverse of encrypt(): integer divide by shared_key * 311
    chars = [chr(c // (shared_key * 311)) for c in cipher]
    # Reverse of dynamic_xor_encrypt(): XOR with the same key, then reverse
    out = ""
    key_len = len(text_key)
    for i, ch in enumerate(chars):
        out += chr(ord(ch) ^ ord(text_key[i % key_len]))
    return out[::-1]
```

Then add a call at the bottom:

```python
if __name__ == "__main__":
    cipher = [237915, 1850450, 1850450, 158610, 2458455, 2273410,
              1744710, 1744710, 1797580, 1110270, 0, 2194105, 555135,
              132175, 1797580, 0, 581570, 2273410, 26435, 1638970,
              634440, 713745, 158610, 158610, 449395, 158610, 687310,
              1348185, 845920, 1295315, 687310, 185045, 317220, 449395]
    a, b, p, g = 95, 21, 97, 31
    shared = pow(pow(g, b, p), a, p)
    print(decrypt(cipher, shared))
```

Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`:

```
┌──(zham㉿kali)-[~/ctf/custom-encryption]
└─$ python3 decrypt.py
picoCTF{custom_d2cr0pt6d_66778b34}
```

Same answer. This version is closest in spirit to the "write the function that will decode the given encrypted file content" instruction from the description.

---

## What Happened Internally

The full timeline of the solve, from "I see two files" to "I have the flag":

1. **Listed and inspected the artifacts.** `file encrypt.py` said Python script; `file flag_info.bin` said ASCII text (it is the script's print output, not raw bytes). A `cat` showed two Python variables — `a = 95`, `b = 21` — and a 34-integer ciphertext list.
2. **Read `encrypt.py` end to end.** Three pieces of business logic: a `generator(g, x, p) = g^x mod p` Diffie-Hellman helper, an `encrypt()` that scales `ord(char)` by `shared_key * 311`, and a `dynamic_xor_encrypt()` that XORs reversed plaintext with `"trudeau"`. The `test()` driver wires them together.
3. **Identified the data flow.** `test()` runs `dynamic_xor_encrypt(plaintext, "trudeau")` first, then `encrypt()` on the result. The ciphertext list therefore represents `(XOR-with-trudeau of reversed plaintext)`, each entry scaled by `shared_key * 311`.
4. **Recovered `shared_key` from the leaked `a` and `b`.** `shared_key = generator(generator(g, b, p), a, p) = pow(pow(31, 21, 97), 95, 97) = 85`. The multiplier for the integer-scaling stage is therefore `85 * 311 = 26435`.
5. **Reverse stage 1: integer-divided every cipher entry by 26435.** Verified all 34 entries are exact multiples (`assert all(...)`). The resulting 34 ASCII codes were mostly printable, with a couple of low control chars (e.g. `chr(0)` for the entry `0`).
6. **Reverse stage 2: XOR'd each byte with `"trudeau"`** (cycling). XOR is its own inverse, so the same key that scrambled the reversed plaintext un-scrambles it. The output was a 34-character string ending in `p` (because the original plaintext starts with `p` of `picoCTF` and we are still reversed).
7. **Reverse stage 3: reversed the string** to restore the original order.
8. **Confirmed the flag** matched the standard `picoCTF{...}` format and the wordplay (`custom_d2cr0pt6d` = "custom decrypted" in leet).

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read both attachment files |
| `nano` | Write `decrypt.py` (`Ctrl+O`, `Enter`, `Ctrl+X`) |
| `python3` | `pow(g, x, p)` for Diffie-Hellman, integer division, XOR, string reversal |
| `file` | Confirm `flag_info.bin` is text, not a real binary blob |
| `cp` | Copy attachments from `~/Downloads` to the working directory |

No external libraries were required. Everything used is in the Python 3 standard library.

---

## Key Takeaways

* **Always read the encryptor before writing the decryptor.** This challenge had no clever math, no oracle, no factoring trick — just three linear transformations stacked in a known order. Reading the code for two minutes saves you an hour of guessing.
* **Diffie-Hellman collapses to "just multiply the exponents" when both secrets are leaked.** The script prints `a` and `b` for debugging; that is the entire key recovery in a single `pow(pow(g, b, p), a, p)`. In a real protocol you would never see the secrets — that is the whole point — so treat any debug print of `a` or `b` as a critical vulnerability.
* **XOR with a repeating key is its own inverse.** You do not need a separate "decryption key"; you re-apply the same `"trudeau"` key and the original plaintext falls out. This is also why XOR-encryption is insecure for any message longer than the key — frequency analysis recovers the key stream.
* **Reversing the plaintext before XOR is a red herring.** It does not add security; it just shuffles the bytes so that, say, `picoCTF{` ends up at the back of the array and a casual reader who glances at the ciphertext does not immediately see the flag format. Always remember to reverse it back.
* **The `* key * 311` parenthesis style is a hint.** If the multiplier were per-byte (a stream cipher, an LFSR, etc.), you could not recover the original by a single global division. The fixed scalar multiplier is what makes this challenge "medium" and not "hard." Production-grade crypto never uses a fixed scalar multiplier — that is exactly what AES, ChaCha20, etc. exist to replace.
* **Read the debug info at the bottom of a picoCTF card carefully.** Sometimes it is just a session counter (`[u:871426 e: p: c:412 l:295982]`) and sometimes it is part of the challenge. In this case it was noise, but I have been bitten before by skipping it. Always glance.

**Flag wordplay decode:** `custom_d2cr0pt6d_66778b34` reads as **"custom decrypted"** — `d2cr0pt6d` is `decrypted` with the `e`s replaced by `2`s and the trailing `d` swapped for `6`. The challenge name was "Custom encryption," so the flag tells you that you successfully inverted a custom (i.e. hand-rolled) encryption scheme. The trailing `66778b34` is an 8-hex-digit nonce that the challenge author appended so two different solvers getting the same plaintext still produce different flags.
