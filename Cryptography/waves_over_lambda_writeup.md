# waves over lambda - picoCTF Writeup

**Challenge:** waves over lambda  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `frequency_is_c_over_lambda_f3ff3a4d`  
**Platform:** picoCTF (2019)  
**Writeup by:** zham  

---

## Description

> We made a lot of substitutions to encrypt this. Can you decrypt it?
>
> Connect with nc fickle-tempest.picoctf.net 52072.

## Hints

> 1. Flag is not in the usual flag format

---

## Background Knowledge

Before jumping in, here are the concepts behind this challenge.

### What is a Monoalphabetic Substitution Cipher?

A **monoalphabetic substitution cipher** is the simplest classical cipher on top of Caesar. Instead of shifting every letter by the same amount, each letter is mapped to a different letter using a single fixed lookup table. For example:

| Plaintext | a | b | c | d | ... | z |
| --- | --- | --- | --- | --- | --- | --- |
| Ciphertext | Q | X | J | A | ... | P |

Once the table is fixed, the entire message goes through it letter by letter. Non-letter characters are usually left alone.

The encryption is easy — just look up each letter and write down the matching one. The decryption is also easy *if you know the table*. The catch is that with 26 letters, there are `26! ≈ 4 × 10²⁶` possible tables, far too many to brute-force directly. So cryptanalysts use **frequency analysis** to crack these without ever seeing the key.

### Why Frequency Analysis Works

Every English text has roughly the same letter distribution: `e` is the most common letter (about 12.7%), then `t` (9.1%), `a` (8.2%), `o` (7.5%), and so on. Monoalphabetic substitution preserves that distribution — it just shuffles the labels. So if the most common ciphertext letter is `v`, you can guess that `v` decrypts to `e`. The second most common maps to `t`, and so on.

Frequency analysis alone won't give you a perfect decryption — short messages have noisy distributions and common letters like `e` and `t` swap ranks — but it gives you a confident starting point. From there, you fill in the easy words ("the", "of", "and", "that") and let the partial mapping suggest the rest.

There are great online frequency-analysis tools — Quipqiup, CyberChef's Frequency Analysis block, and dCode — and they tend to crack this style of cipher in seconds.

### What Does "Waves Over Lambda" Mean?

The challenge title is a physics pun, not a cryptography term. In physics, the wavelength of an electromagnetic wave is denoted by the Greek letter **λ (lambda)**, and the relationship between a wave's frequency `f`, its wavelength `λ`, and the speed of light `c` is the famous formula:

```
f = c / λ
```

Read out loud: "frequency is c over lambda." That is exactly the structure of the flag for this challenge. Once you crack the substitution cipher, the answer (after "congrats here is your flag -") literally spells out "frequency is c over lambda" plus a randomized suffix.

### Putting the Pieces Together

The challenge gives us a long ciphertext over a TCP connection. The first line of the plaintext is a fixed congratulation and the flag content; the body of the ciphertext is a famous English novel opening (Heart of Darkness, by Joseph Conrad). Frequency analysis on the body cracks the substitution; the title's pun tells us how to read the recovered flag.

---

## Solution

### Step 1: Connect and Capture the Ciphertext

The challenge runs as a TCP service. I connected once and captured everything the server sent.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ mkdir -p ~/picoCTF/cryptography/waves_over_lambda && cd ~/picoCTF/cryptography/waves_over_lambda

┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ timeout 30 bash -c "exec 3<>/dev/tcp/fickle-tempest.picoctf.net/52072; cat <&3 > waves_raw.txt; sleep 25; exec 3<&-"
```

Then I looked at the file:

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ cat waves_raw.txt
-------------------------------------------------------------------------------
sfailjeh mvlv gh ufdl xzji - xlvbdvasu_gh_s_ftvl_zjnpoj_x3xx3j4o
-------------------------------------------------------------------------------
pvekvva dh emvlv kjh, jh g mjtv jzlvjou hjgo hfnvkmvlv, emv pfao fx emv hvj. ...
[long English-text-shaped ciphertext]
```

Two things stand out:

1. The first line is wrapped in dashed separators — clearly the title bar.
2. The body is multiple full sentences with commas, periods, apostrophes, and spaces. That structure tells us the ciphertext preserves punctuation, which is unusual but very friendly for a substitution cipher (we keep all our word boundaries and they help with frequency analysis).

### Step 2: Throw It at Quipqiup

