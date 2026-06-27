# Vigenere — picoCTF Writeup

**Challenge:** Vigenere  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{D0NT_US3_V1G3N3R3_C1PH3R_ae82272q}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Can you decrypt this message?
>
> Decrypt this message using this key "CYLAB".
>
> Download the file: message.txt

## Hints

> 1. https://en.wikipedia.org/wiki/Vigen%C3%A8re_cipher

---

## Background Knowledge (Read This First!)

### What is the Vigenere Cipher?

The Vigenere cipher is a classical polyalphabetic substitution cipher — basically a fancier, stronger version of the Caesar cipher. Instead of shifting every letter by the same amount, it uses a repeating key, and each letter of the key tells you how much to shift that particular character.

Think of it like this:

- Caesar cipher = "shift every letter by 3"
- Vigenere cipher = "shift the first letter by 5, the second by 14, the third by 11, the fourth by 0, then repeat the key"

### How the Math Works

Each letter of the alphabet is assigned a number (A = 0, B = 1, C = 2, ... Z = 25).

**Encryption:**
```
cipher = (plain + key) mod 26
```

**Decryption:**
```
plain = (cipher - key) mod 26
```

### Example

Key: `CYLAB` (C=2, Y=24, L=11, A=0, B=1) — the key keeps repeating.

Plaintext:  `pico`
Key (repeating): `CYLA`
Ciphertext: each letter shifted by its matching key letter
```
p(15) + C(2) = 17 = r
i(8)  + Y(24) = 32 mod 26 = 6 = g
c(2)  + L(11) = 13 = n
o(14) + A(0)  = 14 = o
```

So `pico` becomes `rgno` — and this is exactly how the challenge ciphertext starts. Confirms we're on the right track.

### Important Rule

Vigenere only affects **letters**. Numbers (`0`, `1`, `2`...) and symbols (`{`, `_`, `}`) are passed through untouched, and they also do NOT consume a key position. The key index only advances when we hit a letter.

### Why Vigenere is Weak

Even though it was called "le chiffre indéchiffrable" (the indecipherable cipher) for hundreds of years, modern computers and frequency analysis (like Kasiski examination) can break it pretty easily. Never use it for real security.

---

## Solution — Step by Step

### Step 1 — Get the Challenge Files

I downloaded `message.txt` from the picoCTF challenge page into my shared downloads folder on Kali.

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Read the Ciphertext

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat message.txt
rgnoDVD{O0NU_WQ3_G1G3O3T3_A1AH3S_cc82272b}
```

Looks like a picoCTF flag scrambled up. The format `picoCTF{...}` is hidden in there somewhere.

### Step 3 — Create a Python Decryption Script

I'll write a small Python script that walks through each character, applying the Vigenere decryption rule.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano vigenere_solve.py
```

In nano, I pasted the following:

```python
def vigenere_decrypt(ciphertext, key):
    plaintext = []
    key_index = 0

    for ch in ciphertext:
        if ch.isalpha():
            # Get the shift from the current key letter (A=0, B=1, ...)
            shift = ord(key[key_index % len(key)].upper()) - ord('A')
            if ch.isupper():
                decrypted = chr((ord(ch) - ord('A') - shift) % 26 + ord('A'))
            else:
                decrypted = chr((ord(ch) - ord('a') - shift) % 26 + ord('a'))
            plaintext.append(decrypted)
            key_index += 1  # Only advance the key when we consumed a letter
        else:
            # Numbers and symbols pass through unchanged, key does not advance
            plaintext.append(ch)

    return ''.join(plaintext)


cipher = "rgnoDVD{O0NU_WQ3_G1G3O3T3_A1AH3S_cc82272b}"
key = "CYLAB"

print("Ciphertext :", cipher)
print("Key        :", key)
print("Plaintext  :", vigenere_decrypt(cipher, key))
```

Save and exit: `Ctrl+O`, `Enter`, then `Ctrl+X`.

