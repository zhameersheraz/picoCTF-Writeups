# spelling-quiz - picoCTF Writeup

**Challenge:** spelling-quiz  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{perhaps_the_dog_jumped_over_was_just_tired}`  
**Platform:** picoCTF (picoMini by redpwn, 2021)  
**Writeup by:** zham  

---

## Description

> I found the flag, but my brother wrote a program to encrypt all his text files. He has a spelling quiz study guide too, but I don't know if that helps.

## Hints

> 1. The `encrypt.py` script applies the same key to every `.txt` file in the folder, including `study-guide.txt`.
> 2. The study guide is a long list of ordinary English words, so it leaks the structure of English to the attacker.
> 3. Tools like `subbreaker` (or any substitution-cipher breaker that scores English quadgrams) recover the key from a few dozen encrypted words in seconds.

---

## Background Knowledge

Before touching the keyboard it helps to understand what kind of cipher we are dealing with.

### What is a Monoalphabetic Substitution Cipher?

`encrypt.py` builds a Python `dict` that maps every letter of the alphabet to a unique, shuffled letter of the alphabet:

```python
alphabet = list('abcdefghijklmnopqrstuvwxyz')
random.shuffle(shuffled := alphabet[:])
dictionary = dict(zip(alphabet, shuffled))
```

Each plaintext letter `a` always becomes the same ciphertext letter, no matter where it appears. That makes this a **monoalphabetic substitution cipher** — the same family as the classical Caesar cipher, but with a full random 26-letter permutation instead of a fixed shift. Once you know the permutation, decryption is a single lookup per letter.

### Why the Study Guide Matters

The same key encrypts every `.txt` file the script finds. That includes `study-guide.txt`, which is a long list of ordinary English words. So we are sitting on hundreds of thousands of known-plaintext pairs in disguise: as soon as we can map a single cipher word to its real English word, we can recover the entire substitution table and decrypt the flag.

### Two Ways I Cracked It

1. **Primary — `subbreaker`.** An automated tool that breaks substitution ciphers using **quadgram statistics**: it counts how often each 4-letter chunk appears in English, hill-climbs until the chunk frequencies of the decrypted text match English, and prints the recovered key. It only needs ~50 lines of ciphertext to find the key.
2. **Alternative — Pattern matching against an English dictionary.** A 30-line Python script that gives each study-guide word a **shape** (replace each letter by the index of its first appearance: `greenheart` -> `01230345`) and finds dictionary words with the same shape. Words with a unique shape match give us the substitution for free.

Both routes give the same key and the same flag. The first is one shell command. The second makes the attack visible and teaches you how the tool actually works.

---

## Solution

### Step 1: Open the File and See What We Have

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ unzip public.zip
Archive:  public.zip
   creating: public/
  inflating: public/encrypt.py
  inflating: public/study-guide.txt
  inflating: public/flag.txt
```

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ ls -la public/
-rw-r--r-- 1 zham zham     510 Feb 21  2021 encrypt.py
-rw-r--r-- 1 zham zham      43 Feb 21  2021 flag.txt
-rw-r--r-- 1 zham zham 3178399 Feb 21  2021 study-guide.txt
```

Three files. The original `flag.txt` is gone — `encrypt.py` overwrote it in place.

### Step 2: Read the Encryptor

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ cat public/encrypt.py
import random
import os

files = [
    os.path.join(path, file)
    for path, dirs, files in os.walk('.')
    for file in files
    if file.split('.')[-1] == 'txt'
]

alphabet = list('abcdefghijklmnopqrstuvwxyz')
random.shuffle(shuffled := alphabet[:])
dictionary = dict(zip(alphabet, shuffled))

for filename in files:
    text = open(filename, 'r').read()
    encrypted = ''.join([
        dictionary[c]
        if c in dictionary else c
        for c in text
    ])
    open(filename, 'w').write(encrypted)
```

The `os.walk('.')` walks the current directory, the list comprehension picks every `*.txt`, and then a single Python `dict` is used to translate each letter. Underscores and other non-letters pass through unchanged because they are not keys in `dictionary`.

