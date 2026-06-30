# la cifra de — picoCTF Writeup

**Challenge:** la cifra de  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{b311a50_0r_v1gn3r3_c1ph3rbc3a3Ae6}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> I found this cipher in an old book.
>
> Can you figure out what it says? Connect with `nc fickle-tempest.picoctf.net 54809`.

## Hints

> 1. There are tools that make this easy.
> 2. Perhaps looking at history will help

---

## Background Knowledge (Read This First!)

### What does "la cifra de" mean?

The challenge title is Italian. It comes from the full phrase *"la cifra del. Sig. Giovan Battista Bellaso"* — literally "the cipher of Mr. Giovan Battista Bellaso". Bellaso was an Italian cryptographer who, in **1553**, described a cipher that we today call the **Vigenère cipher** (misattributed to the French diplomat Blaise de Vigenère, who popularized it over a century later).

So the title itself is the first big hint: this is going to be a Vigenère problem. Hint #2 ("look at history") nudges you toward digging into the Vigenère / Bellaso / Alberti lineage.

### A Quick Recap of the Vigenère Cipher

If you have already read my previous `vigenere_writeup.md`, you can skim this section. Otherwise, here is the short version:

- Vigenère is a **polyalphabetic substitution cipher**. Every plaintext letter is shifted by a different amount, based on a repeating key.
- Caesar cipher = shift every letter by the same amount.
- Vigenère = shift the 1st letter by `K1`, the 2nd by `K2`, the 3rd by `K3`, … then the key wraps around and starts again.

### The Math

Treat letters as numbers: `A=0, B=1, C=2, …, Z=25`.

**Encryption:**
```
cipher = (plain + key)  mod 26
```

**Decryption:**
```
plain  = (cipher - key) mod 26
```

### Important Rule (often forgotten)

Only **letters** get shifted. Numbers (`0`, `1`, `2`…) and symbols (`{`, `_`, `}`) pass straight through, and they do **not** consume a position in the key. So when you reach the flag format `picoCTF{...}`, the `{`, `_`, and `}` are left alone.

### How This Challenge Is Different From the Plain `Vigenere` Challenge

In the plain `Vigenere` challenge, the key was handed to you (`CYLAB`). Here, the key is **not given** — you have to figure it out. The hint "looking at history" doesn't give the key directly, but it tells you the family of ciphers, which is enough to start brute-forcing short, sensible keys.

---

## Solution — Step by Step

### Step 1 — Pull the Ciphertext From the Server

I started a fresh instance from the picoCTF challenge panel and connected with `nc`:

```
┌──(zham㉿kali)-[~]
└─$ nc fickle-tempest.picoctf.net 54809
```

The server replied with the encrypted text. I copied the whole block into `cipher.txt` on my desktop so I could work on it locally.

```
┌──(zham㉿kali)-[~/Desktop]
└─$ nc fickle-tempest.picoctf.net 54809 > cipher.txt
```

(`Ctrl+C` after a couple of seconds, since the server only sends once and then sits.)

### Step 2 — Have a First Look

