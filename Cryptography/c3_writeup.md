# C3 - picoCTF Writeup

**Challenge:** C3  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{adlibs}`  
**Platform:** picoCTF (2019)  
**Writeup by:** zham  

---

## Description

> This is the Custom Cyclical Cipher!
>
> Download the ciphertext [here]. Download the encoder [here].
>
> Enclose the flag in our wrapper for submission. If the flag was "example" you would submit "picoCTF(example)".

## Hints

> 1. Modern crypto schemes don't depend on the encoder to be secret, but this one does.

---

## Background Knowledge

Before we crack the cipher, here is what is going on under the hood.

**Substitution ciphers.** A substitution cipher swaps each character of the plaintext for another character. The Caesar cipher is the simplest version (shift every letter by a fixed amount). C3 is a more elaborate substitution, but the idea is the same: replace one character with another using a rule.

**Keyed vs. keyless substitution.** Classical substitution ciphers use a key (Caesar's shift value, a keyword alphabet, etc.). If the encoder script itself is the secret, the cipher is "keyless" — there is nothing to crack except the algorithm. The hint tells us this is exactly the situation: the encoder is the secret.

**Two lookup tables.** The encoder reads each character of the plaintext, finds it in `lookup1`, then writes out a character at a *computed* position in `lookup2`. The twist is that the position depends on the previous plaintext character (`prev`). That makes it a **cyclical** cipher — each output character depends on the previous one.

**Reversing cyclical ciphers.** Even though each step depends on the previous one, the relation is fully reversible as long as we know `prev` (we start at 0). So we can just walk the ciphertext in order, recovering each plaintext character one at a time.

**Self-referential scripts.** The recovered plaintext turns out to be a small Python 2 script that prints characters at perfect-cube indices (1, 8, 27, 64, ...). The script's first-line comments (`#asciiorder`, `#fortychars`, `#selfinput`, `#pythontwo`) describe the cipher and crucially tell us what input to feed it — itself.

**`1 / 1` in Python 2.** The line `b = 1 / 1` looks innocent but is a Python 2 giveaway: integer division gives `1`, not `1.0`. The script is intentionally Python 2 syntax (note `print chars[i]` without parentheses).

---

## Solution

I downloaded both files into a working folder and got to work.

### Step 1 — Inspect what we have

```bash
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ mkdir c3 && cd c3
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ file *
ciphertext.txt: ASCII text, with no line terminators
encoder.py:     Python script, ASCII text executable
```

I read the encoder first.

```bash
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ cat encoder.py
```

```python
import sys
chars = ""
from fileinput import input
for line in input():
  chars += line

lookup1 = "\n \"#()*+/1:=[]abcdefghijklmnopqrstuvwxyz"
lookup2 = "ABCDEFGHIJKLMNOPQRSTabcdefghijklmnopqrst"

out = ""

prev = 0
for char in chars:
  cur = lookup1.index(char)
  out += lookup2[(cur - prev) % 40]
  prev = cur

sys.stdout.write(out)
```

So the encoder:

1. Slurps the whole input file into `chars`.
2. For each character, looks up its position in `lookup1` (40 chars total).
3. Writes `lookup2[(cur - prev) % 40]`, where `prev` was the position of the *previous* plaintext character (0 for the first).
4. Updates `prev = cur` and continues.

That last step is what makes it cyclical.

### Step 2 — Reverse the encoder

Because `prev` is known at every step (we start at 0), I can walk the ciphertext backwards:

- Given ciphertext char `c`, find `cur = lookup2.index(c)` — this is the value `(cur - prev) % 40` that the encoder produced.
- So the original position is `plain_idx = (cur + prev) % 40`.
- Plaintext char is `lookup1[plain_idx]`.
- Update `prev = plain_idx` and move on.

I wrote a decoder. (Showing the nano flow as requested.)

```bash
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ nano decoder.py
```

Paste:

```python
#!/usr/bin/env python3
import sys

ciphertext = sys.stdin.read().rstrip("\n")

lookup1 = '\n "#()*+/1:=[]abcdefghijklmnopqrstuvwxyz'
lookup2 = 'ABCDEFGHIJKLMNOPQRSTabcdefghijklmnopqrst'

prev = 0
out = ""
for char in ciphertext:
    cur = lookup2.index(char)
    plain_idx = (cur + prev) % 40
    out += lookup1[plain_idx]
    prev = plain_idx

print(out)
```

Save: `Ctrl+O`, `Enter`, then `Ctrl+X` to exit.

Run it against the ciphertext:

```bash
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ python3 decoder.py < ciphertext.txt
#asciiorder
#fortychars
#selfinput
#pythontwo

chars = ""
from fileinput import input
for line in input():
    chars += line
b = 1 / 1

for i in range(len(chars)):
    if i == b * b * b:
        print chars[i] #prints
        b += 1 / 1
```

The "ciphertext" turns out to be a tiny Python 2 script. The four comments at the top describe the cipher itself, and `#selfinput` is the giveaway: feed the script to itself.

### Step 3 — Run the recovered script on itself

The script loops through the input file and prints the character at every perfect-cube index (`b*b*b`): 1, 8, 27, 64, 125, 216, ...

