# Nice netcat... — picoCTF Writeup

**Challenge:** Nice netcat...  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{g00d_k1tty!_n1c3_k1tty!_e9c85}`  

---

## Description

> There is a nice program that you can talk to by using this command in a shell:
> `$ nc wily-courier.picoctf.net 53800`, but it doesn't speak English...

**Hint 1:** `You can practice using netcat with this picoGym problem: what's a netcat?`

---

## Background Knowledge (Read This First!)

### What is ASCII?

**ASCII** (American Standard Code for Information Interchange) is a standard that maps every printable character to a number between 0 and 127. For example:

| Number | Character |
|--------|-----------|
| 112 | `p` |
| 105 | `i` |
| 99 | `c` |
| 111 | `o` |
| 67 | `C` |
| 84 | `T` |
| 70 | `F` |
| 123 | `{` |
| 125 | `}` |

So when the server sends numbers, it's sending the flag — just in ASCII decimal format instead of plain text.

### What is `chr()` in Python?

`chr(n)` converts an ASCII number back to its character. So `chr(112)` → `'p'`, `chr(105)` → `'i'`, and so on. This is the reverse of `ord()` which converts a character to its number.

### Why does the challenge say "it doesn't speak English"?

Because instead of printing the flag as readable text, the server outputs each character as its **decimal ASCII value** — one number per line. It's still the flag, just encoded as numbers.

---

## What the Server Sends

```
┌──(zham㉿kali)-[~]
└─$ nc wily-courier.picoctf.net 53800
112 
105 
99 
111 
67 
84 
70 
123 
103 
...
125 
10
```

Each number is one character of the flag. `10` at the end is the ASCII code for a newline (`\n`).

---

## Solution — Step by Step

### Step 1 — Connect and receive the numbers

```
┌──(zham㉿kali)-[~]
└─$ nc wily-courier.picoctf.net 53800
112 105 99 111 67 84 70 123 103 48 48 100 95 107 49 116 116 121 33 95 110 49 99 51 95 107 49 116 116 121 33 95 101 57 99 56 53 125 10
```

### Step 2 — Decode the ASCII numbers to text

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "
nums = [112,105,99,111,67,84,70,123,103,48,48,100,95,107,49,116,116,121,33,95,110,49,99,51,95,107,49,116,116,121,33,95,101,57,99,56,53,125,10]
print(''.join(chr(n) for n in nums))
"
picoCTF{g00d_k1tty!_n1c3_k1tty!_e9c85}
```

✅ Got the flag! 🎯

---

## Breaking Down the Decode

Each number maps to one character:

| Number | Character | Number | Character |
|--------|-----------|--------|-----------|
| 112 | `p` | 103 | `g` |
| 105 | `i` | 48 | `0` |
| 99 | `c` | 48 | `0` |
| 111 | `o` | 100 | `d` |
| 67 | `C` | 95 | `_` |
| 84 | `T` | 107 | `k` |
| 70 | `F` | ... | ... |
| 123 | `{` | 125 | `}` |

Combined: `picoCTF{g00d_k1tty!_n1c3_k1tty!_e9c85}`

---

## Alternative Method — Pipe nc output directly into Python

You can decode on the fly by piping the netcat output straight into Python:

```
┌──(zham㉿kali)-[~]
└─$ nc wily-courier.picoctf.net 53800 | python3 -c "
import sys
nums = [int(x) for x in sys.stdin.read().split()]
print(''.join(chr(n) for n in nums))
"
picoCTF{g00d_k1tty!_n1c3_k1tty!_e9c85}
```

This connects, receives all the numbers, and decodes them automatically — no manual copying needed.

### Alternative Method 2 — CyberChef

1. Paste all the numbers into **CyberChef** → https://gchq.github.io/CyberChef/
2. Add the **"From Charcode"** recipe (set base to Decimal)
3. Output shows the flag immediately

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` | Connect to the server and receive numbers | ⭐ Easy |
| Python `chr()` | Convert ASCII decimal numbers to characters | ⭐ Easy |
| Pipe `\|` (optional) | Auto-decode the nc output on the fly | ⭐ Easy |
| CyberChef (optional) | GUI decoding via "From Charcode" recipe | ⭐ Easy |

---

## Key Takeaways

- **ASCII decimal numbers are just characters in disguise** — whenever you see a stream of numbers between 32-126, try decoding them as ASCII
- **`chr(n)`** converts any ASCII number to its character in Python — the opposite of `ord(c)`
- **Piping `nc` output into Python** is a clean way to automate decoding without manual copy-pasting
- The description "it doesn't speak English" was the hint — the server was speaking numbers (ASCII) instead of letters
- The flag `g00d_k1tty!_n1c3_k1tty!` → "good kitty! nice kitty!" — petting the netcat 🐱
