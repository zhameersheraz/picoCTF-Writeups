# vault-door-6 — picoCTF Writeup

**Challenge:** vault-door-6  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Flag:** `picoCTF{n0t_mUcH_h4rD3r_tH4n_x0r_fdd45f2}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This vault uses an XOR encryption scheme.
> The source code for this vault is here: VaultDoor6.java

**Hint:** If X ^ Y = Z, then Z ^ Y = X. Write a program that decrypts the flag based on this fact.

---

## Background Knowledge (Read This First!)

### What does `checkPassword()` do?

This is the shortest `checkPassword` in the series. The whole check is one tight loop:

```java
for (int i=0; i<32; i++) {
    if (((passBytes[i] ^ 0x55) - myBytes[i]) != 0) {
        return false;
    }
}
```

For every byte of your input it asks one question:

> *"Does `(passBytes[i] XOR 0x55) - myBytes[i]` equal 0?"*

The answer is 0 **only** when `passBytes[i] ^ 0x55 == myBytes[i]`, which is the same as saying *"the password byte XORed with `0x55` must equal the stored byte."* If every byte passes, the vault opens.

### What is XOR?

XOR (e**X**clusive **OR**, written `^`) is a bitwise operation that compares two bits and returns `1` if they differ, `0` if they're the same.

| a | b | a ^ b |
|:-:|:-:|:-:|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

Three properties matter for this challenge:

1. **Identity:** `x ^ 0 = x`
2. **Self-inverse:** `x ^ x = 0`
3. **Reversible with the same key:** if `x ^ K = y`, then `y ^ K = x`

That third property is the entire challenge. The hint gives it away:

> *"If X ^ Y = Z, then Z ^ Y = X."*

### Why "XOR encryption" isn't really encryption

Minion #3091 read the first 250 pages of *Applied Cryptography* by Bruce Schneier and decided XOR was the strongest cipher in existence. (The source comment jokes that "nothing important" lives in the last 750 pages.) XOR with a single fixed key is technically reversible by anyone who knows the key — and the key (`0x55`) is sitting right in the source. It's obfuscation, not encryption. Don't use XOR with a known constant to protect anything real.

---

## Reading the Source Code

The interesting part of `VaultDoor6.java`:

```java
public boolean checkPassword(String password) {
    if (password.length() != 32) {
        return false;
    }
    byte[] passBytes = password.getBytes();
    byte[] myBytes = {
        0x3b, 0x65, 0x21, 0x0a, 0x38, 0x00, 0x36, 0x1d,
        0x0a, 0x3d, 0x61, 0x27, 0x11, 0x66, 0x27, 0x0a,
        0x21, 0x1d, 0x61, 0x3b, 0x0a, 0x2d, 0x65, 0x27,
        0x0a, 0x33, 0x31, 0x31, 0x61, 0x60, 0x33, 0x67,
    };
    for (int i=0; i<32; i++) {
        if (((passBytes[i] ^ 0x55) - myBytes[i]) != 0) {
            return false;
        }
    }
    return true;
}
```

The check is `passBytes[i] ^ 0x55 == myBytes[i]`. Flip it around and you get `passBytes[i] == myBytes[i] ^ 0x55`. XOR each entry of `myBytes` with `0x55` and you have the password.

---

## Solution — Step by Step

### Step 1 — Derive the formula

From the hint:

```
if  X ^ Y = Z,  then  Z ^ Y = X
```

In our code:
```
passBytes[i] ^ 0x55  =  myBytes[i]
myBytes[i] ^ 0x55   =  passBytes[i]      ← what we want
```

So every password byte is just `myBytes[i] XOR 0x55`.

### Step 2 — Decode the first 4 bytes by hand

A worked example helps cement the idea. Manually XOR the first four `myBytes` entries with `0x55`:

| `myBytes[i]` | hex | binary | `^ 0x55` (binary) | `^ 0x55` (hex) | char |
|---:|---:|---|:-:|---:|:-:|
| 0x3b | 0x3b | `0011 1011` | `0110 1110` | 0x6e | `n` |
| 0x65 | 0x65 | `0110 0101` | `0011 0000` | 0x30 | `0` |
| 0x21 | 0x21 | `0010 0001` | `0111 0100` | 0x74 | `t` |
| 0x0a | 0x0a | `0000 1010` | `0101 1111` | 0x5f | `_` |

Password starts with `n0t_`. Doing all 32 by hand is tedious — let Python take it from here.

### Step 3 — Write the inverse solver

`nano solve_vd6.py`, paste this, save with `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-6]
└─$ nano solve_vd6.py
```

```python
#!/usr/bin/env python3
import re

with open("VaultDoor6.java") as f:
    src = f.read()

# Pull the byte[] myBytes = { ... }; initializer out of the source.
m = re.search(r"byte\[\]\s+myBytes\s*=\s*\{([^}]+)\}\s*;", src, re.S)
tokens = re.findall(r"0x[0-9a-fA-F]+", m.group(1))
assert len(tokens) == 32

myBytes = [int(t, 16) for t in tokens]