I emulated the Python 2 logic directly in Python 3 (no need to install Python 2 just for this):

```bash
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ nano solve.py
```

```python
#!/usr/bin/env python3
ciphertext = open('ciphertext.txt').read().rstrip('\n')

lookup1 = '\n "#()*+/1:=[]abcdefghijklmnopqrstuvwxyz'
lookup2 = 'ABCDEFGHIJKLMNOPQRSTabcdefghijklmnopqrst'

# Step 1: decode the ciphertext
prev = 0
recovered = ""
for char in ciphertext:
    cur = lookup2.index(char)
    plain_idx = (cur + prev) % 40
    recovered += lookup1[plain_idx]
    prev = plain_idx

# Step 2: emulate the python2 logic on the recovered script itself
b = 1
flag = ""
for i in range(len(recovered)):
    if i == b * b * b:
        flag += recovered[i]
        b += 1

print(f"picoCTF{{{flag}}}")
```

Save: `Ctrl+O`, `Enter`, then `Ctrl+X` to exit.

Run it:

```bash
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ python3 solve.py
picoCTF{adlibs}
```

### Alternative solve — Python 2 directly

If you have Python 2 installed (Kali does not by default; you can grab it with `apt install python2`), you can save the recovered text as a `.py` file and run it on itself. After saving the recovered script to `recovered.py`:

```bash
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ python2 recovered.py < recovered.py
a
d
l
i
b
s
```

Each character prints on its own line because of the `print` statement in the script. Concatenate: `adlibs`. Wrap it: `picoCTF{adlibs}`.

### Alternative solve — one-liner

Once you have `decoder.py`, you can do the second step inline without writing a separate file:

```bash
┌──(zham㉿kali)-[~/picoCTF/C3]
└─$ python3 decoder.py < ciphertext.txt > recovered.txt && \
  python3 -c "
s = open('recovered.txt').read()
b, flag = 1, ''
for i in range(len(s)):
    if i == b*b*b:
        flag += s[i]; b += 1
print('picoCTF{' + flag + '}')
"
picoCTF{adlibs}
```

---

## What Happened Internally

A short timeline of the whole decode:

1. **Read ciphertext** — `DLSeGAGDgBNJDQJDCF...` (237 chars). The encoder produced this by walking the plaintext char-by-char, taking the current plaintext character's index in `lookup1` (`cur`), subtracting the previous plaintext character's index (`prev`), and writing `lookup2[(cur - prev) % 40]`.
2. **First decode step (i=0)** — `prev = 0`. First ciphertext char is `D`. `lookup2.index('D') = 3`. So `(cur - 0) % 40 = 3` → `cur = 3`. `lookup1[3] = '#'`. `prev` becomes 3.
3. **Second decode step (i=1)** — ciphertext `L`. `lookup2.index('L') = 11`. `(cur - 3) % 40 = 11` → `cur = 14`. `lookup1[14] = 'a'`. `prev` becomes 14.
4. **Continuing** this loop for all 237 characters produces the full recovered script (`#asciiorder` ... `b += 1 / 1`).
5. **Self-extraction** — the recovered script, when run on itself, prints chars at indices `1^3, 2^3, 3^3, 4^3, 5^3, 6^3` = `1, 8, 27, 64, 125, 216`.
6. **Pick those chars from the recovered text** — index 1 is `a`, index 8 is `d`, index 27 is `l`, index 64 is `i`, index 125 is `b`, index 216 is `s`. Concatenated: `adlibs`.

---

## Tools Used

| Tool             | Purpose                                                                 |
|------------------|-------------------------------------------------------------------------|
| `wget` / browser | Download ciphertext and encoder from the challenge page.               |
| `cat`            | Quick look at the encoder to understand the algorithm.                  |
| `python3`        | Reverse the encoder (decoder.py) and emulate the recovered Python 2 logic. |
| `nano`           | Edit the decoder / solver scripts.                                      |
| (optional) `python2` | Run the recovered script natively if you prefer literal execution.   |

---

## Key Takeaways

- **Read the encoder carefully.** Even when you are told the encoder is the secret, you usually get it as part of the challenge. Spend five minutes reading every line before writing any code.
- **Cyclical does not mean irreversible.** Anything that is a pure function of `prev` and the current char can be walked backwards step by step, as long as you know the initial `prev` (here, 0).
- **Comments are clues.** The four header comments in the recovered script (`#asciiorder`, `#fortychars`, `#selfinput`, `#pythontwo`) describe the cipher and tell you what to do next. `#selfinput` is the one that matters: feed the script to itself.
- **Watch for Python 2 tells.** `1 / 1`, `print x` without parens, and integer division all hint at Python 2. You can usually emulate the logic in Python 3 instead of installing a second interpreter.
- **Read the prompt's flag wrapper hint.** The challenge tells you to wrap with `picoCTF(...)`. Easy to forget when you are staring at output.

Flag decoded: `picoCTF{adlibs}` — *ad libs*, the improvised lines a performer fills in on the spot. Fitting for a cipher that hands you the stage directions (the encoder) and expects you to deliver the punchline yourself.
