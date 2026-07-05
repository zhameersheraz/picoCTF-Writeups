# unpackme.py — picoCTF Writeup

**Challenge:** unpackme.py
**Category:** Reverse Engineering
**Difficulty:** Medium
**Points:** 100
**Flag:** `picoCTF{175_chr157m45_5274ff21}`
**Platform:** picoCTF 2023
**Writeup by:** zham

---

## Description

> Can you get the flag?
>
> Reverse engineer this `Python` program.

## Hints

**Hint 1:** packing

---

## Background Knowledge

Before we start reversing the script, here are a few concepts worth knowing.

**1. Python's `exec()` function.**
`exec()` runs a string as if it were Python code. When you see `exec(plain.decode())` at the bottom of a script, the file is hiding its real logic inside an encrypted blob and only runs it after decryption. The "real" program never lives in the file's source — you have to decrypt to see it.

**2. Fernet symmetric encryption.**
Fernet is part of Python's `cryptography` library. It takes a 32-byte key (base64-encoded into a 44-character URL-safe string) and produces a token that contains the ciphertext, an IV, a timestamp, and an HMAC for integrity. Without the key you cannot read the payload; with the key it is one line to decrypt.

**3. `base64.b64encode` vs `base64.b64decode`.**
The script does **not** base64-decode the password to make a Fernet key. It `b64encode`s the 32-byte string `correctstaplecorrectstaplecorrec` and feeds the resulting base64 text straight to `Fernet(...)`. Fernet only accepts base64-encoded 32-byte keys, so this round-trip is just to put the bytes into the format Fernet expects.

**4. Why we don't need to run `exec`.**
The whole point of this challenge is that running the script blind asks you for a password you don't know. Instead, we replicate the same Fernet decryption in our own script and **print** the plaintext instead of executing it. That gives us the hidden source code (including the password and the flag) without any risk of side effects.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The downloaded challenge file is `unpackme.py`.

### 1. Move the file into a working directory and look at it

```
┌──(zham㉿kali)-[~/unpackme]
└─$ mkdir -p ~/unpackme && cd ~/unpackme

┌──(zham㉿kali)-[~/unpackme]
└─$ cp ~/Downloads/unpackme.py .

┌──(zham㉿kali)-[~/unpackme]
└─$ cat unpackme.py
```

```python
import base64
from cryptography.fernet import Fernet



payload = b'gAAAAABkzWGO_8MlYpNM0n0o718LL-w9m3rzXvCMRFghMRl6CSZwRD5DJOvN_jc8TFHmHmfiI8HWSu49MyoYKvb5mOGm_Jn4kkhC5fuRiGgmwEpxjh0z72dpi6TaPO2TorksAd2bNLemfTaYPf9qiTn_z9mvCQYV9cFKK9m1SqCSr4qDwHXgkQpm7IJAmtEJqyVUfteFLszyxv5-KXJin5BWf9aDPIskp4AztjsBH1_q9e5FIwIq48H7AaHmR8bdvjcW_ZrvhAIOInm1oM-8DjamKvhh7u3-lA=='

key_str = 'correctstaplecorrectstaplecorrec'
key_base64 = base64.b64encode(key_str.encode())
f = Fernet(key_base64)
plain = f.decrypt(payload)
exec(plain.decode())
```

Three things jump out:

- A hard-coded Fernet `payload` blob.
- A hard-coded key string `correctstaplecorrectstaplecorrec`.
- `exec(plain.decode())` at the end — meaning the *real* program is inside the encrypted blob.

### 2. Decrypt the payload without executing it

The cleanest approach is to copy the same five lines into a new file, swap the `exec(...)` for `print(...)`, and let Python show us what would have been executed.

```
┌──(zham㉿kali)-[~/unpackme]
└─$ nano decrypt.py
```

In `nano` I pasted:

```python
import base64
from cryptography.fernet import Fernet

payload = b'gAAAAABkzWGO_8MlYpNM0n0o718LL-w9m3rzXvCMRFghMRl6CSZwRD5DJOvN_jc8TFHmHmfiI8HWSu49MyoYKvb5mOGm_Jn4kkhC5fuRiGgmwEpxjh0z72dpi6TaPO2TorksAd2bNLemfTaYPf9qiTn_z9mvCQYV9cFKK9m1SqCSr4qDwHXgkQpm7IJAmtEJqyVUfteFLszyxv5-KXJin5BWf9aDPIskp4AztjsBH1_q9e5FIwIq48H7AaHmR8bdvjcW_ZrvhAIOInm1oM-8DjamKvhh7u3-lA=='

key_str = 'correctstaplecorrectstaplecorrec'
key_base64 = base64.b64encode(key_str.encode())
f = Fernet(key_base64)
plain = f.decrypt(payload)
print(plain.decode())
```

