# Based — picoCTF Writeup

**Challenge:** Based  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{learning_about_converting_values_DF19A0E8}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham

---

## Description

> To get truly 1337, you must understand different data encodings, such as hexadecimal or binary. Can you get the flag from this program to prove you are on the way to becoming 1337?
> Connect with `nc fickle-tempest.picoctf.net 63326`.

## Hints

> I hear python can convert things.

> It might help to have multiple windows open.

---

## Background Knowledge (Read This First!)

**What's a "base" in computing?** Normally we count in base 10 (decimal) — ten digits, 0 through 9. Computers can represent the exact same numbers using other counting systems:

- **Binary (base 2):** only digits `0` and `1`. Every value is built from powers of 2.
- **Octal (base 8):** digits `0`–`7`. Often prefixed with `o` or a leading `0`.
- **Hexadecimal (base 16):** digits `0`–`9` plus `a`–`f`. Each hex digit covers exactly 4 bits, which is why it's the go-to shorthand for binary in programming.

**Why does ASCII matter here?** Every letter on a keyboard has a numeric code behind it (the ASCII table). The letter `s`, for example, is the number `115` in decimal. That same `115` can be written as `01110011` in binary, `163` in octal, or `73` in hex — they're all the same number, just written differently. This challenge gives me a string of letters encoded in one of these bases, and I have to convert it back to plain text.

**Why "multiple windows"?** The program is timed — I get 45 seconds per round, and it throws several rounds at me back to back, each in a different base. There's no time to do the math by hand. The fix is to keep a second terminal open running Python, so I can paste in the encoded value, get the decoded word instantly, and paste it straight back into the `nc` session.

---

## Solution

### Step 1 — Connect and open a second window for Python

```
┌──(zham㉿kali)-[~]
└─$ nc fickle-tempest.picoctf.net 63326
Let us see how data is stored
socket
Please give the 01110011 01101111 01100011 01101011 01100101 01110100 as a word.
...
you have 45 seconds.....
Input:
```

The line right above the binary string (`socket` in this example) is actually a giveaway of the answer for that specific round — but the program reuses this pattern with a fresh random word and a fresh encoding every connection, so I can't rely on memorizing one answer. I decode properly each time.

In a second terminal, I keep Python ready to convert whatever encoding shows up:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "print(''.join(chr(int(b,2)) for b in '01110011 01101111 01100011 01101011 01100101 01110100'.split()))"
socket
```

Binary decoded to `socket`. I paste that back into the `nc` window:

```
Input:
socket
```

### Step 2 — Octal round

The next round used octal, marked with an `o` prefix on each value:

```
Please give me the  o163 o164 o162 o145 o145 o164 as a word.
Input:
```

Octal digits go up to 7 only, and each group converts to one ASCII character:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "print(''.join(chr(int(o,8)) for o in '163 164 162 145 145 164'.split()))"
street
```

Answer: `street`.

### Step 3 — Hex round

```
Please give me the 7461626c65 as a word.
Input:
```

This time it's one continuous hex string with no separators. Python's `bytes.fromhex()` handles that directly:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "print(bytes.fromhex('7461626c65').decode())"
table
```

Answer: `table`.

### Step 4 — Flag

After the third correct answer in a row, the program gives up the flag:

```
table
You've beaten the challenge
Flag: picoCTF{learning_about_converting_values_DF19A0E8}
```

---

## What Happened Internally

1. The server picks a random word and a random base (binary, octal, or hex) and prints the word encoded in that base.
2. It starts a 45-second timer and waits for my input.
3. I take the encoded string, run it through the matching Python decode function in a second terminal, and get back the original word.
4. I paste that word into the `nc` session before time runs out.
5. The server checks my answer, and if correct, moves on to the next round with a new random word and possibly a different base.
6. After enough correct rounds in a row, it prints the flag — proof that I can read data regardless of which base it's stored in.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nc` | Connect to the remote challenge server |
| Python 3 (`int(x, base)`, `bytes.fromhex()`) | Convert binary/octal/hex strings back to readable ASCII |
| A second terminal window | Run conversions fast enough to beat the 45-second timer |

---

## Key Takeaways

- Binary, octal, and hexadecimal are all just different ways of writing the same numbers — and by extension, the same ASCII characters.
- Python's `int(string, base)` converts a string in any base straight to its decimal value; from there `chr()` turns that decimal value back into a readable character.
- `bytes.fromhex(...).decode()` is the fastest way to flip a hex string back into plain text without manually splitting it into byte pairs.
- Under time pressure, having a second window ready with the right one-liners pre-typed (or in shell history) saves critical seconds — speed matters as much as correctness here.
- Flag wordplay: `learning_about_converting_values` says exactly what the challenge tested — moving values between number bases (binary, octal, hex) and back to plain text.
