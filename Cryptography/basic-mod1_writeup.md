# basic-mod1 — picoCTF Writeup

**Challenge:** basic-mod1  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{R0UND_N_R0UND_ADD17EC2}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> We found this weird message being passed around on the servers, we think we have a working decryption scheme.
>
> Download the message here.
>
> Take each number mod 37 and map it to the following character set: 0-25 is the alphabet (uppercase), 26-35 are the decimal digits, and 36 is an underscore.
>
> Wrap your decrypted message in the picoCTF flag format (i.e. `picoCTF{decrypted_message}`)

## Hints

> 1. Do you know what mod 37 means?
> 2. mod 37 means modulo 37. It gives the remainder of a number after being divided by 37.

---

## Background Knowledge (Read This First!)

### The modulo operator, again

`a mod b` returns the remainder when `a` is divided by `b`. In Python it is written `a % b`. Quick example:

```
350 mod 37 = ?
37 * 9 = 333
350 - 333 = 17
-> 350 mod 37 = 17
```

Modulo is sometimes described as "rounding" a number down into the range `0 .. b-1`, which is exactly what this challenge puns on in its flag.

### The character set used by this challenge

Each integer 0 through 36 maps to exactly one printable character:

| Range | Maps to |
|------|---------|
| 0–25  | A–Z (uppercase alphabet) |
| 26–35 | the decimal digits 0–9 |
| 36    | underscore `_` |

Note the difference from `basic-mod2`:

| Challenge | Modulus | Alphabet | Index range |
|-----------|---------|----------|-------------|
| basic-mod1 | 37 | A–Z (uppercase) | 0–25 |
| basic-mod2 | 41 | a–z (lowercase) | 1–26 |

Watch the offset! In mod1, index 0 means **A**. In mod2, index 1 means **a**.

### Why this is a substitution cipher

Each character is encoded as `c = index + k * 37` for some integer `k`. Taking `c mod 37` strips the `k * 37` and reveals the original index. That is why the challenge can hide the message behind huge numbers like 374 or 369 — modulo always rounds them back to a value in 0..36.

---

## Solution — Step by Step

### Step 1 — Read the message

```
┌──(zham㉿kali)-[~/Downloads]
└─$ cat message.txt
350 63 353 198 114 369 346 184 202 322 94 235 114 110 185 188 225 212 366 374 261 213
```

Twenty-two integers, space-separated.

### Step 2 — Write a quick decoder

Python has `%` and `chr()` built in. Save this as `solve.py`:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ nano solve.py
```

Paste the content:

```python
#!/usr/bin/env python3
"""basic-mod1 decoder."""

def to_char(n):
    """0..25 -> A..Z, 26..35 -> 0..9, 36 -> '_'."""
    if 0 <= n <= 25:
        return chr(ord('A') + n)
    if 26 <= n <= 35:
        return str(n - 26)
    if n == 36:
        return '_'
    raise ValueError(f"value {n} not in 0..36")

message = "350 63 353 198 114 369 346 184 202 322 94 235 114 110 185 188 225 212 366 374 261 213"
nums = [int(x) for x in message.split()]

plain = ""
for n in nums:
    r = n % 37
    c = to_char(r)
    plain += c
    print(f"{n:>4}  % 37 = {r:>2}   ->  {c}")

print()
print("Decrypted message:", plain)
print("Flag:           ", "picoCTF{" + plain + "}")
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

### Step 3 — Run the decoder

```
┌──(zham㉿kali)-[~/Downloads]
└─$ python3 solve.py
 350  % 37 = 17   ->  R
  63  % 37 = 26   ->  0
 353  % 37 = 20   ->  U
 198  % 37 = 13   ->  N
 114  % 37 =  3   ->  D
 369  % 37 = 36   ->  _
 346  % 37 = 13   ->  N
 184  % 37 = 36   ->  _
 202  % 37 = 17   ->  R
 322  % 37 = 26   ->  0
  94  % 37 = 20   ->  U
 235  % 37 = 13   ->  N
 114  % 37 =  3   ->  D
 110  % 37 = 36   ->  _
 185  % 37 =  0   ->  A
 188  % 37 =  3   ->  D
 225  % 37 =  3   ->  D
 212  % 37 = 27   ->  1
 366  % 37 = 33   ->  7
 374  % 37 =  4   ->  E
 261  % 37 =  2   ->  C
 213  % 37 = 28   ->  2

Decrypted message: R0UND_N_R0UND_ADD17EC2
Flag:            picoCTF{R0UND_N_R0UND_ADD17EC2}
```

Flag: **`picoCTF{R0UND_N_R0UND_ADD17EC2}`**

---

## Alternative Methods

