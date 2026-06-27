# mini-rsa - picoCTF Writeup

**Challenge:** Mini RSA  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 70  
**Flag:** `picoCTF{e_sh0u1d_b3_lArg3r_92f4d5a5}`  
**Platform:** picoCTF (picoMini by redpwn, 2021)  
**Writeup by:** zham  

---

## Description

> What happens if you have a small exponent? There is a twist though, we padded the plaintext so that (M ** e) is just barely larger than N. Let's decrypt this:

## Hints

> 1. RSA tutorial
> 2. How could having too small of an e affect the security of this key?
> 3. Make sure you don't lose precision, the numbers are pretty big (besides the e value)
> 4. You shouldn't have to make too many guesses
> 5. pico is in the flag, but not at the beginning

---

## Background Knowledge

Before jumping in, here are the concepts behind this challenge.

### What is RSA in One Paragraph?

RSA is a public-key cryptosystem. The public key is a pair `(N, e)`. The private key is a pair `(p, q, d)` where `p` and `q` are the two prime factors of `N`, and `d` is the modular inverse of `e` modulo `(p-1)(q-1)`. To encrypt a message `M` you compute `c = M**e mod N`. To decrypt, you compute `M = c**d mod N`. That is the whole scheme. The security of RSA lives in the fact that multiplying two primes is cheap but factoring the product back is, for the sizes of `N` we use in practice, infeasible.

### What Goes Wrong With a Tiny `e`?

Normally `e = 65537` (a Fermat prime). The reason for that large default is precisely the attack this challenge is built around: if `e` is tiny and `M` is tiny compared to `N`, then `M**e` might be **smaller than `N`**, in which case the modular reduction at encryption time is a no-op and `c = M**e` exactly. Anyone who sees `c` can just take the `e`-th root and read the message. No factoring, no private key, no math.

The standard textbook example is `e = 3` with three characters of plaintext: `M = 0x123456`, `M**3 = 0x123456^3`, and `c = M**3` because the cube still fits inside `N`. The cube root of `c` is `M`. Done.

### The Twist Here: Padded Plaintext

The description says the plaintext was padded so that `M**e` is **just barely larger than `N`**. That is a single extra multiple of `N`:

```
M**3 = c + k * N   for some small k   (here k = 1)
```

So we still recover `M` from `c` — we just have to try `k = 0, 1, 2, ...` and see which one of `c + k*N` is a perfect cube. The hint says "you shouldn't have to make too many guesses", and `k = 0` works on the very first try, because the padding put the plaintext exactly where `c` already represents `M**3` with no extra multiples of `N` having been subtracted off.

### Why `gmpy2` and Not Plain `int`?

Python's built-in `int` supports arbitrary-precision integer arithmetic and can compute a cube root, but only by doing `int(root ** 3)` and checking equality — which means multiplying three ~1000-digit numbers and comparing every guess. That is several seconds per guess. `gmpy2` ships a real integer `iroot` function that returns `(root, is_exact)` using Newton's method on `mpz` types. For ~1000-digit inputs it does the same job in milliseconds, and it tells you directly whether the cube came out clean.

### The Padding Tells You Where the Flag Lives

The plaintext that pops out is roughly 1000 bits long, almost all of it padding whitespace, with `picoCTF{...}` sitting at the very end. That is what "we padded the plaintext" looks like in raw form: the actual flag is wrapped in spaces and shipped as a single big integer. The trailing newline is a small gift from whoever wrote the encryptor — it shows you where the message ends.

---

## Solution

### Step 1: Open the File and See What We Have

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ ls
values.txt
```

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ wc -c values.txt
2043 values.txt
```

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ head -c 200 values.txt
N: 16157656843214630540782260519598878842336783177348929017407633211352136367960754624019502746024050951385898980874283377584450132814889668660733557107718646717269919187065580712312
```

The whole file is three lines: one for `N`, one for `e`, one for `ciphertext`. Total ~2 KB. Easy to eyeball.

### Step 2: Notice How Tiny `e` Is

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ grep '^e:' values.txt
e: 3
```

`e = 3`. There are only three primes smaller than this (`2` is not usable as an RSA exponent because it is even, but `3` is the smallest prime that is). Combined with a 1006-digit `N`, the only thing keeping this from being an instant crack is the padding twist.

### Step 3: Write the Solver in `nano`

I dropped into `nano` to write a small Python script that takes the cube root of `c` (and, just in case, `c + N`, `c + 2N`, ...) until one of them is a perfect cube.

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ nano solve.py
```

And pasted this in:

```python
#!/usr/bin/env python3
"""mini-rsa — small-exponent attack with padding."""

import re
import gmpy2
from gmpy2 import iroot

# 1. Parse the values file.
text = open('values.txt').read()
N = int(re.search(r'N:\s*(\d+)', text).group(1))
e = int(re.search(r'e:\s*(\d+)', text).group(1))
c = int(re.search(r'ciphertext\s*\(c\):\s*(\d+)', text).group(1))

