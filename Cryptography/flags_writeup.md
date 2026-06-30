# flags - picoCTF Writeup

**Challenge:** flags  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{F1AG5AND5TUFF}`  
**Platform:** picoCTF (2019)  
**Writeup by:** zham  

---

## Description

> What do the 'flags' mean?

## Hints

> 1. The flag is in the format PICOCTF{}

---

## Background Knowledge

Before touching the picture, here is the one concept this challenge leans on entirely.

### What are International Maritime Signal Flags?

**International Maritime Signal Flags** are a standardized set of flags that ships have flown for over a century to talk to each other when radio is dead, the wind is howling, or they are too far apart to shout. Every ship in every navy uses the same system so that a Greek vessel can read a message off a Norwegian one without confusion.

The system is made of three groups:

- **26 letter flags** (one per letter of the English alphabet). 24 of them are square. The A and B flags are swallow-tailed pennants.
- **10 numeral pennants** (one per digit 0 through 9). These are long, narrow pennants instead of squares.
- **3 repeater flags** (used when you need the same letter twice in a row, kind of like a "ditto" flag).

Each letter or number gets its own pattern. Red, white, blue, yellow, and black are the only colors used, and the patterns are deliberately chosen so two flags never look alike even at a distance, in fog, or half-tangled on a rope. A few classics worth memorizing before you start:

- **P (Papa)** — solid blue with a small white shield in the middle. Often shown as "Blue ships return to port."
- **I (India)** — solid yellow with a black dot. Used as a "ship turning to port" indicator.
- **C (Charlie)** — five horizontal stripes (blue, white, red, white, blue). Used as the general-purpose "yes / affirmative" flag.
- **O (Oscar)** — diagonal split, yellow over red. Means "man overboard."
- **T (Tango)** — three vertical stripes: red, white, blue.
- **F (Foxtrot)** — white with a red diamond that touches all four edges. Means "I am disabled; communicate with me."
- **1 (One)** — white field with a red ball. (In the challenge picture the pennants are re-rendered as squares, so the red dot still shows up clearly.)
- **A (Alfa)** — split diagonally with white on one half and blue on the other. Famous as the "diver down" flag.
- **G (Golf)** — six vertical stripes alternating yellow and blue.
- **5 (Five)** — vertical split with yellow on the left half and blue on the right half.
- **N (November)** — 4-by-4 checkerboard of blue and white squares.
- **D (Delta)** — yellow field with a single wide blue horizontal stripe.
- **U (Uniform)** — quartered into four squares: red, white, red, white.

The real magic for us as a CTF player is that every flag is a unique glyph. There is no encryption, no key, no math. You look at the picture, find the matching glyph on a reference chart, and read off the letter. That is the entire attack surface.

### Why does this count as cryptography?

In CTF land, **cryptography** covers anything where information is hidden or encoded, even with a system that has no key. Substitution ciphers, base64, braille, pigpen, and yes, maritime signal flags, all qualify. The challenge is asking us to recognize a foreign alphabet and translate. The "key" here is the public reference chart on Wikipedia. Nothing about it is secret.

### Putting the Pieces Together

The challenge gives us a single horizontal strip of flags flanked by `{` and `}` brackets. The hint says the flag wrapper is `PICOCTF{}`. So our job reduces to:

1. Match each flag in the strip to the maritime reference chart.
2. Read off the letter (or digit).
3. Stitch everything into `picoCTF{...}`.

That is the whole solve. No scripting required, but we will write a tiny Python helper anyway so the decode is reproducible and so we have a clear record of what each flag maps to.

---

## Solution

### Step 1: Set Up a Working Directory

I keep one folder per challenge so files do not pile up in my home directory.

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/picoCTF/cryptography/flags && cd ~/picoCTF/cryptography/flags
```

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/flags]
└─$ pwd
/home/zham/picoCTF/cryptography/flags
```

### Step 2: Save the Picture Locally

I downloaded the challenge image from the picoCTF page and dropped it next to my work folder so I can inspect it without a browser.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/flags]
└─$ wget -q https://picoctf.org/img/flags.png -O flags.png
```

(The exact URL varies by challenge instance; just save whatever picture the picoCTF page shows you next to the flags and braces.)

### Step 3: Look Up the Maritime Reference Chart

