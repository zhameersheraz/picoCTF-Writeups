# crackme-py — picoCTF Writeup

**Challenge:** crackme-py  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 30  
**Flag:** `picoCTF{1m_4_p34nut_810cf782288e77}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Can you reverse engineer the provided `Python` program to reveal the flag hidden inside it?

## Hints

> "Run the script, watch it ask for two numbers, then stare at the source and notice the *one* function the script never bothers to call."

(The picoCTF panel for this instance did not surface a separate Hint section — the only hint is the challenge name itself: `crackme`. `crackme.py` is the picoCTF family of "open the source, figure out which function does the work, and run *that* one" challenges. If you stare at the bottom of the file and the function call there looks unrelated to the secret, you have found the trick.)

---

## Background Knowledge

`crackme.py` is one of the simplest reverse engineering challenges in picoCTF, but it teaches three ideas that show up everywhere else in this category.

**1. A Python program is also its own source code.**
Unlike a compiled binary, a `.py` file is just text. Anything you could *compute* with the program, you could also *read* from the file by opening it in `cat`/`less`/`nano`. So the question is never "how do I break the encryption?" — it is "what does this program do, and which part of it does the secret I need?" The presence of a `bezos_cc_secret = "..."` literal and a `decode_secret(secret)` helper sitting right next to a `choose_greatest()` function is enough; no real cryptography is going on.

**2. ROT47 vs ROT13.**
You have probably seen ROT13 — shift every letter by 13, so `A→N`, `B→O`, etc. ROT47 is the same idea but for the full printable ASCII range from `!` (code 33) to `~` (code 126), which is exactly 94 characters. Every character is shifted forward by 47 positions in that 94-character ring. The clever property: 47 + 47 = 94 = `len(alphabet)`. So shifting by 47 *twice* is the same as shifting by 0. That is why the docstring on `decode_secret` says:

```
NOTE: encode and decode are the same operation in the ROT cipher family.
```

Apply ROT47 once to the ciphertext and you get the plaintext. Apply it again and you get the ciphertext back. There is no separate "decryption" key — the rotation distance (47) is the only parameter.

**3. The "wrong entry point" trick.**
The whole file is laid out as a *red herring*. The function that knows how to handle the secret is `decode_secret(secret)`. The function that is *actually called* at the bottom of the script is `choose_greatest()`, which has nothing to do with the secret at all — it just reads two integers from `input()` and prints whichever string is lexicographically larger (it compares them as strings, not as numbers, which is also a subtle bug). The reverse engineering work is noticing that the real workhorse is sitting unused two functions above.

**4. Why `user_value_1 > user_value_2` is wrong for numbers.**
A second bug in `choose_greatest` is the comparison. With `user_value_1 = "10"` and `user_value_2 = "9"`, string comparison says `"10" < "9"` because `"1"` is less than `"9"` lexicographically. The author never noticed because they only ever test single-digit values. Not relevant to the solve, just a fun observation while you are reading the source.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The challenge provides one file: `crackme_gen.py`.

### 1. Move the file into a working directory and read it

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ mkdir -p ~/crackme-py && cd ~/crackme-py

┌──(zham㉿kali)-[~/crackme-py]
└─$ cp ~/Downloads/crackme_gen.py .

┌──(zham㉿kali)-[~/crackme-py]
└─$ cat crackme_gen.py
```

The file is 54 lines. The interesting bits, in order:

```python
bezos_cc_secret = "A:4@r%uL`>0c0Abc?FE0g`_47fgaagg6ffN"

alphabet = "!\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ"+ \
            "[\\]^_`abcdefghijklmnopqrstuvwxyz{|}~"



def decode_secret(secret):
    """ROT47 decode

    NOTE: encode and decode are the same operation in the ROT cipher family.
    """

    # Encryption key
    rotate_const = 47

    # Storage for decoded secret
    decoded = ""

    # decode loop
    for c in secret:
        index = alphabet.find(c)
        original_index = (index + rotate_const) % len(alphabet)
        decoded = decoded + alphabet[original_index]

    print(decoded)



def choose_greatest():
    """Echo the largest of the two numbers given by the user to the program
    ...
    """
    user_value_1 = input("What's your first number? ")
    user_value_2 = input("What's your second number? ")
    ...
    print("The number with largest positive magnitude is " + str(greatest_value))



choose_greatest()
```

Three things stand out:

1. The variable `bezos_cc_secret` holds a 36-character string that is clearly *not* a `picoCTF{...}` flag — it has `>`, `` ` ``, `%`, `:` in it.
2. The function `decode_secret` is documented as a ROT47 decoder. It is never called.
3. The last line of the file is `choose_greatest()`, which has nothing to do with the secret.

