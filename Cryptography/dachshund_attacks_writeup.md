# Dachshund Attacks — picoCTF Writeup

**Challenge:** Dachshund Attacks  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 80  
**Flag:** `picoCTF{proving_wiener_4755a2a}`  
**Platform:** picoCTF (2023)  
**Writeup by:** zham  

---

## Description

> What if d is too small?
>
> Connect with `nc wily-courier.picoctf.net 64996`.

The server generates a fresh RSA keypair for every connection, encrypts the flag with it, and prints `e`, `n`, and `c`. The catch is that the private exponent `d` it picks is unusually small — that is the entire vulnerability the challenge wants us to exploit.

## Hints

> 1. What do you think about my pet? `dachshund.jpg`

The hint gives us a dachshund picture. In English, a dachshund is often nicknamed a **"wiener dog"** (or **"sausage dog"**). That is the giveaway — the attack we are meant to run is **Wiener's attack**, a 1989 attack by Michael J. Wiener that recovers small RSA private exponents.

---

## Background Knowledge (Read This First!)

If you are new to RSA, read this top-to-bottom. If you have done the easier crypto challenges in picoCTF, the first two sections are review — the third one (Wiener) is the new material.

### 1. RSA in 60 Seconds

RSA is an asymmetric (public-key) encryption scheme. Each user has a public key and a private key.

**Key generation:**
1. Pick two large random primes `p` and `q`.
2. Compute the modulus `n = p * q`.
3. Compute Euler's totient `phi(n) = (p - 1) * (q - 1)`.
4. Pick a public exponent `e` such that `gcd(e, phi(n)) = 1` (the textbook default is `e = 65537`).
5. Compute the private exponent `d = e^(-1) mod phi(n)` — the modular inverse of `e`.

**Encryption** (with public key `(n, e)`): `c = m^e mod n`
**Decryption** (with private key `d`): `m = c^d mod n`

Anyone who knows `d` can decrypt. So in a secure setup, `d` should be about as large as `n` (a 1024–2048 bit number). The whole security story of RSA rests on the difficulty of factoring `n` into `p` and `q` when both are huge.

### 2. Continued Fractions (Just Enough Math)

For two integers `a` and `b` with `a > b > 0`, we can write the fraction `a/b` as a finite **continued fraction** expansion:

```
a/b = q0 + 1 / (q1 + 1 / (q2 + 1 / (q3 + ...)))
```

The sequence `q0, q1, q2, ...` is found by the standard Euclidean algorithm. For example, `e/n` with any RSA `e` and `n` has such an expansion.

Each "truncation" of that expansion produces a **convergent** — a fraction `h_i / k_i` that is, in a precise sense, the closest fraction to `e/n` for that denominator size. Convergents are how we will sneak up on `d`.

### 3. Wiener's Attack — The Whole Idea

In 1989, Michael J. Wiener proved the following:

> If `d < (1/3) * n^(1/4)`, then `n` and `e` alone are enough to recover `d` in polynomial time.

Why does this work? The key identity is

```
e * d = 1 + k * phi(n)         for some positive integer k
```

When `d` is very small, `k` is also very small (because `phi(n) ≈ n`). So the fraction `k/d` is a really good rational approximation to `e/n`:

```
e/n  ≈  k/d
```

Concretely, the convergent denominators of the continued fraction expansion of `e/n` include `d` itself (or a small multiple/divisor of it). We just compute the continued fraction, walk through each convergent denominator `k`, test whether `k` could plausibly be `d`, and confirm by factoring `n`. The very first one that yields a clean `p * q == n` is the answer.

The reason the challenge is called "Dachshund Attacks" is just a pun: dachshund = wiener dog, Wiener = the attack.

---

## Solution — Step by Step

### Step 1 — Connect to the Server and Capture n, e, c

I opened a fresh terminal and connected with netcat. The server printed the three numbers we need and waited.

```
┌──(zham㉿kali)-[~/picoctf/dachshund-attacks]
└─$ nc wily-courier.picoctf.net 64996
Welcome to my RSA challenge!
e: 57742112596171021818394771829706669597197254450621409706602001364434428134062086864188992874789317337643060135718640170417997033377625588215010782049112938027004177715857689263031126861774041532931044132410081978605478127343108083020386069317318844502588267999821544289943400281096341976121750810367364700799
n: 122463251527785036498101502277444562173099008317653503838923362020208273409405641765528168433408177947456183725236130274101804434533587802000370252675277797088782414291785499770213732339274494006736479342160015348724222275326827580698093516102862258936167351793834643331487579914083163096291333246460422487137
c: 33854978077788903186269961968160397437008297293837776059036883917548674222401860874047588663995422751452682743846105270982801971049552777775992648837048316537316126028085634437691096915003587859940015023460864312367390046446328492479044057404037014903830483676000553222741260677875443060491850799647366817570
```

