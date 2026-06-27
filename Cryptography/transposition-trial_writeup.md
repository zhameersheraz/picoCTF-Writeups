# transposition-trial — picoCTF Writeup

**Challenge:** transposition-trial  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{7R4N5P051N6_15_3XP3N51V3_A9AFB178}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Our data got corrupted on the way here. Luckily, nothing got replaced, but every block of 3 got scrambled around! The first word seems to be three letters long, maybe you can use that to recover the rest of the message.
>
> Download the corrupted message [here].

## Hints

> 1. Split the message up into blocks of 3 and see how the first block is scrambled

---

## Background Knowledge (Read This First!)

### What is a Block Transposition Cipher?

A transposition cipher doesn't change the actual letters — it just shuffles their positions. A "block" transposition specifically chops the text into fixed-size chunks and permutes the letters inside each chunk the same way.

A simple example with block size 3 and permutation `[2, 0, 1]` (move last letter to first, shift others right by one):

```
Plaintext:  THE FLAG
Block 1:    T H E  ->  E T H
Block 2:    F L A  ->  A F L
Scrambled:  ETH AFL
```

Every block goes through the exact same shuffle. Because the shuffle is consistent, if you crack one block, you've cracked them all.

### How to Crack It

Two pieces of information you almost always get for free:

1. **Block size** — given in the challenge description ("blocks of 3")
2. **At least one known block** — here it's the first word ("three letters long")

Guess the plaintext of the first block, compare it to the scrambled version, derive the permutation, and replay it across the whole message.

### Why This Isn't Real Encryption

Just like Caesar and Vigenere, block transposition is fun to study but offers zero security. Anyone with the block size and one guessed block can rebuild the entire message. Modern ciphers (AES, ChaCha20) work at the bit level and are designed to make even partial leakage useless.

---

## Solution — Step by Step

### Step 1 — Grab the Corrupted File

Download `message.txt` from the challenge page into my Kali shared downloads folder.

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Read the Corrupted Message

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat message.txt
heTfl g as iicpCTo{7F4NRP051N5_16_35P3X51N3_V9AAB1F8}7
```

I can already tell something is weird — the first three letters `heT` look like a scrambled version of `The`. The rest is a hot mess. 54 characters total, and 54 / 3 = 18 blocks of 3.

### Step 3 — Split Into Blocks of 3 and Derive the Permutation

Mentally slicing the message into 3-character chunks:

```
heT | fl  | g a | s i | icp | CTo | {7F | 4NR | P05 | 1N5 | _16 | _35 | P3X | 51N | 3_V | 9AA | B1F | 8}7
```

The challenge says the first word is 3 letters long, and `heT` strongly suggests `The`. So:

```
Scrambled:  h  e  T
Plaintext:  T  h  e
Position:   0  1  2
```

Mapping each plaintext position to its scrambled position:

- Plaintext `T` (position 0) lives at scrambled position **2**
- Plaintext `h` (position 1) lives at scrambled position **0**
- Plaintext `e` (position 2) lives at scrambled position **1**

So the permutation is `[2, 0, 1]` — meaning: to rebuild a block, take chars at positions 2, 0, 1 in that order.

### Step 4 — Write a Python Decryption Script

I'll loop through every block and apply the same `[2, 0, 1]` rearrangement.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano transposition_solve.py
```

I pasted this into nano:

```python
def unscramble_block(block, perm):
    """Reorder characters in a block using the given permutation."""
    return ''.join(block[i] for i in perm)


def unscramble_message(message, block_size=3, perm=(2, 0, 1)):
    blocks = [message[i:i + block_size] for i in range(0, len(message), block_size)]
    return ''.join(unscramble_block(b, perm) for b in blocks)


cipher = "heTfl g as iicpCTo{7F4NRP051N5_16_35P3X51N3_V9AAB1F8}7"

print("Corrupted  :", cipher)
print("Unscrambled:", unscramble_message(cipher))
```

Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`.

### Step 5 — Run the Script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 transposition_solve.py
Corrupted  : heTfl g as iicpCTo{7F4NRP051N5_16_35P3X51N3_V9AAB1F8}7
Unscrambled: The flag is picoCTF{7R4N5P051N6_15_3XP3N51V3_A9AFB178}
```

Got the flag. The sentence reads "The flag is picoCTF{...}" — exactly what we expected.

---

## What Happened Internally

Here's a per-block walkthrough of the unscrambling, so you can see the pattern:

