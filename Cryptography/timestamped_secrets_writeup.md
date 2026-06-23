# Timestamped Secrets — picoCTF Writeup

**Challenge:** Timestamped Secrets  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{sa3S_sEc9t_91609b3c}`  
**Platform:** picoCTF (2026)  
**Author (challenge):** Yahaya Meddy  
**Writeup by:** zham  

---

## Description

> Someone encrypted a message using AES in ECB mode but they weren't very careful with their key. Turns out it's derived from something as simple as the current time! Can you uncover the key and decrypt the flag?
>
> Download the encrypted message: `message`
> You may also find the encryption script helpful: `code`

## Hints

> `encryption.py` is a redacted example of the program

Two files came with the challenge:

- `encryption.py` — a redacted copy of the program that produced the ciphertext
- `message.txt` — the AES-ECB ciphertext, with a hint about when it was produced

`message.txt` contents:

```
Hint: The encryption was done around 1770242610 UTC
Ciphertext (hex): 71cd3848348a45b82789f710c3321aceab2171e004200b57fe9cc64d4ea33cec
```

---

## Background Knowledge (Read This First!)

If you have never touched AES or key derivation before, read this section — it is the only theory you need for this challenge.

### 1. AES (Advanced Encryption Standard)

AES is a symmetric block cipher. "Symmetric" means the same key is used to encrypt and to decrypt. "Block" means AES chews on the input in fixed 16-byte chunks at a time. AES itself is secure, but it is only as strong as the key. Common key sizes are **128, 192, and 256 bits**. This challenge uses a **128-bit (16-byte)** key.

### 2. ECB Mode (Electronic Codebook)

ECB is the simplest AES mode: encrypt each 16-byte block independently with the same key. It is famous for being the "naive" mode because identical plaintext blocks always produce identical ciphertext blocks (you can literally see the "Tux the penguin" outline when ECB is used to encrypt a bitmap image). ECB is also the easiest mode to attack when the key is weak or guessable, which is exactly the situation here.

### 3. SHA-256 (Secure Hash Algorithm 256-bit)

A hash function turns any input into a fixed-length 256-bit (32-byte) "fingerprint." SHA-256 is one-way: you cannot recover the input from the output. It is fast to compute, which is great for verification, but a problem when people mistakenly use it as a key-derivation function. Here the author did exactly that:

```python
key = sha256(str(timestamp).encode()).digest()[:16]
```

`sha256(...).digest()` returns 32 bytes; `[:16]` keeps only the first 16 bytes. That truncation is the AES-128 key. If we can guess the `timestamp`, we can rebuild the key byte-for-byte.

### 4. Unix Timestamp

`int(time.time())` returns "seconds since 1970-01-01 00:00:00 UTC." The hint value `1770242610` corresponds to roughly **2026-02-04 22:03:30 UTC** — the moment the server claims to have run the encryption.

### 5. The Vulnerability in One Sentence

The AES key is a deterministic function of a single 32-bit-ish number (the Unix timestamp). The hint hands us that number directly, so the key has zero entropy. We just have to recompute the key and decrypt.

---

## Solution — Step by Step

I worked out of `~/ctf/timestamped-secrets` on my Kali VM.

### Step 1 — Make a Working Directory and Drop the Files In

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/timestamped-secrets && cd ~/ctf/timestamped-secrets

┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ cp /media/sf_downloads/encryption.py .
```

I always keep the original `encryption.py` for reference even though it is redacted — the parts that *are* visible are the part that matters.

### Step 2 — Read the Encryption Script

```
┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ cat encryption.py
from hashlib import sha256
import time
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad

def encrypt(plaintext: str, timestamp: int) -> str:
    timestamp = int(time.time())
    key = sha256(str(timestamp).encode()).digest()[:16]
    cipher = AES.new(key, AES.MODE_ECB)
    padded = pad(plaintext.encode(), AES.block_size)
    ciphertext = cipher.encrypt(padded)
    return ciphertext.hex()
...
```

Two things to notice:

1. The function **overwrites** the `timestamp` argument with `int(time.time())`. So the parameter is useless — whatever value you pass in, the key is derived from the *current* time when `encrypt()` is called.
2. The key is `sha256(str(timestamp).encode()).digest()[:16]`. We can rebuild it exactly if we know `timestamp`.

### Step 3 — Look at the Encrypted Message

