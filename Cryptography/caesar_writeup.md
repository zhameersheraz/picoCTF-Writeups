# caesar - picoCTF Writeup

**Challenge:** caesar  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{crossingtherubiconrgqcnbyq}`  
**Platform:** picoCTF (2024)  
**Writeup by:** zham  

---

## Description

> Decrypt this message.

## Hints

> 1. Caesar cipher tutorial

---

## Background Knowledge

Before jumping in, here are the concepts behind this challenge.

### What is a Caesar Cipher?

A **Caesar cipher** is one of the oldest substitution ciphers on record. Legend says Julius Caesar used it for his military correspondence, which is exactly where the cipher gets its name. The idea is brutally simple:

- Pick a single number called a **shift** (or **key**).
- For every letter in the message, shift it forward in the alphabet by that many positions.
- A becomes D if the shift is 3. B becomes E. X loops around to A.
- Non-letter characters usually stay put.

Because only one number controls the whole message, a Caesar cipher has only **25 useful shifts** to try (shifting by 0 just gives back the original, and shifting by 26 wraps back to the same thing). That makes any Caesar-encrypted message breakable in seconds by trying every shift and looking for the one that produces readable English.

In this challenge, the flag wrapper `picoCTF{...}` is left in plaintext. Only the part **inside** the braces is Caesar-encrypted, so our job is to find the one shift that turns the gibberish inside the braces back into a real English phrase.

### What is ROT13?

**ROT13** is a Caesar cipher locked at shift 13. It is one of the most common "hello world" examples of classical cryptography because it has a fun property: applying ROT13 twice returns the original text. That works because the alphabet has 26 letters and 13 + 13 = 26, so every letter is shifted exactly halfway around the alphabet the first time and all the way back the second time.

ROT13 is not secure for anything — it shows up mostly on internet forums as a tongue-in-cheek way to hide spoilers. For our challenge we are not using ROT13 directly, but the same brute-force idea works for any Caesar shift, including the one the challenge picked.

### Putting the Pieces Together

The challenge encrypts one English phrase with one Caesar shift and wraps it in `picoCTF{...}`. The whole attack is "try all 25 shifts, eyeball the result." In practice that means a 7-line Python loop, which is what we will write next.

---

## Solution

### Step 1: Save the Encrypted Message

I grabbed the wrapped ciphertext from the challenge page and saved it to a file so the rest of the work happens in the terminal.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ mkdir -p ~/picoCTF/cryptography/caesar && cd ~/picoCTF/cryptography/caesar

┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ nano message.txt
```

In `nano`, I pasted:

```
picoCTF{hwtxxnslymjwzgnhtswlvhsgdv}
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X` (the usual nano dance).

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ cat message.txt
picoCTF{hwtxxnslymjwzgnhtswlvhsgdv}
```

### Step 2: Strip the Wrapper, Eyeball the Inside

The `picoCTF{...}` wrapper is plaintext. Only the contents of the braces are Caesar-encrypted.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ grep -oP '(?<=picoCTF\{)[^}]+' message.txt
hwtxxnslymjwzgnhtswlvhsgdv
```

Twenty-six letters of gibberish. We know the result should be lowercase English, so we just need to find the right shift.

### Step 3: Write the Brute-Force Solver in `nano`

There are only 25 useful shifts to try, so a tiny Python loop is the cleanest approach.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ nano solve.py
```

In `nano`, I pasted:

```python
#!/usr/bin/env python3

cipher = "hwtxxnslymjwzgnhtswlvhsgdv"