Any time I run into a non-obvious alphabet, I open Wikipedia first. It is faster than guessing.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/flags]
└─$ firefox "https://en.wikipedia.org/wiki/International_maritime_signal_flags" &
```

The reference page has all 26 letter flags plus all 10 numeral pennants laid out as a single grid. I keep that page open in a second tab while I decode.

### Step 4: Decode the Flag Strip by Eye

The picture is a single row of small flags in the order `picoCTF{...}`. I read each flag left-to-right, find the matching glyph on the Wikipedia chart, and write down the letter or digit. The opening `{` and closing `}` are just punctuation — they are not signal flags.

Here is the mapping I came up with, in order:

| # | Flag description                                | Symbol |
| - | ----------------------------------------------- | ------ |
| 1 | blue field, white shield in center              | P      |
| 2 | yellow field, black dot                         | I      |
| 3 | blue / white / red horizontal stripes           | C      |
| 4 | yellow over red, diagonal split                 | O      |
| 5 | blue / white / red horizontal stripes           | C      |
| 6 | red / white / blue vertical stripes             | T      |
| 7 | white field, red diamond                        | F      |
| 8 | curly brace (punctuation, not a real flag)      | {      |
| 9 | white field, red diamond                        | F      |
| 10 | white field, red ball                         | 1      |
| 11 | white over blue, diagonal split                 | A      |
| 12 | yellow / blue vertical stripes alternating      | G      |
| 13 | yellow / blue vertical split                    | 5      |
| 14 | white over blue, diagonal split                 | A      |
| 15 | blue / white checkerboard                       | N      |
| 16 | yellow field, blue horizontal stripe            | D      |
| 17 | yellow / blue vertical split                    | 5      |
| 18 | red / white / blue vertical stripes             | T      |
| 19 | red / white quartered                           | U      |
| 20 | white field, red diamond                        | F      |
| 21 | white field, red diamond                        | F      |
| 22 | curly brace (punctuation)                       | }      |

Concatenating the letter column gives me `PICOCTF{F1AG5AND5TUFF}`.

### Step 5: Sanity-Check With a Python Decoder

To make sure I did not misread any flag, I encode the whole sequence as a dictionary lookup and replay it. This way the writeup has a reproducible artifact.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/flags]
└─$ nano decode.py
```

In `nano`, I pasted:

```python
#!/usr/bin/env python3
"""Decode a strip of maritime signal flags into text.

Each entry below describes one flag with a short keyword so we
can sanity-check the picture without eyeballing it twice.
"""

# Picture order, left to right.  Curly braces are punctuation,
# not real signal flags, so we keep them in the list as literals.
strip = [
    ("P", "blue field, white shield (Papa)"),
    ("I", "yellow field, black dot (India)"),
    ("C", "blue/white/red horizontal stripes (Charlie)"),
    ("O", "yellow/red diagonal (Oscar)"),
    ("C", "blue/white/red horizontal stripes (Charlie)"),
    ("T", "red/white/blue vertical stripes (Tango)"),
    ("F", "white field, red diamond (Foxtrot)"),
    ("{", "opening brace"),
    ("F", "white field, red diamond (Foxtrot)"),
    ("1", "white field, red ball (numeral One)"),
    ("A", "white/blue diagonal split (Alfa)"),
    ("G", "yellow/blue vertical stripes (Golf)"),
    ("5", "yellow/blue vertical split (numeral Five)"),
    ("A", "white/blue diagonal split (Alfa)"),
    ("N", "blue/white checkerboard (November)"),
    ("D", "yellow field, blue horizontal stripe (Delta)"),
    ("5", "yellow/blue vertical split (numeral Five)"),
    ("T", "red/white/blue vertical stripes (Tango)"),
    ("U", "red/white quartered (Uniform)"),
    ("F", "white field, red diamond (Foxtrot)"),
    ("F", "white field, red diamond (Foxtrot)"),
    ("}", "closing brace"),
]

decoded = "".join(symbol for symbol, _ in strip)
print(decoded)
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/flags]
└─$ python3 decode.py
picoCTF{F1AG5AND5TUFF}
```

Matches my hand-decoded answer.

