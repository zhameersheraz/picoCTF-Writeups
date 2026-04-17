# bytemancy 0 — picoCTF Writeup

**Challenge:** bytemancy 0  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{pr1n74813_ch4r5_62360bfd}`  

---

## Description

> Can you conjure the right bytes? The program's source code can be downloaded here.  
> Connect to the program with netcat:  
> `$ nc candy-mountain.picoctf.net 52295`

**Hint 1:** `Solving this with a one-liner will help with the next challenge in this series`

**Tags:** `ASCII`, `encoding`, `netcat`, `general-skills`

---

## Background Knowledge (Read This First!)

### What is ASCII?

**ASCII (American Standard Code for Information Interchange)** is a character encoding standard that maps numbers to characters. Every letter, digit, and symbol you type has a corresponding number behind it.

For example:

| Decimal | Character |
|---------|-----------|
| 65 | A |
| 97 | a |
| 101 | e |
| 32 | (space) |
| 48 | 0 |

When the challenge says **"Send me ASCII DECIMAL 101"**, it's asking: *what character does the number 101 represent?*

You can look this up instantly in your terminal:

```bash
python3 -c "print(chr(101))"
```

Output: `e`

Or use a simple ASCII table reference — decimal 101 = lowercase `e`.

### What does "side-by-side, no space" mean?

The challenge asks for three values (101, 101, 101) placed next to each other with no spaces or separators. Since all three are the same character (`e`), the answer is simply:

```
eee
```

If the three values were different — say 72, 105, 33 — you'd look up each one:
- 72 → `H`
- 105 → `i`
- 33 → `!`

And send: `Hi!`

### What is `nc` (netcat)?

`nc` is netcat, a raw networking tool that opens a TCP connection to a remote server and lets you interact with it directly in your terminal. Here it connects you to the bytemancy challenge service where you send your answer and receive the flag.

---

## Solution — Step by Step

### Step 1 — Connect to the service

```
┌──(zham㉿kali)-[~]
└─$ nc candy-mountain.picoctf.net 52295
⊹──────[ BYTEMANCY-0 ]──────⊹
☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐
Send me ASCII DECIMAL 101, 101, 101, side-by-side, no space.
☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐
⊹─────────────⟡─────────────⊹
==>
```

The server presents a prompt asking you to send three ASCII decimal values side-by-side with no spaces.

### Step 2 — Decode the ASCII values

The server asks for ASCII DECIMAL **101, 101, 101**.

Look up decimal 101 in the ASCII table:

```
101 → e
101 → e
101 → e
```

All three are the same character. Side-by-side with no space = `eee`.

You can verify this in your terminal any time with Python:

```bash
python3 -c "print(chr(101), chr(101), chr(101), sep='')"
```

Output: `eee`

### Step 3 — Send the answer

```
==> eee
picoCTF{pr1n74813_ch4r5_62360bfd}
```

✅ Got the flag! 🎯

> **Fun detail:** The flag contains `pr1n74813_ch4r5` — leet-speak for "printable chars." This is a nod to the concept of **printable ASCII characters** — the letters, digits, and symbols (decimal 32–126) that are visible when printed, as opposed to control characters like newline or null.

---

## One-liner Solution (as the hint suggests)

The hint says *"solving this with a one-liner will help with the next challenge in this series."* Here's a Python one-liner that converts the values and sends the answer automatically:

```bash
python3 -c "print(chr(101)*3)" | nc candy-mountain.picoctf.net 52295
```

Breaking this down:
- `chr(101)` — converts decimal 101 to the character `e`
- `*3` — repeats it three times → `eee`
- `print(...)` — outputs it with a newline (which acts as pressing Enter)
- `|` — pipes that output directly into the nc connection as if you typed it

This technique becomes essential for later bytemancy challenges where the values may be random each time or too many to type manually.

---

## Alternative Methods

### Alternative 1 — Use Python interactively to look up values

```bash
python3
>>> chr(101)
'e'
```

Simple and quick for looking up one-off ASCII values.

### Alternative 2 — Use an ASCII table reference

Any online ASCII table or the `man ascii` command in Linux shows the full mapping:

```bash
man ascii
```

Scroll to decimal 101 → `e`.

### Alternative 3 — Use `printf` in bash

```bash
printf '%d %d %d\n' "'e" "'e" "'e"
```

Or to go the other direction (decimal to character):

```bash
printf "\\$(printf '%03o' 101)"
```

Output: `e`

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` | Connect to the remote challenge service | ⭐ Easy |
| ASCII table / `chr()` | Convert decimal 101 to its character | ⭐ Easy |
| Python one-liner | Automate the decode and send in one command | ⭐ Easy |

---

## Key Takeaways

- **ASCII is just a number-to-character mapping** — every character your keyboard types has a decimal number; `chr()` in Python converts decimal → character instantly
- **`python3 -c "..."` is powerful for quick one-liners** — you don't need to write a full script; inline Python is faster for small tasks like this
- **Piping (`|`) into `nc` automates interaction** — instead of typing answers manually, you can pipe computed answers directly into the netcat session
- **The hint about one-liners is important** — later challenges in the bytemancy series likely send random values each connection, making manual lookup impossible; automating from the start is the right approach
- **Printable ASCII (32–126) is a key concept in CTFs** — many encoding and binary challenges revolve around this range of human-readable characters
