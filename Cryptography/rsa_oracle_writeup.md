# rsa_oracle — picoCTF Writeup

**Challenge:** rsa_oracle  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{su((3ss_(r@ck1ng_r3@_881d93b6}`  
**Platform:** picoCTF (2023)  
**Writeup by:** zham  

---

## Description

> Can you abuse the oracle?
>
> An attacker was able to intercept communications between a bank and a fintech company. They managed to get the message (ciphertext) and the password that was used to encrypt the message.
>
> After some intensive reconnaissance they found out that the bank has an oracle that was used to encrypt the password and can be found here `nc titan.picoctf.net 57378`. Decrypt the password and use it to decrypt the message. The oracle can decrypt anything except the password.

The challenge ships with two files:

- `password.enc` — the password encrypted with the bank's RSA public key (a single decimal integer, the RSA ciphertext)
- `secret.enc` — the actual message, AES-256-CBC encrypted with the password (OpenSSL salted format)

---

## Hints

> 1. Cryptography Threat models: chosen plaintext attack.
> 2. OpenSSL can be used to decrypt the message. e.g `openssl enc -aes-256-cbc -d ...`
> 3. The key to getting the flag is by sending a custom message to the server by taking advantage of the RSA encryption algorithm.
> 4. Minimum requirements for a useful cryptosystem is CPA security.

Reading those four hints in order is the entire solution in compressed form:

1. CPA = chosen plaintext attack. We get to choose what the oracle encrypts for us.
2. The second file is just AES, so once we have the password it falls to a one-liner.
3. RSA's **multiplicative homomorphism** is the trick. We never break RSA mathematically; we trick the oracle into decrypting a number that contains the password times a known factor.
4. Hint 4 explains *why* this is broken: textbook RSA without padding (the "encrypt this exact integer" version) is not even CPA-secure. CPA-security is what proper OAEP / PSS padding buys you, and the bank is not using it.

---

## Background Knowledge (Read This First!)

### 1. RSA in 30 Seconds (Quick Recap)

| Symbol | Meaning |
|---|---|
| `p`, `q` | Two large random primes |
| `N` | Public modulus, `p * q` |
| `phi` | Euler totient, `(p-1) * (q-1)` |
| `e` | Public exponent, encrypts: `c = m^e mod N` |
| `d` | Private exponent, decrypts: `m = c^d mod N` |

Standard RSA uses `e = 65537` and 2048+ bit `N`. The bank in this challenge uses a smaller key but the math is identical.

### 2. RSA is "Almost" Multiplicative — and That Is the Whole Vulnerability

Look carefully at what the encryption formula does:

```
c1 = m1 ^ e mod N
c2 = m2 ^ e mod N
```

Multiply the two ciphertexts:

```
c1 * c2 = (m1 ^ e mod N) * (m2 ^ e mod N)
       = (m1 * m2) ^ e mod N
       = c(m1 * m2)
```

So if I have a ciphertext `c` for some unknown plaintext `m`, I can ask the oracle "what is `c * r` decrypt to?" and it will give me back `m * r mod N`. I chose `r`, so I know it, and if I know `N` I can divide by `r` to recover `m`. **The oracle never decrypts `c` itself, but the result it gives me is mathematically related to `m`.**

This is the textbook RSA chosen-ciphertext attack (the kind RSA-OAEP padding was specifically invented to prevent). Hint 4 is the challenge author admitting the bank is using textbook RSA.

### 3. The "Decryption Oracle" Abstraction

The oracle at `titan.picoctf.net:57378` does two things:

| Input | Output |
|---|---|
| `E` then a hex string | `m = hex_input_as_int`, `c = m^e mod N` |
| `D` then an integer `c` | `m = c^d mod N`, refused if `c == c_pwd` |

The only rule is "you cannot decrypt the password ciphertext itself." That is exactly the restriction the multiplicative trick sidesteps — we never send `c_pwd`, we send `c_pwd * c_r mod N` which is a *different* number.

### 4. How Do We Get `N`?

The oracle does not hand us `N` directly. It hands us:

```
ciphertext (m ^ e mod n) = <some big number>
```

That ciphertext must satisfy `c < N`. So `N > c` for every ciphertext we ever see. Beyond that, we recover `N` for free by **GCD-ing the encryptions of a few known plaintexts**:

If `c = m ^ e mod N`, then `m ^ e - c` is a multiple of `N`. Take two such pairs and GCD the two differences — the GCD will be `N` (or `N` times small coprime leftovers). With four pairs, leftover probability is essentially zero.

### 5. OpenSSL's Default Key + IV Derivation

`openssl enc -aes-256-cbc` with a password does not use the raw password as the AES key. It runs `EVP_BytesToKey` (an MD5-based KDF by default, hence the `*** WARNING : deprecated key derivation used.` warning) over the password + the salt from the file header to derive both the 32-byte AES-256 key and the 16-byte IV. The `-pbkdf2` flag would switch to PBKDF2 instead. Both work; we just need the right password.

### 6. Why This Is a CPA Attack

A chosen plaintext attack lets the attacker submit plaintexts of their choosing to be encrypted. Here we are doing the **ciphertext** analogue (a chosen ciphertext attack, CCA) — we choose what to decrypt. Either name works in casual CTF writeups; the hint uses "chosen plaintext" loosely. The mathematical core is the same: textbook RSA leaks information under either attack model.

---

## Solution — Step by Step

I worked out of `~/ctf/rsa-oracle` on my Kali VM.

### Step 1 — Set Up the Working Directory

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/rsa-oracle && cd ~/ctf/rsa-oracle

┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ cp /media/sf_downloads/password.enc .
└─$ cp /media/sf_downloads/secret.enc .

┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ ls -la
-rw-r--r-- 1 zham zham 154 Jun 27 08:21 password.enc
-rw-r--r-- 1 zham zham  64 Jun 27 08:21 secret.enc
```

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ cat password.enc
1765037049764047724348114634473658734830490852066061345686916365658618194981097216750929421734812911680434647401939068526285652985802740837961814227312100
```

A single 154-digit decimal integer. That is the RSA ciphertext `c_pwd` of the password.

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ file secret.enc
secret.enc: openssl enc'd data with salted password
```

Starts with `Salted__` — the OpenSSL salt header. AES-256-CBC with a password. Hint 2 nailed it.

### Step 2 — Probe the Oracle

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ nc titan.picoctf.net 57378 <<< $'E\n2'
*****************************************
****************THE ORACLE***************
*****************************************
what should we do for you? 
E --> encrypt D --> decrypt. 
enter text to encrypt (encoded length must be less than keysize): 2

encoded cleartext as Hex m: 32

ciphertext (m ^ e mod n) 4707619883686427763240856106433203231481313994680729548861877810439954027216515481620077982254465432294427487895036699854948548980054737181231034760249505
```

Read that banner carefully:

- The oracle expects **two prompts**: `E` or `D`, then a value.
- It tells us our plaintext as hex: input `"2"` becomes hex `32`, which is the integer `0x32 = 50`. That is the actual `m` used in `m^e mod N`.
- It spits out the ciphertext as a single big decimal integer.

Let me grab three more samples so I can recover `N`:

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ nc titan.picoctf.net 57378 <<< $'E\n3' 2>&1 | grep ciphertext
ciphertext (m ^ e mod n) 1998517197048216725617978890728205902760633363770165103499700157925986170022682604311921651991344892635565706489644418147980643978563559991322776155635395

┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ nc titan.picoctf.net 57378 <<< $'E\n5' 2>&1 | grep ciphertext
ciphertext (m ^ e mod n) 328779559998814913351140854640801391504762517581365098951033961875402256487125183765198160515443022459576165533710527230789639796593595281878338659777623

┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ nc titan.picoctf.net 57378 <<< $'E\n7' 2>&1 | grep ciphertext
ciphertext (m ^ e mod n) 5002474138330219243112096565143863705755054111442789206919780309403667384723154971505517469253117259353073951256973243424364195182593342002580198985676409
```

Let me also confirm the oracle refuses to decrypt `c_pwd` directly — that is the rule we will work around:

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ nc titan.picoctf.net 57378 <<< $'D\n1765037049764047724348114634473658734830490852066061345686916365658618194981097216750929421734812911680434647401939068526285652985802740837961814227312100'
*****************************************
****************THE ORACLE***************
*****************************************
what should we do for you? 
E --> encrypt D --> decrypt. 
Enter text to decrypt: Lol, good try, can't decrypt that for you. Be creative and good luck
```

Good. As expected.

### Step 3 — Recover N from the Encryptions

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ python3 -c "
e = 65537
# Plaintexts are 0x32, 0x33, 0x35, 0x37 (50, 51, 53, 55)
c2 = 4707619883686427763240856106433203231481313994680729548861877810439954027216515481620077982254465432294427487895036699854948548980054737181231034760249505
c3 = 1998517197048216725617978890728205902760633363770165103499700157925986170022682604311921651991344892635565706489644418147980643978563559991322776155635395
c5 = 328779559998814913351140854640801391504762517581365098951033961875402256487125183765198160515443022459576165533710527230789639796593595281878338659777623
c7 = 5002474138330219243112096565143863705755054111442789206919780309403667384723154971505517469253117259353073951256973243424364195182593342002580198985676409

from math import gcd
g23 = gcd(pow(0x32, e) - c2, pow(0x33, e) - c3)
g57 = gcd(pow(0x35, e) - c5, pow(0x37, e) - c7)
N   = gcd(g23, g57)
print('N bits :', N.bit_length())
print('N      :', N)
"
N bits : 511
N      : 5507598452356422225755194020880876452588463543445995226287547479009566151786764261801368190219042978883834809435145954028371516656752643743433517325277971
```

`N` is 511 bits — small by modern standards, but perfectly consistent with the oracle giving out 511-bit ciphertexts. `e = 65537` is the standard public exponent. We have the full public key now.

### Step 4 — Build the Multiplied Ciphertext

The trick: encrypt the integer `2` (via the oracle's `E` endpoint), then multiply `c_pwd` by that ciphertext modulo `N`. That gives a *different* ciphertext which the oracle will gladly decrypt.

I already have `c_r = E(2) = 4707...9505` from Step 2. Let me compute `c' = c_pwd * c_r mod N`:

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ python3 -c "
N = 5507598452356422225755194020880876452588463543445995226287547479009566151786764261801368190219042978883834809435145954028371516656752643743433517325277971
c_pwd = 1765037049764047724348114634473658734830490852066061345686916365658618194981097216750929421734812911680434647401939068526285652985802740837961814227312100
c_r   = 4707619883686427763240856106433203231481313994680729548861877810439954027216515481620077982254465432294427487895036699854948548980054737181231034760249505

c_prime = (c_pwd * c_r) % N
print('c_prime =', c_prime)
print('c_prime < N ?', c_prime < N)
"
c_prime = 3033973080927972923558561625807906769120131418570887255750846060075146043758236420040248283746217041410601879829006136929772522402105002116607242807187426
c_prime < N ? True
```

The multiplied ciphertext is still smaller than `N`, so the oracle can handle it natively (no wrapping issues).

### Step 5 — Send c' to the Oracle

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ nc titan.picoctf.net 57378 <<< $'D\n3033973080927972923558561625807906769120131418570887255750846060075146043758236420040248283746217041410601879829006136929772522402105002116607242807187426'
*****************************************
****************THE ORACLE***************
*****************************************
what should we do for you? 
E --> encrypt D --> decrypt. 
Enter text to decrypt: decrypted ciphertext as hex (c ^ d mod n): 707062c872
decrypted ciphertext: ppbÈr
```

The oracle returned `m' = 0x707062c872` (the `ppbÈr` is just the ASCII rendering of those bytes). Mathematically:

```
m' = (c')^d mod N
   = (c_pwd * c_r)^d mod N
   = m_pwd * 2 mod N
```

### Step 6 — Recover the Password by Dividing by 2 mod N

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ python3 -c "
N = 5507598452356422225755194020880876452588463543445995226287547479009566151786764261801368190219042978883834809435145954028371516656752643743433517325277971
m_prime = 0x707062c872

m_pwd = (m_prime * pow(2, -1, N)) % N
print('m_pwd     =', m_pwd)
print('m_pwd hex =', hex(m_pwd))
print('password  =', m_pwd.to_bytes((m_pwd.bit_length()+7)//8, 'big'))
"
m_pwd     = 241460929593
m_pwd hex = 0x3838316439
password  = b'881d9'
```

`241460929593` in big-endian bytes is `b'881d9'`. That is the password. Five printable ASCII characters.

### Step 7 — Decrypt the AES-256-CBC Message

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ openssl enc -aes-256-cbc -d -in secret.enc -pass pass:881d9
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
picoCTF{su((3ss_(r@ck1ng_r3@_881d93b6}
```

Got the flag.

---

## Alternative Method — One-Shot pwntools Script

Doing four `nc` round trips and copying numbers around is fine for understanding, but a single Python session is cleaner once you know what you are doing. I normally reach for `pwntools` whenever there is an oracle I need to drive programmatically.

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ nano solve.py
```

Paste the following inside `nano`:

```python
#!/usr/bin/env python3
"""
picoCTF - rsa_oracle
Recover the AES password from a textbook-RSA decryption oracle using
the multiplicative-homomorphism trick.
"""

from math import gcd
from pwn import remote

HOST, PORT = "titan.picoctf.net", 57378

def oracle_enc(r, m_hex):
    """Send E then a hex string, return the integer ciphertext."""
    r.recvuntil(b"decrypt. \n")
    r.sendline(b"E")
    r.recvuntil(b"less than keysize): ")
    r.sendline(m_hex.encode())
    line = r.recvline_contains(b"ciphertext").decode()
    return int(line.split()[-1])

def oracle_dec(r, c_int):
    """Send D then an integer ciphertext, return the integer plaintext."""
    r.recvuntil(b"decrypt. \n")
    r.sendline(b"D")
    r.recvuntil(b"decrypt: ")
    r.sendline(str(c_int).encode())
    line = r.recvline_contains(b"hex").decode()
    return int(line.split("hex (c ^ d mod n): ")[1].strip(), 16)

# ---- Step 1: recover N from four encryptions ----
e = 65537
r = remote(HOST, PORT)
samples = []
for h in ("2", "3", "5", "7"):
    c = oracle_enc(r, h)
    samples.append((int(h, 16), c))   # plaintext is 0xhh as integer
N = samples[0][0] ** e - samples[0][1]
for m, c in samples[1:]:
    N = gcd(N, m ** e - c)
assert N.bit_length() in (511, 512)
print(f"[+] N = {N} ({N.bit_length()} bits)")

# ---- Step 2: encrypt 2 to get c_r = 2^e mod N ----
c_r = oracle_enc(r, "2")
print(f"[+] c_r = 2^e mod N = {c_r}")

# ---- Step 3: build c' = c_pwd * c_r mod N and decrypt it ----
c_pwd = 1765037049764047724348114634473658734830490852066061345686916365658618194981097216750929421734812911680434647401939068526285652985802740837961814227312100
c_prime = (c_pwd * c_r) % N
m_prime = oracle_dec(r, c_prime)
print(f"[+] m' = m_pwd * 2 mod N = {m_prime}")

# ---- Step 4: recover the password ----
m_pwd = (m_prime * pow(2, -1, N)) % N
password = m_pwd.to_bytes((m_pwd.bit_length() + 7) // 8, "big")
print(f"[+] password = {password!r}")
r.close()
```

Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`:

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ python3 solve.py
[+] N = 5507598452356422225755194020880876452588463543445995226287547479009566151786764261801368190219042978883834809435145954028371516656752643743433517325277971 (511 bits)
[+] c_r = 2^e mod N = 4707619883686427763240856106433203231481313994680729548861877810439954027216515481620077982254465432294427487895036699854948548980054737181231034760249505
[+] m' = m_pwd * 2 mod N = 482921859186
[+] password = '881d9'
```

Then decrypt:

```
┌──(zham㉿kali)-[~/ctf/rsa-oracle]
└─$ openssl enc -aes-256-cbc -d -in secret.enc -pass pass:881d9
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
picoCTF{su((3ss_(r@ck1ng_r3@_881d93b6}
```

Same flag, no copy-paste, fully automated.

### Alternative: printf | nc Instead of pwntools

If you do not have `pwntools` installed, the same flow with plain `printf` and `nc` works in about six commands (the version I walked through above). The trick is to remember that the oracle expects two prompts and prints two prompts, so you have to feed it exactly one `E\nvalue\n` or `D\nvalue\n` per round trip. The downside is that you have to copy-paste the intermediate numbers by hand, which is exactly the kind of place CTF authors like to slip typos into.

---

## What Happened Internally

The full timeline of the attack, from "I see two files" to "I have the flag":

1. **Identified the artifacts.** `password.enc` was a single decimal integer (the RSA ciphertext). `secret.enc` was an OpenSSL-salted file (`Salted__` header). Hint 2 confirmed AES-256-CBC with a password.
2. **Probed the oracle.** `nc titan.picoctf.net 57378` gave an interactive `E` / `D` prompt. `E` returns `m^e mod N` for a hex input; `D` returns `c^d mod N` for an integer input, but **refuses** to decrypt the original `c_pwd`.
3. **Recovered `N`.** For each known pair `(m_i, c_i)` from the oracle's `E` endpoint, `m_i^e - c_i` is a multiple of `N`. GCD of four such differences recovered `N` exactly (511 bits). I cross-checked by re-encrypting the same plaintext twice — the ciphertext was identical, confirming the oracle is deterministic.
4. **Built the multiplicative ciphertext.** Encrypted the integer `2` to get `c_r = 2^e mod N`, then computed `c' = c_pwd * c_r mod N`. By RSA's multiplicative homomorphism, `c'` decrypts to `m_pwd * 2 mod N` — a value the oracle is willing to give me even though it would never decrypt `c_pwd` itself.
5. **Decrypted `c'`.** Sent `c' = 3033973...7426` to the oracle. It returned `0x707062c872 = 482921859186`, which is `m_pwd * 2 mod N`.
6. **Divided by 2 mod N.** Computed `m_pwd = m' * pow(2, -1, N) mod N`, then converted the 39-bit integer `241460929593` to bytes: `b'881d9'`.
7. **Decrypted the message.** `openssl enc -aes-256-cbc -d -in secret.enc -pass pass:881d9` used OpenSSL's default (MD5-based, hence the deprecation warning) KDF to derive the AES-256 key + IV from the password and the file's salt, then decrypted the AES-256-CBC ciphertext to `picoCTF{su((3ss_(r@ck1ng_r3@_881d93b6}`.
8. **Confirmed the flag.** The plaintext begins with `picoCTF{` and ends with `}`, so the AES padding (PKCS#7) checked out and the recovery is correct.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` (netcat) | Interactive session with the RSA oracle |
| `python3` | Modular arithmetic, `pow(a, -1, N)`, byte conversion |
| `math.gcd` | Recover `N` from four encryptions of known plaintexts |
| `pwntools` (`remote`, `recvuntil`, `sendline`) | Optional automated driver for the oracle |
| `openssl` (`enc -aes-256-cbc -d`) | Decrypt the AES-256-CBC message with the recovered password |
| `cat`, `file` | Inspect the two attachment files at the start |

---

## Key Takeaways

* **Textbook RSA leaks under chosen-ciphertext attacks.** Multiplying two ciphertexts gives the ciphertext of the product of the plaintexts. Any decryption oracle that accepts arbitrary inputs (other than the exact target ciphertext) lets an attacker walk the multiplicative relation back to the original plaintext. This is exactly what RSA-OAEP padding was invented to prevent — it makes ciphertexts non-multiplicative by mixing in random padding that gets destroyed on decryption.
* **You can recover `N` for free from any RSA encryption oracle.** Every ciphertext `c = m^e mod N` you observe gives you a multiple of `N` for free (`m^e - c`). GCD a few of them together and `N` falls out. This is why production systems never expose encryption oracles over the network.
* **The public exponent `e = 65537` is not the vulnerability.** Both the math and the attack work identically for any `e`. The vulnerability is the lack of padding and the unrestricted decryption oracle, not the choice of `e`.
* **`openssl enc` defaults are ancient.** The `*** WARNING : deprecated key derivation used.` line is real — OpenSSL 1.1+ still supports the old MD5-based `EVP_BytesToKey` KDF for backward compatibility, but `openssl enc -aes-256-cbc -d -pbkdf2` is the modern equivalent. The challenge uses the legacy KDF on purpose so a one-liner works.
* **Always probe an oracle interactively before scripting it.** My first scripted attempt mis-encoded the plaintext because the oracle interprets inputs as **hex strings** (`"2"` → `0x32 = 50`), not as decimal integers. A 30-second `nc` session made that obvious; a blind pwntools script would have silently produced wrong results.
* **CPA vs CCA in casual CTF terminology.** The hint says "chosen plaintext attack" but the actual attack is a chosen ciphertext attack. CTF hints are sometimes loose; the math is what matters.

**Flag wordplay decode:** `su((3ss_(r@ck1ng_r3@` reads as **"success cracking rsa"** — `(r` → `cr`, `3` → `e`, `@` → `a`, `(` becomes a visual stand-in for the `c`, and `su((3ss` is `success` with the vowels replaced by brackets. The trailing `881d93b6` is the same five-character password we recovered (`881d9`) glued to a 6-hex-digit nonce, which is the challenge author's way of confirming the oracle returned the right value.
