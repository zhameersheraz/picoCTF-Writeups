# convertme.py — picoCTF Writeup

**Challenge:** convertme.py  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{4ll_y0ur_b4535_9c3b7d4d}`  
**Platform:** picoMini 2022  
**Writeup by:** zham  

---

## Description

> Run the Python script and convert the given number from decimal to binary to get the flag.

**Attachment:** `convertme.py` (Python 3 script)

---

## Hints

1. Look up a decimal to binary number conversion app on the web or use your computer's calculator!
2. The `str_xor` function does not need to be reverse engineered for this challenge.
3. If you have Python on your computer, you can download the script normally and run it. Otherwise, use the `wget` command in the webshell.
4. To use `wget` in the webshell, first right click on the download link and select 'Copy Link' or 'Copy Link Address'.
5. Type everything after the dollar sign in the webshell: `$ wget`, then paste the link after the space after `wget` and press enter. This will download the script for you in the webshell so you can run it!
6. Finally, to run the script, type everything after the dollar sign and then press enter: `$ python3 convertme.py`.

---

## Background Knowledge (Read This First!)

### What is the decimal / binary / hex / base-N zoo?

Computers speak binary. Humans speak decimal. The same integer can be written in many "bases" — different ways of grouping powers of a number. For this challenge you only need two:

| Base | Name | Digits | Example (the number 42) |
|------|------|--------|--------------------------|
| 10 | Decimal | 0-9 | `42` |
| 2 | Binary | 0-1 | `101010` |

To convert a decimal number to binary by hand, repeatedly divide by 2 and read the remainders bottom-up. Or, more practically, in Python:

```python
bin(42)        # -> '0b101010'
format(42, 'b')  # -> '101010'
```

The `0b` prefix is Python's way of marking a literal as binary. The `format` form gives you the bare digits, which is what the script wants as input.

### What is XOR?

`XOR` (exclusive or) is a bitwise operation: `a ^ b` is `1` when the two bits differ and `0` when they are the same. Applied to whole bytes, `chr(ord(c1) ^ ord(c2))` flips the bits that differ between two characters. XORing the same data with the same key twice gives you back the original — that is why the script's `str_xor` is reversible.

In this challenge the flag is stored XOR-encrypted with the key `enkidu` (a name from ancient Mesopotamian mythology — Gilgamesh's wild-man companion). The hint tells us we do not need to figure out the key, just run the function as-is.

### Why we can skip the quiz

`str_xor(flag_enc, 'enkidu')` produces the flag regardless of what the random number is, because the flag is computed only when you answer correctly. But the inputs to `str_xor` are constants in the source: `flag_enc` and the literal string `'enkidu'`. Both are right there in the script, so we can call `str_xor` ourselves with the same inputs and skip the prompt entirely.

---

## Solution

### Step 1: Open the script and read what it does

I read the file in my editor. The interesting parts:

```python
import random

def str_xor(secret, key):
    # ... XOR with a repeating key ...
    return "".join([chr(ord(secret_c) ^ ord(new_key_c)) for (secret_c,new_key_c) in zip(secret,new_key)])

flag_enc = chr(0x15) + chr(0x07) + chr(0x08) + ...   # 32 bytes of encrypted flag

num = random.choice(range(10, 101))

print('If ' + str(num) + ' is in decimal base, what is it in binary base?')

ans = input('Answer: ')

try:
    ans_num = int(ans, base=2)
    if ans_num == num:
        flag = str_xor(flag_enc, 'enkidu')
        print('That is correct! Here\'s your flag: ' + flag)
    else:
        print(str(ans_num) + ' and ' + str(num) + ' are not equal.')
except ValueError:
    print('That isn\'t a binary number. ...')
```

So the flow is:
1. A random number between 10 and 100 is picked.
2. We are asked to type its binary form.
3. If correct, the script decrypts `flag_enc` with the key `'enkidu'` and prints it.

The flag is *entirely* independent of the random number — it is just XOR of a constant with a constant.

### Step 2: Either play the game or skip it

The intended path (per the hints) is to:
1. Run the script.
2. Note the number it picks.
3. Convert that number to binary.
4. Type the binary back in to get the flag.

I tried this once and it works, but it is annoying because `random.choice(range(10, 101))` re-rolls every time, so if you mistype, you have to convert the new number.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 convertme.py
If 84 is in decimal base, what is it in binary base?
Answer: 1010100
That is correct! Here's your flag: picoCTF{4ll_y0ur_b4535_9c3b7d4d}
```

(84 decimal = 1010100 binary.)

### Step 3: Or, just call `str_xor` directly