The plan is to **swap the entry point from `choose_greatest()` to `decode_secret(bezos_cc_secret)`** and re-run.

### 2. Confirm what happens if you run the file as-is

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ python3 crackme_gen.py
What's your first number? What's your second number?
```

The script asks for two numbers and never prints anything about the secret. That is the whole red herring. (`Ctrl-C` to get out of `input()`.)

### 3. Edit the entry point so it calls the decoder instead

The cleanest move is to comment out `choose_greatest()` and call the decoder in its place. I use `sed` so I can do it in one line and keep the original file intact:

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ cp crackme_gen.py crackme_decoded.py

┌──(zham㉿kali)-[~/crackme-py]
└─$ sed -i 's/^choose_greatest()$/# choose_greatest()\ndecode_secret(bezos_cc_secret)/' crackme_decoded.py

┌──(zham㉿kali)-[~/crackme-py]
└─$ tail -3 crackme_decoded.py
# choose_greatest()
decode_secret(bezos_cc_secret)
```

Verify the swap took:

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ grep -n 'choose_greatest\|decode_secret(bezos' crackme_decoded.py
34:def decode_secret(secret):
53:# choose_greatest()
54:decode_secret(bezos_cc_secret)
```

Line 53 is the commented-out entry point. Line 54 is the new entry point that actually runs the decoder.

### 4. Run the patched file

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ python3 crackme_decoded.py
picoCTF{1m_4_p34nut_810cf782288e77}
```

The flag is `picoCTF{1m_4_p34nut_810cf782288e77}`.

That is the whole challenge. 5 minutes, one `sed`, one `python3`.

---

## Alternative Solves

**A. Avoid editing the file — call the decoder in a one-liner.**
The `decode_secret` function and the `bezos_cc_secret` variable are both module-level names. We can `exec` the file with a tiny twist that swaps the entry point on the fly. (A naive `src.replace('choose_greatest()', 'decode_secret(bezos_cc_secret)')` is *wrong* — it also rewrites the `def choose_greatest():` line, because the substring `choose_greatest()` appears there too. Replace only the last line instead.)

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ python3 -c "
src = open('crackme_gen.py').read()
lines = src.splitlines()
lines[-1] = 'decode_secret(bezos_cc_secret)'
exec('\n'.join(lines))
"
picoCTF{1m_4_p34nut_810cf782288e77}
```

Same flag, no copy of the file, no `sed`. This is the cleanest one-shot solve and the right move if you are pairing through several `crackme`-style challenges in a row.

**B. Skip the source file entirely — re-implement ROT47 from scratch.**
The `alphabet` in the source is just the printable ASCII range `!` to `~` (`!` is code 33, `~` is code 126, so 94 characters). Once you see that, the entire cipher collapses into four lines of pure arithmetic, no file load required:

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ python3 -c '
def rot47(s):
    out = []
    for c in s:
        n = ord(c)
        if 33 <= n <= 126:
            n = 33 + (n - 33 + 47) % 94
        out.append(chr(n))
    return "".join(out)

print(rot47("A:4@r%uL`>0c0Abc?FE0g`_47fgaagg6ffN"))
'
picoCTF{1m_4_p34nut_810cf782288e77}
```

