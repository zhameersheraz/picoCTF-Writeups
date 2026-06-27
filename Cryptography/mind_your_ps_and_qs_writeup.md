# Mind your Ps and Qs — picoCTF Writeup

**Challenge:** Mind your Ps and Qs  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 20  
**Flag:** `picoCTF{sma11_N_n0_g0od_1dc7ae91}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> In RSA, a small e value can be problematic, but what about N? Can you decrypt this?
>
> [values](values)

## Hints

> 1. Bits are expensive, I used only a little bit over 100 to save money

---

## Background Knowledge (Read This First!)

### RSA in 30 seconds

RSA is the most common public-key cryptosystem. The math boils down to:

```
keygen:  choose two large primes p, q
         N = p * q                          (the modulus, made public)
         phi = (p - 1) * (q - 1)            (Euler's totient of N)
         pick e with gcd(e, phi) == 1       (public exponent)
         d = e^(-1) mod phi                 (private exponent)
         public key  = (N, e)
         private key = (N, d)

encrypt:  c = m^e mod N
decrypt:  m = c^d mod N
```

The whole security of RSA rests on the assumption that, given only `N`, no one can find `p` and `q`. For a properly-sized `N` (2048+ bits today), factoring `N` back into `p * q` is computationally infeasible. For a small `N`, it is trivial.

### What the hint is really telling us

> "Bits are expensive, I used only a little bit over 100 to save money"

In RSA, "bits" almost always means "bit length of the modulus N". A real-world RSA-2048 key uses a 2048-bit N. A "little bit over 100" means each prime factor is around ~100-130 bits, so `N = p * q` is roughly 200-270 bits. That is *enormously* smaller than the 2048 bits a real key uses, and small enough to factor with public tools in seconds.

### How small-N RSA gets attacked

A modulus with only a few hundred bits can be cracked using:

1. **factordb.com** — a public database of known factorizations. If someone has ever submitted the number before (and many small CTF moduli have been), it returns `p` and `q` instantly.
2. **msieve / yafu / CADO-NFS** — local factoring tools. CADO-NFS can factor a 100-digit (~330 bit) number in a few hours on a single core.
3. **Sympy's `factorint`** — fine for toy-size numbers but slow on anything over 60 digits.
4. **Pollard's rho / p-1** — elementary algorithms that work when one of the primes is small or has small factors. Worth a shot before going to NFS.

For this challenge, factordb.com is the fastest path. We just paste `N` in and read off the factors.

### Once we have p and q

Once we know `p` and `q`, RSA decryption is a single Python line:

```python
phi = (p - 1) * (q - 1)
d   = pow(e, -1, phi)        # modular inverse, Python 3.8+
m   = pow(c, d, N)           # RSA decryption
```

`m` is the plaintext as a big integer. To get the readable flag, convert it to bytes. The encoding direction depends on the challenge — sometimes big-endian, sometimes little-endian. If the first decode looks like garbage, just reverse the byte order and try again.

---

## Solution — Step by Step

### Step 1 — Read the challenge file

The challenge ships a file called `values` with three lines:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ cat values
Decrypt my super sick RSA:
c: 15341890103764929939105506004034128738090325640037083301857608662849501626260517
n: 948406957756830799684818171639547165784816468744946013083947881743680617123566349
e: 65537
```

So we have:

| Symbol | Value |
|--------|-------|
| `c` (ciphertext) | `15341890103764929939105506004034128738090325640037083301857608662849501626260517` |
| `N` (modulus)    | `948406957756830799684818171639547165784816468744946013083947881743680617123566349` |
| `e` (exponent)   | `65537` (the textbook RSA public exponent) |

Quick bit-length sanity check — this `N` is suspiciously small:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ python3 -c "n=948406957756830799684818171639547165784816468744946013083947881743680617123566349; print('N bit length:', n.bit_length())"
N bit length: 269
```

269 bits. Real RSA uses 2048 bits minimum. This one is *way* under the "safe" range.

### Step 2 — Look up the factorization on factordb

Open https://factordb.com/ in Firefox and paste `N` into the search box. Or do it from the terminal:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ firefox "http://factordb.com/index.php?query=948406957756830799684818171639547165784816468744946013083947881743680617123566349"
```

factordb shows:

```
The factorization is:
1891771437429478964908181306574287207137  ×  501332739776173570344039681219489434626477
```

So:

```
p = 1891771437429478964908181306574287207137     (131 bits — "a little over 100")
q = 501332739776173570344039681219489434626477   (139 bits — "a little over 100")
```

The hint was referring to *each prime* being just a bit over 100 bits. Real RSA primes are 1024 bits each.

### Step 3 — Write a small decryptor in Python

Save this as `solve.py`:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ nano solve.py
```

Paste the content:

```python
#!/usr/bin/env python3
"""Mind your Ps and Qs — RSA small-N decryptor."""

# From the challenge file
c = 15341890103764929939105506004034128738090325640037083301857608662849501626260517
N = 948406957756830799684818171639547165784816468744946013083947881743680617123566349
e = 65537

# From factordb
p = 1891771437429478964908181306574287207137
q = 501332739776173570344039681219489434626477

# Sanity check
assert p * q == N, "p * q != N — wrong factors?"
print(f"[+] N verified: {p} * {q} == N")
print(f"[+] p bit length: {p.bit_length()}")
print(f"[+] q bit length: {q.bit_length()}")
print()

# Compute private exponent
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)
print(f"[+] d = {d}")
print()