I copied each number into my working notes. `n` is about 1024 bits (so `d` should also be ~1024 bits for safety), and `e` is unusually large for an RSA exponent — that is actually consistent with Wiener's setup, because Wiener's bound becomes a problem precisely when `e` is close to `n` (which makes `d` small).

### Step 2 — Write a Wiener's Attack Script

I wrote a small Python script that does the attack end-to-end. I prefer writing my own so I can see exactly what is happening — but you can also use `RsaCtfTool` (see Alternative Method at the bottom).

```
┌──(zham㉿kali)-[~/picoctf/dachshund-attacks]
└─$ nano wiener.py
```

I pasted the following into `nano`:

```python
#!/usr/bin/env python3
"""
Wiener's Attack on RSA with a small private exponent d.

Strategy:
  1. Build the continued fraction expansion of e/n.
  2. For each convergent (h_i, k_i), treat k_i as a candidate d.
  3. Confirm by checking (e*d - 1) % k_i == 0, then solving
     x^2 - s*x + n == 0 for s = p + q and checking the discriminant.
  4. The first candidate that produces a clean factorization is d.
"""
import sys
from Crypto.Util.number import long_to_bytes

e = 57742112596171021818394771829706669597197254450621409706602001364434428134062086864188992874789317337643060135718640170417997033377625588215010782049112938027004177715857689263031126861774041532931044132410081978605478127343108083020386069317318844502588267999821544289943400281096341976121750810367364700799
n = 122463251527785036498101502277444562173099008317653503838923362020208273409405641765528168433408177947456183725236130274101804434533587802000370252675277797088782414291785499770213732339274494006736479342160015348724222275326827580698093516102862258936167351793834643331487579914083163096291333246460422487137
c = 33854978077788903186269961968160397437008297293837776059036883917548674222401860874047588663995422751452682743846105270982801971049552777775992648837048316537316126028085634437691096915003587859940015023460864312367390046446328492479044057404037014903830483676000553222741260677875443060491850799647366817570


def continued_fraction(num, den):
    """Return the continued fraction expansion of num/den as a list."""
    cf = []
    while den:
        q, r = divmod(num, den)
        cf.append(q)
        num, den = den, r
    return cf


def convergents(cf):
    """Yield (h_i, k_i) convergents of the continued fraction expansion."""
    h_prev, h_curr = 0, 1
    k_prev, k_curr = 1, 0
    for a in cf:
        h_prev, h_curr = h_curr, a * h_curr + h_prev
        k_prev, k_curr = k_curr, a * k_curr + k_prev
        yield h_curr, k_curr


def isqrt(n):
    if n < 0:
        raise ValueError("Square root not defined for negative numbers")
    if n == 0:
        return 0
    x = n
    y = (x + 1) // 2
    while y < x:
        x = y
        y = (x + n // x) // 2
    return x


def is_perfect_square(n):
    if n < 0:
        return False
    s = isqrt(n)
    return s * s == n


def wiener(e, n):
    """Recover d using Wiener's attack. Returns d, or None on failure."""
    cf = continued_fraction(e, n)
    for k, d in convergents(cf):
        if k == 0:
            continue
        if (e * d - 1) % k != 0:
            continue
        phi_candidate = (e * d - 1) // k
        s = n - phi_candidate + 1           # p + q
        disc = s * s - 4 * n                # discriminant
        if disc <= 0:
            continue
        if is_perfect_square(disc):
            t = isqrt(disc)
            if (s + t) % 2 == 0:
                p = (s + t) // 2
                q = (s - t) // 2
                if p * q == n:
                    return d
    return None


def main():
    print("[*] Running Wiener's attack on RSA with small d...")
    d = wiener(e, n)
    if d is None:
        print("[-] Wiener's attack failed.")
        sys.exit(1)

    print(f"[+] Recovered d = {d}")
    print(f"[+] d bit-length: {d.bit_length()}")

    m = pow(c, d, n)
    plaintext = long_to_bytes(m)
    print(f"[+] Decrypted plaintext (raw bytes): {plaintext!r}")
    try:
        flag = plaintext.decode()
    except UnicodeDecodeError:
        flag = plaintext.decode(errors="replace")
    print(f"[+] Flag: {flag}")


if __name__ == "__main__":
    main()
```

Save and exit:

```
^O   (WriteOut)
File Name to Write: wiener.py -> press Enter
^X   (Exit)
```

### Step 3 — Install pycryptodome (for long_to_bytes) and Run

The script needs `Crypto.Util.number.long_to_bytes` to convert the big integer back to bytes. `pycryptodome` provides that:

```
┌──(zham㉿kali)-[~/picoctf/dachshund-attacks]
└─$ pip3 install pycryptodome --break-system-packages
[install output...]
```

