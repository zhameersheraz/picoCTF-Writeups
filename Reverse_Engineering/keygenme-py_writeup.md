# keygenme-py — picoCTF Writeup

**Challenge:** keygenme-py  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 30  
**Flag:** `picoCTF{1n_7h3_kk3y_of_08c46aa4}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Can you get the flag? Reverse engineer this binary.

The challenge gives us a Python script called `keygenme-trial.py`. It pretends to be a "trial version" of an "Arcane Calculator" that needs a license key. Entering the right key unlocks the full version and the flag.

## Hints

> 1. spot the bug in the dynamic part of the key

That is the entire hint. The author is telling us where to look: not at the static prefix, not at the closing brace, but at the *dynamic* 8-character segment in the middle of the key. The bug is in how those 8 characters are checked — they are checked individually, one character at a time, in a way that lets us reverse the algorithm.

---

## Background Knowledge

Five small ideas make the rest of the solve trivial. None of them are advanced.

**1. What a "license key" usually looks like.**
A typical software license key is a fixed-format string. In this challenge the key is `picoCTF{1n_7h3_kk3y_of_xxxxxxxx}` — that is the picoCTF flag format (`picoCTF{...}`) wrapping a wordplay prefix (`1n_7h3_kk3y_of` is leet for "in the key of") and an 8-character dynamic segment (`xxxxxxxx`). The first 23 characters and the last character are **known**. Only the 8 middle characters are unknown, and they are the only thing standing between us and the flag.

**2. SHA-256 hex digest.**
`hashlib.sha256(b"some string").hexdigest()` returns a 64-character hex string — that is, 64 characters each in `[0-9a-f]`. So each "character" of a SHA-256 digest is one of 16 possible values. The 8-character dynamic segment of the license key is 8 such hex characters, picked from specific positions of the digest. There is no randomness — the digest of `"BENNETT"` is the same every time you run it.

**3. `check_key` is not magic — it is just a series of `==` checks.**
Every block in `check_key` is shaped like:

```python
if key[i] != hashlib.sha256(username_trial).hexdigest()[<some index>]:
    return False
else:
    i += 1
```

That is a one-character comparison. It does **not** check a substring; it does **not** check a hash of the whole key; it does not even check the closing `}` (try it — replacing the `}` with `!` still passes). It is a flat list of 8 independent `==` checks, each on one hex character. That makes it trivial to invert: we just compute the digest, copy the right hex characters in the right order, and we have the key.

**4. The order in `check_key` is deliberately scrambled.**
The 8 dynamic checks pull hex digits from positions `[4, 5, 3, 6, 2, 7, 1, 8]` of the digest. That is a *permutation* of `1..8` — every position is used exactly once, but not in numerical order. The author scrambled them on purpose so that you cannot just "spot the bug" by reading the static prefix and guessing the suffix. You have to actually read the checks, note the indices, and reassemble the 8 characters in the same scrambled order. The hint ("spot the bug in the dynamic part of the key") is reminding you that the scrambling is not a real defense — it is decoration.

**5. `cryptography.Fernet` is not relevant to the flag.**
The full version of the program decrypts an embedded Fernet blob using the license key as the Fernet key. That part is real cryptography, but it does not produce the flag — it produces a second Python file that runs the "full version" UI. We do not even need to decrypt it: the flag is the license key itself, and a valid key is enough to submit. (If you do decrypt, you get a file that prints "Welcome to the Arcane Calculator, tron!" and adds a new menu option — no flag inside.)

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The challenge gives us the source file `keygenme-trial.py` directly (no need to download a binary). Everything below runs in plain Python — no Ghidra, no gdb, no `objdump`.

### 1. Save the attachment and read the keygen logic

```
┌──(zham㉿kali)-[~/keygenme]
└─$ mkdir -p ~/keygenme && cd ~/keygenme

┌──(zham㉿kali)-[~/keygenme]
└─$ cp ~/Downloads/keygenme-trial.py .

┌──(zham㉿kali)-[~/keygenme]
└─$ wc -l keygenme-trial.py
226 keygenme-trial.py

