# 2warm — picoCTF Writeup

**Challenge:** 2warm  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{101010}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> Can you convert the number 42 (base 10) to binary (base 2)?

---

## Hints

1. Submit your answer in our competition's flag format. For example, if your answer was '11111', you would submit 'picoCTF{11111}' as the flag.

---

## Background Knowledge (Read This First!)

### What is base 10 vs base 2?

**Base 10** (decimal) is how humans normally count, using the digits 0-9 and powers of ten. **Base 2** (binary) is how computers count, using only 0 and 1 and powers of two. The same integer can be written in many bases; the question is just "how do you group the place values."

### How to convert decimal to binary

There are a few ways. The two fastest:

1. **Use a tool.** Python, your OS calculator in programmer mode, `bc`, or a one-liner with `printf` all do this in a heartbeat.
2. **Do it by hand with repeated division.** Keep dividing the number by 2, the remainders (read bottom-up) give you the binary digits.

For 42, repeated division by 2 goes:
```
42 / 2 = 21 r 0
21 / 2 = 10 r 1
10 / 2 = 5  r 0
 5 / 2 = 2  r 1
 2 / 2 = 1  r 0
 1 / 2 = 0  r 1
```
Reading remainders bottom-up: `101010`. Done.

A quicker mental check: 42 = 32 + 8 + 2 = `2^5 + 2^3 + 2^1` = `101010`.

### The flag format is part of the puzzle

Hint 1 says "submit your answer in our competition's flag format." That means: take the binary string and wrap it in `picoCTF{...}`. The flag is the binary representation, not the decimal one. Many CTF challenges expect you to know this — getting the math right but forgetting the wrapper is a classic first-timer mistake.

---

## Solution

### Step 1: Convert 42 to binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "print(bin(42)[2:])"
101010
```

Or, by hand with `bc`:
```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "obase=2; 42" | bc
101010
```

Or with the GUI calculator in programmer mode. Pick whichever you like.

### Step 2: Wrap it in the flag format

The hint says "if your answer was '11111', you would submit 'picoCTF{11111}' as the flag." So:

```
picoCTF{101010}
```

**Flag:** `picoCTF{101010}`

---

## What Happened Internally (Timeline)

| Step | What I did | What I got |
|---|---|---|
| 1 | Read the prompt — "convert 42 base 10 to base 2" | Need 42 in binary. |
| 2 | Ran `python3 -c "print(bin(42)[2:])"` | `101010`. |
| 3 | Wrapped in flag format | `picoCTF{101010}`. |
| 4 | Submitted. | Accepted. |

That is the entire solve. There is no instance to spin up, no network call, no binary to reverse — just a math problem and a string format.

---

## Alternative Solve Methods

### Method 1: Python one-liner

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "print(format(42, 'b'))"
101010
```

`format(n, 'b')` and `bin(n)[2:]` both work. The `[2:]` strips the `'0b'` prefix that `bin()` adds.

### Method 2: Bash `bc`

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo "obase=2; 42" | bc
101010
```

`bc` is the arbitrary-precision calculator. The `obase=2;` part sets the output base to binary before you give it the input.

### Method 3: Windows calculator

Open `calc.exe`, switch to **Programmer** mode, type 42, click the **BIN** radio. The display flips to `101010`. Faster than opening a terminal if you are already in the GUI.

### Method 4: By hand

If you ever need to do it without a computer (interview, exam, off-grid CTF), the repeated-division method above takes about 30 seconds per number.

### Method 5: Mental shortcut

For small numbers, write down the powers of two and subtract:

| 32 | 16 | 8 | 4 | 2 | 1 |
|----|----|----|----|----|----|
| 1  | 0  | 1  | 0  | 1  | 0  |

`42 - 32 = 10`, `10 - 8 = 2`, `2 - 2 = 0`. So 42 = 32 + 8 + 2 = `101010`.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `python3` | Quick `bin(42)` conversion |
| `bc` (alt) | Bash one-liner for the same conversion |
| Windows Calculator in Programmer mode (alt) | GUI option for the same conversion |
| Brain (alt) | The repeated-division method works offline too |

---

## Key Takeaways

- **Base conversions are a one-liner in any language.** Memorize the Python / shell / calculator incantation so you can do it without thinking.
- **Strip the `0b` prefix.** Python's `bin()` adds it; if you paste `0b101010` into the flag field, you are wrong. Use `bin(n)[2:]` or `format(n, 'b')` to get the bare digits.
- **Read the flag format hint.** "Submit your answer in our competition's flag format" is a hint to *wrap your answer* in `picoCTF{...}`. Skipping the wrapper is the most common failure mode for this style of problem.
- **`bc` is a forgotten gem.** It supports arbitrary precision and arbitrary output bases (`obase=2;`, `obase=16;`, etc.) which is handy for crypto challenges where the numbers get big.
- **Wordplay, as always:** the challenge name is "2warm" — a pun on the binary digit `2`... no wait, the number `2` in front of "warm." The "warm" theme runs through the whole picoCTF 2019 General Skills set: Lets Warm Up, 2warm, Warmed Up, Wave a flag. It is the "intro" track, and the author wants you to feel like you are easing into the competition. The 42 → `101010` mapping has its own pop-culture nod: 42 is the *Answer to the Ultimate Question of Life, the Universe, and Everything* from *The Hitchhiker's Guide to the Galaxy*, and 42 in binary is `101010` — six binary digits that look like a tidy barcode. Spotting the joke is the second puzzle hiding inside the first.