```
┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ cat message.txt
Hint: The encryption was done around 1770242610 UTC
Ciphertext (hex): 71cd3848348a45b82789f710c3321aceab2171e004200b57fe9cc64d4ea33cec
```

The hint says "**around** 1770242610 UTC." That word "around" matters — in a clean solve the timestamp could be the exact value, or it could be off by a handful of seconds (network lag, request queuing, the script running just before the hint was generated). To be safe, I will brute-force a small window of seconds around it.

### Step 4 — Write a Brute-Force Solver in Python

```
┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ nano solve.py
```

Paste the following inside `nano`:

```python
#!/usr/bin/env python3
"""
Timestamped Secrets - picoCTF solver
Brute-force the Unix timestamp used to derive the AES-ECB key, then decrypt.
"""
from hashlib import sha256
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

CIPHERTEXT_HEX = (
    "71cd3848348a45b82789f710c3321ace"
    "ab2171e004200b57fe9cc64d4ea33cec"
)
HINT_TS = 1770242610          # "around" this Unix timestamp
WINDOW = 24 * 60 * 60         # search +/- 1 day (86 400 s)

cipher_bytes = bytes.fromhex(CIPHERTEXT_HEX)

for ts in range(HINT_TS - WINDOW, HINT_TS + WINDOW + 1):
    # Mirror the buggy encrypt(): key = SHA-256(str(ts))[:16]
    key = sha256(str(ts).encode()).digest()[:16]
    cipher = AES.new(key, AES.MODE_ECB)
    try:
        pt = unpad(cipher.decrypt(cipher_bytes), AES.block_size)
    except ValueError:
        continue  # PKCS#7 padding invalid -> wrong key
    if pt.startswith(b"picoCTF{"):
        print(f"[+] ts = {ts}  ({ts - HINT_TS:+d} s from hint)")
        print(f"[+] flag = {pt.decode()}")
        break
else:
    print("[-] No valid key found in window")
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

A few quick notes on the script:

- `unpad()` will throw `ValueError` if the PKCS#7 padding bytes are not valid for a 16-byte block. That is the cleanest "wrong key" signal — no need to second-guess the result.
- I check `pt.startswith(b"picoCTF{")` so the loop breaks on the first valid timestamp, even if multiple second-off guesses happen to decrypt to printable text.
- The `WINDOW` of ±1 day is wildly more than I need. In practice the correct timestamp is within a few seconds. I keep the wide window to be safe and because the script still finishes in well under a second.

### Step 5 — Install `pycryptodome` (if Needed)

Kali ships with Python 3, but the `Crypto` namespace comes from the `pycryptodome` package, not the older `pycrypto`. Install it if you have not already:

```
┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ pip install pycryptodome
```

### Step 6 — Run the Solver

```
┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ python3 solve.py
[+] ts = 1770242610  (+0 s from hint)
[+] flag = picoCTF{sa3S_sEc9t_91609b3c}
```

The flag fell out on the very first iteration. The hint timestamp was exact — the encryptor ran at exactly `1770242610` UTC.

---

## What Happened Internally (Timeline)

| Step | What the program did | What I did |
|------|----------------------|------------|
| 1 | Author ran `encrypt("picoCTF{...}", <ignored>)` on a server at Unix time `1770242610` (2026-02-04 22:03:30 UTC). | — |
| 2 | Inside `encrypt()`, `timestamp = int(time.time())` overwrote the parameter with `1770242610`. | Read `encryption.py` to confirm the key derivation. |
| 3 | `key = sha256("1770242610".encode()).digest()[:16]` produced 16 bytes: `2d49930affba6a8fa556d85a30dc605f`. | — |
| 4 | The plaintext was PKCS#7-padded to 32 bytes and encrypted with `AES-128-ECB`. The 32-byte ciphertext was hexlified to the 64-char string in `message.txt`. | — |
| 5 | The server logged "around 1770242610 UTC" and handed me the hex. | I brute-forced ±1 day of timestamps and tried to `unpad()` each decryption. |
| 6 | At `ts = 1770242610` the PKCS#7 padding was valid and the plaintext started with `picoCTF{`. | Solver printed the flag. |

The whole attack works because the key has **zero entropy** — a single 10-digit number controls all 16 key bytes. Anyone with the hint can decrypt the message in milliseconds.

---

## Alternative Method 1 — Skip the Brute Force, Use the Hint Directly

Because the hint gave us the exact timestamp, we do not even need a loop. One Python expression is enough:

```
┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ python3 -c "
from hashlib import sha256
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
ct  = bytes.fromhex('71cd3848348a45b82789f710c3321aceab2171e004200b57fe9cc64d4ea33cec')
key = sha256(b'1770242610').digest()[:16]
print('key  =', key.hex())
print('flag =', unpad(AES.new(key, AES.MODE_ECB).decrypt(ct), 16).decode())
"
key  = 2d49930affba6a8fa556d85a30dc605f
flag = picoCTF{sa3S_sEc9t_91609b3c}
```

I still recommend keeping the brute-force script around — if the hint is ever off by a few seconds you will be glad you have it.

---

## Alternative Method 2 — Pure OpenSSL on the Command Line

If you do not want to depend on `pycryptodome`, `openssl` can do AES-128-ECB too. The trick is that the `-K` flag expects the key as a hex string (no `0x` prefix), and `-nopad` lets us see the raw bytes including the PKCS#7 padding (which we then strip):

```
┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ KEY=2d49930affba6a8fa556d85a30dc605f

┌──(zham㉿kali)-[~/ctf/timestamped-secrets]
└─$ python3 -c "import sys; sys.stdout.buffer.write(bytes.fromhex('71cd3848348a45b82789f710c3321aceab2171e004200b57fe9cc64d4ea33cec'))" \
    | openssl enc -aes-128-ecb -d -K "$KEY" -nopad 2>/dev/null \
    | tr -d '\04'
picoCTF{sa3S_sEc9t_91609b3c}
```

Why each piece is there:

- `python3 -c "...bytes.fromhex(...).write..."` converts the hex ciphertext into raw bytes for `openssl`'s stdin.
- `openssl enc -aes-128-ecb -d -K "$KEY" -nopad` decrypts AES-128-ECB with the given key, no built-in padding handling.
- `tr -d '\04'` strips the PKCS#7 padding (the plaintext was 29 bytes, so AES added 3 bytes of `0x04`). The output is the readable flag.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Inspect `encryption.py` and `message.txt` |
| `nano` | Write the brute-force solver script |
| Python 3 | Glued the SHA-256 + AES-ECB math together |
| `pycryptodome` (`Crypto.Cipher.AES`, `Crypto.Util.Padding`) | AES decryption and PKCS#7 unpadding |
| `openssl enc -aes-128-ecb` | Alternative AES-128-ECB decryptor (no Python) |
| `tr` | Strip PKCS#7 padding bytes in the openssl alternative |

---

## Key Takeaways

- **Time-based keys are not keys.** Any secret that is a function of the current clock has the same entropy as the clock itself, which is essentially zero. A brute force of a ±1 day window runs in milliseconds.
- **SHA-256 is a hash, not a KDF.** Truncating SHA-256 to 128 bits does not magically create a "secure" AES-128 key. The output is still entirely determined by the input.
- **AES-ECB is fragile on its own.** Even with a strong key, ECB leaks block-level patterns. In this challenge ECB does not make the attack possible, but it does make the attack trivial — no IVs, no chaining, no authentication, just pure block-by-block math.
- **The "around" wording in the hint is generous.** Real challenge hints can be off by a few seconds, so always write a solver that tolerates a small window. ±1 day is overkill, but cheap to search.
- **Always re-derive the key exactly the same way the encryptor did.** Off-by-one mistakes (e.g. forgetting to truncate to 16 bytes, or hashing `bytes(timestamp)` instead of `str(timestamp).encode()`) will silently produce wrong keys and waste your time.

### Flag Wordplay Decode

The flag is `picoCTF{sa3S_sEc9t_91609b3c}`. Reading the inner word in two parts:

- `sa3S` → `s + a3 + S` → `s + AES + S`. Reading "sAES" out loud sounds like **"secrets."** It is a pun on the cipher that was actually used (AES).
- `sEc9t` → leet-speak for **"secret"** (9 stands in for `t`).

So the body decodes to **"sAES sEcret"** — a tongue-in-cheek "AES secret" / "secrets secret," which fits the challenge title "Timestamped Secrets" perfectly: the entire flag is literally a secret that the AES-ECB key was derived from the timestamp. The trailing `91609b3c` is just a per-flag uniqueness nonce.
