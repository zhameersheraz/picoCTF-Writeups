# basic-mod2 — picoCTF Writeup

**Challenge:** basic-mod2  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{1nv3r53ly_h4rd_8a05d939}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> A new modular challenge!
>
> Download the message here.
>
> Take each number mod 41 and find the modular inverse for the result. Then map to the following character set: 1-26 are the alphabet, 27-36 are the decimal digits, and 37 is an underscore.
>
> Wrap your decrypted message in the picoCTF flag format (i.e. `picoCTF{decrypted_message}`)

## Hints

> 1. Do you know what the modular inverse is?
> 2. The inverse modulo z of x is the number, y, that when multiplied by x is 1 modulo z
> 3. It's recommended to use a tool to find the modular inverses

---

## Background Knowledge (Read This First!)

### The modulo operator (a quick refresher)

`a mod b` is the remainder when `a` is divided by `b`. Example: `268 mod 41 = 22`, because `41 * 6 = 246` and `268 - 246 = 22`. In Python that is `268 % 41`.

### What is the modular inverse?

For two integers `a` and `m`, the **modular inverse** of `a` (mod `m`) is the number `y` such that:

```
(a * y) mod m == 1
```

It only exists when `a` and `m` are coprime (their greatest common divisor is 1). Because 41 is prime, every non-zero number mod 41 has an inverse.

Example: the inverse of `3` mod `41` is `14`, because `3 * 14 = 42`, and `42 mod 41 = 1`.

### The character set used by this challenge

The numbers 1 through 37 each stand for one character:

| Range | Maps to |
|------|---------|
| 1–26  | a–z (lowercase alphabet) |
| 27–36 | the decimal digits 0–9 |
| 37    | underscore `_` |

So if we compute the modular inverse and land on `14`, we look up `14 -> n`.

### How we will solve it in one paragraph

For each number in the file: take it mod 41, find the modular inverse of that remainder, look up the inverse in the character table above, and concatenate the letters. That string is the decrypted message — wrap it in `picoCTF{...}` and submit.

---

## Solution — Step by Step

### Step 1 — Read the message

```
┌──(zham㉿kali)-[~/Downloads]
└─$ cat message.txt
268 413 438 313 426 337 272 188 392 338 77 332 139 113 92 239 247 120 419 72 295 190 131
```

Twenty-three integers, space-separated.

### Step 2 — Write a quick decoder in Python

Python ships with everything we need (no extra packages). Save the script as `solve.py`:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ nano solve.py
```

Paste this content:

```python
#!/usr/bin/env python3
"""basic-mod2 decoder."""

def modinv(a, m):
    """Return the modular inverse of a modulo m (m must be prime and a != 0)."""
    # Extended Euclidean Algorithm
    g, x, _ = _egcd(a, m)
    if g != 1:
        raise ValueError(f"{a} has no inverse mod {m}")
    return x % m