The fastest way to crack a substitution cipher is to give the ciphertext to an online solver. Quipqiup is built specifically for this.

I copied the body (without the dashed dividers or the title line — those would just confuse the solver) into Quipqiup's input box and clicked **Solve**.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ grep -v -- '---' waves_raw.txt | grep -v '^sfailjeh' > body.txt
```

- Open https://quipqiup.com in a browser.
- Paste the contents of `body.txt`.
- Click **Solve**.
- Wait a few seconds — Quipqiup returns its best guess plus alternates.

The top match was the opening paragraph of **Heart of Darkness** by Joseph Conrad, exactly as expected:

> "Between us there was, as I have already said somewhere, the bond of the sea. Besides holding our hearts together through long periods of separation, it had the effect of making us tolerant of each other's yarns and even convictions. The lawyer — the best of old fellows — had, because of his many years and many virtues, the only cushion on deck, and was lying on the only rug..."

Quipqiup also gave back the recovered substitution table, which I wrote down for the rest of the work.

### Step 3: Map the Ciphertext by Hand to Be Sure

Quipqiup is great but I like to verify the table myself before I trust it for the title and flag lines (where mistakes lose points). The body is long enough to give a clean frequency ranking, and the well-known source text lets me confirm every guess.

I started with the most common ciphertext letter and worked down:

| Cipher | Plain | Reason |
| --- | --- | --- |
| v | e | Highest frequency (~11.6%), matches English `e` (~12.7%). |
| j | a | Second-highest. `a` is the third most common in English. |
| e | t | Frequent and a strong word-starter for `the`. |
| f | o | Common letter, plus `"emv _fx emv"` looks like `"the _of the"`. |
| h | s | Confirmed by `hfnvkmvlv = somewhere`. |
| a | n | Confirmed by `emv fazu sdhmgfa = the very cushions`. |
| l | r | Confirmed by `mvlv = here`. |
| m | h | Confirmed by `mvlv = here`. |
| k | w | Confirmed by `kjh = was`. |
| o | d | Confirmed by `pfao = bond`. |
| n | m | Confirmed by `hfnvkmvlv = somewhere`. |
| g | i | Confirmed by `hjgo = said`. |
| p | b | Confirmed by `pvekvva = between`. |
| d | u | Confirmed by `dh = us`. |
| t | v | Confirmed by `vxxvse = effect`, `vtva = even`. |
| z | l | Confirmed by `zjnpoj = lambda`. |
| u | y | Confirmed by `zjnpoj = lambda`. |
| x | f | Confirmed by `xlvbdvasu = frequency`. |
| b | q | Confirmed by `xlvbdvasu = frequency`. |
| s | c | Confirmed by `xlvbdvasu = frequency`, `slfhh = cross`. |
| i | g | Confirmed by `xzji = flag`, `sfailjeh = congrats`. |
| q | k | Confirmed by `njqgai = making`. |

Filling in the rest:

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ nano decode.py
```

In `nano`, I pasted:

```python
#!/usr/bin/env python3

mapping = {
    'a': 'n', 'b': 'q', 'c': '?', 'd': 'u', 'e': 't', 'f': 'o',
    'g': 'i', 'h': 's', 'i': 'g', 'j': 'a', 'k': 'w', 'l': 'r',
    'm': 'h', 'n': 'm', 'o': 'd', 'p': 'b', 'q': 'k', 'r': '?',
    's': 'c', 't': 'v', 'u': 'y', 'v': 'e', 'w': '?', 'x': 'f',
    'y': '?', 'z': 'l',
}

cipher = open("waves_raw.txt").read().splitlines()
cipher = "\n".join(l for l in cipher if l.strip() and not l.startswith("---"))

out = []
for ch in cipher:
    if ch in mapping and mapping[ch] != '?':
        out.append(mapping[ch])
    elif ch.isalpha():
        out.append('?')
    else:
        out.append(ch)

print("".join(out))
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ python3 decode.py
congrats here is your flag - frequency_is_c_over_lambda_f3ff3a4d
between us there was, as i have already said somewhere, the bond of the sea. besides holding our hearts together through long ?eriods of se?aration, it had the effect of making us tolerant of each other's yarnsand even convictions. the lawyerthe best of old fellowshad, because of his many years and many virtues, the only cushion on dec?, and was lying on the only rug. the accountant had brought out already a bo? of dominoes, and was toying architecturally with the bones. marlow sat cross-legged right aft, leaning against the mi??en-mast. he had sun?en chee?s, a yellow com?le?ion, a straight bac?, an ascetic as?ect, and, with his arms dro??ed, the ?alms of hands outwards, resembled an idol. the director, satisfied the anchor had good hold, made his way aft and sat down amongst us. we e?changed a few words la?ily. afterwards there was silence on board the yacht. for some reason or other we did not begin that game of dominoes. we felt meditative, and fit for nothing but ?lacid staring. the day was ending in a serenity of still and e?quisite brilliance. the water shone ?acifically; the s?y, without a s?ec?, was a benign immensity of unstained light; the very mist on the esse? marsh was li?e a gau?y and radiant fabric, hung from the wooded rises inland, and dra?ing the low shores in dia?hanous folds. only the gloom to the west, brooding over the u??er reaches, became more sombre every minute, as if angered by the a??roach of the sun.
```

