# vault-door-3 — picoCTF Writeup

**Challenge:** vault-door-3  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Flag:** `picoCTF{jU5t_a_s1mpl3_an4gr4m_4_u_e60bc2}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This vault uses for-loops and byte arrays.
> The source code for this vault is here: VaultDoor3.java

**Hint:** Make a table that contains each value of the loop variables and the corresponding buffer index that it writes to.

---

## Background Knowledge (Read This First!)

### What does `checkPassword()` do?

Unlike vault-door-1 (which had 32 hardcoded `charAt` checks), this one shuffles your password through **four for-loops** that copy characters into a 32-slot `buffer` array, then checks whether the buffer equals a hardcoded string.

In other words:

1. Take your 32-char password.
2. Run it through four loops that copy characters around.
3. The result must equal `jU5t_a_sna_3lpm1cg04e_u_4_m6rb42` for the vault to open.

We don't know the password, but we **do** know what the final buffer should look like — and we know exactly how each loop copies characters. So we work the loops **backwards**.

### Java for-loop syntax used here

The loops use three patterns you'll see again and again in CTFs:

| Pattern | Reads as |
|---|---|
| `for (i = 0; i < 8; i++)` | start at `0`, while `< 8`, increment by `1` → `i = 0,1,2,…,7` |
| `for (; i < 16; i++)` | same loop, reusing `i` from the previous loop |
| `for (; i < 32; i += 2)` | start where `i` is, while `< 32`, increment by `2` → even numbers only |
| `for (i = 31; i >= 17; i -= 2)` | start at `31`, while `>= 17`, decrement by `2` → odd numbers going down |

Together, the four loops cover every slot `0..31` exactly once. No character gets written twice.

### The trick: invert each loop

Every line in `checkPassword()` is of the form

```java
buffer[i] = password[src_index];
```

If we know what `buffer[i]` should be (from the hardcoded target), and we know `src_index` is just a function of `i`, then:

```
password[src_index] = target[i]
```

That single line is the whole challenge. Build the inverse mapping for each loop, fill in the characters, done.

---

## Reading the Source Code

The interesting part of `VaultDoor3.java`:

```java
public boolean checkPassword(String password) {
    if (password.length() != 32) {
        return false;
    }
    char[] buffer = new char[32];
    int i;
    for (i=0; i<8; i++) {
        buffer[i] = password.charAt(i);
    }
    for (; i<16; i++) {
        buffer[i] = password.charAt(23-i);
    }
    for (; i<32; i+=2) {
        buffer[i] = password.charAt(46-i);
    }
    for (i=31; i>=17; i-=2) {
        buffer[i] = password.charAt(i);
    }
    String s = new String(buffer);
    return s.equals("jU5t_a_sna_3lpm1cg04e_u_4_m6rb42");
}
```

The hardcoded target is the last line:

```
jU5t_a_sna_3lpm1cg04e_u_4_m6rb42
```

That's what `buffer` must look like after the loops run. Our job is to back-compute `password`.

---

## Solution — Step by Step

### Step 1 — Make the inverse mapping table

The hint literally tells us to do this. For each loop, list the loop variable `i`, where it writes (`buffer[i]`), and where it reads from (`password[src_index]`).

| Loop | i values | writes `buffer[i]` from | inverse: `password[src]` gets |
|---|---|---|---|
| 1 | 0, 1, …, 7 | `password[i]` | `password[i] = target[i]` |
| 2 | 8, 9, …, 15 | `password[23-i]` | `password[23-i] = target[i]` |
| 3 | 16, 18, …, 30 | `password[46-i]` | `password[46-i] = target[i]` |
| 4 | 31, 29, …, 17 | `password[i]` | `password[i] = target[i]` |

Working through every row, line by line, gives you the password with no code at all (we'll do that in the Alternative Method below). The same logic in Python is even faster.

### Step 2 — Write the inverse solver

`nano solve_vd3.py`, paste this, save with `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-3]
└─$ nano solve_vd3.py
```

```python
#!/usr/bin/env python3
import re

with open("VaultDoor3.java") as f:
    code = f.read()

# Pull the target straight out of the source so the script survives edits.
m = re.search(r's\.equals\(\s*"([^"]+)"\s*\)', code)
target = m.group(1)
assert len(target) == 32

password = [""] * 32

# Loop 1: buffer[i] = password[i]   for i in 0..7
for i in range(0, 8):
    password[i] = target[i]

# Loop 2: buffer[i] = password[23-i] for i in 8..15
# inverse -> password[23-i] = target[i]
for i in range(8, 16):
    password[23 - i] = target[i]

# Loop 3: buffer[i] = password[46-i] for i in 16,18,...,30
# inverse -> password[46-i] = target[i]
for i in range(16, 32, 2):
    password[46 - i] = target[i]

# Loop 4: buffer[i] = password[i] for i in 31,29,...,17
for i in range(31, 16, -2):
    password[i] = target[i]

