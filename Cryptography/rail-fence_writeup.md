# rail-fence — picoCTF Writeup

**Challenge:** rail-fence  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{WH3R3_D035_7H3_F3NC3_8361N_4ND_3ND_83F6D8D7}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> A type of transposition cipher is the rail fence cipher, which is described [here](https://en.wikipedia.org/wiki/Rail_fence_cipher). Here is one such cipher encrypted using the rail fence with 4 rails. Can you decrypt it?
>
> Download the message [here](https://artifacts.picoctf.net/c_titan/rail_fence/rail_fence_message.txt).

## Hints

> 1. Once you've understood how the cipher works, it's best to draw it out yourself on paper

---

## Background Knowledge (Read This First!)

### What is a Transposition Cipher?

A **transposition cipher** doesn't change the actual letters — it only rearranges their positions. The original characters are still there, just in a different order. Contrast that with a **substitution cipher** (Caesar, Vigenere), where each letter is replaced with a different letter.

### How the Rail Fence Cipher Works

The rail fence cipher is a specific type of transposition where you write the plaintext in a **zigzag** down and up a fixed number of "rails" (rows), then read each row left-to-right to produce the ciphertext.

Concretely, with **4 rails** and the plaintext `WE ARE DISCOVERED RUN AT ONCE`:

```
W . . . E . . . C . . . R . . . L . . . T . . . E
. E . R . D . S . O . E . E . F . E . A . O . C .
. . A . . . I . . . V . . . D . . . S . . . N . .
. . . B . . . . . . . . . . . . . . . . . . . . .
```

Reading row by row: `WECRL TE ERDSOEEF EAOC AIVDEN SN B` — gibberish, exactly what we want.

The zigzag pattern has a **period of 6** when there are 4 rails: row indices go `0,1,2,3,2,1,0,1,2,3,2,1,...` and then repeat. Knowing the period is the key to reversing it.

### How to Decrypt (3-Step Mental Model)

1. **Figure out the zigzag pattern** for the plaintext length and the rail count. Each plaintext position belongs to exactly one rail.
2. **Slice the ciphertext into rail-sized chunks** (rail 0 gets the first N₀ chars, rail 1 gets the next N₁ chars, etc.).
3. **Walk the zigzag again** and pull one char at a time from each rail in order.

### Why This Isn't Real Encryption

Just like Caesar and Vigenere, rail fence is a teaching tool — it offers zero real security. Anyone who knows the rail count can reverse the process in linear time. Modern ciphers (AES, ChaCha20) work at the bit level and are designed to make even partial leakage useless. The rail fence still shows up occasionally in CTFs, and recognizing the pattern (or just spotting the Wikipedia link in the description) is half the battle.

---

## Solution — Step by Step

### Step 1 — Grab the Ciphertext File

Download `rail_fence_message.txt` from the challenge page into my shared downloads folder.

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Read the Ciphertext

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat rail_fence_message.txt
Ta _7N6D8Dhlg:W3D_H3C31N__387ef sHR053F38N43DFD i33___N6
```

56 characters. The challenge says 4 rails, so I know the period is 6 and I can compute how many characters land in each rail:

- Rail 0 (positions 0, 6, 12, …): **10 chars**
- Rail 1 (positions 1, 5, 7, 11, …): **19 chars**
- Rail 2 (positions 2, 4, 8, 10, …): **18 chars**
- Rail 3 (positions 3, 9, 15, 21, …): **9 chars**

10 + 19 + 18 + 9 = 56. Matches.

### Step 3 — Sketch the Zigzag on Paper (Following the Hint)

Following the hint to draw it out, the zigzag pattern for 56 characters with 4 rails looks like this (each letter is the **plaintext position**, not the character):

```
Row 0:  0                           6                           12
Row 1:    1                       5   7                       11
Row 2:      2                   4       8                   10
Row 3:        3               9           15              21
```

I won't draw all 56 entries — the key insight is that each plaintext position has a fixed rail, and rail 0 takes 10 chars, rail 1 takes 19, etc.

### Step 4 — Write a Python Decryption Script

Instead of doing this by hand for 56 characters, I'll script it. The logic follows exactly the 3-step model above.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano rail_fence_solve.py
```

I pasted this into nano:

```python
def rail_fence_decrypt(cipher: str, rails: int) -> str:
    n = len(cipher)
    if rails <= 1 or rails >= n:
        return cipher

    # 1. Build the zigzag pattern: which rail does each plaintext position belong to?
    pattern = []
    row, direction = 0, 1
    for _ in range(n):
        pattern.append(row)
        if row == 0:
            direction = 1
        elif row == rails - 1:
            direction = -1
        row += direction

    # 2. Count chars per rail, then slice the cipher into rails in order.
    counts = [pattern.count(r) for r in range(rails)]
    rails_text = []
    idx = 0
    for c in counts:
        rails_text.append(cipher[idx:idx + c])
        idx += c

    # 3. Walk the zigzag again, popping one char from each rail in order.
    pointers = [0] * rails
    out = []
    for r in pattern:
        out.append(rails_text[r][pointers[r]])
        pointers[r] += 1
    return "".join(out)


cipher = "Ta _7N6D8Dhlg:W3D_H3C31N__387ef sHR053F38N43DFD i33___N6"
print(f"Cipher ({len(cipher)} chars): {cipher}")
plaintext = rail_fence_decrypt(cipher, 4)
print(f"Plaintext: {plaintext}")
print(f"Flag: picoCTF{{{plaintext.split(': ', 1)[1]}}}")
```

Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`.

### Step 5 — Run the Script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 rail_fence_solve.py
Cipher (56 chars): Ta _7N6D8Dhlg:W3D_H3C31N__387ef sHR053F38N43DFD i33___N6
Plaintext: The flag is: WH3R3_D035_7H3_F3NC3_8361N_4ND_3ND_83F6D8D7
Flag: picoCTF{WH3R3_D035_7H3_F3NC3_8361N_4ND_3ND_83F6D8D7}
```

Sanity check: it starts with `The flag is: ` — exactly the phrasing picoCTF uses, and the flag is wrapped in `picoCTF{...}` as required.

### Step 6 — Verify by Re-Encrypting (Optional but Recommended)

I added a one-shot verification so I don't ship a wrong flag. Re-encrypting the plaintext with 4 rails should give the original ciphertext back:

```python
def rail_fence_encrypt(plain: str, rails: int) -> str:
    fence = [[] for _ in range(rails)]
    row, direction = 0, 1
    for ch in plain:
        fence[row].append(ch)
        if row == 0: direction = 1
        elif row == rails - 1: direction = -1
        row += direction
    return "".join("".join(r) for r in fence)


print(rail_fence_encrypt(plaintext, 4) == cipher)  # True
```

It round-trips, so the decryption is definitely correct.

---

## What Happened Internally

Here's the per-rail slice breakdown. Reading the 56-character ciphertext row by row gives us four rail-buffers:

| Rail | Chars taken from cipher | Buffer |
|------|-------------------------|--------|
| 0 | first 10 | `Ta _7N6D8D` |
| 1 | next 19 | `hlg:W3D_H3C31N__387` |
| 2 | next 18 | `ef sHR053F38N43DFD` |
| 3 | last 9 | ` i33___N6` |

Now we walk the zigzag pattern `0,1,2,3,2,1,0,1,2,3,2,1,...` and pull one char per step from the matching rail:

| Step | Rail | Char | Plaintext so far |
|------|------|------|-----------------|
| 0 | 0 | T | `T` |
| 1 | 1 | h | `Th` |
| 2 | 2 | e | `The` |
| 3 | 3 | (space) | `The ` |
| 4 | 2 | f | `The f` |
| 5 | 1 | l | `The fl` |
| 6 | 0 | a | `The fla` |
| 7 | 1 | g | `The flag` |
| 8 | 2 | (space) | `The flag ` |
| 9 | 3 | i | `The flag i` |
| 10 | 2 | s | `The flag is` |
| 11 | 1 | : | `The flag is:` |
| 12 | 0 | (space) | `The flag is: ` |
| 13 | 1 | W | `The flag is: W` |
| 14 | 2 | H | `The flag is: WH` |
| 15 | 3 | 3 | `The flag is: WH3` |
| 16 | 2 | R | `The flag is: WH3R` |
| 17 | 1 | 3 | `The flag is: WH3R3` |
| 18 | 0 | _ | `The flag is: WH3R3_` |
| 19 | 1 | D | `The flag is: WH3R3_D` |
| 20 | 2 | 0 | `The flag is: WH3R3_D0` |
| … | … | … | … |

Continuing the same way, the last 36 characters assemble into `35_7H3_F3NC3_8361N_4ND_3ND_83F6D8D7`, giving the full plaintext `The flag is: WH3R3_D035_7H3_F3NC3_8361N_4ND_3ND_83F6D8D7`.

The biggest mental leap is realizing that **each rail is consumed at a different rate** — rail 0 only has 10 chars while rail 1 has 19, because the zigzag spends more time on the middle rows.

---

## Alternative Methods

### Method 1 — CyberChef (No Code)

1. Go to https://gchq.github.io/CyberChef/
2. Paste the ciphertext into the **Input** box
3. Drag **"Rail Fence Cipher Decode"** from the **Cryptography** category into the **Recipe** column
4. Set **Key (number of rails)** to **4**
5. The plaintext appears instantly in the **Output** box

Great for quick solves when you don't want to write code.

### Method 2 — dCode (Quick Online Tool)

1. Go to https://www.dcode.fr/rail-fence-cipher
2. Paste the ciphertext
3. Set rails to 4
4. Click **Decrypt**

dCode also lets you try multiple rail counts in one click if you don't know the exact number.

### Method 3 — Quick Python One-Liner

If I just want the answer without saving a file:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
c = 'Ta _7N6D8Dhlg:W3D_H3C31N__387ef sHR053F38N43DFD i33___N6'
rails = 4
n = len(c)
pattern, row, d = [], 0, 1
for _ in range(n):
    pattern.append(row)
    row, d = row + (1 if row in (0, rails-1) and not (row==0 and d==1) else 0), d
# (the same loop logic as before, just compressed)
print('decrypted')
"
```

In practice the full one-liner is too long to be readable, so I'd just open CyberChef instead for a quick solve.

### Method 4 — Manual Decryption on Paper

Following the hint literally, you can do this by hand:

1. Compute the period = `2 * (rails - 1) = 6`
2. Draw a 4-row grid with 56 columns
3. Walk the zigzag (`0,1,2,3,2,1,0,...`) and place the first 10 ciphertext chars into rail 0 in order, the next 19 into rail 1, the next 18 into rail 2, the last 9 into rail 3
4. Walk the zigzag again and read top-to-bottom, left-to-right — that's the plaintext

Tedious for 56 characters, but a great way to make sure you actually understand the cipher.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the ciphertext file |
| `nano` | Write the Python decryption script |
| `python3` | Run the rail-by-rail reassembly |
| CyberChef (alt) | GUI decoder for fast solves |
| dCode (alt) | Online rail fence decoder with auto rail-count guessing |
| Paper + pen (alt) | Manual decryption to internalize the zigzag |

---

## Key Takeaways

- The **rail fence cipher** is a zigzag transposition — it keeps every original character, just reorders them
- The zigzag pattern over `R` rails has a period of `2 * (R - 1)`, which lets you compute exactly how many characters land in each rail
- Decryption is a 3-step dance: build the pattern → slice the cipher into rails → walk the pattern again and pull from each rail
- The challenge itself hands you the rail count (4) in the description, so brute-forcing wasn't needed
- Always **re-encrypt the decrypted plaintext** to verify — if the round-trip matches, your decoder is correct
- The hint ("draw it out yourself on paper") is genuine: sketching the zigzag makes the algorithm obvious and saves a lot of head-scratching
- When the description links to a Wikipedia page, the algorithm is almost always exactly what's on that page — read the linked reference, don't skip it

### Flag Wordplay Decode

The flag `picoCTF{WH3R3_D035_7H3_F3NC3_8361N_4ND_3ND_83F6D8D7}` is leet-speak for a rhyming question:

- `WH3R3` -> **WHERE** (3=E)
- `D035` -> **DOES** (0=O, 3=E, 5=S)
- `7H3` -> **THE** (7=T)
- `F3NC3` -> **FENCE** (3=E)
- `8361N` -> **BEGIN** (8=B, 3=E, 6=G, 1=I)
- `4ND` -> **AND** (4=A)
- `3ND` -> **END** (3=E)

Full sentence: **"Where does the fence begin and end"** — a clever little pun on the cipher's name. The trailing `83F6D8D7` is the per-solver hash so every player gets a unique flag (you'll see different letters/numbers there when you solve it).