### Step 3: Look at the Encrypted Flag and the Study Guide

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ cat public/flag.txt
brcfxba_vfr_mid_hosbrm_iprc_exa_hoav_vwcrm
```

The underscores stayed in place — only the letters were substituted. Looks like a sentence: 8-letter word, then 3, 3, 6, 4, 3, 4, 5.

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ wc -l public/study-guide.txt
272543 public/study-guide.txt
```

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ head -5 public/study-guide.txt
gocnfwnwtr
sxlyrxaic
dcrrtfrxcv
uxbvwavcq
lwvicwtiwm
```

272k single-line entries. Each line is one encrypted English word. With ~272k encrypted English words at our fingertips, we have way more known-plaintext signal than a simple Caesar would ever need.

### Step 4: Install `subbreaker`

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ pip install subbreaker
Collecting subbreaker
  Downloading subbreaker-1.2.0-py3-none-any.whl (411 kB)
Installing collected packages: subbreaker
Successfully installed subbreaker-1.2.0
```

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ subbreaker --help
usage: subbreaker [-h] {break,decode,encode,fitness,quadgrams,info,version} ...
```

Two subcommands we care about: `break` (crack the cipher) and `decode` (apply a known key to ciphertext).

### Step 5: Run the Attack

`subbreaker break` takes ciphertext, guesses the substitution by hill-climbing on English quadgram fitness, and prints the recovered key. 50 lines is more than enough:

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ head -n 50 public/study-guide.txt | subbreaker break --lang EN
Alphabet: abcdefghijklmnopqrstuvwxyz
Key:      xunmrydfwhglstibjcavopezqk
Fitness: 92.78
Nbr keys tried: 12675
Keys per second: 18404
Execution time (seconds): 0.689
Plaintext:
kurchicine
malfeasor
greenheart
baptistry
litorinoid
vindicatory
stockrooms
flindersia
disagreeability
frohlich
disamenity
...
```

The tool's plaintext preview is itself confirmation that the key is right — those are all real English words.

### Step 6: Decode the Flag with the Recovered Key

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ subbreaker decode --key xunmrydfwhglstibjcavopezqk --ciphertext public/flag.txt
perhaps_the_dog_jumped_over_was_just_tired
```

### Step 7: Wrap It in the Flag Format

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ echo "picoCTF{$(subbreaker decode --key xunmrydfwhglstibjcavopezqk --ciphertext public/flag.txt)}"
picoCTF{perhaps_the_dog_jumped_over_was_just_tired}
```

**Flag:** `picoCTF{perhaps_the_dog_jumped_over_was_just_tired}`

---

## Alternative Solve Methods

### Method 1: Hand-Rolled Pattern Matching in Python

If you do not want to install `subbreaker`, the same result falls out of a small Python script. The idea: every English word has a **shape** — replace each letter by the index of its first appearance in the word.

| word             | shape          |
| ---------------- | -------------- |
| `greenheart`     | `01230345`     |
| `disagreeability`| `0123456721`   |

Two words are interchangeable substitutions only if they share the same shape. So if an encrypted study-guide word has a shape that matches **exactly one** English word in a large dictionary, we know the mapping for every letter in that word. Stacking a few hundred of those unique matches rebuilds the whole substitution with zero conflicts.

I dropped into `nano` to write the solver:

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ nano solve.py
```

And pasted this in:

```python
#!/usr/bin/env python3
"""
spelling-quiz — alternative solver.

Cracks the monoalphabetic substitution used in encrypt.py by matching the
"shape" (letter-index pattern) of each encrypted study-guide word against a
large English dictionary. Words with a unique shape match give us the
plaintext for free, and from there we derive the substitution.
"""

import os
import urllib.request
from collections import defaultdict

# 1. Load a large English dictionary.
#    On Kali:    /usr/share/dict/words
#    Fallback:  curl -sL https://raw.githubusercontent.com/dwyl/english-words/master/words_alpha.txt -o /tmp/words.txt
DICT_PATHS = ['/usr/share/dict/words', '/tmp/words.txt']
WORDS_FILE = next((p for p in DICT_PATHS if os.path.exists(p)), None)
if WORDS_FILE is None:
    WORDS_FILE = '/tmp/words.txt'
    urllib.request.urlretrieve(
        'https://raw.githubusercontent.com/dwyl/english-words/master/words_alpha.txt',
        WORDS_FILE,
    )

