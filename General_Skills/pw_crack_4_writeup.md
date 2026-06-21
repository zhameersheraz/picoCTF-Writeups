# PW Crack 4 — picoCTF Writeup

**Challenge:** PW Crack 4  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 85  
**Flag:** `picoCTF{fl45h_5pr1ng1ng_ae0fb77c}`  
**Platform:** picoCTF 2022 (PicoMini 2022)  
**Writeup by:** zham  

---

## Description

> Can you crack the password to get the flag?
> Download the password checker [here] and you'll need the encrypted [flag] and the [hash] in the same directory too.
> There are 100 potential passwords with only 1 being correct. You can find these by examining the password checker script.

**Hint 1:** `A for loop can help you do many things very quickly.`

**Hint 2:** `The str_xor function does not need to be reverse engineered for this challenge.`

---

## Background Knowledge (Read This First!)

### How this one differs from PW Crack 5

This is the same family of challenge as PW Crack 5 — an MD5 hash to match, an XOR-encrypted flag to decrypt with whichever password matches that hash — but this time there's no separate dictionary file to download. Instead, `level4.py` ships with the candidate list built right into the script itself, sitting in a Python list called `pos_pw_list`, right at the bottom of the file:

```python
pos_pw_list = ["6288", "6152", "4c7a", ... , "d964", "49ec"]
```

100 candidates instead of 65,536 — small enough that brute-forcing all of them takes a fraction of a second.

### Why a `for` loop is the entire challenge

Hint 1 is the whole solve in one sentence. The script's `level_4_pw_check()` function only ever tests **one** password — whatever a human types into `input()`. We don't need to guess which of the 100 strings is correct; we just need to test all 100 automatically and let the MD5 comparison tell us which one matches.

### Why we still don't need to understand `str_xor`

Same as PW Crack 5: XOR is its own inverse. Once we know the correct password (the one whose MD5 hash matches `level4.hash.bin`), feeding it into the exact same `str_xor()` function the script already provides reverses the encryption on the flag — no need to study the bit-level logic, just reuse the function as-is.

---

## Solution — Step by Step

### Step 1 — Get all three files in one folder

Place `level4.py`, `level4.hash.bin`, and `level4.flag.txt.enc` together in the same directory.

### Step 2 — Read the script with nano

```
┌──(zham㉿kali)-[~/pwcrack4]
└─$ nano level4.py
```

Scrolling to the bottom reveals the full `pos_pw_list` of 100 candidate passwords, sitting right there in plain text — already provided, no guessing or generation needed.

Exit with `Ctrl+X`.

### Step 3 — Write a script that loops through all 100 candidates

```
┌──(zham㉿kali)-[~/pwcrack4]
└─$ nano crack.py
```

```python
import hashlib

def str_xor(secret, key):
    new_key = key
    i = 0
    while len(new_key) < len(secret):
        new_key = new_key + key[i]
        i = (i + 1) % len(key)
    return "".join([chr(ord(secret_c) ^ ord(new_key_c)) for (secret_c, new_key_c) in zip(secret, new_key)])

flag_enc = open('level4.flag.txt.enc', 'rb').read()
correct_pw_hash = open('level4.hash.bin', 'rb').read()

def hash_pw(pw_str):
    m = hashlib.md5()
    m.update(pw_str.encode())
    return m.digest()

pos_pw_list = ["6288", "6152", "4c7a", "b722", "9a6e", "6717", "4389", "1a28", "37ac", "de4f", "eb28", "351b", "3d58", "948b", "231b", "973a", "a087", "384a", "6d3c", "9065", "725c", "fd60", "4d4f", "6a60", "7213", "93e6", "8c54", "537d", "a1da", "c718", "9de8", "ebe3", "f1c5", "a0bf", "ccab", "4938", "8f97", "3327", "8029", "41f2", "a04f", "c7f9", "b453", "90a5", "25dc", "26b0", "cb42", "de89", "2451", "1dd3", "7f2c", "8919", "f3a9", "b88f", "eaa8", "776a", "6236", "98f5", "492b", "507d", "18e8", "cfb5", "76fd", "6017", "30de", "bbae", "354e", "4013", "3153", "e9cc", "cba9", "25ea", "c06c", "a166", "faf1", "2264", "2179", "cf30", "4b47", "3446", "b213", "88a3", "6253", "db88", "c38c", "a48c", "3e4f", "7208", "9dcb", "fc77", "e2cf", "8552", "f6f8", "7079", "42ef", "391e", "8a6d", "2154", "d964", "49ec"]

for word in pos_pw_list:
    if hash_pw(word) == correct_pw_hash:
        print("FOUND PASSWORD:", word)
        print(str_xor(flag_enc.decode(), word))
        break
else:
    print("No match found")
```