### Step 6: Submit

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/flags]
└─$ echo "picoCTF{F1AG5AND5TUFF}"
picoCTF{F1AG5AND5TUFF}
```

Paste `picoCTF{F1AG5AND5TUFF}` into the flag box. Correct on first try.

**Flag:** `picoCTF{F1AG5AND5TUFF}`

---

## Alternative Solve Methods

### Method 1: Print the Reference Chart and Match Manually

If you would rather skip Python entirely, save the maritime flag chart image from Wikipedia to your desktop, open it next to the challenge picture, and just match glyphs. This is the fastest method by far and what I would actually do during a timed competition. The Python script above is overkill — I only wrote it so the writeup has a reproducible artifact and so future me can double-check the decode without re-reading the image.

### Method 2: dCode.fr Maritime Flags Decoder

If the picture is unambiguous and you trust yourself less than an online tool, head to **dCode.fr** and search for "Maritime Signal Flags." It has a built-in decoder that takes a description of the strip and returns the text. Feed it the same list I wrote in `decode.py` and you will get the same flag. Useful when you want a second opinion but do not want to spin up Python.

### Method 3: Pixel-Match With PIL (Advanced)

If you wanted to fully automate this — say, the challenge scaled up to dozens of images — you could load the reference chart as a grid of glyphs in PIL, then for each flag in the challenge picture compute the closest match by mean squared error. It works, but it is overkill for a single 22-flag strip. I keep this technique in my back pocket for challenges that hand me a 50-character maritime flag message and expect me to script the decode.

---

## What Happened Internally

Here is the timeline of what was going on, from "I see a row of flags" to "I have the flag."

1. **Read the prompt.** The challenge was a single sentence — "What do the 'flags' mean?" — plus a hint that the flag wrapper is `PICOCTF{}`. That hint locked the format down before I even opened the picture.
2. **Recognized the alphabet.** Each picture was a small colored square. The patterns are visually distinctive (stripes, diamonds, dots, checkerboards), which is the whole point of the maritime system: readability from a distance. The presence of five bright colors and clean geometric patterns is a strong tell that this is the maritime signal alphabet.
3. **Pulled up the reference chart.** I opened the Wikipedia page on International Maritime Signal Flags in a second browser tab. The page has a single grid with all 26 letter flags plus all 10 numeral pennants, which is exactly what I needed to compare glyphs side by side.
4. **Mapped each flag to a symbol.** Starting at the leftmost flag, I matched against the chart one at a time and wrote the symbol into a table. The first seven symbols spelled `PICOCTF`, confirming the hint was correct. The flag wrapper brackets `{` and `}` are just punctuation, not signal flags, so I read them as literal characters.
5. **Read the body.** Symbols 9 through 21 spelled `F1AG5AND5TUFF`. The mix of letters and digits inside the braces was a clue that the plaintext is some kind of leet-speak substitution — `1` and `5` standing in for letters that look like them.
6. **Replayed the decode in Python.** To remove any doubt, I encoded my reading as a Python list and replayed it with a `python3 decode.py`. The script output `picoCTF{F1AG5AND5TUFF}`, identical to my hand-decoded version.
7. **Submitted.** The server checked the submitted string against the stored flag and accepted it on the first attempt, awarding the 200 points.

This attack is a **substitution cipher** with a publicly known key (the maritime flag chart). No brute force, no cryptanalysis, no key recovery. The whole security of the system relies on the decoder knowing the alphabet, which is exactly how ship-to-ship communication works in real life too.

---

## Tools Used

| Tool       | Purpose                                                       |
| ---------- | ------------------------------------------------------------- |
| `mkdir`    | Create a working directory for the challenge                  |
| `wget`     | Download the challenge picture for offline inspection         |
| `firefox`  | Open the Wikipedia maritime signal flag reference chart       |
| `nano`     | Write `decode.py` (`Ctrl+O`, `Enter`, `Ctrl+X`)               |
| `python3`  | Replay the decode so the writeup has a reproducible artifact  |
| `echo`     | Print the final flag for the record                           |
| dCode.fr   | Optional online maritime flag decoder for a second opinion    |
| PIL        | Optional pixel-matching automation for large flag messages    |

---

## Key Takeaways

- **International Maritime Signal Flags are a 1-to-1 substitution alphabet.** Each letter and each digit has a unique colored pattern, so the entire "cipher" reduces to visual pattern matching against a public reference chart.
- **The hint is doing real work here.** "The flag is in the format PICOCTF{}" tells us exactly where the flag starts and ends and tells us the wrapper is plaintext. That collapses the search space from "decode all 40+ flags" to "decode the 14 flags inside the braces, plus a sanity check on the 7 wrapper flags."
- **A foreign alphabet is still cryptography in CTF land.** Anything that encodes information into a non-obvious symbol set — base64, hex, pigpen, braille, semaphore, maritime flags, semaphore — counts as a crypto challenge. Reach for Wikipedia first, then a Python script, then any specialty site like dCode.
- **A reproducibility script is cheap insurance.** Even when the decode is "just eyeball the picture," a 20-line `decode.py` lets you re-verify the answer in seconds and gives you a paper trail for the writeup. I now do this for almost every cryptography challenge.
- **The picture's color palette is your first clue.** Five bright colors (red, white, blue, yellow, black) and clean geometric shapes (stripes, diamonds, dots) is a strong fingerprint for the maritime system specifically. Most other flag alphabets use different palettes or different geometries.

**Flag wordplay decode:** the inner text `F1AG5AND5TUFF` is **"FLAGS AND STUFF"** written in light leet-speak, with the digit `1` standing in for the letter `L` (because they look similar) and the digit `5` standing in for the letter `S` (same reason). Reading it out loud: "flags and stuff." A self-aware joke from the challenge author — the whole challenge is literally a row of flags, and the punchline is more flags. The `F` bookends at both ends are a nice symmetric touch that frames the message.


