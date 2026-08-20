# PW Crack 2 — picoCTF Writeup  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** 200  
**Flag:** `picoCTF{tr45h_51ng1ng_489dea9a}`  
**Platform:** picoMini 2022  
**Writeup by:** zham  

## Description
> Can you crack the password to get the flag?  
>   
> Download the password checker here and you'll need the encrypted flag in the same directory too.

## Hints
> 1. Does that encoding look familiar?  
> 2. The `str_xor` function does not need to be reverse engineered for this challenge.

## Background Knowledge
This is the same challenge as PW Crack 1, but the password is **obfuscated** using `chr(0xNN)` calls instead of a literal string. The intent is to test if you can read Python's `chr()` (a function that returns the character for a Unicode code point) and decode the obfuscation.

- `chr(0x64)` -> the character with code point 0x64 (100 in decimal) -> `'d'`
- `chr(0x65)` -> 0x65 = 101 -> `'e'`
- `chr(0x37)` -> 0x37 = 55 -> `'7'`
- `chr(0x36)` -> 0x36 = 54 -> `'6'`

So the password is `de76`. Hint 1 ("Does that encoding look familiar?") is nudging you to recognize `0x64` as ASCII hex for a printable character.

## Solution

### Step 1 — Read the source with nano

```bash
┌──(zham㉿kali)-[~]
└─$ nano level2.py
```

The relevant line:
```python
if( user_pw == chr(0x64) + chr(0x65) + chr(0x37) + chr(0x36) ):
```

### Step 2 — Decode the password
`chr()` takes a Unicode/ASCII code point. Hex `0x64` is 100 in decimal, which is `'d'` in ASCII. Doing this for each call:

| `chr()` call | Decimal | Character |
|---|---|---|
| `chr(0x64)` | 100 | d |
| `chr(0x65)` | 101 | e |
| `chr(0x37)` | 55  | 7 |
| `chr(0x36)` | 54  | 6 |

So the password is `de76`. Save and exit nano with `Ctrl+X`.

### Step 3 — Run the checker with the password

```bash
┌──(zham㉿kali)-[~]
└─$ echo "de76" | python3 level2.py
Please enter correct password for flag: Welcome back... your flag, user:
picoCTF{tr45h_51ng1ng_489dea9a}
```

Got the flag.

### Alternative: skip the input() and call str_xor directly
The same "Windows-friendly" approach as PW Crack 1 - call the decrypt function directly without going through `input()`:

```bash
┌──(zham㉿kali)-[~]
└─$ python3 -c "
flag_enc = open('level2.flag.txt.enc', 'rb').read()
def str_xor(secret, key):
    new_key = key
    i = 0
    while len(new_key) < len(secret):
        new_key = new_key + key[i]
        i = (i + 1) % len(key)
    return ''.join([chr(ord(s) ^ ord(k)) for (s, k) in zip(secret, new_key)])
pw = chr(0x64) + chr(0x65) + chr(0x37) + chr(0x36)
print(str_xor(flag_enc.decode(), pw))
"
picoCTF{tr45h_51ng1ng_489dea9a}
```

This sidesteps the PowerShell `echo "..."` newline-quirk entirely by not relying on `input()` at all.

### Step 4 — Submit

```bash
┌──(zham㉿kali)-[~]
└─$ echo "picoCTF{tr45h_51ng1ng_489dea9a}"
picoCTF{tr45h_51ng1ng_489dea9a
```

Pasted into the picoCTF flag box. Accepted.

## What Happened Internally
1. The challenge's `level2.py` reads the encrypted flag from `level2.flag.txt.enc`, then asks the user for a password.
2. The comparison `chr(0x64) + chr(0x65) + chr(0x37) + chr(0x36)` evaluates to the string `"de76"` at runtime. The `chr()` obfuscation is just `chr(100) + chr(101) + chr(55) + chr(54)` - it doesn't actually hide anything.
3. If the user enters the right password, the script XOR-decrypts the flag with the password as the key and prints the result.
4. The XOR cipher is reversible with the right key. With `"de76"`, the bytes `0x64 0x65 0x37 0x36` are repeated to match the flag length, then XORed with each encrypted byte.

## Tools Used
| Tool | Purpose |
|------|---------|
| `nano` | View the source (as before) |
| `python3` | Run the checker, or call str_xor directly |
| `chr()` mental decode | Recognize `0x64` etc. as ASCII hex |
| `echo` + pipe (or `python3 -c`) | Feed the password in |

## Key Takeaways
- **`chr()` is not encryption.** It's a one-line lookup table from integer to character. Anyone who can read the code can decode it. Same for `ord()`, `bytes([...])`, base64, hex strings, etc.
- **Hex `0x64` is ASCII `'d'`.** This is the kind of pattern that becomes second nature: 0x20-0x7E is the printable ASCII range, and the digits `0-9` are 0x30-0x39, the letters `a-z` are 0x61-0x7A, `A-Z` are 0x41-0x5A. If you see `chr(0x6X)`, it's almost always a lowercase letter.
- **String concatenation in Python is `+`.** `chr(0x64) + chr(0x65) + chr(0x37) + chr(0x36)` is just `"d" + "e" + "7" + "6"` which is `"de76"`. The `chr()` calls are noise.
- **Wordplay decode**: `tr45h_51ng1ng` = "trash singing" in 1337 (4=a, 5=s, 1=i). Combined with PW Crack 1's "hash ringing" and the rest of the series (PW Crack 3-5), the theme is **vocal music** - the password crackers are all "singing" something. Trash singing = singing badly = bad password. A bit of a self-deprecating joke by the author.
- **Always run the script directly to bypass input() quirks.** Same lesson as PW Crack 1: `python3 -c "..."` is the most reliable way to feed data into Python on Windows.

## Alternative Solve Methods
1. **Patch the comparison** - change the `chr(...)` to `chr(0)` to always fail-equal-fail, or just `if True:`. Overkill.
2. **Use `ord()` in reverse** - the same lookup table, just inverted. `chr(0x64)` is "the character with code 0x64" which is `'d'`.
3. **Print the comparison at runtime** - add a `print(repr(chr(0x64) + chr(0x65) + chr(0x37) + chr(0x36)))` line before the check. Cheating but works.
4. **Just call `str_xor` with each candidate** - if the password is short (4 chars here), brute force 16^4 = 65536 candidates. Overkill but always works.