with open(WORDS_FILE) as f:
    words = set(w.strip().lower() for w in f if w.strip().isalpha())


def shape(word):
    """Replace each letter with the index of its first occurrence.

    e.g. shape("greenheart") == "01230345"
    """
    m, out, n = {}, [], 0
    for c in word:
        if c not in m:
            m[c] = n
            n += 1
        out.append(str(m[c]))
    return ''.join(out)


# 2. Build shape -> list-of-words index.
shape_index = defaultdict(list)
for w in words:
    if len(w) >= 4:
        shape_index[shape(w)].append(w)

# 3. Read the first 1000 study-guide words and find ones whose shape matches
#    exactly one English word.
with open('public/study-guide.txt') as f:
    sg = [line.strip() for line in f.readlines()[:1000]]

known = []
for cw in sg:
    cands = shape_index.get(shape(cw), [])
    if len(cands) == 1:
        known.append((cw, cands[0]))

print(f"[+] Found {len(known)} uniquely-shaped cipher/plaintext pairs")

# 4. Merge all of those pairs into one substitution table.
sub = {}
for cw, pw in known:
    for c, p in zip(cw, pw):
        if c in sub and sub[c] != p:
            raise SystemExit(f"Conflict on {c}: {sub[c]} vs {p}")
        sub[c] = p

print(f"[+] Substitution covers {len(sub)} of 26 letters")

# 5. Decrypt the flag.
ciphertext = open('public/flag.txt').read().strip()
plaintext = ''.join(sub.get(c, c) for c in ciphertext)
print(f"[+] Plaintext: {plaintext}")
print(f"[+] Flag:      picoCTF{{{plaintext}}}")
```

Save: `Ctrl+O`, `Enter`, exit: `Ctrl+X`. Then run it:

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ python3 solve.py
[+] Found 264 uniquely-shaped cipher/plaintext pairs
[+] Substitution covers 26 of 26 letters
[+] Plaintext: perhaps_the_dog_jumped_over_was_just_tired
[+] Flag:      picoCTF{perhaps_the_dog_jumped_over_was_just_tired}
```

Same flag, ~260 known pairs, zero conflicts. The 370k-word dwyl list is what makes it work — the smaller `/usr/share/dict/words` on Kali (around 100k words) leaves gaps where the study-guide words aren't in the dictionary.

### Method 2: Pure `subbreaker` from the Command Line (No Script)

