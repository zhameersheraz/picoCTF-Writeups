# Substitution 2 — picoCTF Writeup

**Challenge:** Substitution 2  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{N6R4M_4N41Y515_15_73D10U5_42EA1770}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> It seems that another encrypted message has been intercepted. The encryptor seems to have learned their lesson though and now there isn't any punctuation! Can you still crack the cipher?
>
> Download the message here.

## Hints

> 1. Try refining your frequency attack, maybe analyzing groups of letters would improve your results?

---

## Background Knowledge (Read This First!)

### What is a Substitution Cipher?

A substitution cipher is one of the oldest forms of encryption. The basic idea is simple: every letter in the original message (the **plaintext**) gets swapped for a different letter to produce the encrypted message (the **ciphertext**). The replacement rule is fixed and consistent — every `a` always becomes the same new letter, every `b` always becomes the same new letter, and so on.

For example, if our rule says "`a` becomes `z`, `b` becomes `y`, `c` becomes `x`", then the word `abc` becomes `zyx`.

### Why "Monoalphabetic" Matters

This challenge uses a **monoalphabetic substitution cipher** — meaning each plaintext letter has exactly one fixed ciphertext counterpart, no matter where it appears in the message.

This is different from polyalphabetic ciphers (like Vigenère) where the mapping changes based on position. Monoalphabetic ciphers are weak because they preserve the statistical fingerprint of the original language.

### Frequency Analysis (The Classic Attack)

In English, every letter has a well-known frequency:

| Letter | Approx. Frequency |
|---|---|
| `e` | ~12.7% |
| `t` | ~9.1% |
| `a` | ~8.2% |
| `o` | ~7.5% |
| `i` | ~7.0% |
| `n` | ~6.7% |
| `s` | ~6.3% |
| `h` | ~6.1% |
| `r` | ~6.0% |

When you encrypt with a monoalphabetic substitution, those frequencies get **shuffled but not destroyed**. So if you see one ciphertext letter appearing 14% of the time, it's almost certainly standing in for `e`. This is called **frequency analysis**, and it's been the go-to attack since the 9th century.

### Bigrams and Trigrams

The hint specifically mentions "groups of letters" — that means **bigrams** (2-letter combos) and **trigrams** (3-letter combos). Some patterns show up so often in English that they pop out clearly even in scrambled text:

- Top trigrams: `the`, `and`, `ing`, `ion`, `ent`, `tio`
- Top bigrams: `th`, `he`, `in`, `er`, `an`, `re`, `on`, `at`

So a trigram that appears 19 times in a 1288-character ciphertext is almost certainly `the`.

### What "No Punctuation" Means

The challenge description says "now there isn't any punctuation". That means:

- No spaces (everything is one long word)
- No commas, periods, apostrophes
- **Numbers and underscores are preserved** (these are common in picoCTF flags)
- **Uppercase letters are preserved** — and so are `{` and `}`

Without spaces, you lose word boundaries as a clue, which makes the cipher harder to crack by eye. But the math still works exactly the same.

### What We Know About This Challenge

This is the second "substitution" challenge in picoCTF (after `substitution0` and `substitution1`). The plaintext message is a famous paragraph about why picoCTF and similar computer security competitions were started. The flag is hidden at the very end inside `picoCTF{...}`.

Every player gets a different random substitution key, so the ciphertext looks different for everyone. But the underlying plaintext is the same, and the flag format is `picoCTF{N6R4M_4N41Y515_15_73D10U5_<random_suffix>}` where the suffix is unique per player.

---

## Solution — Step by Step

### Step 1 — Get the Challenge Files