Save with `Ctrl+O` → `Enter` → `Ctrl+X`.

### Step 4 — Run it

```
┌──(zham㉿kali)-[~/pwcrack4]
└─$ python3 crack.py
FOUND PASSWORD: 973a
picoCTF{fl45h_5pr1ng1ng_ae0fb77c}
```

Found on the first pass through all 100 candidates.

---

## Alternative Method — Patch the original script's `for` loop in directly

Same idea as PW Crack 5: instead of writing a new file, replace the single `input()` check inside `level4.py` with a loop over `pos_pw_list`:

```
┌──(zham㉿kali)-[~/pwcrack4]
└─$ nano level4.py
```

Replace:

```python
def level_4_pw_check():
    user_pw = input("Please enter correct password for flag: ")
    user_pw_hash = hash_pw(user_pw)

    if( user_pw_hash == correct_pw_hash ):
        print("Welcome back... your flag, user:")
        decryption = str_xor(flag_enc.decode(), user_pw)
        print(decryption)
        return
    print("That password is incorrect")
```

With:

```python
def level_4_pw_check():
    for pw in pos_pw_list:
        pw_hash = hash_pw(pw)
        if( pw_hash == correct_pw_hash ):
            print("Welcome back... your flag, user:")
            decryption = str_xor(flag_enc.decode(), pw)
            print(decryption)
            return
    print("No password in the list matched")
```

Save and run:

```
┌──(zham㉿kali)-[~/pwcrack4]
└─$ python3 level4.py
Welcome back... your flag, user:
picoCTF{fl45h_5pr1ng1ng_ae0fb77c}
```

Same flag, no separate script needed.

---

## What Happened Internally

```
Timeline:
1. level4.py ships with 100 candidate passwords baked directly into pos_pw_list
2. Each candidate gets MD5-hashed the same way hash_pw() does it in the original script
3. Compare each resulting hash against the 16 raw bytes in level4.hash.bin
4. Match found at "973a" — the 16th entry in the list
5. str_xor(flag_enc.decode(), "973a") reverses the XOR encryption using the now-known key
6. Flag printed: picoCTF{fl45h_5pr1ng1ng_ae0fb77c}
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `nano` | Read the script, or patch its password-check loop directly | Easy |
| `hashlib.md5()` | Hash each of the 100 candidates the same way the challenge does | Easy |
| `for` loop | Test all 100 candidates automatically instead of one at a time | Easy |
| `str_xor()` (read, not reverse-engineered) | Decrypt the flag once the correct password is identified | Easy |

---

## Key Takeaways

- **A small, fixed candidate list embedded directly in the source is still a dictionary attack — just a much smaller one.** The exact same hash-and-compare technique from PW Crack 5 applies here with zero modification, just a shorter list
- **Hint phrasing is often a direct instruction in disguise** — "a for loop can help you do many things very quickly" isn't a vague tip, it's telling you the entire shape of the solution
- **Reusing a script's own existing functions (`hash_pw`, `str_xor`) is almost always faster than reimplementing the logic from scratch** — they're already correct, and the challenge wants you to call them differently, not rewrite them
- **The same root technique often carries across an entire challenge series** — PW Crack 3, 4, and 5 are all solved with the same hash-compare-then-XOR-decrypt pattern, just scaled to different list sizes
- The flag `fl45h_5pr1ng1ng` reads as "flash springing" — a playful nod to how quickly a brute-force loop chews through a hundred (or tens of thousands of) candidates compared to guessing by hand