result = "".join(password)
print(f"[+] Target buffer      : {target}")
print(f"[+] Recovered password : {result}")
print(f"[+] Flag               : picoCTF{{{result}}}")
```

### Step 3 — Run it

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-3]
└─$ python3 solve_vd3.py
[+] Target buffer      : jU5t_a_sna_3lpm1cg04e_u_4_m6rb42
[+] Recovered password : jU5t_a_s1mpl3_an4gr4m_4_u_e60bc2
[+] Flag               : picoCTF{jU5t_a_s1mpl3_an4gr4m_4_u_e60bc2}
```

### Step 4 — Verify by re-running the loops forward

I never trust a recovered flag blindly, so I re-implement the four loops exactly as Java would run them and check the rebuilt buffer:

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-3]
└─$ python3 verify_vd3.py
rebuilt buffer : jU5t_a_sna_3lpm1cg04e_u_4_m6rb42
target         : jU5t_a_sna_3lpm1cg04e_u_4_m6rb42
checkPassword -> Access granted.
```

"Access granted." matches the Java program output. The flag is good.

---

## Alternative Method — manual reconstruction

If you'd rather see the answer without writing code, fill in the table by hand. The hint walks you through exactly this. Let's do it for loops 2 and 3 (the two reversed ones), since loops 1 and 4 are direct copies.

**Loop 2** (`buffer[i] = password[23-i]` for `i = 8..15`):

| `i` | reads `password[23-i]` | writes `buffer[i]` (= `target[i]`) |
|---:|---|---|
| 8  | `password[15]` | `n` → `password[15] = 'n'` |
| 9  | `password[14]` | `a` → `password[14] = 'a'` |
| 10 | `password[13]` | `_` → `password[13] = '_'` |
| 11 | `password[12]` | `3` → `password[12] = '3'` |
| 12 | `password[11]` | `l` → `password[11] = 'l'` |
| 13 | `password[10]` | `p` → `password[10] = 'p'` |
| 14 | `password[9]`  | `m` → `password[9]  = 'm'` |
| 15 | `password[8]`  | `1` → `password[8]  = '1'` |

So indices 8–15 of the password spell `1mpl3_an`.

**Loop 3** (`buffer[i] = password[46-i]` for `i = 16, 18, …, 30`):

| `i` | reads `password[46-i]` | writes `buffer[i]` (= `target[i]`) |
|---:|---|---|
| 16 | `password[30]` | `c` → `password[30] = 'c'` |
| 18 | `password[28]` | `0` → `password[28] = '0'` |
| 20 | `password[26]` | `e` → `password[26] = 'e'` |
| 22 | `password[24]` | `u` → `password[24] = 'u'` |
| 24 | `password[22]` | `4` → `password[22] = '4'` |
| 26 | `password[20]` | `m` → `password[20] = 'm'` |
| 28 | `password[18]` | `r` → `password[18] = 'r'` |
| 30 | `password[16]` | `4` → `password[16] = '4'` |

Combined with the direct copies from loops 1 and 4, the full password is:

```
jU5t_a_s1mpl3_an4gr4m_4_u_e60bc2
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reading the Java source | Understand how the four loops shuffle the password | Easy |
| A pen-and-paper table (per the hint) | Build the inverse mapping without code | Easy |
| `python3` + `re` | Pull the target string out of the source and run the inverse loops | Easy |
| Forward verification | Re-run the original loops on the recovered password to confirm the flag | Easy |
| `javac` / `java` (optional) | Compile and run `VaultDoor3.java` with the recovered password for a 1:1 Java check | Easy |

---

## Key Takeaways

- **A loop that copies data is reversible.** Every line `buffer[i] = password[src]` is just bookkeeping — once you know what `buffer[i]` should be, you can read the assignment backward to recover the source character.
- **The hint is the recipe.** "Make a table that contains each value of the loop variables and the corresponding buffer index" — that's literally the inverse-mapping table above. When a challenge hints at a technique, it's because that's the intended path.
- **`char[] buffer = new char[32]` is a byte array.** In Java, `char` is a 16-bit unsigned value, but for ASCII text it's effectively a byte. picoCTF just calls it a "byte array" colloquially.
- **Reused loop variables (`for (; i<16; i++)`) are normal.** The author left `i` running across loops to save a line — it doesn't change anything about how we invert them.
- **Always verify the recovered flag.** Re-running the original loops forward is the safest way to catch off-by-one mistakes.
- **Wordplay decode.** The flag reads as a sentence once you translate the leet:

  | Token | Reads as |
  |---|---|
  | `jU5t` | **just** (`5`→`s`) |
  | `a` | **a** |
  | `s1mpl3` | **simple** (`1`→`i`, `3`→`e`) |
  | `an4gr4m` | **anagram** (`4`→`a`) |
  | `4_u` | **4 u** (leet for *"for you"*) |
  | `e60bc2` | a deliberate hex-looking tail; not a word, just a fingerprint |

  Decoded message: *"just a simple anagram for u"*. The challenge is telling you that the buffer isn't the password — it's an anagrammed version of it, and your job is to undo the loops. The hex tail `e60bc2` is just picoCTF's signature random suffix.