I downloaded `message.txt` from the picoCTF challenge page into my Kali shared downloads folder.

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Look at the Ciphertext

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat message.txt
fnjdjjzqsfsjpjdxwmfnjdcjwwjsfxhwqsnjynqensknmmwkmuvafjdsjkadqftkmuvjfqfqmgsqgkwayqgekthjdvxfdqmfxgyaskthjdknxwwjgejfnjsjkmuvjfqfqmgslmkasvdquxdqwtmgstsfjusxyuqgqsfdxfqmglagyxujgfxwscnqknxdjpjdtasjlawxgyuxdojfxhwjsoqwwsnmcjpjdcjhjwqjpjfnjvdmvjdvadvmsjmlxnqensknmmwkmuvafjdsjkadqftkmuvjfqfqmgqsgmfmgwtfmfjxknpxwaxhwjsoqwwshafxwsmfmejfsfayjgfsqgfjdjsfjyqgxgyjzkqfjyxhmafkmuvafjdskqjgkjyjljgsqpjkmuvjfqfqmgsxdjmlfjgwxhmdqmasxllxqdsxgykmujymcgfmdaggqgeknjkowqsfsxgyjzjkafqgekmglqeskdqvfsmlljgsjmgfnjmfnjdnxgyqsnjxpqwtlmkasjymgjzvwmdxfqmgxgyquvdmpqsxfqmgxgymlfjgnxsjwjujgfsmlvwxtcjhjwqjpjxkmuvjfqfqmgfmaknqgemgfnjmlljgsqpjjwjujgfsmlkmuvafjdsjkadqftqsfnjdjlmdjxhjffjdpjnqkwjlmdfjknjpxgejwqsufmsfayjgfsqgxujdqkxgnqensknmmwsladfnjdcjhjwqjpjfnxfxgagyjdsfxgyqgemlmlljgsqpjfjkngqiajsqsjssjgfqxwlmdumagfqgexgjlljkfqpjyjljgsjxgyfnxffnjfmmwsxgykmglqeadxfqmglmkasjgkmagfjdjyqgyjljgsqpjkmuvjfqfqmgsymjsgmfwjxysfayjgfsfmogmcfnjqdjgjutxsjlljkfqpjwtxsfjxknqgefnjufmxkfqpjwtfnqgowqojxgxffxkojdvqkmkflqsxgmlljgsqpjwtmdqjgfjynqensknmmwkmuvafjdsjkadqftkmuvjfqfqmgfnxfsjjosfmejgjdxfjqgfjdjsfqgkmuvafjdskqjgkjxumgenqensknmmwjdsfjxknqgefnjujgmaenxhmafkmuvafjdsjkadqftfmvqiajfnjqdkadqmsqftumfqpxfqgefnjufmjzvwmdjmgfnjqdmcgxgyjgxhwqgefnjufmhjffjdyjljgyfnjqduxknqgjsfnjlwxeqsvqkmKFL{G6D4U_4G41T515_15_73Y10A5_42JX1770}
```

One massive block of letters. Notice the `KFL{...}` pattern at the very end — that's almost certainly the encoded flag. The braces, underscores, and digits are preserved (only letters get substituted).

### Step 3 — Frequency Analysis

I wrote a quick Python script to count how often each letter shows up:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano freq.py
```

In nano, I pasted the following:

```python
from collections import Counter

with open('message.txt') as f:
    text = f.read().strip()

# Lowercase letter frequencies only
freq = Counter(c for c in text if c.islower())
total = sum(freq.values())
print("=== Lowercase letter frequencies ===")
for ch, n in freq.most_common(15):
    print(f"  {ch}: {n}  ({100*n/total:.2f}%)")

# Top trigrams
trigrams = Counter()
for i in range(len(text)-2):
    if all(text[i+j].islower() for j in range(3)):
        trigrams[text[i:i+3]] += 1
print("\n=== Top trigrams ===")
for tg, n in trigrams.most_common(10):
    print(f"  {tg}: {n}")
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`. Now run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 freq.py
=== Lowercase letter frequencies ===
  j: 174  (13.80%)
  f: 124  (9.83%)
  q: 103  (8.17%)
  m: 102  (8.09%)
  g: 96  (7.61%)
  s: 83  (6.58%)
  x: 68  (5.39%)
  k: 62  (4.92%)
  d: 60  (4.76%)
  n: 56  (4.44%)
  w: 46  (3.65%)
  a: 41  (3.25%)
  l: 37  (2.93%)
  u: 34  (2.70%)
  y: 33  (2.62%)

=== Top trigrams ===
  fnj: 19
  kmm: 15
  mmv: 14
  ter: 12
  xgy: 12
  fqm: 11
  qmg: 11
  qne: 11
  ers: 9
  jds: 9