for shift in range(26):
    decoded = ""
    for c in cipher:
        if c.isalpha():
            base = ord("a") if c.islower() else ord("A")
            decoded += chr((ord(c) - base - shift) % 26 + base)
        else:
            decoded += c
    print(f"shift={shift:2d}: picoCTF{{{decoded}}}")
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X`.

### Step 4: Run It and Pick the English One

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ python3 solve.py
shift= 0: picoCTF{hwtxxnslymjwzgnhtswlvhsgdv}
shift= 1: picoCTF{gvswwmrkxlivyfmgsrvkugrfcu}
shift= 2: picoCTF{furvvlqjwkhuxelfrqujtfqebt}
shift= 3: picoCTF{etquukpivjgtwdkeqptisepdas}
shift= 4: picoCTF{dspttjohuifsvcjdposhrdoczr}
shift= 5: picoCTF{crossingtherubiconrgqcnbyq}
shift= 6: picoCTF{bqnrrhmfsgdqtahbnmqfpbmaxp}
shift= 7: picoCTF{apmqqglerfcpszgamlpeoalzwo}
shift= 8: picoCTF{zolppfkdqeboryfzlkodnzkyvn}
shift= 9: picoCTF{ynkooejcpdanqxeykjncmyjxum}
shift=10: picoCTF{xmjnndiboczmpwdxjimblxiwtl}
shift=11: picoCTF{wlimmchanbylovcwihlakwhvsk}
shift=12: picoCTF{vkhllbgzmaxknubvhgkzjvgurj}
shift=13: picoCTF{ujgkkafylzwjmtaugfjyiuftqi}
shift=14: picoCTF{tifjjzexkyvilsztfeixhtesph}
shift=15: picoCTF{sheiiydwjxuhkrysedhwgsdrog}
shift=16: picoCTF{rgdhhxcviwtgjqxrdcgvfrcqnf}
shift=17: picoCTF{qfcggwbuhvsfipwqcbfueqbpme}
shift=18: picoCTF{pebffvatgurehovpbaetdpaold}
shift=19: picoCTF{odaeeuzsftqdgnuoazdscoznkc}
shift=20: picoCTF{nczddtyrespcfmtnzycrbnymjb}
shift=21: picoCTF{mbyccsxqdrobelsmyxbqamxlia}
shift=22: picoCTF{laxbbrwpcqnadkrlxwapzlwkhz}
shift=23: picoCTF{kzwaaqvobpmzcjqkwvzoykvjgy}
shift=24: picoCTF{jyvzzpunaolybipjvuynxjuifx}
shift=25: picoCTF{ixuyyotmznkxahoiutxmwithew}
```

Look at line `shift= 5`. The contents spell out **crossingtherubiconrgqcnbyq** — the famous historical phrase "crossing the Rubicon," followed by a short noise tail. That is our flag.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ echo "picoCTF{crossingtherubiconrgqcnbyq}"
picoCTF{crossingtherubiconrgqcnbyq}
```

### Step 5: Submit

Paste `picoCTF{crossingtherubiconrgqcnbyq}` into the flag box. Correct on first try.

**Flag:** `picoCTF{crossingtherubiconrgqcnbyq}`

---

## Alternative Solve Methods

### Method 1: Pure Shell with `tr`

If you do not want to touch Python, `tr` can decode a single shift inline. For shift 5, the alphabet maps `a->v, b->w, c->x, d->y, e->z, f->a, ..., z->u`. That argument string is just the alphabet rotated 5 positions to the right (`vwxyzabcdefghijklmnopqrstu`).

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ INSIDE='hwtxxnslymjwzgnhtswlvhsgdv'

┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ echo -n "$INSIDE" | tr 'a-z' 'vwxyzabcdefghijklmnopqrstu'
crossingtherubiconrgqcnbyq
```