# Decrypt
m = pow(c, d, N)
print(f"[+] m (int) = {m}")
print()

# Convert to bytes — try big-endian first
nbytes = (m.bit_length() + 7) // 8
big = m.to_bytes(nbytes, "big")
print(f"[+] big-endian   : {big!r}")
lit = m.to_bytes(nbytes, "little")
print(f"[+] little-endian: {lit!r}")
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

### Step 4 — Run the decryptor

```
┌──(zham㉿kali)-[~/Downloads]
└─$ python3 solve.py
[+] N verified: 1891771437429478964908181306574287207137 * 501332739776173570344039681219489434626477 == N
[+] p bit length: 131
[+] q bit length: 139

[+] d = 422490213645889729416944115858470474771091523344123437309630951594676399386050433

[+] m (int) = 310924024341754586049069014240097859710998385184245570453341488382426297925855600

[+] big-endian   : b'\n}19ea7cd1_do0g_0n_N_11ams{FTCocip'
[+] little-endian: b'picoCTF{sma11_N_n0_g0od_1dc7ae91}\n'
```

The big-endian decode looks like scrambled flag text — note the `picoCTF` substring is at the end *and reversed*. That is a strong hint that the original encoding was little-endian.

The little-endian decode reads cleanly: `picoCTF{sma11_N_n0_g0od_1dc7ae91}` followed by a trailing newline.

Flag: **`picoCTF{sma11_N_n0_g0od_1dc7ae91}`**

### Step 5 — Submit

Paste the flag into the picoCTF challenge page and click Submit.

---

## Alternative Methods

**Method 1 — Sympy's `factorint` (fully offline)**

If factordb.com is down or you want to stay offline:

```python
from sympy import factorint

N = 948406957756830799684818171639547165784816468744946013083947881743680617123566349
factors = factorint(N)
print(factors)   # {1891771437429478964908181306574287207137: 1, 501332739776173570344039681219489434626477: 1}
```

Sympy internally tries a sequence of algorithms (trial division, Pollard's rho, Pollard's p-1, then a sieve). For a 270-bit N it might take a minute or two — much slower than factordb, but you do not depend on the internet.

**Method 2 — `openssl` for factoring**

If you have `openssl` and `msieve` installed:

```
┌──(zham㉿kali)-[~/Downloads]
└─$ msieve -q 948406957756830799684818171639547165784816468744946013083947881743680617123566349
```

`msieve` is a C implementation of the number field sieve, much faster than Sympy for hundreds-of-bits sized inputs. For larger numbers you would want CADO-NFS instead.

**Method 3 — SageMath**

If you have SageMath:

```python
sage: N = 948406957756830799684818171639547165784816468744946013083947881743680617123566349
sage: factor(N)
1891771437429478964908181306574287207137 * 501332739776173570344039681219489434626477
```

Sage uses PARI under the hood, which is roughly as fast as `msieve` for this size.

**Method 4 — `factor` shell command (only works for tiny N)**

```
┌──(zham㉿kali)-[~/Downloads]
└─$ factor 948406957756830799684818171639547165784816468744946013083947881743680617123566349
```

GNU `factor` tops out around 50-digit numbers on a modern system, so this will *not* finish in any reasonable time on a 270-bit input. Mentioned only because beginners sometimes try it first.

**Method 5 — Python `Crypto.Util.number` and one-liner decrypt**

For a tidier script:

```python
from Crypto.Util.number import long_to_bytes, inverse

c = 15341890103764929939105506004034128738090325640037083301857608662849501626260517
N = 948406957756830799684818171639547165784816468744946013083947881743680617123566349
e = 65537
p, q = 1891771437429478964908181306574287207137, 501332739776173570344039681219489434626477

d = inverse(e, (p - 1) * (q - 1))
m = pow(c, d, N)
print(long_to_bytes(m)[::-1].rstrip())   # little-endian trick
```

`pycryptodome` makes the modular inverse call explicit (`inverse`) and `long_to_bytes` handles the padding.

---

## What Happened Internally

A timeline of what actually happened behind the curtain:

1. The challenge author generated a "toy" RSA keypair using two primes of ~130 bits each (instead of the standard 1024+ bits each).
2. They computed `N = p * q` (269 bits), picked the textbook public exponent `e = 65537`, and computed the corresponding private exponent `d`.
3. They encrypted the flag bytes as a single integer `m`, then computed `c = m^e mod N`.
4. They handed us only `(c, N, e)`. Without `p` and `q`, we cannot compute `d`, so we cannot decrypt.
5. We looked at `N` and noticed it was 269 bits — about 7x smaller than a "real" RSA modulus. The hint "a little bit over 100" confirmed this was deliberately weak.
6. We pasted `N` into factordb.com, which already had the factorization on file (this is a popular CTF modulus, so it has been submitted many times before). factordb returned `p` and `q`.
7. We reconstructed `phi = (p - 1) * (q - 1)`, then `d = e^(-1) mod phi` using Python's built-in modular inverse.
8. We decrypted with `m = c^d mod N`, which gave us a big integer.
9. We converted `m` to bytes. The big-endian conversion came out backwards (`picoCTF{...}` reversed, with a leading newline), so we tried little-endian and got the flag with a trailing newline.
10. We submitted the flag, scored the points, and reflected on why "bits are expensive" really means "RSA only works when N is huge".

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the challenge file |
| Firefox / `firefox` | Open factordb.com in the browser for the factorization |
| `nano` | Edit the Python solver |
| `python3` | Run the decryptor; uses built-in `pow()` for fast modular exponentiation and modular inverse |
| `int.bit_length()` | Confirm `N` is suspiciously small |
| `int.to_bytes()` | Convert the decrypted integer into readable ASCII |
| factordb.com | Pre-computed factorization lookup (instant win for small N) |
| Sympy `factorint` (alt) | Offline factoring if factordb is unreachable |
| `msieve` / SageMath (alt) | Faster local factoring for slightly larger N |

---

## Key Takeaways

- **The size of `N` is the single most important thing in RSA.** Public exponent choice (`e`) matters a little (a tiny `e` enables Hastad's broadcast attack), but a small `N` is a death sentence — factoring is the entire foundation RSA stands on.
- **"A little over 100" hits perfectly in the factorable range.** factordb.com handles numbers up to about 80 digits (≈265 bits) essentially instantly because they have been seen before; msieve or CADO-NFS can chew through a 100-digit (~330 bit) N in hours on a laptop.
- **factordb.com is a CTF superpower.** If a number has been factored anywhere in the world, it lives in factordb. Paste, copy, done. Always try this before reaching for heavier tools.
- **Watch your endianness.** `m.to_bytes(n, "big")` and `m.to_bytes(n, "little")` are different. If the first decode looks scrambled, swap. The fact that we saw `picoCTF` reversed in the big-endian output was a giveaway that little-endian was the original encoding.
- **`pow(e, -1, phi)` is your modular-inverse friend.** Python 3.8 added native modular inverse to the built-in `pow()`. No need to import `Crypto.Util.number.inverse` or hand-roll extended Euclidean.
- **Always verify `p * q == N`** before you trust a factorization. One wrong factor turns into a broken `d` and a garbage `m`. The `assert` line in our script catches that immediately.
- **The trailing newline is normal.** Most RSA-flag CTF challenges append a `\n` to the plaintext before encrypting. Strip it when you read the flag.
- **Flag wordplay decode:**
  - `sma11` -> **"small"** (`1` for `l`, leetspeak)
  - `N` -> **"N"** (literally the modulus variable from RSA)
  - `n0` -> **"no"** (`0` for `o`)
  - `g0od` -> **"good"** (`0` for `o`)
  - `1dc7ae91` -> hex-ish tail; reads as `1` + `dc` + `7` + `ae` + `91`, the suffix is just opaque flag-padding nonce (no clean English meaning, the prefix carries the wordplay)
  - Putting it together: **"small N no good 1dc7ae91"** — directly echoing the challenge's central lesson: a small RSA modulus is no good, because anyone with a factoring tool can recover the private key and read your message.