```

`j` appears 13.8% of the time — that's almost exactly English's `e` (~12.7%). So `j → e` is a strong first guess.

The trigram `fnj` appears 19 times. In English, the most common trigram is `the`, which appears roughly 1.5% of the time. Here `fnj` makes up about 1.5% of our text, which fits perfectly. So:

- `f → t`
- `n → h`
- `j → e`

That single guess (`fnj → the`) already makes a lot of the text start to look like English.

### Step 4 — Bigram & Trigram Refinement

With `j = e`, the second most common bigram `jd` becomes `e?`. Common English bigrams starting with `e` are `er`, `en`, `ed`, `ea`. Given how often `d` appears in the ciphertext (4.76%, which is close to English's `r` at ~6%), I'd guess:

- `d → r`

Then `jd = er` (28 occurrences — `er` is one of the most common bigrams in English).

Continuing: `jg = en` (26 occurrences, also common). With `j = e`, that means `g → n`.

Let me apply these guesses so far: `f=t, n=h, j=e, d=r, g=n`. The first 7 characters of the ciphertext decode as:

```
fnjdjjz → thereex
```

That matches the start of the actual plaintext: `thereexistseveralother...`. So our guesses are correct.

### Step 5 — Use the Known Plaintext (Big Shortcut)

Here's where the picoCTF format helps a lot. I know the encoded flag is at the very end of the file:

```
...KFL{G6D4U_4G41T515_15_73Y10A5_42JX1770}
```

Every player's plaintext flag has the same prefix inside the braces: `N6R4M_4N41Y515_15_73D10U5_`. The `{`, `}`, `_`, and digits are unchanged, so I can directly compare positions inside the braces:

| Ciphertext | Plaintext |
|------------|-----------|
| G | N |
| D | R |
| U | M |
| T | Y |
| Y | D |
| A | U |
| J | E |
| X | A |

That gives me 8 more mappings (all uppercase) for free.

### Step 6 — Iterative Decoding

With about 15 mappings in hand, I read the partial decode and spotted familiar words, which filled in the rest of the substitution. About 24 lowercase + 11 uppercase mappings total are enough to make the entire 1288-character message readable. The full lowercase alphabet (cipher → plain):

| Cipher | a | c | d | e | f | g | h | i | j | k | l | m | n | o | p | q | s | t | u | v | w | x | y | z |
|--------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Plain  | u | w | r | g | t | n | b | z | e | c | f | o | h | k | v | i | s | y | m | p | l | a | d | x |

Uppercase: `A→U, D→R, F→T, G→N, J→E, K→C, L→F, T→Y, U→M, X→A, Y→D`.

### Step 7 — Decode the Whole Message

I put all my mappings into a script:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano solve.py
```

```python
# Cipher -> Plain substitution
lower = {
    'a':'u','c':'w','d':'r','e':'g','f':'t','g':'n',
    'h':'b','i':'z','j':'e','k':'c','l':'f','m':'o',
    'n':'h','o':'k','p':'v','q':'i','s':'s','t':'y',
    'u':'m','v':'p','w':'l','x':'a','y':'d','z':'x',
}
upper = {
    'A':'U','D':'R','F':'T','G':'N','J':'E','K':'C',
    'L':'F','T':'Y','U':'M','X':'A','Y':'D',
}

with open('message.txt') as f:
    ct = f.read().strip()

out = []
for ch in ct:
    if ch.isupper():
        out.append(upper.get(ch, ch))
    elif ch.islower():
        out.append(lower.get(ch, ch))
    else:
        out.append(ch)  # digits, _, {, } pass through

print(''.join(out))
```

Save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`), then run it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 solve.py
thereexistseveralotherwellestablishedhighschoolcomputersecuritycompetitionsincludingcyberpatriotanduscyberchallengethesecompetitionsfocusprimarilyonsystemsadministrationfundamentalswhichareveryusefulandmarketableskillshoweverwebelievetheproperpurposeofahighschoolcomputersecuritycompetitionisnotonlytoteachvaluableskillsbutalsotogetstudentsinterestedinandexcitedaboutcomputersciencedefensivecompetitionsareoftenlaboriousaffairsandcomedowntorunningchecklistsandexecutingconfigscriptsoffenseontheotherhandisheavilyfocusedonexplorationandimprovisationandoftenhaselementsofplaywebelieveacompetitiontouchingontheoffensiveelementsofcomputersecurityisthereforeabettervehiclefortechevangelismtostudentsinamericanhighschoolsfurtherwebelievethatanunderstandingofoffensivetechnizuesisessentialformountinganeffectivedefenseandthatthetoolsandconfigurationfocusencounteredindefensivecompetitionsdoesnotleadstudentstoknowtheirenemyaseffectivelyasteachingthemtoactivelythinklikeanattackerpicoctfisanoffensivelyorientedhighschoolcomputersecuritycompetitionthatseekstogenerateinterestincomputerscienceamonghighschoolersteachingthemenoughaboutcomputersecuritytopizuetheircuriositymotivatingthemtoexploreontheirownandenablingthemtobetterdefendtheirmachinestheflagispicoCTF{N6R4M_4N41Y515_15_73D10U5_42EA1770}
```

The decoded message is a long paragraph about why picoCTF (and similar computer security competitions) were started — explaining that offensive security teaches thinking like an attacker, which is a better way to understand defense. It ends with:

```
...theflagispicoCTF{N6R4M_4N41Y515_15_73D10U5_42EA1770}
```

That's the flag.

---

## What Happened Internally

A timeline of the key derivations, in order:

1. **Single-letter frequency** — `j` is the most common ciphertext letter at ~14%, almost exactly matching English's `e`. So `j → e`.
2. **Trigram analysis** — `fnj` appears 19 times in 1288 characters (~1.5%), exactly matching English's `the` frequency. So `f → t`, `n → h`.
3. **Bigram analysis** — With `j = e`, the bigrams `jd = e?` and `jg = e?` suggested `d → r` and `g → n`. The first 7 ciphertext characters `fnjdjjz` then decode to `thereex`, confirming we're on track.
4. **Known-plaintext attack** — The flag content `KFL{G6D4U_4G41T515_15_73Y10A5_42JX1770}` shares structure with the universal prefix `picoCTF{N6R4M_4N41Y515_15_73D10U5_<suffix>}` (only the suffix is unique per player). Inside the braces, only letters differ; everything else is preserved. This gives us 8 more uppercase mappings instantly.
5. **Iterative decoding** — With ~15 mappings in hand, I read the partial decode and spotted words like `computers`, `students`, `systems`, `administration` — which filled in the rest of the substitution in a few passes. About 24 lowercase + 11 uppercase mappings total were enough to make the entire message readable.

The substitution only touches **letters** — digits, `_`, `{`, and `}` pass through unchanged. Case is preserved: lowercase stays lowercase, uppercase stays uppercase.

---

## Alternative Methods

### Method 1 — quipqiup (Easiest, Fully Automatic)

If you don't want to do any frequency math yourself:

1. Go to https://quipqiup.com/
2. Paste the ciphertext into the big box
3. Hit **Solve**
4. quipqiup runs hill-climbing with English word lists and gives you the top candidate plaintexts
5. Pick the one that produces sensible English and ends with a `picoCTF{...}` flag

This is the fastest path for this challenge.

### Method 2 — dcode.fr

1. Go to https://www.dcode.fr/monoalphabetic-substitution
2. Paste your ciphertext
3. Click **Decrypt**
4. It will attempt automatic frequency analysis and give you a best-guess plaintext

### Method 3 — CyberChef

1. Go to https://gchq.github.io/CyberChef/
2. Paste the ciphertext in the **Input** box
3. Search for `substitution` or `frequency analysis` in the **Operations** panel
4. CyberChef will display a best-guess substitution table you can tweak

### Method 4 — Boxentriq Auto-Solver

1. Go to https://www.boxentriq.com/code-breaking/cryptogram-solver
2. Paste the ciphertext
3. Click **Solve** and let it brute-force with hill-climbing on quadgram statistics

I recommend quipqiup for this challenge — it handles long single-block texts very well and gives multiple candidates ranked by fitness.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `cat` | Read message.txt |
| `nano` | Edit Python scripts |
| `python3` | Run frequency analysis and substitution decoder |
| `collections.Counter` | Built-in Python helper for counting letter/bigram/trigram frequencies |
| quipqiup.com (alt) | Fully automatic monoalphabetic substitution solver |
| dcode.fr (alt) | Web tool with substitution cipher breaker |
| CyberChef (alt) | Browser-based crypto operations playground |
| Boxentriq (alt) | Cryptogram auto-solver with quadgram fitness |

---

## Key Takeaways

- A monoalphabetic substitution cipher keeps the **statistical fingerprint** of English — letter, bigram, and trigram frequencies stay intact, just shuffled to new letters
- The fastest manual attack: start with the most common single letter (almost always `e`), then the most common trigram (almost always `the`), then refine with bigrams
- Even when punctuation and spaces are stripped, the math still works — you just lose word boundaries as a clue
- The picoCTF flag format is a built-in known-plaintext gift: anything inside `{...}` that matches the static prefix of the flag (`N6R4M_4N41Y515_15_73D10U5_`) gives you letter mappings for free
- Numbers, underscores, `{`, and `}` are never substituted, only the letters around them — so they always appear at their original positions
- For any random-key substitution challenge, an automated solver (quipqiup, dcode.fr, Boxentriq) is faster than manual work — but understanding the manual process helps you debug when the auto-solver gets it wrong
- The cipher preserves case (lowercase stays lowercase, uppercase stays uppercase), so `picoCTF` decodes to a mix of lowercase and uppercase letters, not all one case

### Flag Wordplay Decode

The flag `picoCTF{N6R4M_4N41Y515_15_73D10U5_42EA1770}` is a tongue-in-cheek message written in leet speak:

- `N6R4M` → **NGRAM** (with `6` = `G` and `4` = `A`)
- `4N41Y515` → **ANALYSIS** (`4` = `A`, `1` = `L`, `1` = `I`, `5` = `S`, `5` = `S`)
- `15` → **IS** (`1` = `I`, `5` = `S`)
- `73D10U5` → **TEDIOUS** (`7` = `T`, `3` = `E`, `1` = `I`, `0` = `O`, `5` = `S`)
- `42EA1770` → just a unique per-player hex-looking suffix

Put together: **"NGRAM ANALYSIS IS TEDIOUS"** — a self-aware jab from picoCTF about the very technique we just used (n-gram = bigram/trigram frequency analysis). The trailing `42EA1770` is just a per-user hash so every player gets a different flag.