┌──(zham㉿kali)-[~/keygenme]
└─$ grep -nE "key_part|check_key|hashlib|hexdigest" keygenme-trial.py
17:username_trial = "BENNETT"
20:key_part_static1_trial = "picoCTF{1n_7h3_kk3y_of_"
21:key_part_dynamic1_trial = "xxxxxxxx"
22:key_part_static2_trial = "}"
23:key_full_template_trial = key_part_static1_trial + key_part_dynamic1_trial + key_part_static2_trial
114:def check_key(key, username_trial):
124:        if key[i] != hashlib.sha256(username_trial).hexdigest()[4]:
129:        if key[i] != hashlib.sha256(username_trial).hexdigest()[5]:
134:        if key[i] != hashlib.sha256(username_trial).hexdigest()[3]:
139:        if key[i] != hashlib.sha256(username_trial).hexdigest()[6]:
144:        if key[i] != hashlib.sha256(username_trial).hexdigest()[2]:
149:        if key[i] != hashlib.sha256(username_trial).hexdigest()[7]:
154:        if key[i] != hashlib.sha256(username_trial).hexdigest()[1]:
159:        if key[i] != hashlib.sha256(username_trial).hexdigest()[8]:
```

Three things jump out immediately:

- The username is `"BENNETT"` (line 17). That is the only thing that feeds the hash.
- The key is `picoCTF{1n_7h3_kk3y_of_xxxxxxxx}` (lines 20–23). The first 23 chars and the trailing `}` are known.
- The 8 checks in `check_key` (lines 124–159) read hex digits from positions `[4, 5, 3, 6, 2, 7, 1, 8]` of `sha256(b"BENNETT").hexdigest()`. They are the *only* checks on the dynamic part.

### 2. Compute the sha256 and inspect the hex digits we need

```
┌──(zham㉿kali)-[~/keygenme]
└─$ python3 -c "
import hashlib
h = hashlib.sha256(b'BENNETT').hexdigest()
print('digest  =', h)
print('digest[4]=', h[4])
print('digest[5]=', h[5])
print('digest[3]=', h[3])
print('digest[6]=', h[6])
print('digest[2]=', h[2])
print('digest[7]=', h[7])
print('digest[1]=', h[1])
print('digest[8]=', h[8])
"
digest  = ba6c084a4d888e1f7c3b0fc71d61c4625708bd915b5e0e60eb73e1667251b567
digest[4]= 0
digest[5]= 8
digest[3]= c
digest[6]= 4
digest[2]= 6
digest[7]= a
digest[1]= a
digest[8]= 4
```

The 8 characters in the order `check_key` wants are: `0`, `8`, `c`, `4`, `6`, `a`, `a`, `4`. Sticking them in that order gives the dynamic segment `08c46aa4`.

### 3. Assemble the full key by hand

Static prefix: `picoCTF{1n_7h3_kk3y_of_` (23 chars)  
Dynamic part: `08c46aa4` (8 chars — from step 2)  
Static suffix: `}` (1 char)

Glue them together: `picoCTF{1n_7h3_kk3y_of_08c46aa4}`.

### 4. Sanity-check against `check_key` before submitting

It is always a good idea to replay the check yourself before sending the flag to the server. Let me run the *same* `check_key` body against our key:

```
┌──(zham㉿kali)-[~/keygenme]
└─$ python3 -c "
import hashlib

def check_key(key, username_trial):
    key_part_static1_trial = 'picoCTF{1n_7h3_kk3y_of_'
    key_part_dynamic1_trial = 'xxxxxxxx'
    key_part_static2_trial = '}'
    template = key_part_static1_trial + key_part_dynamic1_trial + key_part_static2_trial
    if len(key) != len(template): return False
    i = 0
    for c in key_part_static1_trial:
        if key[i] != c: return False
        i += 1
    h = hashlib.sha256(username_trial).hexdigest()
    for idx in [4, 5, 3, 6, 2, 7, 1, 8]:
        if key[i] != h[idx]: return False
        i += 1
    return True

print(check_key('picoCTF{1n_7h3_kk3y_of_08c46aa4}', b'BENNETT'))
"
True
```

`True`. The key passes our local re-implementation of `check_key`.

### 5. Confirm the key actually unlocks the trial program

```
┌──(zham㉿kali)-[~/keygenme]
└─$ printf 'c\npicoCTF{1n_7h3_kk3y_of_08c46aa4}\nd\n' | python3 keygenme-trial.py
===============================================
Welcome to the Arcane Calculator, BENNETT!

