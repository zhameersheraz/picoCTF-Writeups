# Glitch Cat — picoCTF Writeup

**Challenge:** Glitch Cat  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{gl17ch_m3_n07_bda68f75}`  

---

## Description

> Our flag printing service has started glitching!
> `$ nc saturn.picoctf.net 53357`

**Hint 1:** `ASCII is one of the most common encodings used in programming`

**Tags:** `nc`, `shell`, `Python`

---

## Background Knowledge (Read This First!)

### What went wrong with the flag printer?

Normally the service would just print the flag as plain text. But it's "glitching" — instead of printing some characters directly, it's outputting them as **Python `chr()` expressions**. The flag is still there, just partially encoded.

### What is `chr()` again?

`chr()` is a Python built-in that converts an integer (ASCII code) into its corresponding character. The numbers here are in hexadecimal (`0x` prefix):

| Expression | Hex | Decimal | Character |
|------------|-----|---------|-----------|
| `chr(0x62)` | 62 | 98 | `b` |
| `chr(0x64)` | 64 | 100 | `d` |
| `chr(0x61)` | 61 | 97 | `a` |
| `chr(0x36)` | 36 | 54 | `6` |
| `chr(0x38)` | 38 | 56 | `8` |
| `chr(0x66)` | 66 | 102 | `f` |
| `chr(0x37)` | 37 | 55 | `7` |
| `chr(0x35)` | 35 | 53 | `5` |

Combined: `bda68f75`

### What is ASCII?

**ASCII** (American Standard Code for Information Interchange) is a standard that maps characters to numbers. For example, the letter `a` is 97, `b` is 98, and so on. `chr()` uses these ASCII values to produce characters. Hint 1 pointed directly at this.

---

## What the Server Sends

```
┌──(zham㉿kali)-[~]
└─$ nc saturn.picoctf.net 53357
'picoCTF{gl17ch_m3_n07_' + chr(0x62) + chr(0x64) + chr(0x61) + chr(0x36) + chr(0x38) + chr(0x66) + chr(0x37) + chr(0x35) + '}'
```

This is valid Python string concatenation. The plain text parts are already readable — only the last 8 characters of the flag are encoded as `chr()` calls. To get the full flag, just evaluate it as Python.

---

## Solution — Step by Step

### Step 1 — Connect and receive the glitched output

```
┌──(zham㉿kali)-[~]
└─$ nc saturn.picoctf.net 53357
'picoCTF{gl17ch_m3_n07_' + chr(0x62) + chr(0x64) + chr(0x61) + chr(0x36) + chr(0x38) + chr(0x66) + chr(0x37) + chr(0x35) + '}'
```

### Step 2 — Evaluate the Python expression to decode it

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "print('picoCTF{gl17ch_m3_n07_' + chr(0x62) + chr(0x64) + chr(0x61) + chr(0x36) + chr(0x38) + chr(0x66) + chr(0x37) + chr(0x35) + '}')"
picoCTF{gl17ch_m3_n07_bda68f75}
```

✅ Got the flag! 🎯

---

## Alternative Method 1 — Decode hex manually

Since each `chr(0x??)` is just one character, you can decode them by hand using the ASCII table:

```
0x62 = 98  = b
0x64 = 100 = d
0x61 = 97  = a
0x36 = 54  = 6
0x38 = 56  = 8
0x66 = 102 = f
0x37 = 55  = 7
0x35 = 53  = 5
```

Combine with the plaintext prefix and suffix:
`picoCTF{gl17ch_m3_n07_` + `bda68f75` + `}` = `picoCTF{gl17ch_m3_n07_bda68f75}`

### Alternative Method 2 — Use Python `eval()`

Since the server output is valid Python, you can evaluate it directly:

```bash
python3 -c "print(eval(\"'picoCTF{gl17ch_m3_n07_' + chr(0x62) + chr(0x64) + chr(0x61) + chr(0x36) + chr(0x38) + chr(0x66) + chr(0x37) + chr(0x35) + '}'\"))"
```

`eval()` executes the string as Python code and returns the result.

### Alternative Method 3 — CyberChef

1. Open **CyberChef** → https://gchq.github.io/CyberChef/
2. For each hex value, use **From Hex** to convert manually
3. Or just use the Python method — it's faster

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` | Connect and receive the glitched flag output | ⭐ Easy |
| `python3 -c` | Evaluate the `chr()` expressions to decode the flag | ⭐ Easy |

---

## Key Takeaways

- **`chr(0x62)` is just a character** — hex ASCII values in `chr()` calls are trivially decoded with Python
- **The server output was valid Python** — when you see `chr()` concatenation, just wrap it in `python3 -c "print(...)"` and run it
- **Hint 1 was direct** — "ASCII is one of the most common encodings used in programming" pointed straight at the `chr()` / ASCII connection
- This challenge teaches you to recognize Python code embedded in server output — a useful skill for more complex CTF challenges
- The flag `gl17ch_m3_n07` → "glitch me not" — the service was supposed to glitch the flag beyond recognition, but it didn't quite work!