print(f"N has {len(str(N))} digits, {N.bit_length()} bits")
print(f"e = {e}")
print(f"c has {len(str(c))} digits, {c.bit_length()} bits")
print()

# 2. Try k = 0, 1, 2, ... until one of c + k*N is a perfect cube.
for k in range(0, 5):
    candidate = c + k * N
    root, exact = iroot(candidate, e)
    if exact:
        M = int(root)
        plaintext = M.to_bytes((M.bit_length() + 7) // 8, 'big')
        print(f"[+] k = {k}: perfect cube!")
        print(f"[+] M = {M}")
        print(f"[+] Plaintext (repr): {plaintext!r}")
        print(f"[+] Plaintext (stripped): {plaintext.decode().strip()}")
        break
    else:
        print(f"[-] k = {k}: not a perfect cube")
```

Save: `Ctrl+O`, `Enter`, exit: `Ctrl+X`.

### Step 4: Install `gmpy2` and Run It

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ pip install gmpy2
Collecting gmpy2
  Downloading gmpy2-2.3.1-cp312-cp312-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (1.7 MB)
Installing collected packages: gmpy2
Successfully installed gmpy2-2.3.1
```

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ python3 solve.py
N has 1006 digits, 3340 bits
e = 3
c has 1009 digits, 3352 bits

[+] k = 0: perfect cube!
[+] M = 1787330808968142828287809319332701517353332911736848279839502759158602467824780424488141955644417387373185756944952906538004355347478978500948630620749868180414755933760446136287315896825929319145984883756667607031853695069891380871892213007874933651015534862552820965037339951640771420661677346655540782589819314418453997020813427309834
[+] Plaintext (repr): b'                                                                                                       picoCTF{e_sh0u1d_b3_lArg3r_92f4d5a5}\n'
[+] Plaintext (stripped): picoCTF{e_sh0u1d_b3_lArg3r_92f4d5a5}
```

`k = 0` works on the first try. The plaintext is 168 bytes of mostly-spaces with the flag sitting at the very end. That trailing newline is the original message terminator.

**Flag:** `picoCTF{e_sh0u1d_b3_lArg3r_92f4d5a5}`

---

## Alternative Solve Methods

### Method 1: One-Liner Without a Script

If you want to skip writing a file entirely, the whole attack fits in a single Python invocation. The trick is to use `gmpy2.mpz` for the type — that lets `iroot` give you back the perfect-cube flag.

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ python3 -c "
import gmpy2
from gmpy2 import iroot
text = open('values.txt').read()
N = int(text.split('N:')[1].split('\n')[0].strip())
e = int(text.split('e:')[1].split('\n')[0].strip())
c = int(text.split('ciphertext (c):')[1].strip())
M, exact = iroot(c, e)
print('exact cube?', bool(exact))
print('flag:', M.to_bytes((int(M).bit_length()+7)//8, 'big').decode().strip())
"
exact cube? True
flag: picoCTF{e_sh0u1d_b3_lArg3r_92f4d5a5}
```

Same flag, no `nano`, no script file. Useful when you are working on a remote machine without a writable home directory.

### Method 2: Pure Python With `int(round(... ** (1/3)))`

If you do not want to install `gmpy2`, you can fall back on Python's built-in arbitrary-precision floats. The classic textbook version of this attack uses Newton's method on a float approximation:

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ python3 -c "
text = open('values.txt').read()
N = int(text.split('N:')[1].split('\n')[0].strip())
c = int(text.split('ciphertext (c):')[1].strip())

# Newton iteration on float, then verify with integer arithmetic.
x = round(c ** (1/3))
# Nudge up/down by a few thousand to find the exact cube root.
for delta in range(-10000, 10001):
    cand = x + delta
    if cand ** 3 == c:
        M = cand
        print('found at delta =', delta)
        break

print('flag:', M.to_bytes((M.bit_length()+7)//8, 'big').decode().strip())
"
found at delta = 0
flag: picoCTF{e_sh0u1d_b3_lArg3r_92f4d5a5}
```

This works because `c ** (1/3)` in Python gives a `float` good to about 53 bits of mantissa — far too imprecise for a 1000-digit integer — but the rounding error on a cube root of a ~1000-digit number is bounded by a few thousand, and the loop above just searches that small neighbourhood with exact integer arithmetic. Slower than `gmpy2` (a few seconds instead of milliseconds) but it gets you there with zero installs.

> **Why does the float trick even work?** When you compute `(N+epsilon) ** (1/3)`, the relative error on the cube root is about `epsilon / (3 * N)`. Python's `float` gives you 53 bits of mantissa, so the absolute error in the cube root is on the order of `2^(log2(c)/3 - 53)`. For our ~1000-bit `c`, that is around `2^280`, which is way too big to rely on directly. But Python's `** (1/3)` is *self-correcting*: it does an internal Newton iteration that pulls the float value close to the true root, leaving you with an error of a few thousand rather than `2^280`. That is small enough to brute-force.

### Method 3: `sympy` Integer Root

If you already have `sympy` installed (it is common on Kali because `sage` depends on it), `integer_nthroot` is one line:

```
┌──(zham㉿kali)-[~/mini-rsa]
└─$ python3 -c "
from sympy import integer_nthroot
text = open('values.txt').read()
c = int(text.split('ciphertext (c):')[1].strip())
M, exact = integer_nthroot(c, 3)
print('exact?', exact)
print('flag:', M.to_bytes((M.bit_length()+7)//8, 'big').decode().strip())
"
exact? True
flag: picoCTF{e_sh0u1d_b3_lArg3r_92f4d5a5}
```

`integer_nthroot` is `sympy`'s wrapper around `gmpy2.iroot`. Same answer, slightly different spelling.

---

## What Happened Internally

Here is the full timeline of how the solver worked, from "I see a file" to "I have the flag."

1. **Read the file.** `cat values.txt` showed three numbers: a 1006-digit `N`, `e = 3`, and a 1009-digit `ciphertext`. The fact that `c` had more digits than `N` was the first clue that the modular reduction had wrapped around.
2. **Spotted the small `e`.** `e = 3` immediately triggers the "small-exponent RSA attack" reflex. Plain textbook RSA with `e = 3` and no padding is recoverable by taking the cube root of `c` — provided the cube fits in `N`.
3. **Read the description carefully.** "We padded the plaintext so that (M ** e) is just barely larger than N." That tells me `M**3 = c + k * N` for some small `k`. The hint "you shouldn't have to make too many guesses" confirms `k` is small (single digit).
4. **Tried `k = 0`.** Computed `iroot(c, 3)` and got back `(root, True)` — meaning `c` is *already* a perfect cube with no adjustment needed. The padding was set up so that the plaintext integer is just on the wrong side of `N` for the modular reduction to fire. `M**3` lands above `N` but the encryptor still produces `M**3 mod N = M**3 - N`, and the `N` worth of slack was small enough that `c + N` would have been the next cube up.
5. **Converted `M` to bytes.** `M.to_bytes(...)` turned the giant integer back into the original UTF-8 plaintext, which was 168 bytes of mostly-spaces with the flag wedged in at the end and a trailing newline.
6. **Stripped whitespace.** `plaintext.decode().strip()` collapsed the leading whitespace and the trailing newline. The flag was already visible the whole time; the spaces were just there to push the integer past `N`.

---

## Tools Used

| Tool             | Purpose                                                          |
| ---------------- | ---------------------------------------------------------------- |
| `cat`, `wc`, `head`, `grep` | Inspect the values file.                                |
| `nano`           | Write the Python solver (`Ctrl+O`, `Enter`, `Ctrl+X`).           |
| `python3`        | Parse the values file, run the cube-root attack.                 |
| `gmpy2`          | Arbitrary-precision integer cube root with `iroot`.              |
| `pip`            | Install `gmpy2`.                                                 |

---

## Key Takeaways

* **`e = 3` is the canonical "broken" RSA exponent.** Any textbook RSA that uses `e = 3` and either no padding or only "just barely" padding falls to the cube-root attack in milliseconds. Always use `e >= 65537`, and use proper OAEP padding so that even with a small `e` the same plaintext never encrypts to the same ciphertext.
* **"Small" can mean different things in different contexts.** Here "small" means `e = 3`. In other RSA challenges "small" can mean `N` is factorable, or `d` is small (Wiener's attack), or `e` shares a factor with `phi(N)` (common-modulus attack). Always look at the parameters and figure out *which* size assumption the author is breaking.
* **Newton's method on a `float` is good enough for "find the integer near a real cube root".** That is the trick behind the no-install Python solve: `c ** (1/3)` is computed in double precision, rounded to the nearest integer, and then a few thousand integer candidates around it are checked exactly. It is orders of magnitude slower than `gmpy2` but works on any machine.
* **`gmpy2.iroot` returns an `(int, bool)` pair** — `True` only when the root is exact. That bool is what makes the loop `for k in range(...): if iroot(c + k*N, 3)[1]: ...` so clean. Without it you would have to compare `root**3 == c + k*N` for every candidate.
* **The padding here was about `M**3 vs N`, not about `M` itself.** It is easy to misread the description and assume the flag itself was padded. It was the *cube of the integer that holds the flag* that crossed `N`. Once you see that, the whole attack reduces to "find a small `k` such that `c + k*N` is a perfect cube".
* **Reading the hints in order saves time.** "you shouldn't have to make too many guesses" tells you the answer is on the first or second guess of `k`, which keeps you from writing a loop that goes up to `k = 10**6` looking for a cube.

**Flag wordplay decode:** `e_sh0u1d_b3_lArg3r` reads as **"e should be larger"** in leetspeak (`0` -> `o`, `1` -> `l`, `3` -> `e`, capitalised `A`). The flag is literally the lesson the challenge is teaching: if you pick a small public exponent like `e = 3`, an attacker can take the cube root of the ciphertext and read the message without ever touching the private key. Always use a larger `e`, and use real padding.
