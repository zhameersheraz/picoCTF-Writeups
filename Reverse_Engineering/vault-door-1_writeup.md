# vault-door-1 — picoCTF Writeup

**Challenge:** vault-door-1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Flag:** `picoCTF{d35cr4mbl3_tH3_cH4r4cT3r5_1ef266}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This vault uses some complicated arrays! I hope you can make sense of it, special agent. The source code for this vault is here: VaultDoor1.java

**Hint:** Look up the `charAt()` method online.

---

## Background Knowledge (Read This First!)

### What does the program do?

`VaultDoor1.java` reads your input, strips off the `picoCTF{` and `}` wrapper, and runs the remaining 32 characters through `checkPassword()`. That method is just 32 hard-coded checks of the form:

```java
password.charAt(i) == 'X'
```

The order of those checks is scrambled in the source. The trick is to read them carefully, sort them by index, and reassemble the password in order.

### Strings are arrays of characters

A Java `String` is really a sequence of `char` values. The first character sits at index `0`, the next at `1`, and so on — exactly like indexing into a Python list.

### What is `charAt()`?

`charAt(i)` returns the character at index `i`.

| Call | Returns |
|---|---|
| `"hello".charAt(0)` | `'h'` |
| `"hello".charAt(1)` | `'e'` |
| `"hello".charAt(4)` | `'o'` |

So when the source says

```java
password.charAt(7) == 'b'
```

it literally means: *the 8th character of `password` must be the letter `b`.*

### How a regex helps

A pattern like `charAt\(\s*(\d+)\s*\)\s*==\s*'([^'])'` grabs every `(index, character)` pair from the source in a single pass, regardless of how the lines are scrambled.

---

## Reading the Source Code

The interesting part of `VaultDoor1.java`:

```java
public boolean checkPassword(String password) {
    return password.length() == 32 &&
           password.charAt(0)  == 'd' &&
           password.charAt(29) == '2' &&
           password.charAt(4)  == 'r' &&
           password.charAt(2)  == '5' &&
           password.charAt(23) == 'r' &&
           // ... 26 more scrambled lines ...
           password.charAt(31) == '6';
}
```

The password is split across 32 separate `charAt` checks with the indices shuffled. We just need to put each character back at the right index.

---

## Solution — Step by Step

### Step 1 — Stage the source

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-1]
└─$ ls
VaultDoor1.java
```

### Step 2 — Pull out every charAt line

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-1]
└─$ grep -n "charAt" VaultDoor1.java
24:               password.charAt(0)  == 'd' &&
25:               password.charAt(29) == '2' &&
26:               password.charAt(4)  == 'r' ...
```

32 lines, each one a single clue. Order is meaningless — the index on the left is what matters.

### Step 3 — Write a tiny solver

`nano solve.py`, paste this, save with `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-1]
└─$ nano solve.py
```

```python
#!/usr/bin/env python3
import re

with open("VaultDoor1.java") as f:
    code = f.read()

# Match:  password.charAt(N) == 'X'   followed by either && or ;
pattern = re.compile(r"password\.charAt\(\s*(\d+)\s*\)\s*==\s*'([^'])'\s*(?:&&|;)")
pairs = pattern.findall(code)

assert len(pairs) == 32, f"expected 32 checks, got {len(pairs)}"

password = [""] * 32
for idx, ch in pairs:
    password[int(idx)] = ch

result = "".join(password)
print(f"[+] Recovered password : {result}")
print(f"[+] Flag               : picoCTF{{{result}}}")
```

