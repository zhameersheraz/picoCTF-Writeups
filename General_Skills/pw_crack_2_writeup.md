# PW Crack 2 — picoCTF Writeup

**Challenge:** PW Crack 2  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{tr45h_51ng1ng_502ec42e}`  

---

## Description

> Can you crack the password to get the flag?
> Download the password checker here and you'll need the encrypted flag in the same directory too.

**Downloads:** `level2.py`, `level2.flag.txt.enc`  
**Hint 1:** `Does that encoding look familiar?`

---

## Background Knowledge (Read This First!)

### What is hardcoded password?

A **hardcoded password** is a password that is written directly into the source code of a program — instead of being stored securely or checked against a database. This is a major security mistake because anyone who can read the source code can immediately find the password.

In this challenge, the password is hardcoded inside the `if` statement of `level_2_pw_check()`.

### What is `chr()` and hex encoding?

`chr()` is a Python function that converts a number into its corresponding ASCII character. For example:
- `chr(65)` → `'A'`
- `chr(0x33)` → `'3'` (0x33 is hex for 51 decimal)

The `0x` prefix means the number is in **hexadecimal**. So `chr(0x33)` just means "give me the character whose ASCII code is 0x33 (= 51 decimal)" which is the character `'3'`.

This is what Hint 1 refers to — "Does that encoding look familiar?" — the password is encoded as hex ASCII values, but it's trivial to decode.

### What is XOR encryption?

The flag is encrypted using **XOR** with the password as the key. XOR is its own inverse — if you XOR the encrypted data with the same key, you get the original back. Since the password is in the source code, we can decrypt the flag directly.

### ⚠️ Important Note

Both `level2.py` and `level2.flag.txt.enc` must be in the **same directory** when you run the script, because the script opens `level2.flag.txt.enc` by filename.

---

## Reading the Source Code

```python
def level_2_pw_check():
    user_pw = input("Please enter correct password for flag: ")
    if( user_pw == chr(0x33) + chr(0x39) + chr(0x63) + chr(0x65) ):
        print("Welcome back... your flag, user:")
        decryption = str_xor(flag_enc.decode(), user_pw)
        print(decryption)
        return
    print("That password is incorrect")
```

The password check is:
```python
user_pw == chr(0x33) + chr(0x39) + chr(0x63) + chr(0x65)
```

Decode each hex value:

| Hex | Decimal | Character |
|-----|---------|-----------|
| 0x33 | 51 | `3` |
| 0x39 | 57 | `9` |
| 0x63 | 99 | `c` |
| 0x65 | 101 | `e` |

Password = **`39ce`**

---

## Solution — Step by Step

### Step 1 — Decode the password from the source code

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "print(chr(0x33) + chr(0x39) + chr(0x63) + chr(0x65))"
39ce
```

### Step 2 — Run the script and enter the password

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 level2.py
Please enter correct password for flag: 39ce
Welcome back... your flag, user:
picoCTF{tr45h_51ng1ng_502ec42e}
```

✅ Got the flag! 🎯

---

## Alternative Method — Bypass the password check entirely

Instead of running the script interactively, you can extract the password and decrypt the flag directly in one Python command — no need to type anything:

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

flag_enc = open('level2.flag.txt.enc', 'rb').read()
pw = chr(0x33) + chr(0x39) + chr(0x63) + chr(0x65)
print(str_xor(flag_enc.decode(), pw))
"
picoCTF{tr45h_51ng1ng_502ec42e}
```

This skips the interactive prompt entirely by copying the password and decryption logic directly.

---

## Why This Is a Security Problem

The challenge demonstrates a classic **insecure coding practice**:

1. **Hardcoded password** — the password `39ce` is written directly in the source code
2. **Obfuscation ≠ security** — encoding it as `chr(0x33) + chr(0x39)...` looks confusing at first glance, but it takes seconds to decode with Python
3. **Source code is readable** — any Python script can be opened and read. Never store passwords in code

A secure program would store a **hashed** version of the password and compare hashes — never the plaintext password itself.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reading source code | Find the hardcoded password | ⭐ Easy |
| `python3 -c` | Decode hex chr() values | ⭐ Easy |
| `python3 level2.py` | Run the script with the cracked password | ⭐ Easy |

---

## Key Takeaways

- **Always read source code** before running a password-protected script — the password is often hardcoded right in the `if` statement
- **`chr(0x33)`** is just a slightly obfuscated way of writing `'3'` — hex encoding makes it look harder than it is
- **Obfuscation is not security** — hiding a password inside `chr()` calls doesn't protect it at all
- **XOR encryption is reversible with the same key** — once you have the password, decrypting the flag is trivial
- The flag `tr45h_51ng1ng` → "trash singing" — a nod to how poorly this password was hidden
