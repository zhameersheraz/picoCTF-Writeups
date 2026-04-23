# PW Crack 1 — picoCTF Writeup

**Challenge:** PW Crack 1  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{545h_r1ng1ng_56891419}`  

---

## Description

> Can you crack the password to get the flag?
> Download the password checker here and you'll need the encrypted flag in the same directory too.

**Downloads:** `level1.py`, `level1.flag.txt.enc`  
**Hint 1:** `To view the file in the webshell, do: $ nano level1.py`

---

## Background Knowledge (Read This First!)

### What is a hardcoded password?

A **hardcoded password** is a password written directly into the source code as a plain string. This is the most basic form of insecure password storage — anyone who opens the file can immediately see it.

In PW Crack 1, the password is stored as a simple string literal `"691d"` with no obfuscation at all. Compare this to PW Crack 2 where it was slightly hidden using `chr()` hex values — here it's completely exposed.

### What is XOR encryption?

The flag file is encrypted using **XOR** with the password as the key. XOR encryption works both ways — encrypting and decrypting use the exact same operation. So if you XOR the encrypted data with the same password, you recover the original plaintext.

### ⚠️ Important Note

Both `level1.py` and `level1.flag.txt.enc` must be in the **same directory** when you run the script, because the script opens the `.enc` file by filename.

---

## Reading the Source Code

```python
def level_1_pw_check():
    user_pw = input("Please enter correct password for flag: ")
    if( user_pw == "691d"):
        print("Welcome back... your flag, user:")
        decryption = str_xor(flag_enc.decode(), user_pw)
        print(decryption)
        return
    print("That password is incorrect")
```

The password is right there: **`691d`** — no encoding, no obfuscation, just a plain string.

---

## Solution — Step by Step

### Step 1 — Read the source code to find the password

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat level1.py
```

Looking at the `if` statement:
```python
if( user_pw == "691d"):
```

Password = **`691d`**

### Step 2 — Run the script and enter the password

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 level1.py
Please enter correct password for flag: 691d
Welcome back... your flag, user:
picoCTF{545h_r1ng1ng_56891419}
```

✅ Got the flag! 🎯

---

## Alternative Method — Bypass the password check entirely

Skip the interactive prompt by decrypting directly in Python:

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

flag_enc = open('level1.flag.txt.enc', 'rb').read()
print(str_xor(flag_enc.decode(), '691d'))
"
picoCTF{545h_r1ng1ng_56891419}
```

---

## PW Crack 1 vs PW Crack 2 — How They Differ

| | PW Crack 1 | PW Crack 2 |
|--|-----------|-----------|
| Password storage | Plain string `"691d"` | Hex encoded `chr(0x33)+chr(0x39)...` |
| Difficulty to find | Immediately visible | Requires decoding hex values |
| Security | Zero | Still zero — just slightly obfuscated |
| Both are | Hardcoded passwords | Hardcoded passwords |

The lesson across both challenges is the same: **no amount of obfuscation makes a hardcoded password secure.**

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `cat level1.py` | Read the source code to find the password | ⭐ Easy |
| `python3 level1.py` | Run the script with the cracked password | ⭐ Easy |
| Python one-liner (optional) | Bypass the prompt and decrypt directly | ⭐ Easy |

---

## Key Takeaways

- **Always read the source code first** — in PW Crack 1 the password is a plain string sitting right in the `if` statement, no tricks needed
- **PW Crack 1 is even simpler than PW Crack 2** — no hex encoding to decode, the password is completely exposed
- **Hardcoded passwords are never secure** — whether plain text or hex-obfuscated, they are always recoverable from the source code
- **XOR decryption = XOR encryption** — the same key that encrypted the flag also decrypts it
- The flag `545h_r1ng1ng` → "sash ringing" — a playful continuation of the "ringing" theme from PW Crack 2's `tr45h_51ng1ng`