Combine the two pieces into one line and you have the whole flag:

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/caesar]
└─$ echo "picoCTF{$(echo -n "$INSIDE" | tr 'a-z' 'vwxyzabcdefghijklmnopqrstu')}"
picoCTF{crossingtherubiconrgqcnbyq}
```

Same flag. `tr` is the right tool when you already know the shift and just want a quick decode.

### Method 2: CyberChef "ROT Brute Force"

If you would rather stay in the GUI, CyberChef has a one-recipe solution. Drop the inner ciphertext into the Input box, then add the **ROT Brute Force** operation. It outputs all 25 possible shifts at once, so the English line jumps out without writing any code. This is my go-to fallback when I am on a machine without Python.

### Method 3: Frequency-Analysis Eyeball

For extra credit (or for when the message is long enough that eyeballing every shift is annoying), you can crack a Caesar cipher without trying every shift explicitly:

1. Count the letters in the ciphertext.
2. In English, `e` is the most common letter by a wide margin, followed by `t`, `a`, `o`, `i`, `n`, `s`, `h`, `r`.
3. The most common letter in the ciphertext is almost certainly `e` shifted by the same shift as everything else.
4. Compute the shift as `(cipher_letter_position - e_position) mod 26` and try it.

For this challenge the ciphertext is only 26 letters long, so the frequency ranking is noisy and this method is overkill — but it scales beautifully to longer messages and to languages where you do not want to brute force.

---

## What Happened Internally

Here is the timeline of what was going on, from "I see a ciphertext" to "I have the flag."

1. **Read the ciphertext.** The challenge handed me `picoCTF{hwtxxnslymjwzgnhtswlvhsgdv}`. The wrapper was preserved so the server could confirm flag shape; only the inside was scrambled.
2. **Stripped the wrapper.** `grep -oP '(?<=picoCTF\{)[^}]+'` extracted `hwtxxnslymjwzgnhtswlvhsgdv`, leaving 26 letters of pure ciphertext to attack.
3. **Ran the brute force.** For each candidate shift `s` from 0 to 25, I computed `D(c) = (c - s) mod 26` for every letter `c` in the ciphertext. Modulo 26 wraps `a` back to `z` and `z` forward to `y`, which is why the alphabet loops instead of throwing an error.
4. **Identified the right shift.** I scanned the 26 outputs for English. `shift=5` produced `crossingtherubiconrgqcnbyq`, a well-known historical reference — that unique "signature" is what made this shift stand out among 25 candidates.
5. **Submitted.** The recovered plaintext `picoCTF{crossingtherubiconrgqcnbyq}` was sent back to the server, which matched it against the stored flag and awarded the 100 points.

The whole attack is called a **brute force** because we tried every possible key. With a 25-key space, that takes microseconds — and that is exactly why the Caesar cipher is never used for anything real.

---

## Tools Used

| Tool        | Purpose                                                          |
| ----------- | ---------------------------------------------------------------- |
| `mkdir`     | Create a working directory for the challenge                     |
| `nano`      | Write `message.txt` and `solve.py` (`Ctrl+O`, `Enter`, `Ctrl+X`) |
| `cat`       | Display the saved ciphertext                                    |
| `grep -oP`  | Strip the `picoCTF{...}` wrapper, keep only the inner ciphertext |
| `python3`   | Run the 26-shift brute force and print every candidate           |
| `tr`        | Shell-only alternative for single-shift decoding                 |
| `echo`      | Print the final flag                                            |
| CyberChef   | Optional GUI alternative using the **ROT Brute Force** recipe   |

---

## Key Takeaways

- A Caesar cipher is one of the simplest classical substitution ciphers, and its keyspace is only 25 shifts — small enough to brute-force by hand or with a 7-line script.
- When the flag wrapper is fixed (`picoCTF{...}`), only the inner portion is encrypted. Pull the inner text out first, then attack.
- Writing a tiny Python loop and iterating all 26 shifts is the fastest, most reliable way to crack any Caesar cipher. The `mod 26` trick is what makes the alphabet wrap cleanly.
- `tr` is the perfect shell-only fallback when you already know the shift, and CyberChef's **ROT Brute Force** is the GUI equivalent when you do not want to write code.
- Frequency analysis (count letters, map the most common one to `e`) is a brute-force-free alternative that scales to long messages.
- For real cryptography, always reach for a real cipher like AES or ChaCha20 — Caesar is a teaching toy.

**Flag wordplay decode:** the decoded phrase is **"crossing the Rubicon"** — the famous 49 BC moment when Julius Caesar marched his army across the Rubicon river, committing to civil war and crossing a "point of no return." A neat wink from the challenge author, because the cipher itself was allegedly invented by that very same Julius Caesar. The ciphertext and the plaintext share a namesake.
