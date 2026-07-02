# vault-door-4 — picoCTF Writeup

**Challenge:** vault-door-4  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Flag:** `picoCTF{jU5t_4_bUnCh_0f_bYt3s_759600abc3}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This vault uses ASCII encoding for the password.
> The source code for this vault is here: VaultDoor4.java

**Hint 1:** Use a search engine to find an "ASCII table".  
**Hint 2:** You will also need to know the difference between octal, decimal, and hexadecimal numbers.

---

## Background Knowledge (Read This First!)

### What does `checkPassword()` do?

This time the author stored the password as a 32-byte array, but every byte was written in a **different number base** to make it harder to read. The check itself is dead simple:

```java
for (int i=0; i<32; i++) {
    if (passBytes[i] != myBytes[i]) {
        return false;
    }
}
```

It just compares your input byte-by-byte against `myBytes`. If we decode every entry of `myBytes` back to its ASCII character, we have the password.

### What is ASCII?

ASCII is a 7-bit encoding that assigns a number to every common character. The printable characters live in the range **32 to 126**:

| Dec | Hex | Oct | Char | | Dec | Hex | Oct | Char | | Dec | Hex | Oct | Char |
|---:|---:|---:|:-:|---|---:|---:|---:|:-:|---|---:|---:|---:|:-:|
| 32  | 20  | 040 | ` ` | | 64  | 40  | 100 | `@` | | 96  | 60  | 140 | `` ` `` |
| 48  | 30  | 060 | `0` | | 65  | 41  | 101 | `A` | | 97  | 61  | 141 | `a` |
| 53  | 35  | 065 | `5` | | 85  | 55  | 125 | `U` | | 117 | 75  | 165 | `u` |
| 95  | 5F  | 137 | `_` | | 110 | 6E  | 156 | `n` | | 121 | 79  | 171 | `y` |

Notice how **the same character has the same value in every base** — only the *notation* changes:

```
'_' = 95  = 0x5F  = 0137
'U' = 85  = 0x55  = 0125
'n' = 110 = 0x6E  = 0156
```

### Decimal, hex, and octal at a glance

| Base | Name | Digits | Prefix in Java | Prefix in Python | Example |
|---|---|---|---|---|---|
| 10 | Decimal | 0–9 | none | none | `95` |
| 16 | Hexadecimal | 0–9, A–F | `0x` | `0x` | `0x5F` |
| 8  | Octal | 0–7 | leading `0` (e.g. `0137`) | `0o` (e.g. `0o137`) | `0137` |

The trap in this challenge: a Java literal like `0142` is **octal**, not decimal. The leading `0` flips the base. Easy to miss if you've never seen it before.

### Four notations in one array

The `myBytes` array mixes all four styles in adjacent rows:

| Row | Notation | Example | Decodes to |
|---|---|---|---|
| 1 | Decimal | `106` | `j` |
| 2 | Hex | `0x55` | `U` |
| 3 | Octal | `0142` | `b` |
| 4 | Char literal | `'a'` | `a` |

Each row is a separate hint that the author wants you to recognise all four notations. Once you do, the whole 32-byte array becomes a plain ASCII string.

---

## Reading the Source Code

The interesting part of `VaultDoor4.java`:

```java
public boolean checkPassword(String password) {
    byte[] passBytes = password.getBytes();
    byte[] myBytes = {
        106 , 85  , 53  , 116 , 95  , 52  , 95  , 98  ,
        0x55, 0x6e, 0x43, 0x68, 0x5f, 0x30, 0x66, 0x5f,
        0142, 0131, 0164, 063 , 0163, 0137, 067 , 065 ,
        '9' , '6' , '0' , '0' , 'a' , 'b' , 'c' , '3' ,
    };
    for (int i=0; i<32; i++) {
        if (passBytes[i] != myBytes[i]) {
            return false;
        }
    }
    return true;
}
```

Four rows, four notations, 32 bytes total. Just decode every entry.

---

## Solution — Step by Step

### Step 1 — Decode by hand (rows 1 and 2 are easy)

**Row 1 — Decimal:** `106 85 53 116 95 52 95 98` → `j U 5 t _ 4 _ b`

**Row 2 — Hex:** `0x55 0x6e 0x43 0x68 0x5f 0x30 0x66 0x5f`

Pull each hex value out and look it up in an ASCII table:

| Hex | Dec | Char |
|---:|---:|:-:|
| `0x55` | 85 | `U` |
| `0x6e` | 110 | `n` |
| `0x43` | 67 | `C` |
| `0x68` | 104 | `h` |
| `0x5f` | 95 | `_` |
| `0x30` | 48 | `0` |
| `0x66` | 102 | `f` |
| `0x5f` | 95 | `_` |

Row 2 → `U n C h _ 0 f _`

So far: `jU5t_4_bUnCh_0f_`

### Step 2 — Decode rows 3 and 4 (octal + char literals)

**Row 3 — Octal:** `0142 0131 0164 063 0163 0137 067 065`

The leading `0` makes these base 8. Convert each to decimal:

| Octal | Decimal | Char |
|---:|---:|:-:|
| `0142` | 1·64 + 4·8 + 2 = **98** | `b` |
| `0131` | 1·64 + 3·8 + 1 = **89** | `Y` |
| `0164` | 1·64 + 6·8 + 4 = **116** | `t` |
| `063`  | 6·8 + 3 = **51** | `3` |
| `0163` | 1·64 + 6·8 + 3 = **115** | `s` |
| `0137` | 1·64 + 3·8 + 7 = **95** | `_` |
| `067`  | 6·8 + 7 = **55** | `7` |
| `065`  | 6·8 + 5 = **53** | `5` |