| Block # | Scrambled (in) | Perm `[2,0,1]` | Unscrambled (out) |
|---------|----------------|----------------|-------------------|
| 0 | `heT` | T, h, e | `The` |
| 1 | `fl ` | (space), f, l | ` fl` |
| 2 | `g a` | a, g, (space) | `ag ` |
| 3 | `s i` | i, s, (space) | `is ` |
| 4 | `icp` | p, i, c | `pic` |
| 5 | `CTo` | o, C, T | `oCT` |
| 6 | `{7F` | F, {, 7 | `F{7` |
| 7 | `4NR` | R, 4, N | `R4N` |
| 8 | `P05` | 5, P, 0 | `5P0` |
| 9 | `1N5` | 5, 1, N | `51N` |
| 10 | `_16` | 6, _, 1 | `6_1` |
| 11 | `_35` | 5, _, 3 | `5_3` |
| 12 | `P3X` | X, P, 3 | `XP3` |
| 13 | `51N` | N, 5, 1 | `N51` |
| 14 | `3_V` | V, 3, _ | `V3_` |
| 15 | `9AA` | A, 9, A | `A9A` |
| 16 | `B1F` | F, B, 1 | `FB1` |
| 17 | `8}7` | 7, 8, } | `78}` |

Concatenated: `The flag is picoCTF{7R4N5P051N6_15_3XP3N51V3_A9AFB178}`

A nice sanity check: the `oCT` from block 5 plus the `F{7` from block 6 lines up perfectly to spell `oCTF{7` — the start of the flag we expect.

---

## Alternative Methods

### Method 1 — CyberChef (No Code)

1. Go to https://gchq.github.io/CyberChef/
2. Paste the corrupted message into the input box
3. Add the **Rail Fence Cipher** decode operation is *not* what we want — instead use a custom recipe with a **"Permutation / Transpose"** block
4. Set the block size to 3 and provide the permutation `[2, 0, 1]`
5. Read the flag from the output

### Method 2 — Quick Python One-Liner

For a fast solve without creating a file:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
msg = 'heTfl g as iicpCTo{7F4NRP051N5_16_35P3X51N3_V9AAB1F8}7'
perm = (2, 0, 1)
out = ''.join(''.join(msg[j+i] for j in perm) for i in range(0, len(msg), 3))
print(out)
"
The flag is picoCTF{7R4N5P051N6_15_3XP3N51V3_A9AFB178}
```

### Method 3 — Manual Unscrambling (Pen and Paper)

For a 3-letter block the permutation `[2, 0, 1]` is small enough to do by hand:

1. Draw a 3-column grid: positions 0, 1, 2
2. For each block, write the scrambled letters in positions 0, 1, 2
3. Read them out in order 2, 0, 1
4. Move to the next block

Tedious for 18 blocks, but a great way to understand what's happening under the hood.

### Method 4 — `tr` + Awk Pipeline

If you want to stay in pure shell:

```bash
echo "heTfl g as iicpCTo{7F4NRP051N5_16_35P3X51N3_V9AAB1F8}7" \
  | fold -w3 \
  | awk '{print substr($0,3,1) substr($0,1,1) substr($0,2,1)}' \
  | tr -d '\n'
echo
```

`fold -w3` splits into 3-char lines, `awk` rearranges each line per the permutation, and `tr -d '\n'` strips the newlines back out.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the corrupted message |
| `nano` | Write the Python unscrambling script |
| `python3` | Run the block-by-block permutation |
| `fold` + `awk` (alt) | Pure-shell alternative without writing a script file |
| CyberChef (alt) | GUI-based permutation decoding |

---

## Key Takeaways

- A **block transposition cipher** preserves all characters — it just rearranges them within fixed-size blocks
- Knowing the block size and a single guessed block is enough to recover the entire permutation
- The trick of the challenge: the first word `The` tells you exactly which 3-letter permutation was used
- The permutation `[2, 0, 1]` is equivalent to a right-rotation by 1 of each block (last char moves to the front)
- Python's list slicing and string joining make permutation decryption a one-liner
- Always look for known-plaintext hints in challenge descriptions — they're the most common shortcut in classical ciphers

### Flag Wordplay Decode

The flag `picoCTF{7R4N5P051N6_15_3XP3N51V3_A9AFB178}` is a leet-speak warning from picoCTF:

- `7R4N5P051N6` -> **TRANSPOSITION** (the cipher we just used)
- `15` -> **IS**
- `3XP3N51V3` -> **EXPENSIVE**

Full sentence: **"Transposition is expensive"** — meaning: block transposition takes a lot of resources for an attacker to break, but it's also a lot of overhead and complexity for the defender. A friendly reminder that the right tool matters when picking a cipher, and classical ciphers belong in classrooms, not in production. The trailing `A9AFB178` is just the per-user hash so every solver gets a unique flag.
