# PW Crack 1 — picoCTF Writeup  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** 200  
**Flag:** `picoCTF{545h_r1ng1ng_fa343060}`  
**Platform:** picoMini 2022  
**Writeup by:** zham  

## Description
> Can you crack the password to get the flag?  
>   
> Download the password checker here and you'll need the encrypted flag in the same directory too.

## Hints
> 1. To view the file in the webshell, do: `$ nano level1.py`  
> 2. To exit nano, press Ctrl and x and follow the on-screen prompts.  
> 3. The `str_xor` function does not need to be reverse engineered for this challenge.

## Background Knowledge
This is a "password cracking" challenge, but it's not really about cracking - the password is **hardcoded in the source code** as a plaintext string. The challenge tests whether you actually read the code instead of trying to brute-force or reverse-engineer the cipher.

The `str_xor` is a simple XOR cipher that takes the encrypted flag and the user-supplied password, repeats the password to match the flag's length, and XORs them byte-by-byte. With the right password, this recovers the plaintext flag. Without the right password, the XOR produces garbage.

Hint 3 explicitly says you don't need to reverse the cipher - you just need to find the password in the source.

## Solution

### Step 1 — Download the files
The challenge gives us two files: `level1.py` (the password checker) and `level1.flag.txt.enc` (the encrypted flag). Put both in the same directory.

```bash
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads && ls level1*
level1.py
level1.flag.txt.enc
```

### Step 2 — Read the source with nano
The hint tells us to use `nano` to look at `level1.py`.

```bash
┌──(zham㉿kali)-[~]
└─$ nano level1.py
```

The relevant snippet:
```python
def level_1_pw_check():
    user_pw = input("Please enter correct password for flag: ")
    if( user_pw == "1e1a"):
        print("Welcome back... your flag, user:")
        decryption = str_xor(flag_enc.decode(), user_pw)
        print(decryption)
        return
    print("That password is incorrect")
```

The password is right there in the source: `1e1a`. Plaintext, hardcoded string comparison. Save with `Ctrl+X`.

### Step 3 — Run the checker with the password
Pipe the password into the script.

```bash
┌──(zham㉿kali)-[~]
└─$ echo "1e1a" | python3 level1.py
Please enter correct password for flag: Welcome back... your flag, user:
picoCTF{545h_r1ng1ng_fa343060}
```

Got it. The flag is on the last line.

### Alternative: skip the input() and call str_xor directly
If piping doesn't behave nicely (e.g. on Windows where `echo` adds a stray `\r`), just call the decrypt function straight from the script.

```bash
┌──(zham㉿kali)-[~]
└─$ python3 -c "
flag_enc = open('level1.flag.txt.enc', 'rb').read()
def str_xor(secret, key):
    new_key = key
    i = 0
    while len(new_key) < len(secret):
        new_key = new_key + key[i]
        i = (i + 1) % len(key)
    return ''.join([chr(ord(s) ^ ord(k)) for (s, k) in zip(secret, new_key)])
print(str_xor(flag_enc.decode(), '1e1a'))
"
picoCTF{545h_r1ng1ng_fa343060}
```

### Step 4 — Submit

```bash
┌──(zham㉿kali)-[~]
└─$ echo "picoCTF{545h_r1ng1ng_fa343060}"
picoCTF{545h_r1ng1ng_fa343060
```

Pasted into the picoCTF flag box. Accepted.

## What Happened Internally
1. The challenge's `level1.py` reads the encrypted flag from `level1.flag.txt.enc` (30 bytes of XOR-encrypted text), then asks the user for a password.
2. If the user-supplied password equals the hardcoded literal `"1e1a"`, the script calls `str_xor(flag_enc.decode(), "1e1a")` to decrypt and print the flag.
3. The XOR cipher is: for each byte of the flag, XOR with the corresponding byte of the (cyclically-repeated) password. With password `1e1a` (bytes `0x31 0x65 0x31 0x61`), this reverses the encryption.
4. The password was never meant to be cracked - it was meant to be **read** in the source. Anyone with a text editor gets it in 5 seconds.

## Tools Used
| Tool | Purpose |
|------|---------|
| `nano` | View the source (as the hint suggested) |
| `python3` | Run the checker, or call str_xor directly |
| `echo` + pipe | Feed the password to `input()` |
| `cat` (optional) | Eyeball the encrypted flag |

## Key Takeaways
- **Always read the source first.** Password-checking challenges in CTFs often leak the answer in plaintext. Don't reach for hashcat until you've grepped for `if user_pw ==`.
- **Hardcoded credentials are a real bug class.** Any "secret" embedded in source code (or in a config file, or in a Git history) is not a secret. This challenge teaches that lesson the gentle way.
- **str_xor is a toy cipher.** XOR with a short repeating key is what you do for hiding a flag from a `cat` of the binary, not for actual security. It breaks to known-plaintext in one shot (the flag starts with `picoCTF{`).
- **Wordplay decode**: `545h_r1ng1ng` = "hash ringing" in 1337 (5=s, 4=a, 1=i, h=h, r=r, n=n, g=g). A "hash ring" is a real thing - a consistent hash ring used in distributed systems. The trailing `fa343060` is per-instance noise.
- **Echo + pipe on Windows can fail.** `echo "1e1a"` on Windows PowerShell can output `1e1a` with no trailing newline by default, but the legacy `cmd` echo can add `\r\n`. When in doubt, call the decrypt function directly with `python3 -c` rather than fighting the shell.

## Alternative Solve Methods
1. **Brute force** - 4-char hex password is only 16^4 = 65536 candidates. Fast to brute force if you can't read the source. But overengineered.
2. **Known-plaintext attack on the XOR** - the flag starts with `picoCTF{`, so XOR the first 8 bytes of the .enc with `picoCTF{` to recover the key prefix, then decrypt. Works without ever reading the source.
3. **Patch the comparison** - change `if( user_pw == "1e1a"):` to `if True:` so any input succeeds. Overkill for this challenge.
