# Not TRUe - picoCTF Writeupg

**Challenge:** Not TRUe  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{th4ts_s0_N0t_TRU3_42aa9b67}`  
**Platform:** picoCTF (2024)  
**Writeup by:** zham  

---

## Description

> no. no. that's Not TRUe. that's impossible!

The title and the description are both doing the same thing: pointing me at the word "TRUe" inside "NoT TRUe." That capitalisation is the whole clue. The challenge is the **NTRU** public-key cryptosystem, with the letters rearranged inside "TRUe." The challenge file ships with an `encrypt.py` (the encryption script) and a `public.txt` (the public key plus ciphertext). My job is to recover the plaintext flag from those two files.

## Hints

> This is a lattice based cryptosystem. Is there a lattice attack that allows you to compute the private key given the public information?

The hint is doing the work for me. "Lattice based" rules out RSA, ECC, Diffie-Hellman — it has to be NTRU, GGH, or one of the LWE-style schemes. The second sentence narrows it: there is a known attack that recovers the private key (not just decrypts a single message). That is exactly the NTRU lattice attack: build a 2D x 2D integer lattice from the public key, run LLL, and read the short vector as `(p*g, f)`.

---

## Background Knowledge

If you have never touched NTRU before, here are the small ideas you need to follow the solve.

**Polynomials, not numbers.** NTRU works entirely inside the ring `R_q = (Z/qZ)[x] / (x^N - 1)`. Every "number" in NTRU is actually a polynomial of degree less than `N`, with coefficients reduced mod `q`. The relation `x^N = 1` means coefficients wrap around: the `x^N` term disappears. Think of it as modular arithmetic where the modulus is a polynomial, not an integer.

**Short polynomials are the secret sauce.** A "ternary" polynomial has every coefficient in `{-1, 0, 1}`. NTRU's private key is a pair of ternary polynomials `(f, g)`. They are tiny — coefficients are essentially nothing. The public key is built out of them in a way that makes the private key look random, but only if you cannot find short vectors in a particular lattice.

**Key generation in one line.** Pick random ternary `f` and `g`. Compute `h = p * f_q_inv * g` in `R_q`, where `f_q_inv` is the inverse of `f` in `R_q` and `p` is a small prime. Multiply both sides by `f` and you get the relation the whole attack rests on:

```
f * h  =  p * g      (mod q, mod x^N - 1)
```

This says "if I had the public key `h` and a short polynomial `f`, I could read off `p*g` immediately, and then I'd be able to decrypt any ciphertext." So recovering `(f, g)` from `h` is exactly breaking NTRU.

**The NTRU lattice is just this equation, packed into a matrix.** A circulant matrix of `h` is a matrix `H` such that `H @ v` equals the polynomial product `h * v`. The standard 2N x 2N NTRU lattice is:

```
M = [ q * I_N    H   ]
    [   0       I_N ]
```

A short row `(a, b)` of `M` satisfies `b` small and `a = q*k + H*b` for some integer vector `k`. Rearranging, `b * h - a ≡ 0 (mod q)`. Plug in the relation we want — `a = p*g` and `b = f` — and you see that `(p*g, f)` is in the lattice. Because `f` and `g` are tiny, this row is much shorter than any "random" lattice row. **LLL finds it.**

**LLL is a black box that finds short vectors.** LLL (Lenstra–Lenstra–Lovász) takes any lattice basis and returns a new basis whose vectors are all short — exponentially shorter than the original in practice. You do not need to understand the algorithm. You just call it and look at the output.

**The parameters here are deliberately toy.** Real NTRU uses `N = 701` and `q = 8192` (or similar). This challenge uses `N = 48` and `q = 509`. LLL on a 96-dimensional lattice takes a fraction of a second. That is the entire point: the hint is telling me this is a toy, and toy parameters mean lattice reduction works.

**Decryption is the easy half.** Once I have `f`, decryption is three steps per ciphertext block:
1. `a = f * ct` mod `q`, then center each coefficient to `[-q/2, q/2]`. The "noise" term `p^2 * g * r` is small enough that centering kills it.
2. `b = a mod p`. The `p^2` factor makes the noise vanish, leaving `b = f * m mod p`.
3. `m = f_p_inv * b mod p`, where `f_p_inv` is `f^(-1)` in `R_p`.

Then convert the recovered polynomial's bits back to ASCII and read the flag.

---

## Step-by-step Solution

### Step 1: Read the public file and the encrypt script

First thing I do is sanity-check the inputs.

```
┌──(zham㉿kali)-[~/picoCTF/nottrue]
└─$ cat public.txt
N = 48
p = 3
q = 509
h = [83, 440, 124, 416, 371, 240, 320, 408, 130, 337, 399, 169, 81, 354,
     221, 446, 309, 97, 148, 403, 506, 163, 195, 379, 91, 354, 24, 11,
     72, 264, 343, 442, 319, 10, 418, 114, 140, 68, 116, 102, 507, 347,
     65, 35, 7, 273, 290, 175]