# passBytes[i] = myBytes[i] ^ 0x55   (XOR is its own inverse)
KEY = 0x55
password_bytes = [b ^ KEY for b in myBytes]
password = "".join(chr(b) for b in password_bytes)

print(f"[+] Recovered password: {password}")
print(f"[+] Flag              : picoCTF{{{password}}}")
```

### Step 4 — Run it

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-6]
└─$ python3 solve_vd6.py
[+] Recovered password: n0t_mUcH_h4rD3r_tH4n_x0r_fdd45f2
[+] Flag              : picoCTF{n0t_mUcH_h4rD3r_tH4n_x0r_fdd45f2}
```

### Step 5 — Verify by re-running the original check

I never trust a recovered flag blindly, so I rebuild the comparison and feed the candidate in:

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-6]
└─$ python3 verify_vd6.py
checkPassword -> Access granted.
flag: picoCTF{n0t_mUcH_h4rD3r_tH4n_x0r_fdd45f2}
```

"Access granted." matches the Java program output. The flag is good.

---

## Alternative Method — XOR in your head

Once you've done a few of these by hand you can read XOR like addition. The trick is to remember that XOR with `0x55` flips bits at positions 0, 2, 4, 6 (the even-indexed bits of every byte). A useful shortcut for the bytes that appear in this challenge:

| `myBytes[i]` | `^ 0x55` | Char | | `myBytes[i]` | `^ 0x55` | Char |
|---:|---:|:-:|---|---:|---:|:-:|
| 0x00 | 0x55 | `U` | | 0x21 | 0x74 | `t` |
| 0x0a | 0x5f | `_` | | 0x27 | 0x72 | `r` |
| 0x11 | 0x44 | `D` | | 0x31 | 0x64 | `d` |
| 0x1d | 0x48 | `H` | | 0x33 | 0x66 | `f` |
| 0x21 | 0x74 | `t` | | 0x36 | 0x63 | `c` |
| 0x21 | 0x74 | `t` | | 0x38 | 0x6d | `m` |
| 0x27 | 0x72 | `r` | | 0x3b | 0x6e | `n` |
| 0x27 | 0x72 | `r` | | 0x3d | 0x68 | `h` |
| 0x2d | 0x78 | `x` | | 0x60 | 0x35 | `5` |
| 0x31 | 0x64 | `d` | | 0x61 | 0x34 | `4` |
| 0x33 | 0x66 | `f` | | 0x65 | 0x30 | `0` |
| 0x36 | 0x63 | `c` | | 0x66 | 0x33 | `3` |
| 0x38 | 0x6d | `m` | | 0x67 | 0x32 | `2` |
| 0x3b | 0x6e | `n` | |  |  |  |
| 0x3d | 0x68 | `h` | |  |  |  |

Read the right column top-to-bottom and you get the password without writing any code at all. Same data the script spits out, just done by hand.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reading the Java source | Spot the single XOR-with-`0x55` operation | Easy |
| Mental bitwise XOR (first 4 bytes) | See what the recovered characters look like before automating | Easy |
| `python3` + `re` | Pull `myBytes` out of the source and XOR with the key | Easy |
| Hand-computed shortcut table | Decode the password with no code at all | Easy |
| Forward verification | Re-run the original check to confirm `Access granted.` | Easy |
| `javac` / `java` (optional) | Compile `VaultDoor6.java` and type the recovered password for a 1:1 Java check | Easy |

---

## Key Takeaways

- **XOR is its own inverse.** Once you know the key, recovering the plaintext is a one-line operation. The hint is literally the formula: `Z ^ Y = X`.
- **Algebra beats memorisation.** Don't try to "decode the cipher" character by character — derive `passBytes[i] = myBytes[i] ^ 0x55` once, then apply it mechanically.
- **`byte` in Java is signed.** `0xa` is `10`, not a negative number — Java just stores it as a signed `byte` for arithmetic. The XOR still works because the bits are the same.
- **"XOR encryption" is not encryption.** XOR with a fixed key is reversible by anyone who can read the source. Real encryption uses algorithms like AES where knowing the algorithm alone doesn't reveal the plaintext.
- **The hint walks you through the solve.** *"If X ^ Y = Z, then Z ^ Y = X"* is the recipe. When a challenge hints at a fact, it's because that fact IS the solve.
- **Always verify the recovered flag.** Re-running the original check catches transcription mistakes (and the `0x0a` vs `0xa` ambiguity in Java hex literals).
- **Wordplay decode.** The flag reads as a sentence once you translate the leet:

  | Token | Reads as |
  |---|---|
  | `n0t` | **not** (`0`→`o`) |
  | `mUcH` | **much** (intentionally odd capitalisation, like `cH4r4cT3r5` in vault-door-1) |
  | `h4rD3r` | **harder** (`4`→`a`, `3`→`e`) |
  | `tH4n` | **than** (`4`→`a`) |
  | `x0r` | **xor** (`0`→`o`) |
  | `fdd45f2` | a deliberate hex-looking tail; not a word, just a fingerprint |

  Decoded message: *"not much harder than xor"*. The challenge is mocking Minion #3091's hubris — XOR with a known key is so trivially reversible that "encryption" is a stretch. The hex tail `fdd45f2` is picoCTF's signature random suffix.
