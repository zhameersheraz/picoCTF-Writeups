# The Numbers — picoCTF Writeup

**Challenge:** The Numbers  
**Category:** Cryptography  
**Difficulty:** Easy  
**Flag:** `PICOCTF{THENUMBERSMASON}`  

---

## Description

> The numbers... what do they mean?
> Download the file: numbers.png

**Hint shown in challenge:** `The flag is in the format PICOCTF{}`

---

## Background Knowledge (Read This First!)

### What is the A1Z26 Cipher?

The A1Z26 cipher is one of the simplest ciphers — it replaces each letter with its position number in the alphabet:

```
A=1, B=2, C=3, D=4 ... Z=26
```

---

## Solution — Step by Step

### Step 1 — Open the Image

I opened the downloaded `numbers.png` file. The image showed a series of numbers:

```
16 9 3 15 3 20 6 { 20 8 5
14 21 13 2 5 18 19 13 1
19 15 14 }
```

✅ The `{` and `}` characters confirm this follows the `PICOCTF{...}` flag format!

### Step 2 — Decode the Numbers

Using A1Z26 where each number = its position in the alphabet:

```
16 → P
9  → I
3  → C
15 → O
3  → C
20 → T
6  → F
20 → T
8  → H
5  → E
14 → N
21 → U
13 → M
2  → B
5  → E
18 → R
19 → S
13 → M
1  → A
19 → S
15 → O
14 → N
```

I used dcode.fr to decode it: 🔗 https://www.dcode.fr/letter-number-cipher

Output: `PICOCTFTHENUMBERSMASON`

### Step 3 — Add the Flag Format

Adding the `{` and `}` brackets as shown in the image:

```
PICOCTF{THENUMBERSMASON}
```

Got the flag! 🎯

---

## Alternative Methods

**Method 1 — Python**
```python
numbers = [16,9,3,15,3,20,6,20,8,5,14,21,13,2,5,18,19,13,1,19,15,14]
flag = ''.join([chr(n + 64) for n in numbers])
print(f"PICOCTF{{{flag}}}")
```

**Method 2 — Manual Decoding**
Simply use the alphabet: `A=1, B=2, C=3 ... Z=26` and count each number to its letter.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Image Viewer | Open and read numbers.png |
| dcode.fr | Decode the A1Z26 number cipher |

---

## Key Takeaways

- A1Z26 is a simple substitution cipher where numbers replace letters (A=1, B=2 ... Z=26)
- The `{` and `}` in the image were literal characters — not encoded
- dcode.fr is a very useful online tool for identifying and decoding many types of ciphers
- When you see numbers in a CTF, always try A1Z26 first — it's one of the most common beginner ciphers
- The flag "THE NUMBERS MASON" is a reference to the famous Call of Duty: Black Ops scene 🎮