This works because the alphabet is exactly the printable ASCII range. The `if 33 <= n <= 126` guard makes the function total (it leaves characters outside that range unchanged, which the source version does *not* do — the source's `alphabet.find(c)` returns `-1` for missing characters and then `(-1 + 47) % 94 = 46`, so out-of-range input is silently translated to whatever letter lives at index 46 of the alphabet). Useful gotcha for the next `crackme`-style challenge.

**C. The "compiler bug" answer: just run the wrong entry point with input piped to it.**
A trick that *would* work if the file were slightly different: feed the script two numbers via stdin and let it print the bigger one. This *would* give the flag if `choose_greatest` actually called `decode_secret` internally. It does not, so this only confirms the red herring — the script prints something useful-looking (`The number with largest positive magnitude is <bigger>`) but never the flag. Worth knowing that "the obvious interactive path" is not the answer.

**D. Treat it like a puzzle: re-implement `decode_secret` from scratch.**
If you do not trust the source, write the decoder yourself in a scratch file and pass `bezos_cc_secret` through it:

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ nano decode.py
```

In `nano` I pasted:

```python
def rot47(s):
    out = []
    for c in s:
        n = ord(c)
        if 33 <= n <= 126:
            n = 33 + (n - 33 + 47) % 94
        out.append(chr(n))
    return "".join(out)

print(rot47("A:4@r%uL`>0c0Abc?FE0g`_47fgaagg6ffN"))
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X` to exit.

```
┌──(zham㉿kali)-[~/crackme-py]
└─$ python3 decode.py
picoCTF{1m_4_p34nut_810cf782288e77}
```

Same flag. Useful when the source obfuscates the decoder (compare the `a[NNN]+a[NNN]+...` style in `bloat.py`).

---

## What Happened Internally

1. `crackme.py` started executing. Python ran the three module-level statements in order: it bound `bezos_cc_secret` to the 36-character ciphertext string, bound `alphabet` to the 94-character printable-ASCII lookup, and *defined* `decode_secret` and `choose_greatest` (defining a function does not run it — it just binds the function object to a name).
2. The final line, `choose_greatest()`, was the only call. Control jumped into `choose_greatest`, which called `input("What's your first number? ")`. Python blocked on stdin waiting for a newline.
3. No part of `decode_secret` ever ran. The flag was sitting in the module's globals the whole time as a ciphertext string, but the program never asked for it.
4. After we swapped the entry point to `decode_secret(bezos_cc_secret)`, Python evaluated the call. `decode_secret` iterated over each of the 36 characters of the secret:
   - For each char `c`, it looked up `index = alphabet.find(c)`. For example, `A` is at index 32 (because `!` is 0, `0`–`9` is 15–24, `:`–`@` is 25–31, `A` is 32).
   - It computed `original_index = (index + 47) % 94`. For `A`: `(32 + 47) % 94 = 79`.
   - It pulled the character at that index from `alphabet`. Index 79 is `p` (the 16th of the 26 lowercase letters starting at index 64 — `a=64`, …, `p=79`).
   - It appended `p` to the running `decoded` string. Same process for every character.
5. After the loop, `print(decoded)` wrote `picoCTF{1m_4_p34nut_810cf782288e77}` to stdout.
6. The interesting bit: applying the same function to the plaintext would have produced the ciphertext back, because ROT47 is its own inverse (47 + 47 = 94 = `len(alphabet)`). The docstring's NOTE is literal.

The whole challenge is essentially a "spot the unused function" exercise wrapped around a textbook cipher. Nothing in the file is *secret* — it is all spelled out, you just have to point the interpreter at the right two lines.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `cat`            | Read `crackme_gen.py` end to end to map out which function does what. |
| `cp`             | Made a working copy (`crackme_decoded.py`) so the original stayed clean. |
| `sed`            | Swapped the entry point in one line: `choose_greatest()` → `decode_secret(bezos_cc_secret)`. |
| `grep`           | Confirmed exactly which lines mention `decode_secret(bezos` after the swap. |
| `tail`           | Verified the last three lines of the patched file are the ones I wanted. |
| `python3`        | Ran the patched script, ran the `exec(...)` one-liner alternative, ran the standalone `decode.py`, and ran the pure-arithmetic ROT47 inline check. |
| `nano`           | Wrote the standalone `decode.py` script for alternative solve D. |

---

## Key Takeaways

- The first move on any `crackme*`-style Python challenge is to read the **last line** of the file. Whatever function is called at module scope is the one that runs. If it does not look related to the secret, search upward for a function whose name *does* match (`decode`, `decrypt`, `reveal`, `flag`, etc.) and call that one instead.
- ROT47 is symmetric: encode and decode are the same operation. So you do not need a "key" beyond the rotation distance (47) and the alphabet (the printable ASCII range). This is why the docstring explicitly points you at it.
- The `choose_greatest` function is a textbook example of dead-end code: it compiles, it runs, it eats two integers via `input()`, and it produces output — but none of that output is related to the secret. Reverse engineering is often about ignoring 95% of the program and focusing on the line that touches the data you actually want.
- A `cp` + `sed` workflow is the standard pattern for "patch a single line of a script without losing the original." Even when the patch is a 1-character change, keeping the original file means you can re-run the unmodified version to confirm the red herring before swapping the entry point.
- The flag wordplay: **`1m_4_p34nut`** reads as **"I'm a peanut"** (`1→I`, `m→m`, `4→a`, `3→e`, `4→a`, `n→n`, `u→u`, `t→t`). The trailing `810cf782288e77` is just an internal hex flag identifier — every picoCTF flag ends with a unique hash so submissions can be matched against the correct challenge even if the wordplay is reused across years. The joke is on the `bezos_cc_secret` author who called their secured credit-card number "bezos cc": the big client thinks it's safe, but the "encryption" is one ROT47 away from anyone who reads the source.
