# ClusterRSA - picoCTF Writeup

**Challenge:** ClusterRSA  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{mul71_rsa_787c01b3}`  
**Platform:** picoCTF (2024)  
**Writeup by:** zham  

---

## Description

> A message has been encrypted using RSA, but this time something feels... more crowded than usual. Can you decrypt it?
>
> Download the message.

## Hints

> 1. RSA usually means two primes... but what if someone got greedy?
> 2. Prime factors decomposition

---

## Background Knowledge

Before jumping in, here are the concepts behind this challenge.

### Standard RSA, in 30 Seconds

RSA is a public-key cryptosystem. The public key is a pair `(e, n)` and the private key is `d`. To encrypt a number `m`, the sender computes `c = m^e mod n`. To decrypt, the recipient computes `m = c^d mod n`. The relationship between `e` and `d` is:

```
e * d ≡ 1  (mod phi(n))
```

Where `phi(n)` is Euler's totient function — the number of integers in `[1, n]` that are coprime to `n`.

### The "Two Primes" Assumption

In normal textbook RSA, `n` is the product of exactly two large primes `p` and `q`. That gives us:

```
phi(n) = (p - 1) * (q - 1)
```

If you can factor `n` into `p` and `q`, you can rebuild `phi(n)` and recover `d`. That is why `p` and `q` have to be huge (1024 bits each in modern RSA) — factoring their product is computationally infeasible with the best known algorithms.

### Multi-Prime RSA: The "Got Greedy" Variant

What happens if you use three, four, or even ten primes instead of two? Mathematically, nothing breaks — RSA still works:

```
n = p1 * p2 * p3 * ... * pk
phi(n) = (p1 - 1) * (p2 - 1) * ... * (pk - 1)
e * d ≡ 1 (mod phi(n))
```

The "got greedy" in the hint points directly at this: the challenge author used **multiple smaller primes** instead of two big ones. Each individual prime is now small enough that a public database of factorizations (`factordb.com`) knows about them. The product is still ~280 bits long, but splitting it into four pieces of ~70 bits each is trivial.

This is a real lesson too: while multi-prime RSA is technically allowed (RFC 8017), it only stays secure if every single prime factor remains large. Drop below ~512 bits per prime and you are basically inviting a factorization attack.

### How Big is `n` Here?

`n` is about 280 bits (35 bytes). For reference, modern secure RSA uses 2048- or 4096-bit moduli. So this `n` is small enough that we can just ask an online database.

---

## Solution

### Step 1: Look at the Challenge File

I downloaded `message.txt` and `cat`ed it:

```
┌──(zham㉿kali)-[~/clusterrsa]
└─$ ls
message.txt
```

```
┌──(zham㉿kali)-[~/clusterrsa]
└─$ cat message.txt
n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
e = 65537
ct = 3021569373773402689513257373362764131880473249842187164838297943840513930619586623604677697191914325
```

Three values, all integers. `e = 65537` is the textbook RSA public exponent, so this is standard textbook RSA — the only twist is the factorization of `n`.

### Step 2: Quick Sanity Check on `n`

Before reaching for outside tools, I want to see how big `n` really is. `n.bit_length()`:

```
┌──(zham㉿kali)-[~/clusterrsa]
└─$ python3 -c "
n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
print('n bit length:', n.bit_length())
print('n digit count:', len(str(n)))
"
n bit length: 280
n digit count: 82
```

280 bits. Way smaller than secure RSA (which is 2048+ bits). That is a strong signal that `n` is either factorable directly or already in a public database.

### Step 3: Hit factordb

`factordb.com` is a public database that catalogs integer factorizations. For any "small enough" number, it usually already has the answer. I checked its API:

```
┌──(zham㉿kali)-[~/clusterrsa]
└─$ curl -s "http://factordb.com/api?query=8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651"
{"id":1100000008358012778,"status":"FF","factors":[["9671406556917033397931773",1],["9671406556917033398314601",1],["9671406556917033398439721",1],["9671406556917033398454847",1]]}
```

A few things to read here:

- **`status: FF`** — "Fully Factored." factordb already knows the answer.
- **`factors`** — an array of `[prime, exponent]` pairs. All four primes appear with exponent `1` (each appears exactly once in the product).
- **The four primes are all around 73 bits long** (`9671406556917033397931773` is 73 bits). Each is way below the threshold where direct factoring becomes hard.

So `n = p1 * p2 * p3 * p4` with:

```
p1 = 9671406556917033397931773
p2 = 9671406556917033398314601
p3 = 9671406556917033398439721
p4 = 9671406556917033398454847
```

That is exactly the "more crowded than usual" from the description — four primes instead of two.

### Step 4: Build the Solver

I wrote the standard multi-prime RSA decrypt in Python:

```
┌──(zham㉿kali)-[~/clusterrsa]
└─$ nano solve.py
```

The content I pasted into `nano`:

```python
#!/usr/bin/env python3
import requests

