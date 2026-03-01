# 13 - picoCTF Writeup

**Challenge:** 13  
**Category:** Cryptography  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{not_too_bad_of_a_problem}`

---

## Description

Cryptography can be easy, do you know what ROT13 is?

`cvpbPGS{abg_gbb_onq_bs_n_ceboyrz}`

---

## Hints

1. This can be solved online if you don't want to do it by hand!

---

## Solution

### Step 1: Identify the Cipher

The ciphertext was given directly in the challenge description:

```
cvpbPGS{abg_gbb_onq_bs_n_ceboyrz}
```

The challenge title is **"13"** and it mentions **ROT13** — so the cipher is clearly **ROT13** (Caesar cipher with a shift of 13).

### Step 2: Decode with ROT13

I used the `caesar` command to decode it:

```bash
echo "cvpbPGS{abg_gbb_onq_bs_n_ceboyrz}" | caesar
```

**Output:**
```
picoCTF{not_too_bad_of_a_problem}
```

Got the flag! 🎯

---

## Why This Works

### What is ROT13?

**ROT13** shifts each letter by **13 positions** in the alphabet. Since the alphabet has 26 letters, ROT13 is its own inverse — applying it twice returns the original text.

**Decoding example:**
```
c → p
v → i
p → c
b → o
P → C
G → T
S → F
```

So `cvpbPGS` becomes `picoCTF`!

### The Vulnerability Chain
```
Ciphertext → ROT13 decode → Flag
```

---

## Alternative Methods

### Method 1: tr command (Linux)
```bash
echo "cvpbPGS{abg_gbb_onq_bs_n_ceboyrz}" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

### Method 2: Python
```python
import codecs
cipher = "cvpbPGS{abg_gbb_onq_bs_n_ceboyrz}"
print(codecs.decode(cipher, 'rot_13'))
```

### Method 3: Online Tool
Use **CyberChef** (https://gchq.github.io/CyberChef/) with the **"ROT13"** recipe.

---

## Flag

```
picoCTF{not_too_bad_of_a_problem}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `caesar` (bsdgames) | Decode the ROT13 cipher |

---

## Key Takeaways

- **ROT13** is a Caesar cipher with a fixed shift of 13
- Applying ROT13 **twice** returns the original text
- The challenge title "13" directly hints at the shift value
- `cvpbPGS` → `picoCTF` is the classic ROT13 giveaway in picoCTF challenges
- The `tr` command is a quick built-in Linux way to apply ROT13 without extra tools