### Step 4 — Run it

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-1]
└─$ python3 solve.py
[+] Recovered password : d35cr4mbl3_tH3_cH4r4cT3r5_1ef266
[+] Flag               : picoCTF{d35cr4mbl3_tH3_cH4r4cT3r5_1ef266}
```

### Step 5 — Verify against the original check

I never trust a recovered flag blindly, so I re-implemented `checkPassword()` in Python and fed the candidate in:

```python
# verify.py
checks = [
    (0,'d'),(29,'2'),(4,'r'),(2,'5'),(23,'r'),(3,'c'),(17,'4'),(1,'3'),
    (7,'b'),(10,'_'),(5,'4'),(9,'3'),(11,'t'),(15,'c'),(8,'l'),(12,'H'),
    (20,'c'),(14,'_'),(6,'m'),(24,'5'),(18,'r'),(13,'3'),(19,'4'),(21,'T'),
    (16,'H'),(27,'e'),(30,'6'),(25,'_'),(22,'3'),(28,'f'),(26,'1'),(31,'6'),
]

candidate = "d35cr4mbl3_tH3_cH4r4cT3r5_1ef266"
ok = len(candidate) == 32 and all(candidate[i] == ch for i, ch in checks)
print("checkPassword ->", "Access granted." if ok else "Access denied!")
```

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-1]
└─$ python3 verify.py
checkPassword -> Access granted.
```

"Access granted." matches the Java program output. The flag is good.

---

## Alternative Method — manual reconstruction

If you'd rather skip scripting, copy the `charAt` lines into a text editor and re-order them by index from `0` to `31`. Concatenate the right-hand letters in order and you get the same password.

| Index | Char | | Index | Char | | Index | Char | | Index | Char |
|---:|:-:|---|---:|:-:|---|---:|:-:|---|---:|:-:|
| 0  | `d` | | 8  | `l` | | 16 | `H` | | 24 | `5` |
| 1  | `3` | | 9  | `3` | | 17 | `4` | | 25 | `_` |
| 2  | `5` | | 10 | `_` | | 18 | `r` | | 26 | `1` |
| 3  | `c` | | 11 | `t` | | 19 | `4` | | 27 | `e` |
| 4  | `r` | | 12 | `H` | | 20 | `c` | | 28 | `f` |
| 5  | `4` | | 13 | `3` | | 21 | `T` | | 29 | `2` |
| 6  | `m` | | 14 | `_` | | 22 | `3` | | 30 | `6` |
| 7  | `b` | | 15 | `c` | | 23 | `r` | | 31 | `6` |

Reading row by row: `d35cr4mbl3_tH3_cH4r4cT3r5_1ef266`.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `cat` / `nano` | Read the Java source | Easy |
| `grep` | Pull every `charAt` line into view at once | Easy |
| `python3` + `re` | Parse the source and rebuild the password | Easy |
| Manual sort | Reconstruct the password without code | Easy |

---

## Key Takeaways

- **Java strings are character arrays.** `charAt(i)` is the equivalent of Python's `s[i]` — once that clicks, "obfuscated" `charAt`-check challenges become a 30-second job.
- **Obfuscation isn't security.** Hiding a password across many lines in random order doesn't make it harder to recover; it just makes it harder to *read*. The password is still right there in the source. This is the same lesson Minion #8728 needed to learn the hard way.
- **Regex + a one-liner beats manual work.** Pulling structured facts out of source code is a CTF superpower. The pattern `charAt\(\s*(\d+)\s*\)\s*==\s*'([^'])'` will work on basically every Vault Door variant in this series.
- **Always verify the recovered flag.** Re-running the original `checkPassword()` (or just submitting it) catches typos before you move on.
- **Wordplay decode.** The flag reads as a sentence once you translate the leet:

  | Token | Reads as |
  |---|---|
  | `d35cr4mbl3` | **descramble** (`3`→`e`, `5`→`s`, `4`→`a`) |
  | `tH3` | **the** (`3`→`e`) |
  | `cH4r4cT3r5` | **characters** (`4`→`a`, `3`→`e`, `5`→`s`) — note the intentional odd capitalisation |
  | `1ef266` | a deliberate hex-looking tail; not a word, just a fingerprint |

  Decoded message: *"descramble the characters."* The challenge is literally telling you what to do.