This is the trial version of Arcane Calculator.
...
___Arcane Calculator___

Menu:
(a) Estimate Astral Projection Mana Burn
(b) [LOCKED] Estimate Astral Slingshot Approach Vector
(c) Enter License Key
(d) Exit Arcane Calculator
What would you like to do, BENNETT (a/b/c/d)? 
Enter your license key: 
Full version written to 'keygenme.py'.

Exiting trial version...
```

The program wrote `keygenme.py` (the decrypted full version) and exited the trial loop. That is what `decrypt_full_version` does on a valid key — it is the program's own "the key is correct" signal.

### 6. Submit the flag

The flag is `picoCTF{1n_7h3_kk3y_of_08c46aa4}`. (Your instance may print a slightly different dynamic suffix if picoCTF rotates the username, but for `BENNETT` this is the answer.)

---

## Alternative Solves

**A. Skip `check_key` entirely — just print the answer from a one-liner.**
If you do not want to copy the loop, you can collapse the whole reverse-engineering step into a single Python expression that does the same thing `check_key` does, in the same order:

```
┌──(zham㉿kali)-[~/keygenme]
└─$ python3 -c "
import hashlib
h = hashlib.sha256(b'BENNETT').hexdigest()
dyn = ''.join(h[i] for i in [4,5,3,6,2,7,1,8])
print(f'picoCTF{{1n_7h3_kk3y_of_{dyn}}}')
"
picoCTF{1n_7h3_kk3y_of_08c46aa4}
```

That is the whole keygen in 4 lines. It works because the indices `[4,5,3,6,2,7,1,8]` are visible right in the source — there is no hidden logic to discover.

**B. Wrap the one-liner into a reusable `keygen.py`.**
If you want a script you can keep around (handy when you re-solve a challenge on a new instance or share with a friend), `nano keygen.py` and paste:

```python
#!/usr/bin/env python3
import hashlib

USERNAME = b"BENNETT"
indices  = [4, 5, 3, 6, 2, 7, 1, 8]

digest = hashlib.sha256(USERNAME).hexdigest()
dynamic = "".join(digest[i] for i in indices)
print(f"picoCTF{{1n_7h3_kk3y_of_{dynamic}}}")
```

Save with Ctrl+O, Enter, Ctrl+X.

```
┌──(zham㉿kali)-[~/keygenme]
└─$ python3 keygen.py
picoCTF{1n_7h3_kk3y_of_08c46aa4}
```

Same answer, packaged.

**C. Brute-force by hand if `hashlib` is unavailable.**
In the unlikely event you cannot import `hashlib` (e.g. the Python build was stripped down), you can recreate `sha256("BENNETT")` by piping to `sha256sum` and grabbing the relevant characters:

```
┌──(zham㉿kali)-[~/keygenme]
└─$ printf 'BENNETT' | sha256sum
ba6c084a4d888e1f7c3b0fc71d61c4625708bd915b5e0e60eb73e1667251b567  -