def _egcd(a, b):
    if a == 0:
        return b, 0, 1
    g, x1, y1 = _egcd(b % a, a)
    return g, y1 - (b // a) * x1, x1

def to_char(n):
    """Map 1..26 -> a..z, 27..36 -> 0..9, 37 -> '_'."""
    if 1 <= n <= 26:
        return chr(ord('a') + n - 1)
    if 27 <= n <= 36:
        return str(n - 27)
    if n == 37:
        return '_'
    raise ValueError(f"value {n} not in 1..37")

message = "268 413 438 313 426 337 272 188 392 338 77 332 139 113 92 239 247 120 419 72 295 190 131"
nums = [int(x) for x in message.split()]

plain = ""
for n in nums:
    r = n % 41
    inv = modinv(r, 41)
    plain += to_char(inv)
    print(f"{n:>4}  % 41 = {r:>2}   inv = {inv:>2}   ->  {to_char(inv)}")

print()
print("Decrypted message:", plain)
print("Flag:           ", "picoCTF{" + plain + "}")
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

### Step 3 — Run the decoder

```
┌──(zham㉿kali)-[~/Downloads]
└─$ python3 solve.py
 268  % 41 = 22   inv = 28   ->  1
 413  % 41 =  3   inv = 14   ->  n
 438  % 41 = 28   inv = 22   ->  v
 313  % 41 = 26   inv = 30   ->  3
 426  % 41 = 16   inv = 18   ->  r
 337  % 41 =  9   inv = 32   ->  5
 272  % 41 = 26   inv = 30   ->  3
 188  % 41 = 24   inv = 12   ->  l
 392  % 41 = 23   inv = 25   ->  y
 338  % 41 = 10   inv = 37   ->  _
  77  % 41 = 36   inv =  8   ->  h
 332  % 41 =  4   inv = 31   ->  4
 139  % 41 = 16   inv = 18   ->  r
 113  % 41 = 31   inv =  4   ->  d
  92  % 41 = 10   inv = 37   ->  _
 239  % 41 = 34   inv = 35   ->  8
 247  % 41 =  1   inv =  1   ->  a
 120  % 41 = 38   inv = 27   ->  0
 419  % 41 =  9   inv = 32   ->  5
  72  % 41 = 31   inv =  4   ->  d
 295  % 41 =  8   inv = 36   ->  9
 190  % 41 = 26   inv = 30   ->  3
 131  % 41 =  8   inv = 36   ->  9

Decrypted message: 1nv3r53ly_h4rd_8a05d939
Flag:            picoCTF{1nv3r53ly_h4rd_8a05d939}
```

Flag: **`picoCTF{1nv3r53ly_h4rd_8a05d939}`**

---

## Alternative Methods

**Method 1 — Python with `sympy` (if installed)**

If you have `sympy` available (`pip install sympy`), `mod_inverse` is one line:

```python
from sympy import mod_inverse
print(mod_inverse(22, 41))   # -> 28
```

**Method 2 — Brute force in Python (no math library at all)**

Since the modulus is tiny (41), we can just try every `y` from 1 to 40 and pick the one that works:

```python
def modinv_brute(a, m):
    for y in range(1, m):
        if (a * y) % m == 1:
            return y
    return None
```

For a beginner this is the easiest to read and reason about.

**Method 3 — SageMath / `pari-gp`**

If you happen to have a CAS lying around:

```
? lift(Mod(22, 41)^(-1))
28
```

**Method 4 — Online calculator**

If you really do not want to write code, sites like https://www.dcode.fr/modular-inverse will find any modular inverse for you.

**Method 5 — Bash + `awk` (no Python)**

For purists, an `awk` one-liner works for the whole pipeline, but it is harder to read:

```bash
awk '{
  for (i = 1; i <= NF; i++) {
    n = $i; r = n % 41;
    # brute the inverse
    for (y = 1; y < 41; y++) if ((r * y) % 41 == 1) inv = y;
    if (inv >= 1 && inv <= 26)      c = sprintf("%c", 96 + inv);
    else if (inv >= 27 && inv <= 36) c = inv - 27;
    else if (inv == 37)              c = "_";
    printf "%s", c
  }
}' message.txt
```

The Python script above is much cleaner; the `awk` version is here just to show the math runs anywhere.

---

## What Happened Internally

A timeline of what actually happened, step by step:

1. The challenge server handed us 23 integers, each one a ciphertext character encoded as a number well above the alphabet range (1–37).
2. The instruction `n mod 41` mapped each number into the range 0–40. That made every value small enough to live in the same finite field.
3. We then asked: for each remainder `r`, what is `y` such that `r * y ≡ 1 (mod 41)`? That `y` is the modular inverse. Since 41 is prime, every non-zero `r` has exactly one inverse in the range 1–40. We found it with the extended Euclidean algorithm.
4. We translated each inverse into a character using the lookup table from the description (1–26 -> a–z, 27–36 -> 0–9, 37 -> `_`).
5. Concatenating the 23 characters gave the string `1nv3r53ly_h4rd_8a05d939`.
6. Wrapping that string in `picoCTF{...}` produced the flag we submitted.
7. The whole scheme is essentially a substitution cipher: each character is represented by a number, that number is encrypted by `c = (r⁻¹ mod 41) substituted for the alphabet index`, and the substitution table is fixed and public. Knowing the table is enough to invert it without a key.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the message file |
| `nano` | Edit the Python solver |
| `python3` | Run the decoder |
| Extended Euclidean algorithm (in `solve.py`) | Compute modular inverses mod 41 |
| `tr` / `sort -u` / `wc` | Sanity checks (not strictly required) |

---

## Key Takeaways

- `n mod m` reduces any integer into the range `0 .. m-1`. It is the entry ticket to every modular-arithmetic puzzle.
- The modular inverse exists for every non-zero `a` mod `m` when `m` is prime. 41 is prime, which is why the challenge picked it.
- The extended Euclidean algorithm is the canonical way to compute modular inverses — `gcd`-style recursion that runs in `O(log m)` time. For tiny moduli like 41, a brute-force scan from 1 to 40 also works and is easier to read.
- The character map is a Caesar-style substitution: the encryption step is a fixed permutation of {1..37}, so decryption is just the inverse permutation. Once you see the table, the math is one lookup per character.
- Numbered character sets like this are common in picoCTF — once you recognize the "lookup table by index" pattern, you can solve an entire family of challenges (basic-mod1, basic-mod2, etc.) with the same script template.
- The `sympy.numbers.mod_inverse` function is a great shortcut when you have `sympy` installed — it does exactly what our `modinv` does, but with a one-line call.
- Flag wordplay decode:
  - `1nv3r53ly` -> **"inversely"** (`1`=i, `n`=n, `v`=v, `3`=e, `r`=r, `5`=s, `3`=e, `l`=l, `y`=y)
  - `h4rd` -> **"hard"** (`h`=h, `4`=a, `r`=r, `d`=d)
  - `8a05d939` -> hex-ish suffix (looks like an 8-char hex hash; it is just a uniqueness tail, not a real digest)
  - Putting it together: **"inversely hard"** — a wink at the fact that modular inversion is the tricky part of the challenge.
