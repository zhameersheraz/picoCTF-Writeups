# Serpentine — picoCTF Writeup

**Challenge:** Serpentine  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 50  
**Flag:** `picoCTF{7h3_r04d_l355_7r4v3l3d_aa2340b2}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Find the flag in the Python script!
> [Download Python script]

**Hint 1:** `Try running the script and see what happens`

**Hint 2:** `In the webshell, try examining the script with a text editor like nano`

**Hint 3:** `To exit nano, press Ctrl and x and follow the on-screen prompts.`

**Hint 4:** `The str_xor function does not need to be reverse engineered for this challenge.`

---

## Background Knowledge (Read This First!)

### What is XOR, and why does it matter here?

**XOR (exclusive or)** is a bitwise operation: comparing two bits, the result is `1` if they're different and `0` if they're the same. The property that makes XOR useful for simple encryption is that it's its own inverse — XOR-ing the same value with the same key twice always gets you back to where you started: `data XOR key XOR key == data`. That means the exact same function used to *encrypt* something can be reused, unchanged, to *decrypt* it — run it again with the same key.

### What `str_xor` in this script actually does

```python
def str_xor(secret, key):
    new_key = key
    i = 0
    while len(new_key) < len(secret):
        new_key = new_key + key[i]
        i = (i + 1) % len(key)
    return "".join([chr(ord(secret_c) ^ ord(new_key_c)) for (secret_c, new_key_c) in zip(secret, new_key)])
```

The key (`'enkidu'`, 6 characters) is shorter than the encrypted flag (40 characters), so the first loop just repeats the key over and over, cycling character by character, until it's stretched long enough to line up one-to-one with every character of the secret. The second part XORs each character of the secret against the matching stretched-key character.

### Why hint 4 matters

You don't need to understand *why* this XOR-and-key-stretching trick works at the bit level — you just need to recognize that the exact same function, called with the exact same key, undoes itself. The script already hands you a perfectly working decryption function; the only thing missing is someone actually calling it.

### The trap in the menu

The script's `print_flag()` function is real, complete, and correct — it's sitting right there in the source. But option `b` in the menu doesn't call it. It just prints a fake apology message claiming the function is "misplaced." It isn't missing — it's just never invoked.

---

## Solution — Step by Step

### Step 1 — Run the script and try the "Print flag" option

```
┌──(zham㉿kali)-[~]
└─$ python3 serpentine.py
...
a) Print encouragement
b) Print flag
c) Quit

What would you like to do? (a/b/c) b

Oops! I must have misplaced the print_flag function! Check my source code!
```

The script is telling us directly to go read the source.

### Step 2 — Open the source with nano

```
┌──(zham㉿kali)-[~]
└─$ nano serpentine.py
```

Reading through it reveals three things we need:
- `flag_enc` — a long string built from individual `chr(0x..)` hex byte values
- the key: `'enkidu'`
- the working `str_xor(secret, key)` function, and a `print_flag()` function that calls `str_xor(flag_enc, 'enkidu')` — but nothing in `main()` ever calls `print_flag()`

Exit without changes for now: `Ctrl+X`.

### Step 3 — Edit the script to actually call `print_flag()`

```
┌──(zham㉿kali)-[~]
└─$ nano serpentine.py
```

Scroll to the very bottom and find:

```python
if __name__ == "__main__":
  main()
```

Replace `main()` with `print_flag()`:

```python
if __name__ == "__main__":
  print_flag()
```

Save with `Ctrl+O` → `Enter`, then exit with `Ctrl+X`.

### Step 4 — Run the edited script

```
┌──(zham㉿kali)-[~]
└─$ python3 serpentine.py
picoCTF{7h3_r04d_l355_7r4v3l3d_aa2340b2}
```

Flag decoded.

---

## Alternative Method — Decode without touching the original file

If you'd rather not edit the challenge file at all, copy just the three pieces you need into a new script:

```
┌──(zham㉿kali)-[~]
└─$ nano decode.py
```

```python
def str_xor(secret, key):
    new_key = key
    i = 0
    while len(new_key) < len(secret):
        new_key = new_key + key[i]
        i = (i + 1) % len(key)
    return "".join([chr(ord(secret_c) ^ ord(new_key_c)) for (secret_c, new_key_c) in zip(secret, new_key)])

flag_enc = chr(0x15) + chr(0x07) + chr(0x08) + chr(0x06) + chr(0x27) + chr(0x21) + chr(0x23) + chr(0x15) + chr(0x5c) + chr(0x01) + chr(0x57) + chr(0x2a) + chr(0x17) + chr(0x5e) + chr(0x5f) + chr(0x0d) + chr(0x3b) + chr(0x19) + chr(0x56) + chr(0x5b) + chr(0x5e) + chr(0x36) + chr(0x53) + chr(0x07) + chr(0x51) + chr(0x18) + chr(0x58) + chr(0x05) + chr(0x57) + chr(0x11) + chr(0x3a) + chr(0x0f) + chr(0x0a) + chr(0x5b) + chr(0x57) + chr(0x41) + chr(0x55) + chr(0x0c) + chr(0x59) + chr(0x14)

print(str_xor(flag_enc, 'enkidu'))
```

Save with `Ctrl+O` → `Enter` → `Ctrl+X`, then:

```
┌──(zham㉿kali)-[~]
└─$ python3 decode.py
picoCTF{7h3_r04d_l355_7r4v3l3d_aa2340b2}
```

Same result, zero changes to the original challenge file.

---

## What Happened Internally

```
Timeline:
1. Run serpentine.py normally, select 'b' — script lies, claims print_flag is "misplaced"
2. nano serpentine.py — reveals flag_enc, key 'enkidu', and a fully working print_flag()
   function that's simply never called anywhere in main()
3. Edit the if __name__ == "__main__": block to call print_flag() instead of main()
4. Re-run the script — str_xor(flag_enc, 'enkidu') runs exactly as written, no
   reverse-engineering needed (XOR with the same key undoes itself)
5. Flag printed directly to stdout
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `python3` | Run the script, both before and after editing | Easy |
| `nano` | Read the source, then edit the entry point | Easy |
| XOR self-inverse property | Lets the script's own encryption function double as its decryption function | Medium |

---

## Key Takeaways

- **XOR is its own inverse** — the same function and key that encrypted something will decrypt it again if you just call it a second time; you never had to understand the bit math, only recognize this property
- **A menu lying to you isn't the same as a function being missing** — always check the actual source before trusting what a program's interface tells you
- **You don't always need to write new logic** — sometimes the entire solve is just calling a function that's already sitting there, fully correct, unused
- **`nano`'s `Ctrl+O` (save) / `Ctrl+X` (exit) is often the fastest way to patch a script in place** during a CTF, far quicker than rewriting it from scratch
- The flag `7h3_r04d_l355_7r4v3l3d` decodes to "the road less traveled" — fitting, since the menu's "intended" path (option `b`) is a dead end, and the actual flag only shows up once you step off that path and go straight to the source
