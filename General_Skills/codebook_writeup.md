# Codebook — picoCTF Writeup

**Challenge:** Codebook  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{c0d3b00k_455157_7d102d7a}`  

---

## Description

> Run the Python script `code.py` in the same directory as `codebook.txt`.
> Download `code.py` and `codebook.txt`

**Hint 1:** `On the webshell, use ls to see if both files are in the directory you are in`

**Tags:** `shell`, `Python`

---

## Background Knowledge (Read This First!)

### What is a codebook?

A **codebook** is a lookup table used to encode or decode messages. In this challenge, `codebook.txt` contains a string of characters, and the script uses specific **index positions** from that string to construct a password.

### What is string indexing?

In Python, every character in a string has a numbered position starting from 0:

```
a z b y c x d w e v f u g t h s i r j q k p l o m n
0 1 2 3 4 5 6 7 8 9 ...
```

So `codebook[4]` gives the character at position 4 — which is `c`.

### How is the password built?

The script constructs the password like this:

```python
password = codebook[4] + codebook[14] + codebook[13] + codebook[14] +\
           codebook[23]+ codebook[25] + codebook[16] + codebook[0]  +\
           codebook[25]
```

Using `codebook.txt` = `azbycxdwevfugthsirjqkplomn`:

| Index | Character |
|-------|-----------|
| `[4]` | `c` |
| `[14]` | `h` |
| `[13]` | `t` |
| `[14]` | `h` |
| `[23]` | `o` |
| `[25]` | `n` |
| `[16]` | `i` |
| `[0]` | `a` |
| `[25]` | `n` |

Password = **`chthonian`** (meaning "relating to the underworld" — a fitting CTF word!)

### ⚠️ Important Note

Both `code.py` and `codebook.txt` **must be in the same directory** when you run the script. The script opens `codebook.txt` by filename — if it's in a different folder, you'll get a `FileNotFoundError`.

---

## Solution — Step by Step

### Step 1 — Make sure both files are in the same directory

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ls
code.py  codebook.txt
```

Both files are present in `/media/sf_downloads`. ✅

### Step 2 — Run the script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 code.py
picoCTF{c0d3b00k_455157_7d102d7a}
```

✅ Got the flag! 🎯

---

## What Happened Under the Hood

The script read `codebook.txt`:
```
azbycxdwevfugthsirjqkplomn
```

Then assembled the password by picking characters at specific indices:
```
[4]→c  [14]→h  [13]→t  [14]→h  [23]→o  [25]→n  [16]→i  [0]→a  [25]→n
= chthonian
```

Then used `chthonian` as the XOR key to decrypt `flag_enc` — revealing the flag.

---

## Alternative Method — Solve without running the script

You can extract the password and decrypt the flag entirely in Python without needing to run `code.py`:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
def str_xor(secret, key):
    new_key = key
    i = 0
    while len(new_key) < len(secret):
        new_key = new_key + key[i]
        i = (i + 1) % len(key)
    return ''.join([chr(ord(s) ^ ord(k)) for s,k in zip(secret,new_key)])

flag_enc = chr(0x13) + chr(0x01) + chr(0x17) + chr(0x07) + chr(0x2c) + chr(0x3a) + chr(0x2f) + chr(0x1a) + chr(0x0d) + chr(0x53) + chr(0x0c) + chr(0x47) + chr(0x0a) + chr(0x5f) + chr(0x5e) + chr(0x02) + chr(0x3e) + chr(0x5a) + chr(0x56) + chr(0x5d) + chr(0x45) + chr(0x5d) + chr(0x58) + chr(0x31) + chr(0x5e) + chr(0x05) + chr(0x5f) + chr(0x53) + chr(0x5a) + chr(0x10) + chr(0x5f) + chr(0x0e) + chr(0x13)
codebook = open('codebook.txt').read()
password = codebook[4]+codebook[14]+codebook[13]+codebook[14]+codebook[23]+codebook[25]+codebook[16]+codebook[0]+codebook[25]
print(str_xor(flag_enc, password))
"
picoCTF{c0d3b00k_455157_7d102d7a}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `ls` | Verify both files are in the same directory | ⭐ Easy |
| `python3 code.py` | Run the script to get the flag | ⭐ Easy |
| Reading source code | Understand how the password is constructed | ⭐ Easy |

---

## Key Takeaways

- **Both files must be in the same directory** — when a script opens a file by name (not path), it looks in the current working directory only
- **String indexing starts at 0** — `codebook[4]` is the 5th character, not the 4th
- **Reading the source code reveals the password construction** — understanding what the script does means you can solve it even without running it
- The flag `c0d3b00k_455157` → "codebook assist" — the codebook file was the key to unlocking the flag
- The password `chthonian` means "relating to the underworld in Greek mythology" — a classic CTF flavour word