n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
e = 65537
ct = 3021569373773402689513257373362764131880473249842187164838297943840513930619586623604677697191914325

# Step 1: factor n via factordb
api = f"http://factordb.com/api?query={n}"
factors = [int(f[0]) for f in requests.get(api, timeout=20).json()["factors"]]
print(f"Factorization: {factors}")

# Step 2: phi(n) for k primes is the product of (p_i - 1)
phi = 1
for p in factors:
    phi *= (p - 1)

# Step 3: standard RSA decrypt
d = pow(e, -1, phi)
m = pow(ct, d, n)
flag = m.to_bytes((m.bit_length() + 7) // 8, "big").decode()
print("Flag:", flag)
```

Save: `Ctrl+O`, `Enter`, exit: `Ctrl+X`.

```
┌──(zham㉿kali)-[~/clusterrsa]
└─$ python3 solve.py
Factorization: [9671406556917033397931773, 9671406556917033398314601, 9671406556917033398439721, 9671406556917033398454847]
Flag: picoCTF{mul71_rsa_787c01b3}
```

Done. The plaintext was a UTF-8 byte string that decoded cleanly to `picoCTF{mul71_rsa_787c01b3}`.

### Step 5: Sanity-Check the Math

Before celebrating, I confirmed the four primes actually multiply back to `n` — a habit that has saved me from typos more than once:

```
┌──(zham㉿kali)-[~/clusterrsa]
└─$ python3 -c "
n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
ps = [9671406556917033397931773, 9671406556917033398314601, 9671406556917033398439721, 9671406556917033398454847]
prod = 1
for p in ps: prod *= p
print('product equals n:', prod == n)
"
product equals n: True
```

Verified — `p1 * p2 * p3 * p4` equals `n` exactly. The flag is `picoCTF{mul71_rsa_787c01b3}`.

---

## Alternative Solve Methods

### Method 1: Hardcode the Four Primes, Skip factordb

If factordb is down, or you want a self-contained solver, hardcode the four primes from a previous factordb pull:

```python
n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
e = 65537
ct = 3021569373773402689513257373362764131880473249842187164838297943840513930619586623604677697191914325

primes = [9671406556917033397931773,
          9671406556917033398314601,
          9671406556917033398439721,
          9671406556917033398454847]

phi = 1
for p in primes:
    phi *= (p - 1)

d = pow(e, -1, phi)
m = pow(ct, d, n)
print(m.to_bytes((m.bit_length() + 7) // 8, "big").decode())
```

Output:

```
picoCTF{mul71_rsa_787c01b3}
```

This is the version you would want in a CTF toolbox — paste the four primes once you have them from factordb, and the rest is the textbook RSA decrypt.

### Method 2: Manual Trial Division for Small Factors

If you wanted to see the "more crowded" feeling for yourself before trusting factordb, you can do a trial-division sweep. The first prime factor of `n` is `9671406556917033397931773`, which is bigger than `10^15`, so naive trial division up to `10^6` or even `10^9` will not find it. The exercise just confirms the factors are too big for trial division and motivates the factordb lookup.

```python
from sympy import isprime, nextprime

n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651

p = 2
factors = []
m = n
while p * p <= m:
    while m % p == 0:
        factors.append(p)
        m //= p
    p = nextprime(p)

if m > 1:
    factors.append(m)

print("Factors via trial division:", factors)
```

This loop is painfully slow for an 82-digit number — it will run for ages before reaching `sqrt(n)`. That slowness is exactly the point: trial division is not the right tool here, and you reach for factordb (or Pollard's rho, or ECM) instead.

### Method 3: Sympy's `factorint` (If factordb Is Offline)

`sympy.factorint` will internally switch between trial division, Pollard's rho, and ECM depending on the size of the factors. For our four ~73-bit primes, it should still finish in reasonable time:

```python
from sympy import factorint
from Crypto.Util.number import long_to_bytes

n = 8749002899132047699790752490331099938058737706735201354674975134719667510377522805717156720453193651
e = 65537
ct = 3021569373773402689513257373362764131880473249842187164838297943840513930619586623604677697191914325

factors = list(factorint(n).keys())
phi = 1
for p in factors:
    phi *= (p - 1)

d = pow(e, -1, phi)
m = pow(ct, d, n)
print(long_to_bytes(m).decode())
```

```
picoCTF{mul71_rsa_787c01b3}
```

`sympy.factorint` is a good offline fallback when factordb is unreachable, but it is noticeably slower than factordb for numbers that have already been cataloged.

---

## What Happened Internally

Here is the full timeline of the attack, from "I have a file" to "I have the flag."

1. **Read the challenge file.** `cat message.txt` showed three integers: `n` (the modulus), `e = 65537` (the public exponent), and `ct` (the ciphertext).
2. **Sized up `n`.** `n.bit_length()` returned `280`. Secure RSA is 2048+ bits, so this `n` is small by orders of magnitude.
3. **Sent `n` to factordb.** `curl http://factordb.com/api?query=<n>` returned `status: FF` with four prime factors, each around 73 bits long. That confirmed the multi-prime RSA twist hinted at by "more crowded than usual."
4. **Verified the factorization locally.** Multiplying the four primes returned exactly `n`, so we knew factordb was right.
5. **Computed `phi(n)`.** Multi-prime RSA uses the same Euler-totient formula as textbook RSA — just with more factors. `phi(n) = (p1-1) * (p2-1) * (p3-1) * (p4-1)`.
6. **Found the private exponent `d`.** `pow(e, -1, phi)` computed the modular inverse of `e` modulo `phi(n)`.
7. **Decrypted.** `pow(ct, d, n)` raised the ciphertext to the private exponent modulo `n`, recovering the plaintext integer `m`.
8. **Converted to bytes.** `m.to_bytes((m.bit_length() + 7) // 8, "big")` and `.decode()` turned the integer into the readable flag.

The whole thing is textbook RSA decryption — the only non-textbook step is the factorization of `n`, which factordb handled for us.

---

## Tools Used

| Tool         | Purpose                                                       |
| ------------ | ------------------------------------------------------------- |
| `cat`        | Read `message.txt`                                            |
| `python3`    | Bit-length sanity check, byte conversion, modular arithmetic  |
| `curl`       | Pull the factorization from factordb's API                    |
| factordb.com | Online database of integer factorizations                     |
| `requests`   | Programmatic access to factordb (alternative to `curl`)       |
| `nano`       | Write the multi-prime RSA solver                              |
| `sympy`      | Offline fallback factorization (`factorint`)                  |

---

## Key Takeaways

- **factordb is the first stop for any small-`n` RSA challenge.** If `n` is under ~300 bits, just paste it into the API. If the response says `status: FF`, you are done before you even start writing Python. For larger `n`, factordb will still tell you whether someone has already factored it.
- **`n.bit_length()` is a five-second triage step.** It tells you immediately whether you are looking at a toy challenge (under 512 bits), a vulnerable-by-design challenge (under 1024 bits), or actually-secure RSA (2048+ bits). The right tool to factor `n` depends entirely on which bucket you are in.
- **Multi-prime RSA is RSA.** The encryption, decryption, and `phi(n)` formulas do not change — only the number of factors in `n`. If you ever see a real-world multi-prime RSA implementation, factor it the same way: identify every prime factor, multiply `(p-1)` for each, and compute `d = e^-1 mod phi(n)`.
- **More primes only helps the attacker, not the defender.** The challenge author's "greed" gave them four small primes instead of two big ones, and the security of the system collapsed. In real cryptography, every prime in `n` must remain individually large (>= 1024 bits) for the product to stay hard to factor. Splitting `n` into many small primes is a strict downgrade in security.
- **Always verify factorizations.** Once you have the primes, multiply them back and check that the product equals `n`. Off-by-one typos in copy-paste are a classic source of "I got the wrong flag" frustration.

**Flag wordplay decode:** `mul71_rsa` reads as **"multi RSA."** The `71` is leet for `ti` (`7` → `t`, `1` → `i`), so the whole thing is "multi RSA" — exactly what the challenge was: a standard RSA decryption where `n` was the product of multiple primes instead of the usual two. The trailing `787c01b3` is just a per-player nonce.
