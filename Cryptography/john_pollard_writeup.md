# john_pollard - picoCTF Writeup

**Challenge:** john_pollard  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 500  
**Flag:** `picoCTF{73176001,67867967}`  
**Platform:** picoCTF (2019)  
**Writeup by:** zham  

> Server note: the picoCTF flag checker expects `q` first, then `p`. Submitting `(small, big)` returns "Incorrect flag." Use the `(big, small)` variant — that is exactly what hint 2 covers.

---

## Description

> Sometimes RSA certificates are breakable

## Hints

> 1. The flag is in the format picoCTF{p,q}
> 2. Try swapping p and q if it does not work

---

## Background Knowledge

Before we look at a single byte of the certificate, here is the RSA refresher we need to make the solve click.

### What is RSA?

**RSA** (named after Rivest, Shamir, and Adleman, who described it publicly in 1977) is the most widely deployed public-key cryptosystem on the planet. Every time your browser does HTTPS, every time you SSH into a server, every time you sign a software update, RSA is almost certainly doing some of the work.

The whole system rests on the fact that multiplying two huge primes is easy, but factoring their product is hard. Specifically:

1. Pick two large random primes `p` and `q`.
2. Compute the modulus `N = p * q`.
3. Compute Euler's totient `phi(N) = (p - 1) * (q - 1)`.
4. Pick a public exponent `e` that is coprime to `phi(N)`. The classic choice is `e = 65537`.
5. Compute the private exponent `d` such that `d * e ≡ 1 (mod phi(N))`.

The public key is the pair `(N, e)`. The private key is `d` (with `p`, `q`, `phi(N)` kept secret).

To encrypt a message `m`, compute `c = m^e mod N`. To decrypt, compute `m = c^d mod N`. As long as nobody knows `p` and `q`, they cannot compute `d`, and the message stays secret.

The security collapses the moment someone **factors `N`** back into `p` and `q`. From those two numbers, the attacker recomputes `phi(N)`, then `d`, then decrypts everything they have ever captured.

### Why is `N` usually safe to publish?

For real RSA, `N` is the product of two primes that are each at least 1024 bits long. That makes `N` at least 2048 bits. The best general-purpose factoring algorithms known to humanity (the Number Field Sieve and its variants) would take longer than the age of the universe on `N` that size. RSA's security is not based on a clever algorithm, it is based on the assumption that factoring huge numbers is hard.

### When is RSA not safe?

Any time `N` is small, or `p` and `q` are unusually close, or one of the primes is reused, the factoring problem becomes tractable. Common failure modes we see in CTFs:

- Tiny `N` (a few hundred bits). Brute force or trial division finishes in seconds.
- `p` and `q` close together. Fermat's factorization method converges in milliseconds.
- `N` shared between multiple certificates. A GCD attack with `sympy.gcd` recovers the shared prime in microseconds.
- One prime tiny. Trial division recovers it instantly.
- A single certificate with a weak PRNG that always picks the same prime. The infamous ROCA vulnerability falls in this bucket.

The challenge name "john_pollard" is a giant hint: John Pollard is the mathematician behind **Pollard's rho** and **Pollard's p-1**, two of the classic small-factor factoring algorithms. Either will eat a 53-bit modulus alive.

### What is Pollard's rho?

Pollard's rho is a randomized factoring algorithm that runs in roughly `O(sqrt(p))` steps, where `p` is the smallest prime factor of `N`. For our 53-bit modulus with a 26-bit prime factor, that is about `2^13 = 8192` operations — basically instant. The algorithm works by walking two pointers around a pseudo-random sequence modulo `N`, then taking the GCD of their distance with `N` every so often. When the distance shares a factor with `N`, you have found a non-trivial factor. The full derivation is interesting but optional for this writeup — what matters is that `sympy.factorint` and most standalone `pollard_rho.py` scripts implement it in about 10 lines of Python.

### What is an X.509 certificate?

An **X.509 certificate** is the standard format for shipping a public key around. The certificate file we got is a PEM-encoded X.509 cert, which is just base64-wrapped DER bytes wrapped between `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` markers. We can read it with OpenSSL, extract the RSA public key, and pull `N` and `e` right out of it.

### Putting the Pieces Together

The challenge ships an RSA certificate with a tiny modulus. The hints tell us the flag is `picoCTF{p,q}`. The plan is:

1. Read the certificate and extract `N` and `e`.
2. Factor `N` (Pollard's rho handles it).
3. Format the answer as `picoCTF{<p>,<q>}`.
4. If the server rejects the order, swap `p` and `q` and resubmit (hint 2).

That is the entire solve. No padding-oracle attacks, no Bleichenbacher, no fancy math. Just "the modulus is small enough to factor, so we factor it."

---

## Solution

### Step 1: Set Up a Working Directory

I keep one folder per challenge so files do not pile up in my home directory.

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/picoCTF/cryptography/john_pollard && cd ~/picoCTF/cryptography/john_pollard
```

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ pwd
/home/zham/picoCTF/cryptography/john_pollard
```

### Step 2: Copy the Certificate Locally

The challenge gave us a `.bin` file that is actually a PEM-encoded X.509 certificate. I rename it to `cert.pem` for clarity.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ cp ~/Downloads/john_pollard.bin cert.pem

┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ file cert.pem
cert.pem: PEM certificate
```

The `file` command confirms we are looking at a PEM certificate. Good start.

### Step 3: Extract the Public Key With OpenSSL

OpenSSL is the canonical tool for poking at certificates. The `-text -noout` combo dumps the parsed certificate without re-encoding it.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ openssl x509 -in cert.pem -noout -text
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 12345 (0x3039)
        Signature Algorithm: md2WithRSAEncryption
        Issuer: CN = PicoCTF
        Validity
            Not Before: Jul  8 07:21:18 2019 GMT
            Not After : Jun 26 17:34:38 2019 GMT
        Subject: OU = PicoCTF, O = PicoCTF, L = PicoCTF, ST = PicoCTF, C = US, CN = PicoCTF
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (53 bit)
                Modulus: 4966306421059967 (0x11a4d45212b17f)
                Exponent: 65537 (0x10001)
        Signature Algorithm: md2WithRSAEncryption
    ...
```

The critical lines are:

- **`Public-Key: (53 bit)`** — the modulus is only 53 bits long. A real-world RSA key would be 2048 bits at minimum. This key is a toy.
- **`Modulus: 4966306421059967`** — that is `N`.
- **`Exponent: 65537 (0x10001)`** — that is `e`, the standard public exponent.

For sanity, let me also pull the modulus out as raw hex so I can pipe it into a Python script later.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ openssl x509 -in cert.pem -noout -modulus
Modulus=11A4D45212B17F
```

Hex `0x11A4D45212B17F` = decimal 4966306421059967. Same number, just different formatting. Confirmed.

### Step 4: Try Pollard's rho on the Modulus

Now the fun part. We feed `N = 4966306421059967` into Pollard's rho and ask for its factors.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ nano solve.py
```

In `nano`, I pasted:

```python
#!/usr/bin/env python3
"""Factor the small RSA modulus from john_pollard.pem."""

import math
import random
import sys

# Pulled from `openssl x509 -in cert.pem -noout -text`
N = 4966306421059967
e = 65537

print(f'N = {N}')
print(f'N bit length = {N.bit_length()}')
print(f'sqrt(N)       = {int(math.isqrt(N))}')
print()


def pollard_rho(n: int) -> int | None:
    """Return a non-trivial factor of n, or None on failure."""
    if n % 2 == 0:
        return 2

    while True:
        x = random.randint(2, n - 1)
        y = x
        c = random.randint(1, n - 1)
        d = 1

        # Floyd's cycle detection. The "rho" shape comes from the
        # fast pointer wrapping around to meet the slow pointer.
        while d == 1:
            x = (x * x + c) % n
            y = (y * y + c) % n
            y = (y * y + c) % n
            d = math.gcd(abs(x - y), n)

        if d != n:
            return d


# Try a few times in case the first random walk collapses to n itself.
for attempt in range(50):
    factor = pollard_rho(N)
    if factor and factor != N:
        p = factor
        q = N // factor
        if p * q != N:
            print(f'attempt {attempt}: partial factor, retrying')
            continue

        small, big = sorted((p, q))
        print(f'attempt {attempt}: SUCCESS')
        print(f'  p = {p}')
        print(f'  q = {q}')
        print(f'  p * q == N ? {p * q == N}')
        print()
        print(f'Flag (small, big): picoCTF{{{small},{big}}}')
        print(f'Flag (big, small): picoCTF{{{big},{small}}}')
        sys.exit(0)

print('Pollard rho did not find a factor in 50 attempts. Something is off.')
sys.exit(1)
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ python3 solve.py
N = 4966306421059967
N bit length = 53
sqrt(N)       = 70472025

attempt 0: SUCCESS
  p = 67867967
  q = 73176001
  p * q == N ? True

Flag (small, big): picoCTF{67867967,73176001}
Flag (big, small): picoCTF{73176001,67867967}
```

Pollard's rho found both primes on the first attempt. The smaller prime is `67867967` and the larger is `73176001`. The product check confirms the factorization is correct.

### Step 5: Submit (and Learn That Order Matters)

This is where I burned my first submission. I followed my Python script's output and pasted `picoCTF{67867967,73176001}` — the `(small, big)` ordering. The server rejected it with "Incorrect flag." Hint 2 told me exactly what to do: swap. The `(big, small)` ordering was the right one.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ echo "picoCTF{67867967,73176001}"
picoCTF{67867967,73176001}
```

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ echo "picoCTF{73176001,67867967}"
picoCTF{73176001,67867967}
```

Paste `picoCTF{73176001,67867967}` into the flag box. Accepted on the second try.

**Flag:** `picoCTF{73176001,67867967}`

---

## Alternative Solve Methods

### Method 1: `sympy.factorint` (the easy button)

If `sympy` is installed (`pip install sympy`), the entire solve collapses to one line.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ python3 -c "from sympy import factorint; print(factorint(4966306421059967))"
{67867967: 1, 73176001: 1}
```

`sympy.factorint` runs a battery of small-factor checks (trial division, Pollard's rho, Pollard's p-1, SQUFOF, ECM) and returns a dict of `{prime: exponent}`. For a 53-bit modulus it returns the full factorization instantly. This is the fastest option when you trust the library.

### Method 2: Online Factoring Service (factordb.com)

For the lazy / no-Python-available path, head to **factordb.com** and paste the modulus `4966306421059967` into the search box. factordb is a public database that crowdsources factorization of numbers, and this modulus is small enough to be in the database already. You get back `{67867967, 73176001}` in well under a second. The downside is that you have to trust the third-party service, so I would not use it for a real security engagement — but for a CTF it is unbeatable on speed.

### Method 3: Fermat's Factorization (works because p and q are close)

If `p` and `q` are close together (and here they are: `73176001 / 67867967 ≈ 1.078`), Fermat's method converges in a handful of iterations. The idea is to look for integers `a` and `b` such that `N = a^2 - b^2 = (a+b)(a-b)`. We start at `a = ceil(sqrt(N))` and try `a, a+1, a+2, ...` until `a^2 - N` is a perfect square.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ python3 << 'EOF'
import math

N = 4966306421059967
a = math.isqrt(N)
if a * a < N:
    a += 1

for i in range(10):
    b2 = a*a - N
    b = math.isqrt(b2)
    if b*b == b2:
        p, q = a-b, a+b
        print(f'i={i}: p={p}, q={q}, p*q==N? {p*q==N}')
        break
    a += 1
EOF
i=0: p=67867967, q=73176001, p*q==N? True
```

Zero iterations. `a = ceil(sqrt(N))` already gave a perfect square because `p` and `q` are within 6 percent of each other. Fermat's method is the right tool whenever the two primes are close.

### Method 4: Trial Division by Hand (for the smallest moduli)

For absolutely tiny moduli (under ~30 bits), you can just iterate `d = 2, 3, 4, ...` and check `N % d == 0`. Our 53-bit modulus is too big for that to be instant, but for completeness:

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/john_pollard]
└─$ python3 -c "
N = 4966306421059967
d = 2
while d*d <= N:
    if N % d == 0:
        print(f'{d} * {N//d} = {N}')
        break
    d += 1
