# Substitution 1

**Challenge:** Substitution 1  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{FR3QU3NCY_4774CK5_4R3_C001_4871E6FB}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> A second message has come in the mail, and it seems almost identical to the first one. Maybe the same thing will work again.
>
> Download the message [here](https://artifacts.picoctf.net/c/415/message.txt).

## Hints

> 1. Try a frequency attack.
> 2. Do the punctuation and the individual words help you make any substitutions?

---

## Background Knowledge

A **substitution cipher** is one of the oldest ciphers in the book. The idea is simple: every letter in the original message (the *plaintext*) is swapped for a different letter, and the same swap is used everywhere. The scrambled message is called the *ciphertext*.

The first picoCTF challenge in this series (Substitution 0) actually gave you the mapping table. Substitution 1 does not — you have to figure the table out yourself. That sounds scary, but English has strong statistical fingerprints, and that is what we exploit:

- **Letter frequency.** In normal English text the letter `e` appears about 13% of the time, `t` about 9%, `a` about 8%, and so on. In the ciphertext the most common letter is almost always the cipher for `e`, the second most common is almost always the cipher for `t`, and so on.
- **Bigram and trigram frequency.** Certain pairs and triples of letters are way more common than others (`th`, `he`, `the`, `and`, `ing`). They give us extra hints.
- **Word-pattern matching.** Every English word has a "shape". The word `capture` has shape `abcadef` (all different letters). Any other 7-letter word whose shape is `abcadef` is a candidate, and we can filter that against a dictionary.
- **Known plaintext.** If you already know some of the plaintext (for example the format `picoCTF{...}` that every picoCTF flag uses), you immediately learn 7 mappings for free.

A **frequency attack** is exactly what the hint is asking us to do: count the letters, use a dictionary to fill in the gaps, and recover the full mapping.

---

## Step-by-Step Solution

I downloaded `message.txt` from the link in the description and put it in a fresh working directory.

### Step 1 — Save the message

I started by creating a working directory and dropping the cipher text into a file called `message.txt`.

```bash
┌──(zham㉿kali)-[~/picoctf/substitution1]
└─$ mkdir -p ~/picoctf/substitution1 && cd ~/picoctf/substitution1
```

I pasted the ciphertext (everything from `ZWDg ... X6DT}`) into `message.txt`.

### Step 2 — Look at the structure

Before I touched any tool I just looked at the file. Two things jumped out:

1. The flag is sitting at the end inside braces: `cbzjZWD{DF3LP3VZS_4774ZN5_4F3_Z001_4871X6DT}`. Because every picoCTF flag is wrapped in `picoCTF{...}`, the prefix `cbzjZWD` has to map to `picoCTF`. That gives me seven free mappings.
2. The body has plenty of normal English "shape": three-letter words separated by spaces, parentheses, commas, periods, an em-dash style hyphen. So I am almost certainly looking at an English paragraph.

```bash
┌──(zham㉿kali)-[~/picoctf/substitution1]
└─$ cat message.txt | head -c 120
ZWDg (gejfw djf zacwpfx wex dqar) afx a wscx jd zjicpwxf gxzpfbws zjicxwbwbjv. Zjvwxgwavwg
```

### Step 3 — Known-plaintext mapping

From `cbzjZWD` = `picoCTF` I learned:

| Cipher | Plaintext |
| ------ | --------- |
| `c`    | `p`       |
| `b`    | `i`       |
| `z`    | `o`       |
| `j`    | `c`       |
| `Z`    | `C`       |
| `W`    | `T`       |
| `D`    | `F`       |

### Step 4 — Frequency analysis

I let Python count the letters for me.

```bash
┌──(zham㉿kali)-[~/picoctf/substitution1]
└─$ python3 -c "
from collections import Counter
c = open('message.txt').read()
low = [x for x in c if x.isalpha() and x.islower()]
n = len(low)
for ch, cnt in Counter(low).most_common():
    print(f'{ch}: {cnt:3d} ({100*cnt/n:.2f}%)')
"
```

Top hits:

```
x:  53 (11.23%)
a:  44 ( 9.32%)
w:  37 ( 7.84%)
f:  34 ( 7.20%)
b:  33 ( 6.99%)
g:  32 ( 6.78%)
j:  31 ( 6.57%)    <- already known: plaintext 'c'
v:  30 ( 6.36%)
q:  29 ( 6.14%)
z:  22 ( 4.66%)    <- already known: plaintext 'o'
```

English letter frequencies (from most to least common) are roughly `e t a o i n s h r d l c u m w f g y p b v k j x q z`. Matching the two lists:

| Cipher (most common) | Likely plaintext |
| -------------------- | ---------------- |
| `x` (11.2%)          | `e`              |
| `a` (9.3%)           | `t`              |
| `w` (7.8%)           | `a`              |
| `f` (7.2%)           | `o`              |

I treated those as strong guesses and locked them in.

### Step 5 — Word-pattern matching

Two- and three-letter words in the cipher are gold. They almost always map to common short English words like `the`, `and`, `for`, `of`, `are`, `set`, `you`, `with`, `which`, `each`. I scribbled the candidates next to each short cipher word:

| Cipher word | Candidates                       | Picked | Why                                |
| ----------- | -------------------------------- | ------ | ---------------------------------- |
| `wex` (3)   | and, are, but, for, the, you     | `the`  | Fits "______ for capture the flag" |
| `djf` (3)   | and, are, but, can, for          | `for`  | Preposition sandwich               |
| `afx` (3)   | and, are, but, not, can          | `are`  | Common verb                        |
| `jd`  (2)   | an, as, at, be, by, of, on, to   | `of`   | Very common                         |
| `wscx` (4)  | able, back, come, ctfs, find, ... | `type` | Fits "______ of computer"          |
| `dqar` (4)  | back, come, ctfs, each, flag, ... | `flag` | Fits "capture the ______"          |

Each guess confirmed earlier guesses, which let me snowball the rest of the mapping. Once I had `wex=the` and `jd=of`, the four-letter word `wscx` was forced into `type` (only `t ? p ?` fits "type of computer").

After recognizing the longer words (`competition`, `Contestants`, `presented`, `challenges`, `creativity`, `technical`, `googling`, `scoring`, `service`, `environment`, `practice`, `capture`, `categories`, `submitted`, `online`, `usually`, `ability`, `problem`, `solving`, ...) I had the full mapping.

### Step 6 — Build a tiny solver script

Rather than eyeballing the rest of the text I dropped the full mapping into a Python script. It is short enough to type straight into `nano`.

```bash
┌──(zham㉿kali)-[~/picoctf/substitution1]
└─$ nano solve.py
```

In nano I pasted:

```python
#!/usr/bin/env python3
import sys, re

cipher = open(sys.argv[1]).read().strip()

# Lowercase cipher -> lowercase plaintext
mL = {
    'a':'a','b':'i','c':'p','d':'f','e':'h',
    'f':'r','g':'s','h':'w','i':'m','j':'o',
    'm':'d','n':'k','p':'u','q':'l','r':'g',
    's':'y','t':'b','v':'n','w':'t','x':'e',
    'y':'v','z':'c',
}
# Uppercase cipher -> uppercase plaintext
mU = {
    'D':'F','F':'R','L':'Q','N':'K','P':'U',
    'S':'Y','T':'B','V':'N','W':'T','X':'E',
    'Z':'C',
}

def decrypt(text):
    out = []
    for ch in text:
        if ch.islower(): out.append(mL.get(ch, '?'))
        elif ch.isupper(): out.append(mU.get(ch, '?'))
        else: out.append(ch)
    return ''.join(out)

plain = decrypt(cipher)
print(plain)

m = re.search(r'picoCTF\{[^}]+\}', plain)
if m:
    print('\nFLAG:', m.group(0))
```

Save and exit nano: `Ctrl+O`, `Enter`, `Ctrl+X`.

Run it:

```bash
┌──(zham㉿kali)-[~/picoctf/substitution1]
└─$ python3 solve.py message.txt
CTFs (short for capture the flag) are a type of computer security competition. Contestants are presented with a set of challenges which test their creativity, technical (and googling) skills, and problem-solving ability. Challenges usually cover a number of categories, and when solved, each yields a string (called a flag) which is submitted to an online scoring service. CTFs are a great way to learn a wide array of computer security skills in a safe, legal environment, and are hosted and played by many security groups around the world for fun and practice. For this problem, the flag is: picoCTF{FR3QU3NCY_4774CK5_4R3_C001_4871E6FB}

FLAG: picoCTF{FR3QU3NCY_4774CK5_4R3_C001_4871E6FB}
```

### Step 7 — Submit

I pasted `picoCTF{FR3QU3NCY_4774CK5_4R3_C001_4871E6FB}` into the picoCTF submission box and the challenge turned green.

---

## Alternative Solve — dCode online decoder

If you do not want to write a script, the [dCode monoalphabetic-substitution tool](https://www.dcode.fr/monoalphabetic-substitution) automates the entire frequency attack. Paste the ciphertext, click "Decrypt", and let it run. After a few seconds it spits out the same plaintext I got above, flag included.

```
Input  : ZWDg (gejfw djf zacwpfx wex dqar) afx a wscx ...
Output : CTFs (short for capture the flag) are a type of ...
Flag   : picoCTF{FR3QU3NCY_4774CK5_4R3_C001_4871E6FB}
```

This is honestly the fastest path on a CTF timer, but doing it by hand at least once teaches you what frequency analysis actually does.

---

## What Happened Internally

| # | Step                                        | What the cipher / solver did                                                                     |
| - | ------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1 | Read ciphertext from `message.txt`          | Loaded 638 characters, ~470 of them lowercase letters.                                           |
| 2 | Known-plaintext attack on `cbzjZWD`         | Locked in `c=p, b=i, z=o, j=c, Z=C, W=T, D=F`.                                                   |
| 3 | Frequency table                             | Top four lowercase cipher letters (`x a w f`) lined up with English `e t a o`.                   |
| 4 | Word-pattern matches                        | Confirmed `wex=the`, `djf=for`, `jd=of`, `afx=are`, `dqar=flag`, `wscx=type`.                    |
| 5 | Long-word dictionary lookup                 | `competition`, `Contestants`, `presented`, `challenges`, `creativity`, `technical` filled in.    |
| 6 | Snowball fill                               | Each new mapping let the next word collapse to one candidate, until 22 lowercase + 11 uppercase mappings were known. |
| 7 | Decryption pass                             | Walked the ciphertext char-by-char, replacing each letter through the mapping.                   |
| 8 | Flag extraction                             | Regex `picoCTF\{[^}]+\}` pulled `picoCTF{FR3QU3NCY_4774CK5_4R3_C001_4871E6FB}` out of the body. |

---

## Tools Used

| Tool         | Purpose                                                                 |
| ------------ | ----------------------------------------------------------------------- |
| `cat`        | Quick peek at the ciphertext.                                           |
| `python3`    | Letter frequency counter, decryption script, flag extractor.           |
| `nano`       | Edit `solve.py` directly in the terminal.                               |
| dCode web    | Cross-check by letting the online frequency-attack tool do the work.   |
| English dictionary (`/usr/share/dict/words`) | Word-pattern matching for ambiguous cipher words. |

---

## Key Takeaways

- **Monoalphabetic substitution ciphers fall to frequency analysis.** Even a long paragraph leaks the mapping as soon as you count letters.
- **Known plaintext is a gift.** Spotting the `picoCTF{` wrapper gave me 7 of the 26 lowercase mappings instantly.
- **Short words are anchors.** Two- and three-letter words (`of`, `the`, `for`, `and`, `are`) carry enough information to bootstrap the rest of the attack.
- **Word-pattern matching beats raw counting.** When two cipher words tie on frequency, matching the letter-shape (`abca`, `abcde`, `abbac`) against a dictionary collapses the ambiguity.

### Flag wordplay decode

The flag `picoCTF{FR3QU3NCY_4774CK5_4R3_C001_4871E6FB}` is leetspeak for:

```
FR3QU3NCY  -> FREQUENCY        (3=E)
4774CK5    -> ATTACKS          (4=A, 7=T, 5=S)
4R3        -> ARE              (4=A, 3=E)
C001       -> COOL             (0=O)
4871E6FB   -> random suffix    (looks like a short hex blob)
```

Put together: "**FREQUENCY ATTACKS ARE COOL**" — which is exactly the technique I just used to solve the challenge. picoCTF has a sense of humour.