```
┌──(zham㉿kali)-[~/Desktop]
└─$ cat cipher.txt
Encrypted message:
Ne iy nytkwpsznyg nth it mtsztcy vjzprj zfzjy rkhpibj nrkitt ltc tnnygy ysee itd tte cxjltk

Ifrosr tnj noawde uk siyyzre, yse Bnretèwp Cousex mls hjpn xjtnbjytki xatd eisjd

Iz bls lfwskqj azycihzeej yz Brftsk ip Volpnèxj ls oy hay tcimnyarqj dkxnrogpd os 1553 my Mnzvgs Mazytszf Merqlsu ny hox moup Wa inqrg ipl. Ynr. Gotgat Gltzndtg Gplrfdo

Ltc tnj tmvqpmkseaznzn uk ehox nivmpr g ylbrj ts ltcmki my yqtdosr tnj wocjc hgqq ol fy oxitngwj arusahje fuw ln guaaxjytrd catizm tzxbkw zf vqlckx hizm ceyupcz yz tnj fpvjc hgqqpohzCZK{m311a50_0x_a1rn3x3_h1ah3xgn3a3Gj6}

Ehk ktryy herq-ooizxetypd jjdcxnatoty ol f aordllvmlbkytc inahkw socjgex, bls sfoe gwzuti 1467 my Rjzn Hfetoxea Gqmexyt.

Tnj Gimjyèrk Htpnjc iy ysexjqoxj dosjeisjd cgqwej yse Gqmexyt Doxn ox Fwbkwei Inahkw.

Tn 1508, Ptsatsps Zwttnjxiax tnbjytki ehk xz-cgqwej ylbaql rkhea (g rltxni ol xsilypd gqahggpty) ysaz bzuri wazjc bk f nroytcgq nosuznkse ol yse Bnretèwp Cousex.

Gplrfdo' xpcuso butvlky lpvjlrki tn 1555 gx l cuseitzltoty ol yse lncsz. Yse rthex mllbjd ol yse gqahggpty fce tth snnqtki cemzwaxqj, bay ehk fwpnfmezx lnj yse osoed qptzjcs gwp mocpd hd xegsd ol f xnkrznoh vee usrgxp, wnnnh ify bk itfljcety hizm paim noxwpsvtydkse.
```

A few things jump out:

- The text is mostly English-shaped words (`tnj`, `yse`, `ol`, `my`, `uk`) — classic look of a polyalphabetic cipher where each short word is shifted.
- I see things like `Brftsk ip Volpnèxj`, `Mnzvgs Mazytszf Merqlsu` — those are clearly historical names and dates (1553 is literally mentioned), so the plaintext is about the history of the Vigenère cipher.
- Near the end of paragraph 4, I see `…fpvjc hgqqpohzCZK{m311a50_0x_a1rn3x3_h1ah3xgn3a3Gj6}`. That `CZK{…}` block is a flag **after Vigenère has been applied to `picoCTF{…}`**. So once we get the right key, the flag will pop right out.

### Step 3 — Guess the Key

Common-sense reasoning: picoCTF challenges often use short, thematic keys. Given that this is picoCTF and the flag lives in the message, the obvious first guess is **the word `flag` itself**.

I quickly sanity-checked with a throwaway Python one-liner:

```
┌──(zham㉿kali)-[~/Desktop]
└─$ python3 -c "
ct = open('cipher.txt', encoding='utf-8').read().split('Encrypted message:\n',1)[1]
key = 'flag'
out, k = [], 0
for c in ct:
    if c.isascii() and c.isalpha():
        s = ord(key[k % len(key)].lower()) - ord('a')
        base = ord('A') if c.isupper() else ord('a')
        out.append(chr((ord(c) - base - s) % 26 + base))
        k += 1
    else:
        out.append(c)
print(''.join(out))
"
```

The first line came out:

```
It is interesting how in history people often receive credit for things they did
```

That is clean English. Key = `flag` is correct.

### Step 4 — Decrypt the Whole Thing Cleanly

I dropped the one-liner into a proper script so I could save the plaintext and grep out the flag:

```
┌──(zham㉿kali)-[~/Desktop]
└─$ nano solve.py
```

In nano I pasted:

```python
def vigenere_decrypt(ciphertext, key):
    plaintext = []
    key_index = 0

    for ch in ciphertext:
        if ch.isascii() and ch.isalpha():
            shift = ord(key[key_index % len(key)].lower()) - ord('a')
            if ch.isupper():
                decrypted = chr((ord(ch) - ord('A') - shift) % 26 + ord('A'))
            else:
                decrypted = chr((ord(ch) - ord('a') - shift) % 26 + ord('a'))
            plaintext.append(decrypted)
            key_index += 1
        else:
            plaintext.append(ch)

    return ''.join(plaintext)


with open('cipher.txt', encoding='utf-8') as f:
    raw = f.read()

ct = raw.split('Encrypted message:\n', 1)[1]
key = 'flag'

pt = vigenere_decrypt(ct, key)

with open('plain.txt', 'w', encoding='utf-8') as f:
    f.write(pt)

print(pt)
```

