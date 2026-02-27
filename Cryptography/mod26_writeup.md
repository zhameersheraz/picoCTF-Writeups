# Mod 26 - picoCTF Writeup

**Challenge:** Mod 26  
**Category:** Cryptography  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{next_time_I'll_try_2_rounds_of_rot13_45559abd}`

---

## Description

Cryptography can be easy, do you know what ROT13 is?

Download the file: values.txt

---

## Hints

1. This can be solved online if you don't want to do it by hand!

---

## Solution

### Step 1: Examine the File

I navigated to the downloaded file and used `exiftool` to check the file type:

```bash
cd /media/sf_downloads
exiftool values.txt
```

**Output:**
```
File Name                       : values.txt
File Type                       : TXT
MIME Type                       : text/plain
```

✅ It's a plain text file.

### Step 2: Read the File Contents

I used `strings` to read the content of the file:

```bash
strings values.txt
```

**Output:**
```
cvpbPGS{arkg_gvzr_V'yy_gel_2_ebhaqf_bs_ebg13_45559noq}
```

✅ This looks like **ROT13** — the structure `cvpbPGS{...}` resembles `picoCTF{...}` shifted by 13!

### Step 3: Decode with ROT13

I used the `caesar` command to decode it:

```bash
echo "cvpbPGS{arkg_gvzr_V'yy_gel_2_ebhaqf_bs_ebg13_45559noq}" | caesar
```

**Output:**
```
picoCTF{next_time_I'll_try_2_rounds_of_rot13_45559abd}
```

Got the flag! 🎯

---

## Why This Works

### What is ROT13?

**ROT13** (Rotate by 13) is a special case of the Caesar cipher that shifts each letter by **13 positions** in the alphabet.

Since the alphabet has 26 letters, applying ROT13 **twice** returns the original text — making it its own inverse!

```
a → n → a  (shift 13, shift 13 again = back to start)
```

**Example:**
```
p → c
i → v
c → p
o → b
→ picoCTF becomes cvpbPGS
```

### The Vulnerability Chain
```
values.txt → strings → ROT13 encoded text → caesar decode → Flag
```

---

## Alternative Methods

### Method 1: Using tr command (Linux)
```bash
echo "cvpbPGS{arkg_gvzr_V'yy_gel_2_ebhaqf_bs_ebg13_45559noq}" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

### Method 2: Python
```python
import codecs
cipher = "cvpbPGS{arkg_gvzr_V'yy_gel_2_ebhaqf_bs_ebg13_45559noq}"
print(codecs.decode(cipher, 'rot_13'))
```

### Method 3: Online Tool
Use **CyberChef** (https://gchq.github.io/CyberChef/) with the **"ROT13"** recipe.

---

## Flag

```
picoCTF{next_time_I'll_try_2_rounds_of_rot13_45559abd}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `exiftool` | Check file type and metadata |
| `strings` | Read file contents |
| `caesar` (bsdgames) | Decode the ROT13 cipher |

---

## Key Takeaways

- **ROT13** is a Caesar cipher with a fixed shift of 13
- Applying ROT13 **twice** gives back the original text
- The `caesar` command from `bsdgames` tries all 25 shifts — ROT13 is just one of them
- The flag hints at "2 rounds of rot13" — meaning applying ROT13 twice would give back the ciphertext
- ROT13 is **not encryption** — it offers zero security and is trivially reversible
- The `tr` command is a quick built-in Linux way to apply ROT13 without installing anything
