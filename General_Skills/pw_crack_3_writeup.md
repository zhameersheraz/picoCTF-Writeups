# PW Crack 3 — picoCTF Writeup

**Challenge:** PW Crack 3  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 75  
**Flag:** `picoCTF{m45h_fl1ng1ng_cd6ed2eb}`  
**Platform:** picoCTF 2022 (PicoMini 2022)  
**Writeup by:** zham  

---

## Description

> Can you crack the password to get the flag?
> Download the password checker [here] and you'll need the encrypted [flag] and the [hash] in the same directory too.
> There are 7 potential passwords with 1 being correct. You can find these by examining the password checker script.

**Hint 1:** `To view the level3.hash.bin file in the webshell, do: $ bvi level3.hash.bin`

**Hint 2:** `To exit bvi type :q and press enter.`

**Hint 3:** `The str_xor function does not need to be reverse engineered for this challenge.`

---

## Background Knowledge (Read This First!)

### The smallest of the PW Crack series

This is the same family as PW Crack 4 and PW Crack 5 — an MD5 hash to match, an XOR-encrypted flag to decrypt with whatever password matches that hash — but scaled down to just 7 candidates, sitting in `level3.py` itself:

```python
pos_pw_list = ["f09e", "4dcf", "87ab", "dba8", "752e", "3961", "f159"]
```

With only 7 options, you could even test them by hand one at a time — but scripting it is just as fast and removes any chance of a typo.

### Why hint 1 mentions `bvi`

`bvi` is a **binary** version of the `vi` text editor — built for viewing and editing raw, non-text bytes safely. `level3.hash.bin` is a raw 16-byte MD5 digest, not readable text, so a normal `cat` or `nano` would just show garbage or risk corrupting it if accidentally edited. `bvi` displays the file in hex without that risk, though for this challenge we don't actually need to read the hash by eye — we let Python compare it byte-for-byte instead.

---

## Solution — Step by Step

### Step 1 — Get all three files in one folder

Place `level3.py`, `level3.hash.bin`, and `level3.flag.txt.enc` together in the same directory.

### Step 2 — Read the script with nano

```
┌──(zham㉿kali)-[~/pwcrack3]
└─$ nano level3.py
```

The `pos_pw_list` of 7 candidates sits in plain text at the bottom of the file. Exit with `Ctrl+X`.

### Step 3 — Write a script to test all 7 candidates

```
┌──(zham㉿kali)-[~/pwcrack3]
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

flag_enc = open('level3.flag.txt.enc', 'rb').read()
correct_pw_hash = open('level3.hash.bin', 'rb').read()

def hash_pw(pw_str):
    m = hashlib.md5()
    m.update(pw_str.encode())
    return m.digest()

pos_pw_list = ["f09e", "4dcf", "87ab", "dba8", "752e", "3961", "f159"]

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
┌──(zham㉿kali)-[~/pwcrack3]
└─$ python3 crack.py
FOUND PASSWORD: 87ab
picoCTF{m45h_fl1ng1ng_cd6ed2eb}
```

Found on the third candidate.

---

## Alternative Method — Skip the hash entirely with a known-plaintext attack

Every picoCTF flag starts with the same 8 characters: `picoCTF{`. Since XOR with a repeating key is what encrypted the flag, we can use that fixed, known prefix to recover the key directly — without ever touching `level3.hash.bin` or the candidate list at all:

```
┌──(zham㉿kali)-[~/pwcrack3]
└─$ nano recover_key.py
```

```python
enc = open('level3.flag.txt.enc', 'rb').read()
known = b'picoCTF{'
key_guess = bytes([a ^ b for a, b in zip(enc, known)])
print(key_guess)
```

Save and run:

```
┌──(zham㉿kali)-[~/pwcrack3]
└─$ python3 recover_key.py
b'87ab87ab'
```

The recovered bytes repeat every 4 characters — `87ab` — confirming the key length and its value in one step, purely from the structure of the encryption itself. From here, running the original script with that password produces the same flag as Step 4.

This works because XOR-ing the encrypted bytes against the plaintext we already know (the flag header) cancels out the ciphertext and leaves the repeating key exposed — no password list, no hash comparison, no guessing required.

---

## What Happened Internally

```
Timeline:
1. level3.py ships with 7 candidate passwords baked into pos_pw_list
2. Each candidate gets MD5-hashed the same way hash_pw() does it in the original script
3. Compare each resulting hash against the 16 raw bytes in level3.hash.bin
4. Match found at "87ab" — the 3rd entry in the list
5. str_xor(flag_enc.decode(), "87ab") reverses the XOR encryption using the known key
6. Flag printed: picoCTF{m45h_fl1ng1ng_cd6ed2eb}

(Alternative path: XOR-ing the ciphertext against the known "picoCTF{" prefix
recovers the same key "87ab" directly, with no list or hash check needed at all)
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `nano` | Read the script and write the cracking script | Easy |
| `hashlib.md5()` | Hash each of the 7 candidates the same way the challenge does | Easy |
| `bvi` (mentioned in hints, not strictly needed) | Safely view raw binary files like the hash without corrupting them | Easy |
| `for` loop | Test all 7 candidates automatically | Easy |
| Known-plaintext XOR recovery | Recover the key directly from the fixed `picoCTF{` flag prefix | Medium |

---

## Key Takeaways

- **A 7-item list is still a dictionary attack** — the size of the candidate list changes nothing about the technique, only how long it takes to run
- **Knowing the plaintext format of what you're decrypting is itself a weapon** — because every flag starts with `picoCTF{`, that fixed prefix alone is enough to recover an XOR key, no password list or hash file required
- **`bvi` exists for a reason** — regular text editors can silently corrupt binary files; tools built for raw bytes (`bvi`, `xxd`, `od`) are the safe way to inspect them
- **The same underlying weakness (a short, repeating XOR key) can be broken multiple ways** — brute-forcing candidates and exploiting known plaintext both work here precisely because the key never changes per byte
- The flag `m45h_fl1ng1ng` reads as "mash flinging" — continuing the same "throwing things at a target until something sticks" joke that runs through the whole PW Crack series, this time about mashing buttons (or candidates) rather than hashes