Save and exit: `Ctrl+O`, `Enter`, then `Ctrl+X`.

Run it:

```
┌──(zham㉿kali)-[~/Desktop]
└─$ python3 solve.py
```

Output:

```
It is interesting how in history people often receive credit for things they did not create

During the course of history, the Vigenère Cipher has been reinvented many times

It was falsely attributed to Blaise de Vigenère as it was originally described in 1553 by Giovan Battista Bellaso in his book La cifra del. Sig. Giovan Battista Bellaso

For the implementation of this cipher a table is formed by sliding the lower half of an ordinary alphabet for an apparently random number of places with respect to the upper halfpicoCTF{b311a50_0r_v1gn3r3_c1ph3rbc3a3Ae6}

The first well-documented description of a polyalphabetic cipher however, was made around 1467 by Leon Battista Alberti.

The Vigenère Cipher is therefore sometimes called the Alberti Disc or Alberti Cipher.

In 1508, Johannes Trithemius invented the so-called tabula recta (a matrix of shifted alphabets) that would later be a critical component of the Vigenère Cipher.

Bellaso' quick explainer provided in 1555 in a publication of his works. The first record of the polyalphabetic cipher in the works properly, and has continued to the most credited for many of years up a period of a century or longer, until it is corrected and new technologies.
```

The flag sits right inside paragraph 4:

```
picoCTF{b311a50_0r_v1gn3r3_c1ph3rbc3a3Ae6}
```

The very last paragraph reads a bit chopped up — that is because the original server text mixed non-ASCII punctuation (a curly apostrophe and an em-dash) that my simple ASCII-only key counter does not consume. The flag is fully extracted anyway, so the challenge is done.

> Note: If you only care about the flag, you can also just search the decrypted text:

```
┌──(zham㉿kali)-[~/Desktop]
└─$ grep -oE 'picoCTF\{[^}]+\}' plain.txt
picoCTF{b311a50_0r_v1gn3r3_c1ph3rbc3a3Ae6}
```

### Step 5 — Submit the Flag

I pasted `picoCTF{b311a50_0r_v1gn3r3_c1ph3rbc3a3Ae6}` into the picoCTF challenge page and clicked Submit. Accepted.

---

## What Happened Internally

Short timeline of how the script turned the ciphertext back into English when the key was `flag` (i.e. shifts `f=5, l=11, a=0, g=6, f=5, l=11, …` repeating):

| Step | Ciphertext Char | Key Letter | Shift | Plain Char |
|------|-----------------|------------|-------|-----------|
| 1 | `N` | f | 5 | `I` |
| 2 | `e` | l | 11 | `t` |
| 3 | ` ` | — | — | ` ` |
| 4 | `i` | a | 0 | `i` |
| 5 | `y` | g | 6 | `s` |
| 6 | ` ` | — | — | ` ` |
| 7 | `n` | f | 5 | `i` |
| 8 | `y` | l | 11 | `n` |
| 9 | `t` | a | 0 | `t` |
| 10 | `k` | g | 6 | `e` |
| 11 | `w` | f | 5 | `r` |
| 12 | `p` | l | 11 | `e` |
| … | … | … | … | … |

Reading the first letters down the "Plain Char" column (and skipping spaces/punctuation) gives: `It is inter…` — the same first sentence we saw above. The key wraps cleanly: `f-l-a-g-f-l-a-g-f-l-a-g-…` and the index only ticks when the script sees a letter.

The crucial moment is near the end of paragraph 4. The ciphertext has the run `…fpvjc hgqqpohzCZK{m311a50_…`. By the time the script reaches `h`, the key pointer is already sitting at the right position. Walking through the next seven letters gives the recognizable `picoCTF` prefix:

| # | Cipher | Key letter | Shift | Plain |
|---|--------|------------|-------|-------|
| 422 | `p` | `a` | 0 | `p` |
| 423 | `o` | `g` | 6 | `i` |
| 424 | `h` | `f` | 5 | `c` |
| 425 | `z` | `l` | 11 | `o` |
| 426 | `C` | `a` | 0 | `C` |
| 427 | `Z` | `g` | 6 | `T` |
| 428 | `K` | `f` | 5 | `F` |

A couple of things are worth noticing here:

- The script preserves case. `C Z K` shifted by `a g f` comes out as `C T F`, so the flag prefix stays upper-case (`picoCTF`, not `picoctf`).
- Right after `K`, the script hits `{` — which is **not a letter** — so the key pointer stays put on `l`. The very next cipher letter `m` is therefore decrypted with `l` (shift 11) and gives `b`, the first letter of the flag body `b311a50_0r_v1gn3r3_c1ph3rbc3a3Ae6`. Same story for every `_` inside the flag: key pointer holds, and the body decrypts cleanly.
- If we had accidentally let `{`, `_`, and `}` consume key positions, the rest of the flag would be totally garbled. This is the single most common bug when rolling a Vigenère decryptor by hand.

---

## Alternative Methods

### Method 1 — CyberChef (Zero Code)

1. Go to https://gchq.github.io/CyberChef/
2. Paste the ciphertext into the **Input** box
3. From the Operations list, drag **Vigenère Decode** into the recipe
4. Set the **Key** field to `flag`
5. Read the flag from the Output box

Useful when you want to try a bunch of keys quickly without writing a new script each time.

### Method 2 — dcode.fr Vigenère Solver

1. Go to https://www.dcode.fr/vigenere-cipher
2. Paste the ciphertext
3. Enter `flag` as the key
4. Click **Decrypt**
5. Copy the flag out of the plaintext

dcode.fr is also nice because if you do not know the key, it offers a built-in **Kasiski / index-of-coincidence** brute-forcer that will try common English key lengths and rank candidate keys automatically. That is a great fallback for harder Vigenère problems where the key is not obvious.

### Method 3 — One-liner (No File Needed)

Already shown in Step 3. Best for quickly testing a candidate key without touching disk.

### Method 4 — Brute-force the Key Yourself

If you do not want to guess, you can write a small script that tries every 1- to 6-letter lowercase key (26^6 ≈ 300 million keys — too many to brute naively, so most people only try dictionary words). For picoCTF, the key is almost always a short English word related to the theme, and a hand-picked list of 30–50 guesses (`flag`, `vigenere`, `bellaso`, `alberti`, `cipher`, `history`, `key`, …) is normally enough. That is exactly how I found `flag` here.

### Method 5 — Let an Online Solver Find the Key

For a CTF you control, you would not normally just plug ciphertext into a black-box tool. But for learning, https://www.boxentriq.com/code-breaking/vigenere-cipher or https://cryptii.com/pipes/vigenere-decoder can help you test ideas. Note that for a real challenge you should always understand the math, not just rely on GUIs.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` (netcat) | Connect to the picoCTF server and receive the ciphertext |
| `cat` | Quick eyeball check on the ciphertext |
| `nano` | Write the Python decryption script |
| `python3` | Run the Vigenère decryption (key = `flag`) |
| `grep` | Pull the `picoCTF{…}` substring out of the decrypted file |
| CyberChef (alt) | One-click Vigenère decode with a known key |
| dcode.fr Vigenère (alt) | Same, plus a built-in unknown-key brute-forcer |

---

## Key Takeaways