Hint 2 says "The `str_xor` function does not need to be reverse engineered for this challenge." I read that as: the function is the actual decryption — just call it. So I copied `str_xor` and the `flag_enc` literal into a small script and ran it on the `'enkidu'` key:

```python
def str_xor(secret, key):
    new_key = key
    i = 0
    while len(new_key) < len(secret):
        new_key = new_key + key[i]
        i = (i + 1) % len(key)
    return "".join([chr(ord(secret_c) ^ ord(new_key_c)) for (secret_c, new_key_c) in zip(secret, new_key)])

flag_enc = chr(0x15) + chr(0x07) + chr(0x08) + chr(0x06) + chr(0x27) + chr(0x21) + chr(0x23) + chr(0x15) + chr(0x5f) + chr(0x05) + chr(0x08) + chr(0x2a) + chr(0x1c) + chr(0x5e) + chr(0x1e) + chr(0x1b) + chr(0x3b) + chr(0x17) + chr(0x51) + chr(0x5b) + chr(0x58) + chr(0x5c) + chr(0x3b) + chr(0x4c) + chr(0x06) + chr(0x5d) + chr(0x09) + chr(0x5e) + chr(0x00) + chr(0x41) + chr(0x01) + chr(0x13)

print(str_xor(flag_enc, 'enkidu'))
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 convertme_decode.py
picoCTF{4ll_y0ur_b4535_9c3b7d4d}
```

Either path lands on the same flag. The "skip" path is faster and you do not have to re-roll the random number.

**Flag:** `picoCTF{4ll_y0ur_b4535_9c3b7d4d}`

---

## What Happened Internally (Timeline)

| Step | What I did | What the script did |
|---|---|---|
| 1 | Read the file | Picked `num = random.choice(range(10, 101))` and printed it. |
| 2 | Either answered the binary quiz OR copied the function and key | Compared `int(ans, 2)` to `num`; if equal, called `str_xor(flag_enc, 'enkidu')`. |
| 3 | — | Built `new_key` by repeating `'enkidu'` to match the length of `flag_enc`. |
| 4 | — | XORed each byte of `flag_enc` against the corresponding byte of `new_key` to produce the plaintext flag. |

---

## Alternative Solve Methods

### Method 1: The intended path (answer the quiz)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 convertme.py
If 84 is in decimal base, what is it in binary base?
Answer: 1010100
That is correct! Here's your flag: picoCTF{4ll_y0ur_b4535_9c3b7d4d}
```

This is what the hints push you toward. It teaches the binary conversion part directly.

### Method 2: Skip the random and call `str_xor` directly

See the body of Step 3 above. Pull the function and constants out of the script and call it once. This is the "I read source code" path, and it is faster when you just want the flag.

### Method 3: Do it in one line in Python

If you want to be cute:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "exec(open('convertme.py').read().split('num = random')[0] + \"\nprint(str_xor(flag_enc, 'enkidu'))\")"
picoCTF{4ll_y0ur_b4535_9c3b7d4d}
```

That trims the source down to just the parts before the `random.choice(...)` line and tacks on a direct call to `str_xor`. The flag pops out without any input.

### Method 4: Let Python do the conversion for you

If you want to actually play the quiz and never think about binary, use Python interactively:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "import random; num = random.choice(range(10, 101)); print(num, '->', format(num, 'b'))"
84 -> 1010100
```

Copy the binary and paste it into the running script.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Python 3 | Run the script, do the XOR decryption, and the binary conversion |
| `bin()` / `format(n, 'b')` | Decimal to binary conversion |
| Text editor | Read the script and copy out the relevant constants |

---

## Key Takeaways

- **Read the source.** In scripting challenges, the answer is almost always sitting in a constant. Skipping the interactive part is a fair shortcut when the goal is just to capture the flag.
- **`int(ans, base=2)` is the right way to parse binary in Python** — do not roll your own converter.
- **XOR is its own inverse.** `a ^ k ^ k == a`. That is why storing the flag XOR-encrypted with a hard-coded key is not security — anyone with the script has the key.
- **The `str_xor` function extends a short key by repeating it** until it matches the secret length. This is the classic Vigenère-style keystream trick (although XOR-based, not character-shift-based). It is reversible only if you know the key — and the key `'enkidu'` is right there in the source.
- **Wordplay, as always:** the flag decodes from leetspeak as **all your bases** — a reference to the famous 1989 meme *"All your base are belong to us"* from the Sega game *Zero Wing*. The challenge name `convertme.py` is the setup: convert *me* (a number) from one base to another. The punchline is that the flag itself is about converting between bases. The author has a sense of humor about it.
