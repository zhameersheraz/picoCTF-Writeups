# convertme.py — picoCTF Writeup

**Challenge:** convertme.py  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{4ll_y0ur_b4535_9c3b7d4d}`  

---

## Description

> Run the Python script and convert the given number from decimal to binary to get the flag.
> Download Python script

**Hint 1:** `Look up a decimal to binary number conversion app on the web or use your computer's calculator!`

**Tags:** `base`, `Python`

---

## Background Knowledge (Read This First!)

### What is Decimal?

**Decimal** is the normal number system everyone uses daily — base 10, using digits `0-9`. When we say "54", that's a decimal number.

### What is Binary?

**Binary** is base 2 — it uses only `0` and `1`. Every decimal number can be represented in binary by breaking it down into powers of 2.

To convert 54 to binary:
```
54 = 32 + 16 + 4 + 2
   = 1×32 + 1×16 + 0×8 + 1×4 + 1×2 + 0×1
   = 110110
```

### How to convert instantly with Python

The fastest way to convert any decimal number to binary in the terminal:

```bash
python3 -c "print(bin(54)[2:])"
```

- `bin(54)` → `'0b110110'` (Python adds `0b` prefix to show it's binary)
- `[2:]` → strips the `0b` prefix, leaving just `110110`

### What does the script do?

The script picks a **random number between 10 and 100**, asks you to convert it to binary, checks your answer, and if correct — decrypts and prints the flag using XOR.

---

## Solution — Step by Step

### Step 1 — Run the script

```
┌──(zham㉿kali)-[~]
└─$ python3 /media/sf_downloads/convertme.py
If 54 is in decimal base, what is it in binary base?
Answer:
```

The script gives a random number — in this case `54`.

### Step 2 — Convert the number to binary

Open a second terminal and run:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "print(bin(54)[2:])"
110110
```

### Step 3 — Enter the binary answer

```
Answer: 110110
That is correct! Here's your flag: picoCTF{4ll_y0ur_b4535_9c3b7d4d}
```

✅ Got the flag! 🎯

---

## Quick Conversion Reference

Since the number is always between 10 and 100, here are some common conversions:

| Decimal | Binary |
|---------|--------|
| 10 | 1010 |
| 16 | 10000 |
| 32 | 100000 |
| 42 | 101010 |
| 54 | 110110 |
| 64 | 1000000 |
| 100 | 1100100 |

For any number the script gives, just run:
```bash
python3 -c "print(bin(NUMBER)[2:])"
```

---

## Alternative Methods

### Alternative 1 — Use `bc` in bash

```bash
echo "obase=2; 54" | bc
```

`bc` is a command-line calculator. `obase=2` sets the output base to binary.

### Alternative 2 — Use online converter

Go to https://www.rapidtables.com/convert/number/decimal-to-binary.html, type the number, and get the binary instantly — as Hint 1 suggested.

### Alternative 3 — Patch the script to skip the question

Since the number is stored in the variable `num`, you can patch the script to print the binary automatically:

```bash
python3 -c "
import random
num = random.choice(range(10,101))
print('Number:', num)
print('Binary:', bin(num)[2:])
"
```

Run this to see both the number and its binary answer at once — then run the actual script and type the answer.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `python3 convertme.py` | Run the challenge script | ⭐ Easy |
| `python3 -c "print(bin(N)[2:])"` | Convert decimal to binary instantly | ⭐ Easy |
| `echo "obase=2; N" \| bc` (optional) | Alternative bash conversion | ⭐ Easy |

---

## Key Takeaways

- **`bin(n)[2:]`** is the Python one-liner for decimal to binary — memorize it for CTFs
- **Binary is base 2** — each digit represents a power of 2 from right to left
- The `0b` prefix Python adds to binary numbers is just a label — always strip it with `[2:]` when you need just the digits
- The flag `4ll_y0ur_b4535` → "all your bases" — a reference to the classic internet meme "All your base are belong to us", also a nod to number base conversion
