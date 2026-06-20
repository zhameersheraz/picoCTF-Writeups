# ASCII Numbers — picoCTF Writeup

**Challenge:** ASCII Numbers  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{45c11_n0_qu35710n5_1ll_t311_y3_n0_l135_445d4180}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Convert the following string of ASCII numbers into a readable string:
> `0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x34 0x35 0x63 0x31 0x31 0x5f 0x6e 0x30 0x5f 0x71 0x75 0x33 0x35 0x37 0x31 0x30 0x6e 0x35 0x5f 0x31 0x6c 0x6c 0x5f 0x74 0x33 0x31 0x31 0x5f 0x79 0x33 0x5f 0x6e 0x30 0x5f 0x6c 0x31 0x33 0x35 0x5f 0x34 0x34 0x35 0x64 0x34 0x31 0x38 0x30 0x7d`

**Hint 1:** `CyberChef is a great tool for any encoding but especially ASCII.`

**Hint 2:** `Try CyberChef's 'From Hex' function`

---

## Background Knowledge (Read This First!)

### What is ASCII?

**ASCII (American Standard Code for Information Interchange)** is a table that maps every printable character — letters, digits, punctuation, symbols — to a number between 0 and 127. A computer never actually stores the letter `p`; it stores the number `112`, and the terminal/font renders `112` as the letter `p` on screen.

### What does `0x70` mean?

That's the same number `112`, just written in **hexadecimal** instead of decimal. The `0x` prefix is the standard way to mark "this number is in base 16." So `0x70` and `112` are the exact same value — the challenge just chose to format every ASCII code as hex instead of decimal.

| Hex | Decimal | ASCII Character |
|---|---|---|
| `0x70` | 112 | `p` |
| `0x69` | 105 | `i` |
| `0x63` | 99 | `c` |
| `0x6f` | 111 | `o` |
| `0x43` | 67 | `C` |
| `0x54` | 84 | `T` |
| `0x46` | 70 | `F` |
| `0x7b` | 123 | `{` |

Already spelling out `picoCTF{` — the flag format itself is the proof we're decoding correctly.

### Why convert by hand at all?

There's no SSH server here — the entire challenge is just this one hex string. The "solve" is purely a decoding problem: turn each hex byte back into the character it represents, in order.

---

## Solution — Step by Step

### Step 1 — Write a small decode script

```
┌──(zham㉿kali)-[~]
└─$ mkdir ascii_numbers && cd ascii_numbers

┌──(zham㉿kali)-[~/ascii_numbers]
└─$ nano decode.py
```

Paste this in:

```python
hex_values = "0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x34 0x35 0x63 0x31 0x31 0x5f 0x6e 0x30 0x5f 0x71 0x75 0x33 0x35 0x37 0x31 0x30 0x6e 0x35 0x5f 0x31 0x6c 0x6c 0x5f 0x74 0x33 0x31 0x31 0x5f 0x79 0x33 0x5f 0x6e 0x30 0x5f 0x6c 0x31 0x33 0x35 0x5f 0x34 0x34 0x35 0x64 0x34 0x31 0x38 0x30 0x7d"

flag = "".join(chr(int(byte, 16)) for byte in hex_values.split())
print(flag)
```

Save with `Ctrl+O` → `Enter` → exit with `Ctrl+X`.

`int(byte, 16)` tells Python "this string is a base-16 number" — it converts `"0x70"` to the integer `112`. `chr()` then turns that integer back into its character. Joining all 56 results together rebuilds the flag.

### Step 2 — Run it

```
┌──(zham㉿kali)-[~/ascii_numbers]
└─$ python3 decode.py
picoCTF{45c11_n0_qu35710n5_1ll_t311_y3_n0_l135_445d4180}
```

Flag decoded in one shot.

---

## Alternative Methods

### CyberChef (the method picoCTF's own hints point to)

1. Go to https://gchq.github.io/CyberChef/
2. Paste the full hex string into the input box
3. Drag the **'From Hex'** recipe into the recipe pane
4. The output box immediately shows the flag in plaintext

No terminal needed at all — this is the intended beginner-friendly path, and exactly why the hints call it out by name.

### One-liner, no script file

If you don't want to open `nano` for something this short:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c 'print("".join(chr(int(b,16)) for b in "0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x34 0x35 0x63 0x31 0x31 0x5f 0x6e 0x30 0x5f 0x71 0x75 0x33 0x35 0x37 0x31 0x30 0x6e 0x35 0x5f 0x31 0x6c 0x6c 0x5f 0x74 0x33 0x31 0x31 0x5f 0x79 0x33 0x5f 0x6e 0x30 0x5f 0x6c 0x31 0x33 0x35 0x5f 0x34 0x34 0x35 0x64 0x34 0x31 0x38 0x30 0x7d".split()))'
picoCTF{45c11_n0_qu35710n5_1ll_t311_y3_n0_l135_445d4180}
```

### `xxd` for a pure Bash one-liner

```
┌──(zham㉿kali)-[~]
└─$ echo "70696f6f4354467b3435633131..." | xxd -r -p
```

(Strip the `0x` and spaces from each byte first to feed `xxd` a continuous hex string.) `xxd -r -p` reverses a plain hex dump back into raw bytes, which the terminal then prints as text.

---

## What Happened Internally

```
Timeline:
1. Challenge gives 56 space-separated hex bytes, each one ASCII code in base 16
2. nano decode.py — write a script that parses each "0xNN" token as base-16
3. int(byte, 16) — converts hex string to its integer value (e.g. "0x70" -> 112)
4. chr() — converts that integer back into the character it represents (112 -> 'p')
5. "".join(...) — concatenates all 56 decoded characters in original order
6. python3 decode.py — prints the rebuilt flag string
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `nano` | Write the decode script | Easy |
| `python3` | Run the hex-to-ASCII conversion | Easy |
| `int(x, 16)` | Parse a hex string as its integer value | Easy |
| `chr()` | Convert an integer back to its character | Easy |
| CyberChef `From Hex` | GUI alternative — official hinted tool | Easy |
| `xxd -r -p` | CLI alternative — reverses hex dump to raw bytes | Easy |

---

## Key Takeaways

- **Hex is just another base** — `0x70` and `112` are the same number, only the formatting differs; converting hex to ASCII is really converting hex to decimal, then decimal to character
- **The flag format is a built-in checksum** — seeing `picoCTF{` appear from the first 8 bytes confirms the decode method is correct before you even finish
- **CyberChef is worth knowing for encoding puzzles** — drag-and-drop recipes like `From Hex` solve problems like this with zero code, and picoCTF's own hints often point straight at it
- **Python's `chr()` / `ord()` and `int(x, 16)` are the core building blocks** for any character-encoding challenge — almost every ASCII/hex/decimal conversion problem reduces to these two functions
- The flag `45c11_n0_qu35710n5_1ll_t311_y3_n0_l135` decodes to "ASCII, no questions, I'll tell ye no lies" — a pun on the old saying "ask no questions and I'll tell you no lies," swapping "ask" for "ASCII" since the whole challenge is about reading ASCII without question