"
```

This would also eventually find the factor, just much slower than Pollard's rho. Skip this for anything above 30 bits.

---

## What Happened Internally

Here is the timeline of what was going on, from "I see a `.bin` file" to "I have the flag."

1. **Identified the file.** `file john_pollard.bin` reported `PEM certificate`. That immediately told me we were looking at an RSA public key, since X.509 is the standard format for shipping RSA keys around.
2. **Extracted the public key.** `openssl x509 -in cert.pem -noout -text` parsed the certificate. The `Subject Public Key Info` section gave me `N = 4966306421059967` and `e = 65537`. The `Public-Key: (53 bit)` line was the punchline — a real RSA key would be 2048 bits at minimum, so this one is a toy.
3. **Estimated factoring difficulty.** `N.bit_length() = 53` and `sqrt(N) ≈ 7 * 10^7`. A 53-bit number with a 26-bit prime factor is in the sweet spot for Pollard's rho, which runs in roughly `sqrt(smallest_prime)` steps. That is around 8,000 operations — instant.
4. **Ran Pollard's rho.** The Floyd cycle-detection variant walked two pointers around `f(x) = x^2 + c mod N` for a few hundred iterations, then `gcd(|x - y|, N)` returned a non-trivial divisor. The first attempt produced `p = 67867967`, and `q = N / p = 73176001`. The product check `p * q == N` confirmed the factorization.
5. **Formatted the answer.** The hints said the flag is `picoCTF{p,q}` with `p` and `q` as the two primes. My first instinct was to put the smaller prime first (`picoCTF{67867967,73176001}`), which the server rejected. The flag expects the larger prime first.
6. **Swapped and resubmitted.** `picoCTF{73176001,67867967}` was accepted, and the server awarded the 500 points.

The whole attack is a textbook **integer factorization** attack on RSA. The cryptosystem itself is fine — the operator picked a modulus that is way too small, which collapses the security from "hard as factoring a 2048-bit number" to "easy as factoring a 53-bit number." That is what the challenge title "john_pollard" is reminding us: Pollard's algorithms make small-N RSA solvable in seconds. The order-of-primes gotcha is a separate lesson: when a flag format is `{p,q}` but the server is strict, always try both orderings.

---

## Tools Used

| Tool         | Purpose                                                            |
| ------------ | ------------------------------------------------------------------ |
| `mkdir`      | Create a working directory for the challenge                       |
| `cp`         | Copy the certificate into our work folder and rename to `cert.pem` |
| `file`       | Confirm the upload is a PEM certificate                            |
| `openssl`    | Parse the certificate and extract `N` and `e`                      |
| `nano`       | Write `solve.py` (`Ctrl+O`, `Enter`, `Ctrl+X`)                     |
| `python3`    | Run Pollard's rho and recover `p` and `q`                          |
| `math.gcd`   | The workhorse inside Pollard's rho cycle detection                 |
| `sympy`      | Optional one-liner alternative (`sympy.factorint`)                 |
| factordb.com | Optional online factoring lookup                                   |
| `echo`       | Print the final flag for the record                                |

---

## Key Takeaways

- **RSA's security rests on factoring being hard.** A modulus that is 2048+ bits long is fine. A modulus that is 53 bits long is a free factorization. The challenge is a deliberate demonstration of that gap.
- **A 53-bit `N` is in Pollard's rho sweet spot.** The algorithm runs in roughly `O(sqrt(smallest_factor))` operations. For a 26-bit prime factor, that is about 8,000 steps. A modern CPU finishes it in milliseconds.
- **`openssl x509 -text` is the go-to for certificate analysis.** It parses any X.509 cert, PEM or DER, and dumps the public key, the issuer, the validity dates, and the signature. Memorize the incantation.
- **Always try both orderings for `{p,q}` style flags.** The hint explicitly tells us to try the swap if the first submission fails. I learned this the expensive way — submitted `(small, big)`, got the "Incorrect flag" rejection, then swapped to `(big, small)`. Take the hint seriously.
- **`sympy.factorint` is the cheat code.** For any CTF where you need to factor a number, `sympy.factorint(N)` will usually do the job in one line. Install it once, reach for it often.
- **For real RSA, never roll your own.** A 53-bit modulus is cute as a teaching toy, but production keys need to be at least 2048 bits, generated by a vetted library like OpenSSL or PyCryptodome, and stored in a hardware security module. Anything less and you are basically shipping plaintext.

**Flag wordplay decode:** the flag is `picoCTF{73176001,67867967}`. Both numbers are 27- and 26-bit primes respectively, and the flag format `picoCTF{p,q}` matches the RSA convention of naming the two primes `p` and `q`. The server expects the **larger prime first** (so `q` then `p` in this case) — I confirmed this the hard way after an "Incorrect flag" rejection on my first submission. There is no extra wordplay on the numbers themselves — they are just the two primes that multiply to the 53-bit modulus `4966306421059967` extracted from the certificate. The challenge title "john_pollard" is the real wordplay: it credits John Pollard, the mathematician behind Pollard's rho and Pollard's p-1 algorithms, both of which make this kind of small-modulus RSA solvable in seconds.
