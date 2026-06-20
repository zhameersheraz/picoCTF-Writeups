# PW Crack 5 — picoCTF Writeup

**Challenge:** PW Crack 5  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{h45h_sl1ng1ng_fffcda23}`  
**Platform:** picoCTF 2022 (PicoMini 2022)  
**Writeup by:** zham  

---

## Description

> Can you crack the password to get the flag?
> Download the password checker [here] and you'll need the encrypted [flag] and the [hash] in the same directory too. Here's a [dictionary] with all possible passwords based on the password conventions we've seen so far.

**Hint 1:** `Opening a file in Python is crucial to using the provided dictionary.`

**Hint 2:** `You may need to trim the whitespace from the dictionary word before hashing. Look up the Python string function, strip`

**Hint 3:** `The str_xor function does not need to be reverse engineered for this challenge.`

---

## Background Knowledge (Read This First!)

### Why we can't just "decrypt" an MD5 hash

MD5 is a **one-way hash function** — it's designed so that turning a password into its hash is easy, but there's no mathematical way to go backwards from a hash to the original password. The only way to "crack" a hash like this is to guess candidate passwords, hash each guess the exact same way, and see if any of them produce an identical hash. That's a **dictionary attack** — and it's exactly what the challenge hands us the tools for: a 65,536-line dictionary file containing every 4-character hex string from `0000` to `ffff`.

### Why hint 2 (`.strip()`) actually matters

When Python reads a text file line by line, each line still has its trailing newline character (`\n`) attached. If you hash `"eee0\n"` instead of `"eee0"`, you get a completely different MD5 digest — hashing is byte-for-byte sensitive, so a single invisible character ruins the match. `.strip()` removes that leading/trailing whitespace before hashing, so the dictionary word matches exactly what was used to generate the real password's hash.

### The script's own joke: "THIS FUNCTION WILL NOT HELP YOU FIND THE FLAG"

`level5.py` includes `str_xor()`, the same key-stretching XOR function from the Serpentine challenge, with a comment from the author claiming it won't help. It's lying for fun — `str_xor` is exactly what turns the encrypted flag bytes back into plaintext, once we have the real password as the XOR key. Hint 3 confirms this: you don't need to understand the bit-level XOR math, just trust that calling it with the correct password works, because XOR undoes itself when run twice with the same key.

### Why the script as-given can't brute-force anything

`level5.py` only ever asks for **one** password via `input()`, checks it once, and quits. To try all 65,536 dictionary entries, we need to either rewrite that one check into a loop ourselves, or write a small standalone script that does the same job.

---

## Solution — Step by Step

### Step 1 — Get all four files in one folder

Place `level5.py`, `level5.hash.bin`, `level5.flag.txt.enc`, and `dictionary.txt` together in the same directory — the script expects to find the hash and encrypted flag files sitting right next to it.

### Step 2 — Read the script with nano

```
┌──(zham㉿kali)-[~/pwcrack5]
└─$ nano level5.py
```

Key things to note:
- `correct_pw_hash = open('level5.hash.bin', 'rb').read()` — a raw 16-byte MD5 digest, not text
- `hash_pw(pw_str)` — MD5-hashes whatever string it's given
- `str_xor(flag_enc.decode(), user_pw)` — decrypts the flag using the password as the XOR key, only runs if the hash matches

Exit with `Ctrl+X`.

### Step 3 — Write a script that tries every dictionary word

```
┌──(zham㉿kali)-[~/pwcrack5]
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

flag_enc = open('level5.flag.txt.enc', 'rb').read()
correct_pw_hash = open('level5.hash.bin', 'rb').read()

def hash_pw(pw_str):
    m = hashlib.md5()
    m.update(pw_str.encode())
    return m.digest()

with open('dictionary.txt', 'r') as f:
    for line in f:
        word = line.strip()
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
┌──(zham㉿kali)-[~/pwcrack5]
└─$ python3 crack.py
FOUND PASSWORD: eee0
picoCTF{h45h_sl1ng1ng_fffcda23}
```

Out of 65,536 candidate passwords, `eee0` is the one whose MD5 hash matches — and using it as the XOR key cleanly decrypts the flag.

---

## Alternative Method — Patch the original script instead

Rather than writing a separate cracker, you can edit `level5.py` directly so it loops through the dictionary instead of asking for one manual `input()`:

```
┌──(zham㉿kali)-[~/pwcrack5]
└─$ nano level5.py
```

Replace this:

```python
def level_5_pw_check():
    user_pw = input("Please enter correct password for flag: ")
    user_pw_hash = hash_pw(user_pw)

    if( user_pw_hash == correct_pw_hash ):
        print("Welcome back... your flag, user:")
        decryption = str_xor(flag_enc.decode(), user_pw)
        print(decryption)
        return
    print("That password is incorrect")
```

With this:

```python
def level_5_pw_check():
    with open('dictionary.txt', 'r') as f:
        for line in f:
            user_pw = line.strip()
            user_pw_hash = hash_pw(user_pw)
            if( user_pw_hash == correct_pw_hash ):
                print("Welcome back... your flag, user:")
                decryption = str_xor(flag_enc.decode(), user_pw)
                print(decryption)
                return
    print("That password is incorrect")
```

Save and run it exactly as before:

```
┌──(zham㉿kali)-[~/pwcrack5]
└─$ python3 level5.py
Welcome back... your flag, user:
picoCTF{h45h_sl1ng1ng_fffcda23}
```

Same result — this version reuses every other function in the original script untouched, only the one input-handling function changes.

---

## What Happened Internally

```
Timeline:
1. level5.py needs a password whose MD5 hash matches the 16 bytes in level5.hash.bin
2. dictionary.txt holds all 65,536 four-character hex strings (0000 - ffff) as candidates
3. .strip() removes the trailing newline each dictionary line carries, so hashes line up exactly
4. hash_pw() runs MD5 on each candidate, comparing against correct_pw_hash byte-for-byte
5. Match found at "eee0" — confirmed correct because the hashes are identical
6. str_xor(flag_enc.decode(), "eee0") — XOR with the correct key reverses the encryption
7. Flag printed: picoCTF{h45h_sl1ng1ng_fffcda23}
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `nano` | Read the original script, or patch it directly | Easy |
| `hashlib.md5()` | Hash each dictionary candidate the same way the challenge does | Easy |
| `.strip()` | Remove trailing newlines from dictionary lines before hashing | Easy |
| Dictionary attack (`for line in file`) | Try every candidate password against the target hash | Medium |
| `str_xor()` (read, not reverse-engineered) | Decrypt the flag once the correct password/key is found | Easy |

---

## Key Takeaways

- **Hashes are one-way — cracking always means guessing, not decrypting** — a dictionary attack only works because hashing is deterministic: the same input always produces the same hash, so you can test candidates and compare
- **Whitespace is invisible but not weightless to a hash function** — a single trailing `\n` left over from reading a text file will silently break every comparison, even though the word "looks" identical on screen
- **A function the author tells you to ignore can still be the key to the whole challenge** — `str_xor`'s comment was a joke, not a warning; always verify a claim like that against what the code actually does
- **When a script only checks one password at a time, that's a sign you need to either loop it yourself or write your own driver around its functions** — reusing the script's own logic (hash_pw, str_xor) is faster and safer than reimplementing it from scratch
- The flag `h45h_sl1ng1ng` reads as "hash slinging" — a nod to the "Hash-Slinging Slasher," and a fitting title for a challenge that's entirely about throwing thousands of hash attempts at a target until one sticks