┌──(zham㉿kali)-[~/keygenme]
└─$ DIGEST=$(printf 'BENNETT' | sha256sum | cut -d' ' -f1)
┌──(zham㉿kali)-[~/keygenme]
└─$ DYNAMIC="${DIGEST:4:1}${DIGEST:5:1}${DIGEST:3:1}${DIGEST:6:1}${DIGEST:2:1}${DIGEST:7:1}${DIGEST:1:1}${DIGEST:8:1}"
┌──(zham㉿kali)-[~/keygenme]
└─$ echo "picoCTF{1n_7h3_kk3y_of_${DYNAMIC}}"
picoCTF{1n_7h3_kk3y_of_08c46aa4}
```

Bash's `${var:offset:length}` substring expansion is doing the same job as Python's `digest[i]` here. Same flag, no Python required.

---

## What Happened Internally

1. We opened `keygenme-trial.py` and read top-to-bottom. The two important globals were the username `"BENNETT"` and the three-part key template `picoCTF{1n_7h3_kk3y_of_` + 8 dynamic chars + `}`.
2. We located `check_key` (line 114) and read the 8 dynamic-character checks (lines 124–159). Each check compared one position of our key against one hex digit of `sha256(b"BENNETT").hexdigest()` at a fixed index.
3. We wrote `python3 -c "import hashlib; print(hashlib.sha256(b'BENNETT').hexdigest())"` and got `ba6c084a4d888e1f7c3b0fc71d61c4625708bd915b5e0e60eb73e1667251b567`.
4. We pulled characters at positions `[4, 5, 3, 6, 2, 7, 1, 8]` in that order — i.e. `0`, `8`, `c`, `4`, `6`, `a`, `a`, `4` — and concatenated them as the dynamic segment `08c46aa4`.
5. We prepended the static prefix `picoCTF{1n_7h3_kk3y_of_` and appended `}` to produce the 32-character key `picoCTF{1n_7h3_kk3y_of_08c46aa4}`.
6. We replayed the `check_key` body locally and confirmed `True`. We also fed the key to the real `keygenme-trial.py` via `printf` and watched the program print `Full version written to 'keygenme.py'` — that is the program's own signal that the key was accepted.
7. (Optional.) The decrypted `keygenme.py` does not contain the flag. It is a second copy of the Arcane Calculator with a different username (`"tron"`) and the previously locked menu option `(b)` now unlocked. The flag remains the license key we reconstructed in step 5.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `cp`             | Moved the downloaded `keygenme-trial.py` into a working directory. |
| `wc -l`          | Quick size check — 226 lines, nothing scary.                   |
| `grep -nE`       | Jumped straight to the lines that mention `key_part`, `check_key`, `hashlib`, `hexdigest` so I did not have to read the whole file. |
| `python3 -c "..."` | Computed `sha256("BENNETT")` and inspected the 8 hex digits we need. |
| `printf '...' \| python3 keygenme-trial.py` | Confirmed the key unlocks the trial program end-to-end. |
| `nano keygen.py` | Saved the reusable 4-line keygen script (Alternative Solve B). |
| `sha256sum` + bash `${var:offset:length}` | Pure-shell fallback if Python is unavailable (Alternative Solve C). |

---

## Key Takeaways

- **Read the validation, do not just look at the format.** A license key check that splits the key into 8 independent single-character `==` comparisons is trivially reversible: just compute each character and paste them back in. The "scrambled order" (`[4,5,3,6,2,7,1,8]`) is decoration, not a defense. Whenever you see `check == get_hash()[i]` for individual `i`, the algorithm is invertible.
- **The hint told us exactly where to look.** "Spot the bug in the dynamic part of the key" meant *do not waste time on the prefix or the closing brace* — they are already given to us in `key_part_static1_trial` and `key_part_static2_trial`. The "bug" is that the dynamic part is checked 8 characters at a time instead of as one whole substring (or, better, as a real signature), so we can rebuild it character by character.
- **The closing `}` is not actually checked.** Try replacing the `}` with `!` — `check_key` still returns `True`, because the loop only runs over the 8 dynamic indices and then `return True`s. The length check (`len(key) != len(template)`) only enforces the *length* of 32, not the value of the last byte. This is the author's "bug" called out in the hint. We still submit a key ending in `}` because that is the picoCTF flag format, but it is good to know you could submit `!` and it would be accepted by the program.
- **`sha256` is deterministic and not a secret.** Anyone who knows the username can compute the digest and the license key. Real software license keys do not work this way — they are usually signed by the vendor with a private key so that only the vendor can produce valid keys. Here, the entire security model is "the user does not know the username" — which they do, because it is hardcoded as a global. The "keygen" in the challenge title is literally just a `sha256` computation.
- **The flag's leet:** `1n_7h3_kk3y_of` reads as **in the key of** (`1→i`, `n→n`, `_→_`, `7→t`, `h→h`, `3→e`, `_→_`, `k→k`, `k→k`, `3→e`, `y→y`, `_→_`, `o→o`, `f→f`). Combined with the rest of the flag, `picoCTF{1n_7h3_kk3y_of_<8 hex chars>}` becomes "picoCTF { in the key of <hex> }" — a music-theory pun ("in the key of D minor", etc.) plus a programming pun ("the key is in the SHA-256"). The 8 hex characters themselves (`08c46aa4` here) do not carry wordplay; they are a unique-per-instance identifier so that every picoCTF user gets a different flag and the platform can detect share-spoofing. Your suffix will likely be different from mine; the wordplay up to the closing brace is what stays the same.