### Step 4 — Run the Script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 vigenere_solve.py
Ciphertext : rgnoDVD{O0NU_WQ3_G1G3O3T3_A1AH3S_cc82272b}
Key        : CYLAB
Plaintext  : picoCTF{D0NT_US3_V1G3N3R3_C1PH3R_ae82272q}
```

Got the flag.

---

## What Happened Internally

A quick timeline of what my script did, letter by letter:

| Step | Ciphertext Char | Key Letter | Shift | Plain Char |
|------|----------------|------------|-------|-----------|
| 1 | `r` | C (1st) | 2 | `p` |
| 2 | `g` | Y (2nd) | 24 | `i` |
| 3 | `n` | L (3rd) | 11 | `c` |
| 4 | `o` | A (4th) | 0 | `o` |
| 5 | `D` | B (5th) | 1 | `C` |
| 6 | `V` | C (1st, wraps) | 2 | `T` |
| 7 | `D` | Y (2nd) | 24 | `F` |
| 8 | `{` | — (skipped) | — | `{` |
| 9 | `O` | L (3rd) | 11 | `D` |
| 10 | `0` | — (skipped) | — | `0` |
| 11 | `N` | A (4th) | 0 | `N` |
| 12 | `U` | B (5th) | 1 | `T` |
| ... | ... | ... | ... | ... |
| Last letter | `b` | L | 11 | `q` |

The key keeps cycling: `C Y L A B C Y L A B C Y L A B ...` — and the script only advances the key pointer when it actually consumes a letter. Numbers, underscores, and braces are all passed through as-is.

---

## Alternative Methods

### Method 1 — CyberChef (Easiest, No Code)

1. Go to https://gchq.github.io/CyberChef/
2. Drag `message.txt` into the input box, or paste the ciphertext directly
3. Add the **Vigenere Decode** operation from the recipe list
4. Type `CYLAB` as the key
5. The output is the flag instantly

### Method 2 — dcode.fr (Web Tool)

1. Go to https://www.dcode.fr/vigenere-cipher
2. Paste the ciphertext
3. Enter `CYLAB` as the known key
4. Click "Decrypt"
5. Read the flag

### Method 3 — One-liner in Python

If you don't want to write a file, just run this in the terminal:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
ct = 'rgnoDVD{O0NU_WQ3_G1G3O3T3_A1AH3S_cc82272b}'
key = 'CYLAB'
out, k = [], 0
for c in ct:
    if c.isalpha():
        s = ord(key[k % len(key)].upper()) - ord('A')
        base = ord('A') if c.isupper() else ord('a')
        out.append(chr((ord(c) - base - s) % 26 + base))
        k += 1
    else:
        out.append(c)
print(''.join(out))
"
picoCTF{D0NT_US3_V1G3N3R3_C1PH3R_ae82272q}
```

### Method 4 — Manual Decryption (The Hard Way)

You can do it by hand if you really want — write out a Vigenere square (a 26x26 grid of alphabets, each row shifted by one), find the row of the first key letter, and trace down to the ciphertext letter, then read the column header as the plaintext. Painful for a flag this long, but it works for short ones.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the contents of message.txt |
| `nano` | Write the Python decryption script |
| `python3` | Run the Vigenere decryption logic |
| CyberChef (alt) | Quick GUI-based Vigenere decode |
| dcode.fr (alt) | Online Vigenere solver |

---

## Key Takeaways

- The Vigenere cipher is a polyalphabetic substitution cipher using a repeating key
- Decryption formula: `plain = (cipher - key) mod 26` for each letter
- The key index only advances on letters — numbers and symbols (`{`, `_`, `}`) are untouched and do not consume a key position
- When you see a flag-like string starting with `rgnoDVD` or similar scrambled text, plus a key, the Vigenere pattern `pico` -> `rgno` with key `C` is a dead giveaway
- Python's `chr()` / `ord()` and the modulo operator are the bread and butter for writing quick cipher scripts
- CyberChef and dcode.fr are great go-to tools when you want a one-click solve
- Even the "indecipherable cipher" can be cracked with frequency analysis and modern computing — never use it for anything real

### Flag Wordplay Decode

The flag `picoCTF{D0NT_US3_V1G3N3R3_C1PH3R_ae82272q}` is a warning written in leet speak:

- `D0NT` -> **DON'T**
- `US3` -> **USE**
- `V1G3N3R3` -> **VIGENERE**
- `C1PH3R` -> **CIPHER**

Put together: **"Don't use Vigenere cipher"** — a cheeky reminder from picoCTF that classical ciphers are fun to learn but offer no real security. The trailing `ae82272q` is just a unique hash so every player gets a different flag.