Save and exit: `Ctrl+O`, `Enter`, then `Ctrl+X`.

```
┌──(zham㉿kali)-[~/unpackme]
└─$ python3 decrypt.py
```

Output:

```
pw = input('What's the password? ')

if pw == 'batteryhorse':
  print('picoCTF{175_chr157m45_5274ff21}')
else:
  print('That password is incorrect.')
```

Now we have the password (`batteryhorse`) and the flag in plain sight.

### 3. Confirm the flag by running the real script

To prove the flag actually works, I ran the original `unpackme.py` and fed it the password:

```
┌──(zham㉿kali)-[~/unpackme]
└─$ echo "batteryhorse" | python3 unpackme.py
What's the password? picoCTF{175_chr157m45_5274ff21}
```

The flag matches: `picoCTF{175_chr157m45_5274ff21}`.

---

## Alternative Solves

If you don't want to touch the script, a one-liner works just as well. The `-c` flag lets you pass Python code directly so you don't have to create `decrypt.py`:

```
┌──(zham㉿kali)-[~/unpackme]
└─$ python3 -c "import base64; from cryptography.fernet import Fernet; \
key=base64.b64encode(b'correctstaplecorrectstaplecorrec'); \
print(Fernet(key).decrypt(b'gAAAAABkzWGO_8MlYpNM0n0o718LL-w9m3rzXvCMRFghMRl6CSZwRD5DJOvN_jc8TFHmHmfiI8HWSu49MyoYKvb5mOGm_Jn4kkhC5fuRiGgmwEpxjh0z72dpi6TaPO2TorksAd2bNLemfTaYPf9qiTn_z9mvCQYV9cFKK9m1SqCSr4qDwHXgkQpm7IJAmtEJqyVUfteFLszyxv5-KXJin5BWf9aDPIskp4AztjsBH1_q9e5FIwIq48H7AaHmR8bdvjcW_ZrvhAIOInm1oM-8DjamKvhh7u3-lA==').decode())"
```

Same decrypted source comes out.

If you have access to a `.pyc` (compiled bytecode) version of the challenge instead of the `.py`, you can decompile it directly with `uncompyle6` or `decompyle3` and skip the Fernet dance entirely. That's the more common "unpack" route for packed Python — here the source is plaintext, so Fernet is the obstacle.

---

## What Happened Internally

1. The script imported `base64` and `Fernet`.
2. It base64-encoded the 32-byte string `correctstaplecorrectstaplecorrec` to produce the Fernet-shaped key `Y29ycmVjdHN0YXBsZWNvcnJlY3RzdGFwbGVjb3JyZWM=`.
3. `Fernet(key)` built a cipher object from that key.
4. `f.decrypt(payload)` ran AES-CBC + HMAC verification and returned the original plaintext bytes (the inner Python program).
5. `plain.decode()` turned the bytes into a string.
6. `exec(string)` ran that string as live Python — prompting for a password, comparing it to `batteryhorse`, and printing the flag if it matched.

By replacing step 6 with `print(...)` we stopped the program at step 5 and read its source instead of letting it run.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `cat`            | Quick read of `unpackme.py`.                                   |
| `nano`           | Wrote the small `decrypt.py` helper.                           |
| `python3`        | Imported `cryptography.fernet`, decrypted the payload.         |
| `cryptography`   | Provided the `Fernet` primitive for symmetric decryption.      |
| `base64`         | Converted the raw 32-byte key into a Fernet-compatible string. |
| `echo + pipe`    | Sent the recovered password into `unpackme.py` non-interactively. |
| `uncompyle6` / `decompyle3` (optional) | Mentioned for the `.pyc` variant of the challenge. |

---

## Key Takeaways

- `exec(plaintext)` is a red flag in any script you're asked to "just run." Treat it like `eval` — read the input before letting it execute.
- Fernet tokens always start with the bytes `gAAAAA` (the version byte). If you see that pattern in a challenge, you're probably one `decrypt` call away from the answer.
- The decryption key isn't always hidden — sometimes it sits in plain sight in the same file as the ciphertext. Don't assume you need to brute-force anything.
- Swapping `exec` for `print` is a useful habit when reversing obfuscated Python. It gives you the source instead of the side effects.
- The flag's leetspeak: `175_chr157m45` reads as **its_christmas** (1→i, 7→t, 5→s, then `chr` + 1→i, 5→s, 7→t, `m`, 4→a, 5→s). So the flag's wordplay is `picoCTF{its_christmas_<hash>}` — a holiday greeting from picoCTF.