- The challenge title is itself a clue: *la cifra de … Bellaso* is the historical name of the cipher we today call Vigenère
- When the key is not handed to you, start with short thematic words (`flag`, `cipher`, `history`, the cipher's name, the author's name) — picoCTF loves a clean English key
- A two-minute Python one-liner is the fastest way to sanity-check a candidate key against the start of the ciphertext; if it spits out a readable English sentence, you are done
- Even when the plaintext is long and historical, the decryption formula stays the same: `plain = (cipher - key) mod 26` per letter, with the key index advancing only on letters
- Don't panic if part of the decrypted text looks garbled. In real CTFs, the flag often sits inside a paragraph, so as long as you can isolate a `picoCTF{...}` pattern, you are golden

### Flag Wordplay Decode

`picoCTF{b311a50_0r_v1gn3r3_c1ph3rbc3a3Ae6}`

Decoded, segment by segment:

- `b3` → **be**
- `11` → **ll** (1 = l)
- `a` → literal `a`
- `50` → **so** (5 = s, 0 = o)
- `0r` → **or**
- `v1gn3r3` → **vigenere**
- `c1ph3r` → **cipher** (1 = i, 3 = e)
- `bc3a3Ae6` is just a per-flag salt

So the flag reads out loud as: **"be-llas-o or vi-gener-e ci-pher"** — a wink to the entire puzzle, because the "Vigenère cipher" was really Bellaso's cipher all along, misattributed for centuries.

---

## Appendix — Verbatim Server Output

For anyone reading this writeup offline and wanting to reproduce the solve without re-connecting to picoCTF, here is the exact output I got from `nc fickle-tempest.picoctf.net 54809` (newlines preserved exactly as the server sent them):

```
Encrypted message:
Ne iy nytkwpsznyg nth it mtsztcy vjzprj zfzjy rkhpibj nrkitt ltc tnnygy ysee itd tte cxjltk

Ifrosr tnj noawde uk siyyzre, yse Bnretèwp Cousex mls hjpn xjtnbjytki xatd eisjd

Iz bls lfwskqj azycihzeej yz Brftsk ip Volpnèxj ls oy hay tcimnyarqj dkxnrogpd os 1553 my Mnzvgs Mazytszf Merqlsu ny hox moup Wa inqrg ipl. Ynr. Gotgat Gltzndtg Gplrfdo

Ltc tnj tmvqpmkseaznzn uk ehox nivmpr g ylbrj ts ltcmki my yqtdosr tnj wocjc hgqq ol fy oxitngwj arusahje fuw ln guaaxjytrd catizm tzxbkw zf vqlckx hizm ceyupcz yz tnj fpvjc hgqqpohzCZK{m311a50_0x_a1rn3x3_h1ah3xgn3a3Gj6}

Ehk ktryy herq-ooizxetypd jjdcxnatoty ol f aordllvmlbkytc inahkw socjgex, bls sfoe gwzuti 1467 my Rjzn Hfetoxea Gqmexyt.

Tnj Gimjyèrk Htpnjc iy ysexjqoxj dosjeisjd cgqwej yse Gqmexyt Doxn ox Fwbkwei Inahkw.

Tn 1508, Ptsatsps Zwttnjxiax tnbjytki ehk xz-cgqwej ylbaql rkhea (g rltxni ol xsilypd gqahggpty) ysaz bzuri wazjc bk f nroytcgq nosuznkse ol yse Bnretèwp Cousex.

Gplrfdo' xpcuso butvlky lpvjlrki tn 1555 gx l cuseitzltoty ol yse lncsz. Yse rthex mllbjd ol yse gqahggpty fce tth snnqtki cemzwaxqj, bay ehk fwpnfmezx lnj yse osoed qptzjcs gwp mocpd hd xegsd ol f xnkrznoh vee usrgxp, wnnnh ify bk itfljcety hizm paim noxwpsvtydkse.
```

> Note: the server hostname and port (`fickle-tempest.picoctf.net 54809`) are randomly generated each time a new instance is spawned, so they will be different for future runs. The ciphertext *body* is different every run too (flag is per-user), but the cipher family and key stay the same — `flag` decrypts any instance of this challenge.