Row 3 → `b Y t 3 s _ 7 5`

**Row 4 — Char literals:** `'9' '6' '0' '0' 'a' 'b' 'c' '3'` → `9 6 0 0 a b c 3`

Stitching all four rows together:

```
jU5t_4_b | UnCh_0f_ | bYt3s_75 | 9600abc3
```

Password: **`jU5t_4_bUnCh_0f_bYt3s_759600abc3`**

### Step 3 — Skip the busywork with a tiny solver

Doing the conversions by hand is fine for one challenge, but if you're lazy (like me) let Python read the source and handle the bases. `nano solve_vd4.py`, paste this, save with `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-4]
└─$ nano solve_vd4.py
```

```python
#!/usr/bin/env python3
import re

with open("VaultDoor4.java") as f:
    src = f.read()

# Strip the byte[] myBytes = { ... }; initializer out of the Java source.
m = re.search(r"byte\[\]\s+myBytes\s*=\s*\{([^}]+)\}\s*;", src, re.S)
body = m.group(1)

tokens = [t.strip() for t in body.split(",") if t.strip()]
assert len(tokens) == 32

values = []
for tok in tokens:
    if tok.startswith("'") and tok.endswith("'"):
        values.append(ord(tok[1]))                # char literal
    elif tok.startswith("0x") or tok.startswith("0X"):
        values.append(int(tok, 16))               # hex
    elif tok.startswith("0") and tok != "0" and tok.lstrip("0").isdigit():
        values.append(int(tok, 8))                # octal (Java-style leading 0)
    else:
        values.append(int(tok, 10))               # decimal

password = "".join(chr(v) for v in values)
print(f"[+] myBytes (dec)     : {values}")
print(f"[+] Recovered password: {password}")
print(f"[+] Flag              : picoCTF{{{password}}}")
```

### Step 4 — Run it

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-4]
└─$ python3 solve_vd4.py
[+] myBytes (dec)     : [106, 85, 53, 116, 95, 52, 95, 98, 85, 110, 67, 104, 95, 48, 102, 95, 98, 89, 116, 51, 115, 95, 55, 53, 57, 54, 48, 48, 97, 98, 99, 51]
[+] Recovered password: jU5t_4_bUnCh_0f_bYt3s_759600abc3
[+] Flag              : picoCTF{jU5t_4_bUnCh_0f_bYt3s_759600abc3}
```

### Step 5 — Verify by re-running the original check

I never trust a recovered flag blindly, so I rebuild `myBytes` exactly as Java would and compare every byte:

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-4]
└─$ python3 verify_vd4.py
checkPassword -> Access granted.
flag: picoCTF{jU5t_4_bUnCh_0f_bYt3s_759600abc3}
```

"Access granted." matches the Java program output. The flag is good.

---

## Alternative Method — ASCII table lookup

If you'd rather avoid any scripting, copy the `myBytes` array into your editor and convert each row using an online ASCII table. The trick is just to remember which base each row uses:

| Row | What it looks like | How to convert | Example |
|---|---|---|---|
| 1 | `106` | Read as decimal | `106 → j` |
| 2 | `0x55` | Drop `0x`, read as hex | `0x55 → 85 → U` |
| 3 | `0142` | **Leading zero = octal** | `0142 → 98 → b` |
| 4 | `'a'` | Strip quotes, look up the letter | `'a' → a` |

A common beginner mistake is reading row 3 as decimal (`0142` would be the integer `142`, which is outside printable ASCII). When you see a Java integer that starts with `0` and only uses digits `0`–`7`, it's octal.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| An ASCII table (online reference) | Convert each numeric value to its character | Easy |
| Mental math / Python `int(s, 0)` | Convert between decimal, hex, and octal | Easy |
| `python3` + `re` | Extract the `myBytes` initializer and decode every token | Easy |
| Forward verification | Re-build `myBytes` and confirm every byte matches | Easy |
| `javac` / `java` (optional) | Compile `VaultDoor4.java` and type the recovered password for a 1:1 Java check | Easy |

---

## Key Takeaways

- **The same number can wear many outfits.** `95`, `0x5F`, `0137`, and `'_'` all mean the exact same byte. Once you can recognise the four Java notations, you can read any array like this in seconds.
- **Leading zero in Java = octal.** This is the trap. `065` is **53** (`'5'`), not 65 (`'A'`). Always check the prefix before doing the conversion.
- **The hints walk you through the solve.** *"Use a search engine to find an ASCII table"* and *"know the difference between octal, decimal, and hexadecimal"* — that's literally the recipe: look up each value in an ASCII table, paying attention to the base.
- **`byte` in Java is just a number.** `byte[]` is identical to any other integer array for this challenge; the word "byte array" in the description is just picoCTF being friendly.
- **Always verify the recovered flag.** Re-running the comparison yourself catches transcription mistakes (especially the easy ones in row 3).
- **Wordplay decode.** The flag reads as a sentence once you translate the leet:

  | Token | Reads as |
  |---|---|
  | `jU5t` | **just** (`5`→`s`) |
  | `4` | **a** (`4`→`a`) |
  | `bUnCh` | **bunch** |
  | `0f` | **of** (`0`→`o`) |
  | `bYt3s` | **bytes** (`Y`→`y`, `3`→`e`) |
  | `759600abc3` | a deliberate hex-looking tail; not a word, just a fingerprint |

  Decoded message: *"just a bunch of bytes"*. The challenge is literally telling you what the vault stores — a 32-byte array of mixed-base ASCII codes — and your job is to read it. The hex tail `759600abc3` is picoCTF's signature random suffix.
