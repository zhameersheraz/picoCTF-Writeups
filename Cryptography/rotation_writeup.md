# rotation - picoCTF Writeup

**Challenge:** rotation  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{r0tat1on_d3crypt3d_a4b7d759}`  
**Platform:** picoCTF (2019)  
**Writeup by:** zham  

---

## Description

> You will find the flag after decrypting this file.
>
> Download the encrypted flag [here].

## Hints

> 1. Sometimes rotation is right.

---

## Background Knowledge

Before we crack this one, here is what is going on.

**Caesar / ROT ciphers.** A Caesar cipher shifts every letter by a fixed number of positions in the alphabet. ROT-13 is the most famous example — shift each letter by 13 and you get a (sort of) reversible cipher. The same idea, but with a different shift, is what we are dealing with here.

**Alphabetic vs. full-ASCII rotation.** There are two common variants:
- *Alphabetic-only* rotation — only letters shift; digits and symbols stay put.
- *Full-ASCII* rotation — every character (across the full printable ASCII range, typically 32-126) shifts by the same amount.

The hint ("Sometimes rotation is right") does not tell us which one is used, so we have to figure that out from the ciphertext itself.

**Frequency / known-plaintext attack.** When we already know the first seven characters of the plaintext must be `picoCTF`, we can use them to derive the key. Compare the first letter of the ciphertext (`x`) with the expected plaintext letter (`p`): the shift is whatever maps `x` to `p`. Apply the same shift everywhere and check the result.

**Why letters-only usually wins here.** picoCTF flags mix letters, digits, and `_`. If the cipher rotated digits too, we would expect to see weird symbol substitutions in the flag body, which would break the `picoCTF{...}` wrapper convention. Letters-only rotation preserves the flag wrapper, so it is almost always the right guess for this challenge family.

---

## Solution

I started by inspecting the ciphertext to see what we are dealing with.

### Step 1 — Look at the ciphertext

```bash
┌──(zham㉿kali)-[~/picoCTF/rotation]
└─$ mkdir rotation && cd rotation
┌──(zham㉿kali)-[~/picoCTF/rotation]
└─$ cat ciphertext.txt
xqkwKBN{z0bib1wv_l3kzgxb3l_i4j7l759}
┌──(zham㉿kali)-[~/picoCTF/rotation]
└─$ wc -c ciphertext.txt
37 ciphertext.txt
```

37 bytes total. The structure looks suspiciously like a flag: a mix of upper/lowercase letters, digits, and underscores, wrapped in `{...}`.

### Step 2 — Figure out the rotation

I know picoCTF flags always start with `picoCTF{`. Let me line those up:

| Ciphertext | x   | q   | k   | w   | K   | B   | N   | {   |
|------------|-----|-----|-----|-----|-----|-----|-----|-----|
| Plaintext  | p   | i   | c   | o   | C   | T   | F   | {   |

Computing the shift (letter-by-letter, preserving case):

- `x` -> `p`: shift of **-8** (or +18) in lowercase
- `q` -> `i`: shift of **-8** (matches)
- `K` -> `C`: shift of **-8** (matches in uppercase)
- `B` -> `T`: shift of **+18** (which is -8 mod 26, matches)

So the rotation is **-8 within the alphabet**, applied independently to uppercase and lowercase, leaving digits and symbols untouched.

### Step 3 — Build the decoder

```bash
┌──(zham㉿kali)-[~/picoCTF/rotation]
└─$ nano solve.py
```

```python
#!/usr/bin/env python3
ciphertext = open('ciphertext.txt').read().strip()

shift = 8  # backwards
out = ""
for c in ciphertext:
    if 'a' <= c <= 'z':
        out += chr((ord(c) - ord('a') - shift) % 26 + ord('a'))
    elif 'A' <= c <= 'Z':
        out += chr((ord(c) - ord('A') - shift) % 26 + ord('A'))
    else:
        out += c

print(f"picoCTF{{{out[7:-1]}}}" if out.startswith('picoCTF{') else out)
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

Run it:

```bash
┌──(zham㉿kali)-[~/picoCTF/rotation]
└─$ python3 solve.py
picoCTF{r0tat1on_d3crypt3d_a4b7d759}
```

### Alternative solve — CyberChef

If you prefer a GUI tool, CyberChef (https://gchq.github.io/CyberChef) handles this in seconds:

1. Paste the ciphertext as input.
2. Add the **ROT13 Brute Force** recipe (Operations > Cryptography > ROT13 Brute Force).
3. Scan the output for the one that starts with `picoCTF{` — that will be the **ROT8** entry.

### Alternative solve — `tr` one-liner

For a pure shell approach, you can pipe the ciphertext through `tr` with two alphabets — the source alphabet shifted by 8 and the destination in normal order:

```bash
┌──(zham㉿kali)-[~/picoCTF/rotation]
└─$ cat ciphertext.txt | \
  tr 'A-Za-z' 'I-ZA-ih-za-i'
picoCTF{r0tat1on_d3crypt3d_a4b7d759}
```

How it works:
- `'A-Za-z'` is the source alphabet (uppercase A-Z then lowercase a-z).
- `'I-ZA-ih-za-i'` is the destination. The first 18 chars (`I-ZA-i`) shift uppercase A-R to S-Z plus wraparound, and the second half (`h-za-i`) does the same for lowercase a-r to s-z plus wraparound — net effect: rotate by 18 forward = rotate by 8 backward.

Digits and symbols are untouched because they are not listed in either alphabet of `tr`.

---

## What Happened Internally

A short timeline of the decode:

1. **Inspect ciphertext** — `xqkwKBN{z0bib1wv_l3kzgxb3l_i4j7l759}`. The `{...}` wrapper and the mix of letters/digits/underscores scream "flag". I recognize the `picoCTF{...}` format immediately.
2. **Known-plaintext match** — by lining up `xqkwKBN{` with `picoCTF{`, I derive a shift of -8 in the alphabet (or +18, equivalently).
3. **Apply shift** — for each char, if it is `a-z`, shift by -8 with wraparound mod 26. Same for `A-Z`. Anything else (digits, `_`, `{`, `}`) passes through unchanged.
4. **Read the result** — `picoCTF{r0tat1on_d3crypt3d_a4b7d759}` — "rotation decrypted" with leetspeak substitutions (`0` for `o`, `1` for `i`, `3` for `e`, `4` for `a`, `7` for `t`).

---

## Tools Used

| Tool       | Purpose                                                       |
|------------|---------------------------------------------------------------|
| `cat`      | View the ciphertext.                                          |
| `wc -c`    | Confirm the ciphertext size (37 bytes).                       |
| `python3`  | Decode the ROT cipher and print the flag.                     |
| `nano`     | Edit `solve.py`.                                              |
| (optional) CyberChef | GUI option — drop in ciphertext, brute force ROT.      |
| (optional) `tr`      | Pure-shell one-liner decode.                          |

---

## Key Takeaways

- **Known plaintext is a cheat code.** Whenever you know (or can guess) part of the message, derive the key from that. picoCTF flags give you the first 8 characters for free (`picoCTF{`).
- **Rotation only works within a class.** A pure alphabetic Caesar cipher rotates only letters; digits and symbols pass through. If a ciphertext has digits/symbols that look "normal", you almost certainly have a letters-only rotation, not a full-ASCII one.
- **Try the small range first.** With a single-character alphabet of 26, you only have to try 26 shifts. Brute force or eyeball the small space before writing a complicated analysis.
- **`tr` is your shell friend.** For one-off character-class substitutions, `tr 'src' 'dst'` is faster than spinning up Python or CyberChef.
- **Look at the flag's wordplay.** picoCTF authors usually pick a flag body that reflects the challenge. `r0tat1on_d3crypt3d` literally spells out "rotation decrypted" in leetspeak — that is your hint that you solved it right.

Flag decoded: `picoCTF{r0tat1on_d3crypt3d_a4b7d759}` — *rotation decrypted*. The cipher author was kind enough to tell you exactly what you just did, in leetspeak.