ct = [[266, 103, 183, ...], [482, 430, 145, ...], ...]   # 6 blocks of length 48
```

Tiny parameters, six ciphertext blocks, each a polynomial of degree 47 with coefficients in `Z/509Z`. That fits the standard NTRU-48 toy instance.

Then I peek at `encrypt.py` to confirm the math:

```
┌──(zham㉿kali)-[~/picoCTF/nottrue]
└─$ cat encrypt.py | head -50
from random import randint
from sage.all import *

N = 48
p = 3
q = 509

R = PolynomialRing(ZZ, 'x')
x = R.gen()
R_modq = PolynomialRing(Integers(q), 'x').quotient(x**N - 1, 'xbar')
R_modp = PolynomialRing(Integers(p), 'x').quotient(x**N - 1, 'xbar')

def gen_poly():
    return R([randint(-1,1) for _ in range(N)])

def gen_msg(text):
    binary_str = ''.join(format(ord(char), '08b') for char in text)
    ...
    polynomials = [R([int(bit) for bit in chunk]) for chunk in chunks]
    return polynomials

def encrypt(h, m):
    r = gen_poly()
    return R_modq(p*(h*r) + m)

def generate_keys():
    while True:
        f = gen_poly()
        g = gen_poly()
        try:
            f_p_inv = R_modp(f)**-1
            f_q_inv = R_modq(f)**-1
            break
        except:
            continue
    h = R_modq(p*(f_q_inv*g))
    ...