Now run the attack:

```
┌──(zham㉿kali)-[~/picoctf/dachshund-attacks]
└─$ python3 wiener.py
[*] Running Wiener's attack on RSA with small d...
[+] Recovered d = 502271248598975531268517606331940404610443657385558190400420672471850406675
[+] d bit-length: 249
[+] Decrypted plaintext (raw bytes): b'picoCTF{proving_wiener_4755a2a}\n'
[+] Flag: picoCTF{proving_wiener_4755a2a}
```

Done. `d` is only 249 bits long, while `n` is about 1024 bits — that is exactly the "small `d`" condition Wiener's attack requires.

---

## What Happened Internally (Timeline)

A walk-through of what the script actually does, in order:

1. **Input the challenge parameters.** `e`, `n`, and `c` come straight from the server.
2. **Build the continued fraction expansion of `e/n`.** This uses the Euclidean algorithm: keep dividing `e` by `n`, take the quotient, swap, repeat.
3. **Walk the convergents.** Each convergent is computed by the recurrence `h_i = q_i * h_{i-1} + h_{i-2}` (and same for `k_i`), starting from `h_{-1} = 1, h_0 = q_0` and `k_{-1} = 0, k_0 = 1`.
4. **Test each convergent denominator `k` as a candidate `d`.** For each candidate `d`:
   - Check `(e*d - 1) % k == 0`. If yes, set `phi_candidate = (e*d - 1) / k`.
   - Compute `s = p + q = n - phi_candidate + 1`.
   - Compute the discriminant `s^2 - 4n`. If it is a perfect square, we can factor `n` and we have our `d`.
5. **Found `d = 502...675` after just a handful of convergents** (Wiener's theorem guarantees it appears early).
6. **Decrypt.** `m = pow(c, d, n)` in Python does the modular exponentiation efficiently.
7. **Convert integer to bytes** with `long_to_bytes`, then decode UTF-8, and we get the flag.

---

## Alternative Method — RsaCtfTool

If you do not want to write the attack by hand, the all-in-one tool `RsaCtfTool` already implements Wiener's attack (plus many others). It will autodetect the vulnerability.

```
┌──(zham㉿kali)-[~]
└─$ git clone https://github.com/RsaCtfTool/RsaCtfTool.git
┌──(zham㉿kali)-[~]
└─$ cd RsaCtfTool
┌──(zham㉿kali)-[~/RsaCtfTool]
└─$ pip3 install -r requirements.txt --break-system-packages
┌──(zham㉿kali)-[~/RsaCtfTool]
└─$ python3 RsaCtfTool.py --private -n 122463251527785036498101502277444562173099008317653503838923362020208273409405641765528168433408177947456183725236130274101804434533587802000370252675277797088782414291785499770213732339274494006736479342160015348724222275326827580698093516102862258936167351793834643331487579914083163096291333246460422487137 -e 57742112596171021818394771829706669597197254450621409706602001364434428134062086864188992874789317337643060135718640170417997033377625588215010782049112938027004177715857689263031126861774041532931044132410081978605478127343108083020386069317318844502588267999821544289943400281096341976121750810367364700799 --uncipherfile cipher.txt
```

(You would first write `c` into `cipher.txt`.) RsaCtfTool tries Wiener first, recovers `d`, and prints the plaintext — same answer, less typing.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` (netcat) | Connect to the challenge server and grab `e`, `n`, `c` |
| Python 3 | Write and run the Wiener's attack script |
| `pycryptodome` | `Crypto.Util.number.long_to_bytes` to convert integer plaintext to bytes |
| `nano` | Edit the Python script in the terminal |
| `RsaCtfTool` (alternative) | Automated RSA attack toolkit that already implements Wiener |

---

## Key Takeaways

- The flag **"proving wiener"** is a pun. Michael J. Wiener **proved** the theorem that makes RSA insecure when `d` is small — so the flag reads "proving Wiener" as both "validating Wiener's result" and a nod to the dachshund = "wiener dog" pun in the title.
- RSA's security rests on three independent assumptions: `n` is hard to factor, `d` is large enough, and `e` and `phi(n)` are coprime with `phi(n)` having no small factors. Wiener's attack breaks the second assumption directly.
- The rule of thumb: always pick `d` to be roughly the same size as `n` (1024+ bits for a 1024-bit modulus). Implementations that pick `d` from a small range for performance are broken by Wiener.
- Continued fractions are surprisingly powerful — they are the same trick behind other attacks like the rational approximation used in Coppersmith's method.
- When in doubt on a CTF challenge, glance at the puns. "Dachshund" → wiener dog → Wiener. Crypto challenge authors love a good pun.

**Flag:** `picoCTF{proving_wiener_4755a2a}`