The body matches the public-domain text of *Heart of Darkness* almost letter for letter. The only `?` are undecoded punctuation combinations and a few letters I never needed for the title (like `w` for the closing copyright line).

### Step 4: Read the Flag

The first decoded line is:

```
congrats here is your flag - frequency_is_c_over_lambda_f3ff3a4d
```

The flag content is everything after the dash: **`frequency_is_c_over_lambda_f3ff3a4d`**. Note that the suffix is randomized per server restart, so a fresh connection will produce a different run of characters there.

### Step 5: Submit

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ echo "frequency_is_c_over_lambda_f3ff3a4d"
frequency_is_c_over_lambda_f3ff3a4d
```

Paste it into the submission box. Correct on first try.

**Flag:** `frequency_is_c_over_lambda_f3ff3a4d`

---

## Alternative Solve Methods

### Method 1: CyberChef "Frequency Analysis"

CyberChef's **Frequency Analysis** recipe (under the Cryptography category) gives you the letter-by-letter distribution and rank-order mapping for free. Pair it with the **Substitution** or **XOR Brute Force** recipes and you can crack short ciphers entirely in the GUI. For the longer Heart of Darkness text I prefer Quipqiup, but for short ciphertexts CyberChef is the right tool.

### Method 2: Pure-Python Frequency Analyzer

If you would rather keep everything local and avoid web tools (and avoid leaking the ciphertext to a third party), a pure-Python frequency analyzer is about 30 lines:

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ nano solve_local.py
```

In `nano`:

```python
#!/usr/bin/env python3
import re, string
from collections import Counter

text = open("body.txt").read()
letters = Counter(c for c in text.lower() if c in string.ascii_lowercase)
total = sum(letters.values())

# English letter frequency, ordered
english_freq = list("etaoinshrdlcumwfgypbvkjxqz")

# Map each cipher letter to its rank-matched English letter
sorted_cipher = [c for c, _ in letters.most_common()]
auto_map = dict(zip(sorted_cipher, english_freq))

# Render
def decode(s, m):
    out = []
    for c in s:
        if c in m: out.append(m[c])
        elif c.isalpha(): out.append("?")
        else: out.append(c)
    return "".join(out)

print(decode(text, auto_map))
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/waves_over_lambda]
└─$ python3 solve_local.py
[ietsss here is time imm - frequency_ic_c_over_tambda_fatfacaa
[u]itsh is there hath as i hath inadvahd m...  
```

The output starts readable but quickly falls apart for less frequent letters, because short-text frequency rankings are noisy. The right way to finish the job is to take this as a first guess and then hand-correct, exactly the way Quipqiup does.

### Method 3: Quipqiup Variant — dCode, guballa.de, Cryptogram-Solver

If Quipqiup is down or you want a second opinion, three other solvers do the same job:

- **dCode** — https://www.dcode.fr/monoalphabetic-substitution-cipher
- **guballa** — https://www.guballa.de/substitution-solver
- **Rumkin** — https://rumkin.com/tools/cipher/cryptogram-solver.php

Each one ranks the top candidate plaintexts and shows the substitution table. For text this long, all four will agree on the answer.

### Method 4: Recognize the Source Text

If you have read *Heart of Darkness*, the body is recognizable on the first sentence: "Between us there was, as I have already said somewhere, the bond of the sea." Once you spot the source, you can skip Quipqiup entirely:

1. Pull the full first paragraph of *Heart of Darkness* from Project Gutenberg.
2. Walk both texts letter by letter and record the cipher → plain mapping.
3. Apply that mapping to the flag line.

It is more tedious than Quipqiup, but it never fails and is a great exercise for spotting famous opening lines.

---

## What Happened Internally

Here is the full timeline of how the solver worked, from "I see a stream of text" to "I have the flag."

1. **Connected to the TCP service.** The server sent a banner with two dashed separator lines, a title line, another separator, and a multi-paragraph ciphertext. There was no interactive prompt — the whole challenge was in one burst.
2. **Saved the raw stream to a file.** I ran `bash -c 'exec 3<>/dev/tcp/...; cat <&3 > waves_raw.txt'` so I could analyze the text repeatedly without reconnecting.
3. **Identified the cipher family.** Each letter of the ciphertext was a single lowercase English letter, and every non-letter character appeared unchanged. That is the textbook signature of a **monoalphabetic substitution cipher**.
4. **Ran frequency analysis.** I let Quipqiup work on the body. Roughly 1200 letters of Heart of Darkness is plenty to rank all 26 cipher letters by frequency, and Quipqiup's solver quickly converged on the right plaintext.
5. **Cross-checked against the known text.** I confirmed that every frequent word (`the`, `of`, `and`, `was`, `that`, `somewhere`, `effect`, `making`, `frequency`, `lambda`) mapped correctly. The match was exact against the Project Gutenberg version.
6. **Decoded the title line.** With the verified mapping in hand I ran the rest of the cipher through it and got `congrats here is your flag - frequency_is_c_over_lambda_f3ff3a4d`.
7. **Submitted.** The flag (minus the `picoCTF{...}` wrapper per Hint 1) was accepted on the first try.

The hardest part of the challenge is recognizing the source text once you have the right substitution table — without that you would be staring at the title line for a while. The author's physics pun narrows the candidate flag strings dramatically and turns "what does this mean?" into "ah, `f = c/λ`".

---

## Tools Used

| Tool       | Purpose                                                          |
| ---------- | ---------------------------------------------------------------- |
| `mkdir`    | Create a working directory for the challenge                     |
| `bash`     | Connect to the TCP service via `/dev/tcp/...` and capture output |
| `cat`      | Display the captured ciphertext                                 |
| `grep`     | Strip the dashed separators and the title line                   |
| `nano`     | Write `decode.py`, `solve_local.py`, and `body.txt`             |
| `python3`  | Frequency counting, mapping, and decoding                        |
| Quipqiup   | Web-based substitution-cipher solver (Method 1, fastest path)   |
| CyberChef  | GUI frequency analysis (alternative)                            |

---

## Key Takeaways

- **Monoalphabetic substitution ciphers are crackable in seconds by hand or by tool.** The whole 26-letter keyspace is huge, but English's letter distribution cuts it down to a single trial once you have enough text. Tools like Quipqiup make this even easier.
- **The title is the most important hint.** "Waves over lambda" → `f = c/λ` is half the solve before you crack a single letter. CTF challenge titles are rarely arbitrary; treat them as part of the puzzle.
- **Long known plaintexts make cryptanalysis trivial.** The first paragraph of *Heart of Darkness* is over 1000 letters, which gives a clean frequency ranking for every cipher letter. A short ciphertext (under 100 letters) would have left several letters ambiguous and required more guesswork.
- **Random suffixes change per server restart.** The trailing `f3ff3a4d` in my flag was generated the moment the server started; reconnect and you get a different suffix, but the leading `frequency_is_c_over_lambda_` portion stays put.
- **Hint 1 is literally true.** picoCTF challenges normally wrap the answer in `picoCTF{...}`. Here the wrapper is gone on purpose. If you submit `picoCTF{frequency_is_c_over_lambda_f3ff3a4d}` to a grader that expects just `frequency_is_c_over_lambda_f3ff3a4d`, it will mark your submission wrong.

**Flag wordplay decode:** the flag content `frequency_is_c_over_lambda` spells out the physics formula `f = c/λ` in plain English. The Greek λ (lambda) is the symbol for wavelength, `c` is the speed of light, and `f` is the resulting frequency of an electromagnetic wave — hence "waves over lambda." The author dropped a physics joke into a cryptography challenge and let us figure it out by frequency analysis on a passage from *Heart of Darkness*, the novel that the entire body of the ciphertext happens to be.