```

Two things confirm the attack plan:

- `h = p * f_q_inv * g` in `R_q`. This is the exact NTRU key-generation step. The relation `f * h = p * g` holds, so the lattice attack will work.
- Encryption is `ct = p * (h * r) + m` in `R_q`. Standard NTRU encryption.

### Step 2: Pick the right tooling

SageMath is the canonical NTRU tool, but it is enormous to install. For a 2N = 96 dimensional lattice, **`fpylll`** (the Python bindings for FPLLL) is more than enough. I install it on my Kali VM.

```
┌──(zham㉿kali)-[~/picoCTF/nottrue]
└─$ pip install fpylll --break-system-packages
Collecting fpylll
  Downloading fpylll-0.6.4-cp311-cp311-manylinux_2_17_x86_64.whl (41.6 MB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 41.6/41.6 MB 39.9 MB/s eta 0:00:00
Successfully installed fpylll-0.6.4
```

I also need `sympy` for the polynomial inverse mod `p` and `numpy` for the circulant matrix.

```
┌──(zham㉿kali)-[~/picoCTF/nottrue]
└─$ pip install sympy numpy --break-system-packages
...
Successfully installed mpmath-1.3.0 numpy-2.4.0 sympy-1.14.0
```

### Step 3: Write the solver

The solver has four pieces: polynomial multiplication, the circulant matrix of `h`, the lattice + LLL step, and decryption. I write it to `solve.py` with `nano`.

```
┌──(zham㉿kali)-[~/picoCTF/nottrue]
└─$ nano solve.py
```

```python
#!/usr/bin/env python3
"""NTRU lattice attack on picoCTF 'Not TRUe'."""
import numpy as np
from fpylll import IntegerMatrix, LLL
from sympy import Poly, GF
from sympy.abc import x as sx

# Public parameters from public.txt
N, p, q = 48, 3, 509

h = [83, 440, 124, 416, 371, 240, 320, 408, 130, 337, 399, 169, 81, 354,
     221, 446, 309, 97, 148, 403, 506, 163, 195, 379, 91, 354, 24, 11,
     72, 264, 343, 442, 319, 10, 418, 114, 140, 68, 116, 102, 507, 347,
     65, 35, 7, 273, 290, 175]

ct = [[266, 103, 183, 99, 223, 103, 1, 377, 148, 10, 152, 500, 110, 255,
       445, 248, 419, 283, 163, 260, 241, 433, 441, 481, 166, 159, 355,
       153, 270, 357, 163, 481, 52, 498, 96, 444, 433, 1, 266, 176, 263,
       264, 98, 111, 28, 280, 193, 363],
      [482, 430, 145, 33, 17, 508, 275, 281, 114, 483, 458, 143, 277, 276,
       186, 492, 277, 176, 415, 123, 274, 239, 195, 163, 275, 82, 497,
       364, 494, 113, 161, 187, 229, 495, 85, 169, 58, 496, 300, 421, 191,
       21, 214, 377, 488, 439, 327, 169],
      [197, 239, 59, 404, 152, 444, 314, 168, 193, 326, 390, 51, 139, 296,
       4, 357, 64, 476, 12, 313, 207, 401, 78, 492, 401, 75, 399, 95, 27,
       178, 422, 242, 105, 125, 287, 11, 93, 397, 71, 495, 275, 57, 322,
       413, 363, 489, 54, 45],
      [304, 172, 179, 19, 250, 416, 209, 0, 431, 123, 387, 404, 102, 474,
       445, 174, 229, 464, 425, 204, 233, 426, 362, 179, 252, 377, 208,
       87, 451, 334, 257, 214, 222, 239, 415, 66, 299, 469, 215, 283, 87,
       444, 35, 451, 141, 185, 404, 269],
      [287, 283, 174, 167, 453, 203, 271, 389, 278, 264, 483, 181, 453, 147,
       385, 254, 338, 330, 46, 77, 229, 247, 323, 97, 323, 232, 74, 99,
       364, 165, 240, 222, 405, 107, 482, 508, 174, 97, 86, 397, 468, 353,
       377, 56, 407, 203, 296, 119],
      [280, 434, 358, 332, 155, 154, 267, 179, 181, 383, 435, 233, 157, 250,
       39, 181, 37, 52, 179, 361, 132, 254, 20, 183, 99, 393, 274, 401,
       144, 178, 423, 187, 44, 376, 186, 256, 485, 377, 222, 433, 87, 470,
       451, 153, 216, 248, 23, 117]]


def poly_mult(a, b, N, mod=None):
    """Polynomial product of a and b in Z[x]/(x^N - 1), optionally mod m."""
    out = [0] * N
    for i, ai in enumerate(a):
        for j, bj in enumerate(b):
            out[(i + j) % N] += ai * bj
    if mod is not None:
        out = [c % mod for c in out]
    return out


def circulant(h, N):
    """Circulant matrix H where (H @ v) is the polynomial h * v mod x^N - 1."""
    M = np.zeros((N, N), dtype=int)
    for i in range(N):
        M[i] = np.roll(h, i)        # row i = h rolled right by i
    return M


def f_p_inv(f, N, p):
    """Inverse of f in (Z/pZ)[x] / (x^N - 1) via sympy."""
    fpoly = Poly(sum(int(c) * sx ** i for i, c in enumerate(f)), sx, domain=GF(p))
    modulus = Poly(sx ** N - 1, sx, domain=GF(p))
    inv = fpoly.invert(modulus)
    coeffs = [0] * N
    for monom, c in inv.terms():
        coeffs[int(monom[0])] = int(c) % p
    return coeffs


def decrypt(block, f, N, p, q):
    """NTRU decryption: a = f * ct mod q (centered), then m = f_p_inv * (a mod p)."""
    a = poly_mult(f, block, N, mod=q)
    a = [((c + q // 2) % q) - q // 2 for c in a]
    b = [c % p for c in a]
    inv = f_p_inv(f, N, p)
    return poly_mult(inv, b, N, mod=p)


def bits_to_text(bits):
    s = ''.join(str(b) for b in bits)
    out = ''
    for i in range(0, len(s), 8):
        byte = s[i:i + 8]
        if len(byte) < 8:
            break
        out += chr(int(byte, 2))
    return out


# Build the NTRU lattice and run LLL
H = circulant(h, N)
M = np.zeros((2 * N, 2 * N), dtype=int)
M[:N, :N] = q * np.eye(N, dtype=int)
M[N:, :N] = H
M[N:, N:] = np.eye(N, dtype=int)

Mi = IntegerMatrix.from_matrix(M.tolist())
LLL.reduction(Mi)
Mred = np.array([[Mi[i, j] for j in range(2 * N)] for i in range(2 * N)],
                 dtype=int)

# Hunt for a row whose second half is ternary (that is f)
for row in Mred:
    f_cand = row[N:]
    if all(x in (-1, 0, 1) for x in f_cand):
        f = f_cand.tolist()
        break
else:
    raise RuntimeError("No ternary candidate found")

# Decrypt each block
flag_bits = []
for block in ct:
    m = decrypt(block, f, N, p, q)
    flag_bits.extend(int(b) for b in m)

print(bits_to_text(flag_bits))
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

A note on the circulant matrix: row `i` is `h` rolled right by `i` positions. This makes `(H @ v)` exactly the polynomial product `h * v` reduced mod `x^N - 1`. The other direction (`np.roll(h, -i)`) does not work for this attack — I tried it and LLL no longer finds the right short vector. Worth knowing if you ever write this from scratch.

### Step 4: Run the solver

```
┌──(zham㉿kali)-[~/picoCTF/nottrue]
└─$ python3 solve.py
picoCTF{th4ts_s0_N0t_TRU3_42aa9b67}
```

LLL finishes in about a second, the private-key hunt finds a ternary row on the first try, and decryption spits the flag out as the only line of output.

### Step 5: Verify the recovered private key

Before I submit, I sanity-check that the recovered `f` actually satisfies the NTRU relation `f * h ≡ p * g (mod q)`. I extend the script briefly to print `g` and verify.

```
┌──(zham㉿kali)-[~/picoCTF/nottrue]
└─$ python3 -c "
from solve import poly_mult, N, p, q, h
from fpylll import IntegerMatrix, LLL
import numpy as np

H = (np.zeros((N, N), dtype=int),)
import solve as S
H = S.circulant(h, N)
M = np.zeros((2*N, 2*N), dtype=int)
M[:N, :N] = q * np.eye(N, dtype=int); M[N:, :N] = H; M[N:, N:] = np.eye(N, dtype=int)
Mi = IntegerMatrix.from_matrix(M.tolist()); LLL.reduction(Mi)
Mred = np.array([[Mi[i, j] for j in range(2*N)] for i in range(2*N)], dtype=int)
for row in Mred:
    if all(x in (-1, 0, 1) for x in row[N:]):
        p_g = row[:N].tolist()
        f = row[N:].tolist()
        break
g = [c // p for c in p_g]
fh = poly_mult(f, h, N, mod=q)
fh_c = [((c + q//2) % q) - q//2 for c in fh]
print('f*h mod q (centered):', fh_c)
print('p*g expected       :', p_g)
print('Match:', fh_c == p_g)
"
f*h mod q (centered): [0, -3, 0, 3, 3, 3, 3, 0, 3, -3, -3, -3, -3, 0, 3, 3, 3, 3, -3, 0, -3, -3, 3, 3, 0, -3, -3, 3, 0, -3, -3, 3, -3, 3, 0, -3, -3, -3, 3, 3, -3, -3, 0, 0, 0, 0, 0, 3]
p*g expected       : [0, -3, 0, 3, 3, 3, 3, 0, 3, -3, -3, -3, -3, 0, 3, 3, 3, 3, -3, 0, -3, -3, 3, 3, 0, -3, -3, 3, 0, -3, -3, 3, -3, 3, 0, -3, -3, -3, 3, 3, -3, -3, 0, 0, 0, 0, 0, 3]
Match: True
```

`f` and `g` are both ternary, and `f * h ≡ p * g (mod q)` holds exactly. The recovered key is the genuine private key, not a lattice artefact. Submit.

---

## Alternative Solve Methods

**Method 1: Sage one-liner (if you have it).**

If you already have SageMath installed, the entire attack collapses to a few lines because Sage has built-in polynomial rings and an LLL method on matrices:

```python
# Sage
N, p, q = 48, 3, 509
h = [...]  # load from public.txt

P.<x> = PolynomialRing(Integers(q))
H = matrix(ZZ, [[h[(j-i) % N] for j in range(N)] for i in range(N)])

B = block_matrix([[q * identity_matrix(N), H],
                  [zero_matrix(N),       identity_matrix(N)]])
B = B.LLL()

for row in B:
    f_cand = row[N:]
    if all(c in (-1, 0, 1) for c in f_cand):
        f = list(f_cand)
        # decrypt ct[i] using f as in solve.py
        break
```

Sage's `Poly.invert()` makes the `f_p_inv` step a one-liner too.

**Method 2: Pure Python with a hand-rolled LLL.**

For a 96-dim lattice you can implement the LLL algorithm in about 80 lines of pure Python. Educational only — I would not actually run it because `fpylll` is C-optimised and finishes faster than I can type. But it works, and you learn a lot about how LLL picks short vectors.

**Method 3: Brute-force instead of LLL.**

For this parameter set, `f` has 48 ternary coefficients, so the key space is `3^48 ≈ 2 × 10^22`. That is too big to brute-force directly. But you could restrict to "balanced" ternary keys (equal numbers of `-1`, `0`, `1`) and search a smaller slice. This is more of a thought experiment than a real attack — LLL is the right tool.

**Method 4: Use an online lattice-reduction service.**

Some CTF players submit the lattice to an online NTRU solver (e.g. the one at `https://latticehacker.cryptography.io/` if you can find a current one) and paste back the recovered key. Fine for a CTF, but you do not learn anything.

---

## What Happened Internally

A timeline of how the solve played out, step by step.

- **t = 0s — Read `public.txt`.** I saw `N = 48`, `p = 3`, `q = 509`, an `h` array, and a `ct` array with 6 blocks of length 48. The numbers are tiny, the structure is NTRU. LLL will work.
- **t = 30s — Read `encrypt.py`.** Confirmed `h = p * f_q_inv * g` (the standard NTRU relation) and `ct = p * h * r + m`. Confirmed the lattice attack is the right move.
- **t = 1 min — Installed `fpylll`, `sympy`, `numpy`.** All three are pip-installable, no system packages needed. Total install time about 20 seconds.
- **t = 2 min — Wrote `solve.py`.** Five small functions: polynomial multiply, circulant matrix, polynomial inverse mod `p`, decrypt, bits-to-text. The lattice construction is six lines.
- **t = 3 min — First run.** LLL reduced the 96 x 96 lattice in well under a second. The first reduced row whose second half was ternary turned out to be a valid `f`. Decryption of all 6 blocks yielded the flag in one line of output: `picoCTF{th4ts_s0_N0t_TRU3_42aa9b67}`.
- **t = 4 min — Verification.** Computed `f * h mod q` and confirmed it equals `p * g` for the recovered `g`. The match was exact. The recovered key is the real private key, not an accidental short vector.
- **t = 5 min — Submit.** Submitted the flag, got the points.

The whole challenge is roughly 5 minutes of real work after you understand the math. Most of the time is spent understanding NTRU the first time you see it; the actual code is short.

---

## Tools Used

| Tool | Purpose | Notes |
| --- | --- | --- |
| `cat` | Read `public.txt` and `encrypt.py` | First two commands I run on every challenge |
| `nano` | Write `solve.py` | Plain editor, no IDE needed for a 100-line script |
| `pip` | Install `fpylll`, `sympy`, `numpy` | `--break-system-packages` because Kali is PEP-668 locked by default |
| `python3` | Run `solve.py` | Python 3.11 in my VM |
| `fpylll` | LLL lattice reduction | Python bindings for the FPLLL C library; fastest LLL implementation available |
| `sympy` | Polynomial inverse mod `p` | `Poly.invert()` does the heavy lifting for `f_p_inv` |
| `numpy` | Circulant matrix construction | `np.roll` and `np.eye` are enough |

---

## Key Takeaways

- **"Lattice-based" is the magic phrase.** When a challenge description or hint mentions lattices, the public-key scheme is almost certainly NTRU, GGH, or an LWE variant. All three have known attacks when parameters are small. The hint tells me which side of the attack to focus on: in this case, recovering the private key, not just decrypting a message.
- **The NTRU lattice attack is short.** Build the 2N x 2N matrix `M = [qI, H; 0, I]`, call LLL, scan the reduced basis for a ternary row. That is the entire attack. The math is the hard part to learn, not the code.
- **Circulant matrices encode polynomial multiplication.** Once you see that `(H @ v)` is exactly the polynomial product `h * v`, the NTRU lattice writes itself. `H_{i,j} = h[(j-i) mod N]` (or equivalently, row `i` of `H` is `h` rolled right by `i`).
- **Decryption has a built-in noise cancellation.** Multiplying the ciphertext by `f` introduces a `p^2` factor on the noise term. Reduce mod `p` and the noise vanishes. Then a single multiplication by `f_p_inv` recovers the message.
- **Toy parameters are the giveaway.** `N = 48` and `q = 509` are not "hardened NTRU." They are deliberately vulnerable so LLL finishes in a second. Standardized NTRU uses `N = 701` or larger to push the lattice into a regime where no known attack finishes.
- **Verify the recovered key before submitting.** Computing `f * h ≡ p * g (mod q)` is a 30-second sanity check that distinguishes "I found a short vector" from "I found the private key." Always do it.

**Flag wordplay decode.** The flag is `th4ts_s0_N0t_TRU3`, leetspeak for "that's so Not TRUe." The challenge title is "Not TRUe" with the inner word capitalised as a hint, and the flag doubles down on it: the system the challenge is built on is NTRU, so "Not TRUe" is a pun — the security of NTRU is, in fact, Not True for these parameters. The `42aa9b67` suffix is the per-instance hash that picoCTF appends so every challenger's flag is unique.