**Method 1 — Pure Python one-liner**

If you just want the answer:

```python
nums = [350, 63, 353, 198, 114, 369, 346, 184, 202, 322, 94, 235, 114, 110, 185, 188, 225, 212, 366, 374, 261, 213]
print("picoCTF{" + "".join(chr(65 + n % 37) if n % 37 < 26 else "_" if n % 37 == 36 else str(n % 37 - 26) for n in nums) + "}")
```

**Method 2 — `awk` in the terminal**

If you do not want to touch Python:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ awk '{for(i=1;i<=NF;i++){r=$i%37; if(r<26)c=sprintf("%c",65+r); else if(r==36)c="_"; else c=r-26; printf "%s", c}}' message.txt
R0UND_N_R0UND_ADD17EC2
```

**Method 3 — Bash loop**

```bash
msg=""
for n in 350 63 353 198 114 369 346 184 202 322 94 235 114 110 185 188 225 212 366 374 261 213; do
    r=$((n % 37))
    if [ "$r" -lt 26 ]; then
        msg+=$(printf "\\$(printf '%03o' $((65 + r)))")
    elif [ "$r" -eq 36 ]; then
        msg+="_"
    else
        msg+="$((r - 26))"
    fi
done
echo "picoCTF{$msg}"
```

**Method 4 — Online calculator**

Paste the numbers into any modular arithmetic playground, or use CyberChef with a "MOD 37" -> "From Decimal" -> "Substitute" pipeline.

**Method 5 — Mental / pen-and-paper**

With only 22 numbers and a mod of 37, you can absolutely do this by hand. Compute each remainder, then either find the corresponding letter (A=0, B=1, ...) or the corresponding digit/underscore. It is slow but it works and you will never forget how modulo works again.

---

## What Happened Internally

A timeline of what actually happened:

1. The challenge server handed us 22 space-separated integers, each much larger than the alphabet (some are even >37, like 369 and 374).
2. The challenge description told us the encoding rule: each character is just its index in {0..36} plus some multiple of 37. That multiple was thrown away by the `mod 37` step.
3. For each number `n`, we computed `r = n % 37`. This pulled the value back into the printable range. Example: `369 % 37 = 36` -> underscore.
4. We then translated each `r` using the table 0..25 -> A..Z, 26..35 -> 0..9, 36 -> `_`. Example: `r = 17` -> `R`.
5. Concatenating the 22 characters gave us the plaintext `R0UND_N_R0UND_ADD17EC2`.
6. Wrapping the plaintext in `picoCTF{...}` produced the flag we submitted.
7. The whole scheme is the inverse of `encode(c) = c + 37 * random_k`. Adding any multiple of 37 to a character index does not change its identity once you take mod 37 — so the encoding is "lossy" by design and trivial to invert.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the message file |
| `nano` | Edit the Python solver |
| `python3` | Run the decoder |
| `awk` | Alternative one-liner decode |
| Bash arithmetic `$((...))` | Hand-compute `n % 37` if scripting Python feels heavy |

---

## Key Takeaways

- Modulo is just "the leftover after division". Once you accept that, taking any number mod 37 always returns something in `0..36`.
- The character set in this challenge is **uppercase** and **starts at index 0**, unlike `basic-mod2` which is lowercase and starts at 1. Mixing those up is the #1 beginner mistake — always double-check the offset.
- "Huge" ciphertext numbers like 369 or 374 are no obstacle: `369 % 37 == 36`. The challenge is testing that you understand what `%` actually does.
- This challenge and `basic-mod2` are siblings. Solve one, and the script template works for the other with two lines changed (the modulus and the alphabet offset).
- A pure-bash or pure-awk decode is a great exercise: it shows the math runs anywhere with no Python dependency. For CTFs, Python is usually faster, but for portability, `awk` is hard to beat.
- CyberChef's recipe chain (From Decimal -> Substitute -> ...) is worth knowing if you would rather click than code.
- Flag wordplay decode:
  - `R0UND` -> **"Round"** (`R`=R, `0`=O, `U`=U, `N`=N, `D`=D)
  - `N` -> **"n"** (a stylized "and")
  - `R0UND` -> **"round"** (same trick, lowercase implied)
  - `ADD17EC2` -> hex/leetspeak tail (most naturally read as a "hex-style" suffix; you can loosely leet-decode it as `ADD` + `1`(i) + `7`(t) + `E` + `C` + `2` to approximate "ADDITIONS" or similar, but it is really just opaque flag padding)
  - Putting it together: **"Round n round, ADD17EC2"** — a wink at the modulo operation itself: every number got "rounded" into the 0–36 range, again and again, to reveal the message.