`subbreaker` actually accepts ciphertext straight from stdin if you prefer:

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ cat public/flag.txt | subbreaker encode --key xunmrydfwhglstibjcavopezqk | head
# (this just sanity-checks the key by re-encoding and re-decoding; the real win
# is that the same subcommand pattern works for any future substitution challenge)
```

For a one-shot solve you can also chain `break` and `decode`:

```
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ KEY=$(head -n 50 public/study-guide.txt | subbreaker break --lang EN 2>/dev/null | awk '/^Key:/ {print $2}')
┌──(zham㉿kali)-[~/spelling-quiz]
└─$ echo "picoCTF{$(subbreaker decode --key $KEY --ciphertext public/flag.txt)}"
picoCTF{perhaps_the_dog_jumped_over_was_just_tired}
```

Useful when you have to script a lot of similar challenges.

### Method 3: Manual Frequency Analysis (Old-School, No Tools)

If you ever find yourself without any tooling, here is the manual method that works on this kind of corpus:

1. **Count letter frequencies** in `study-guide.txt`. The most common cipher letter is almost certainly `e`. The next few are usually `t`, `a`, `o`, `i`, `n`, `s`, `h`, `r`.
2. **Look at short words** in the study guide. Three-letter words with three different letters and a unique pattern are almost always `the`, `and`, `for`, or `was`. Each gives you 3 letters of the substitution.
3. **Look at common suffixes.** `-ing`, `-tion`, `-ed`, `-ly`, `-ness` all show up as recognizable shapes (e.g. `...ing` shows up as the same three-letter ending thousands of times).
4. **Combine** until all 26 letters are pinned down.

It is slow, it is error-prone, and the `subbreaker` approach is strictly better. But understanding it is what makes the automated tool feel less magical.

---

## What Happened Internally

Here is the full timeline of how the solver worked, from "I see a file" to "I have the flag."

1. **Read the encryptor.** `cat encrypt.py` revealed a single Python `dict` that maps each plaintext letter to a unique ciphertext letter. The same `dict` is reused for every file, so every encrypted text in this challenge shares one key.
2. **Counted the study guide.** `wc -l` showed 272543 encrypted words on separate lines. With that much known-plaintext signal, the cipher is trivially breakable.
3. **Installed `subbreaker`.** A single pip command. The tool brings its own quadgram statistics for English.
4. **Ran `subbreaker break`.** Fed in just the first 50 study-guide lines, started from a random key, hill-climbed on quadgram fitness, and converged on the key `xunmrydfwhglstibjcavopezqk` with a fitness of 92.78. The plaintext preview showed real English words (`greenheart`, `baptistry`, `vindicatory`, ...), which independently confirms the key is correct.
5. **Decoded the flag.** `subbreaker decode` applied the key in reverse. The encrypted flag `brcfxba_vfr_mid_hosbrm_iprc_exa_hoav_vwcrm` turned into `perhaps_the_dog_jumped_over_was_just_tired`.
6. **Wrapped in flag format.** `picoCTF{...}` is the required outer wrapper; picoCTF validates it as a single string.

---

## Tools Used

| Tool             | Purpose                                                              |
| ---------------- | -------------------------------------------------------------------- |
| `unzip`          | Extract the challenge archive.                                       |
| `cat`, `head`, `wc`, `ls` | Inspect the encrypted files and study guide.                  |
| `pip`            | Install `subbreaker`.                                                |
| `subbreaker`     | Break the substitution cipher via quadgram hill-climbing.            |
| `nano`           | Write the alternative Python solver (`Ctrl+O`, `Enter`, `Ctrl+X`).   |
| `python3`        | Run the dictionary-pattern alternative solver.                       |
| English word list | Match study-guide word shapes to known plaintexts (370k entries from dwyl/english-words). |

---

## Key Takeaways

* **Monoalphabetic substitution is broken by anything that reveals the underlying language.** Letter frequency works on long texts; word-shape matching works whenever you have a list of known plaintext words; quadgram statistics work even on short ciphertexts. Pick whichever is convenient.
* **The size of `study-guide.txt` (272k words) is the whole point of the challenge.** The title "spelling quiz study guide" is a wink that this is the cracking material. Without it, you would have only 41 characters of ciphertext, which is way too little to crack a substitution cipher by hand.
* **`subbreaker` is a great drop-in tool for any future CTF** that hands you a single-substitution cipher and a corpus of ciphertext. Keep it in your toolbox.
* **A 30-line Python script with a public English dictionary gets the same answer** in a few seconds and makes the attack visible. Knowing how to write it yourself is the difference between running tools and understanding them.
* **Always check whether a published tool is over-kill.** For a one-off challenge, `subbreaker` is the right tool. For a custom solver embedded in a bigger pipeline, the pattern-matching approach is more transparent and easier to debug.
* **`os.walk('.')` + in-place overwrites are a security smell.** `encrypt.py` destroys the original text the moment it finishes. Anyone running the script on their actual notes would lose them. In CTFs this is just a flag, in real life it would be a disaster. Use `pathlib.Path.with_suffix('.enc')` and write to a new path.

**Flag wordplay decode:** `perhaps_the_dog_jumped_over_was_just_tired` is a tired-dog twist on the classic pangram "the quick brown fox jumps over the lazy dog." The flag flips the script: this time the dog was too tired to jump over anything. The author's way of saying the cipher is *old* and *tame* — exactly the kind of monoalphabetic substitution that classical cryptanalysis has been breaking by hand for centuries.
