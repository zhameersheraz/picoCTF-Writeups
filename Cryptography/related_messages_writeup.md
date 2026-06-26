# Related Messages — picoCTF Writeup

**Challenge:** Related Messages  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{m3ssage_w1th_typ0}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> Oops! I have a typo in my first message so i sent it again! I used RSA twice so this is secure right?
>
> `chall.py`, `output.txt`

## Hints

> 1. How are the two messages related?
> 2. Franklin Reiter \_\_\_\_\_\_\_\_\_ \_\_\_\_\_\_\_\_\_ attack.

The challenge ships with two files:

- `chall.py` — the script that generated the ciphertext
- `output.txt` — the public modulus `N`, the two ciphertexts `c1` and `c2`, and the difference `m1 - m2`

`chall.py`:

```python
from Crypto.Util.number import getPrime, inverse, bytes_to_long, long_to_bytes, GCD

Message = bytes_to_long(b"[redacted]")
Message_fixed = bytes_to_long(b"[redacted]")
e = 0x11
p = getPrime(1024)
q = getPrime(1024)
phi = (p-1) * (q-1)
d = inverse(e, phi)
N = p*q

ciphertext = pow(Message, e, N)
ciphertext2 = pow(Message_fixed, e, N)

print(ciphertext, ciphertext2)
print(Message - Message_fixed)
print(N)
```

`output.txt` (truncated for readability):

```
3486364849772584627692611749053367200656673358261596068549224442954489368512244047032432842601611650021333218776410522726164792063436874469202000304563253268152374424792827960027328885841727753251809392141585739745846369791063025294100126955644910200403110681150821499366083662061254649865214441429600114378725559898580136692467180690994656443588872905046189428367989340123522629103558929469463071363053880181844717260809141934586548192492448820075030490705363082025344843861901475648208157572346004443100461870519699021342998731173352225724445397168276113254405106732294978648428026500248591322675321980719576323749
201982790559548563915678784397933493721879152787419243871599124287434576744055997870874349538398878336345269929647585648144070475012256331468688792105087899416655051702630953882466457932737483198442642588375981620937494661378586614008496182135571457352400128892078765628319466855732569272509655562943410536265866312968101366413636251672211633011159836642751480632253423529271185888171036917413867011031963618529122680143291205470937752671602494831117301480813590683791618751348224964277861127486155552153012612562009905595646626759034581358425916638671884927506025703373056113307665093346439014722219878575598308124
-3
17334845546772507565250479697360218105827285681719530148909779921509619103084219698006014339278818598859177686131922807448182102049966121282308256054696565796008642900453901629937223685292142986689576464581496406676552201407729209985216274086331582917892470955265888718120511814944341755263650688063926284195007148056359887333784052944201212155189546062807573959105963160320187551755272391293705288576724811668369745107148481856135696249862795476376097454818009481550162364943945249601744881676746859305855091288055082626399929893610275614840617858985993338556889612804266896309310999363054134373435198031731045253881
```

So we have:

| Symbol | Meaning | Source |
|---|---|---|
| `N` | RSA modulus, ~2048 bits | line 4 of `output.txt` |
| `e` | public exponent | `0x11 = 17` from `chall.py` |
| `c1` | ciphertext of `Message` (with typo) | line 1 |
| `c2` | ciphertext of `Message_fixed` (corrected) | line 2 |
| `m1 - m2` | integer difference of the two plaintexts | line 3, value `-3` |

The relationship is screaming at us: the two plaintexts differ by a constant (3), they share the same `(N, e)`, and `e = 17` is tiny. That is the textbook setup for the **Franklin-Reiter Related Message Attack**.

---

## Background Knowledge (Read This First!)

If you have never seen polynomial GCDs over `Z/NZ` before, read this section slowly — it is the only theory you need for this challenge.

### 1. RSA in 30 Seconds (Quick Recap)

| Symbol | Meaning |
|---|---|
| `p`, `q` | Two large random primes (1024 bits each here) |
| `N` | Public modulus, `p * q` (~2048 bits) |
| `phi` | Euler totient, `(p-1) * (q-1)` |
| `e` | Public exponent, encrypts: `c = m^e mod N` |
| `d` | Private exponent, decrypts: `m = c^d mod N` |

RSA is "secure" only as long as factoring `N` is hard and recovering `d` from `(N, e)` is hard. This challenge leaves `N` unbreakable but lets `d` slip out through a side door.

### 2. Why "Two RSA" Is Not "Double RSA"

The author reused the **same `(N, e)`** for both messages. Encryption is `c = m^e mod N`, and that operation is **deterministic** for fixed `(N, e)`. Reusing keys does not give any "double encryption" benefit unless the author mixes in padding, randomness, or a different exponent. Here none of that happens — the security model is simply RSA on each message independently. That means the *relationship* between the two plaintexts is the only thing we have to exploit, and the relationship is screaming at us from line 3 of `output.txt`.

### 3. What "Related Message" Means

Two messages are *related* if you can write one as a known linear function of the other:

```
m2 = a * m1 + b       for known integers a, b
```

In our case `m1 - m2 = -3`, so `m2 = m1 + 3`, i.e. `a = 1, b = 3`. The relationship is trivial, the author basically handed it to us, and hint 1 is there to make sure you read it.

### 4. Why RSA Breaks When Plaintexts Are Related (Franklin-Reiter)

The attack was published by Matthew Franklin and Michael K. Reiter in 1996. The intuition is dead simple once you accept one shift in viewpoint: instead of treating `m` as an integer and `^e` as a modular exponentiation, treat `m` as a **root of a polynomial**.

Pick `x = m1`. Then:

```
c1 = m1^e mod N
c2 = (m1 + 3)^e mod N
```

In the polynomial ring `(Z/NZ)[x]`, define:

```
f(x) = x^e       - c1
g(x) = (x + 3)^e - c2
```

Both `f` and `g` vanish at `x = m1`, because plugging in `m1` recovers exactly `c1` and `c2`. So `m1` is a **common root** of `f` and `g`. Two polynomials with a common root are not coprime — they share a non-trivial greatest common divisor. Computing `gcd(f, g)` over `(Z/NZ)[x]` therefore yields a polynomial that also has `m1` as a root.

If everything lines up (small `e`, small `b`, the polynomials are not coprime for any other reason), that GCD is **exactly** the linear polynomial `(x - m1)`. Reading off the constant term gives `m1` directly. We never factored `N`, we never recovered `d`. We just exploited that two RSA encryptions of related plaintexts share a polynomial root.

### 5. Why `e = 17` Is Important

The polynomials `f` and `g` have degree `e`. Computing their GCD with the Euclidean algorithm runs in `O(e^2)` polynomial operations. With `e = 17`, that is 17 squarings in `(Z/NZ)[x]`, each polynomial being at most 17 coefficients wide. Trivially fast. If `e = 65537`, the same attack still works in theory but the polynomials get huge — that is the territory of the **Coppersmith short-pad attack**, a related but heavier cousin of Franklin-Reiter.

### 6. Polynomial GCD Over `Z/NZ` — Why It Still Works Even Though `Z/NZ` Is Not a Field

`Z/NZ` is not a field when `N` is composite (every `p` is a zero-divisor). In a field, the Euclidean algorithm terminates cleanly; in a ring with zero-divisors, the leading coefficient of `b` can be non-invertible, and `gcd` becomes a much messier notion. In practice, two things happen:

1. The Euclidean steps almost always succeed because the leading coefficients randomly happen to be coprime to `N`.
2. If a step fails (the leading coefficient shares a factor with `N`), we have just stumbled onto a non-trivial factor of `N` — which is itself a full RSA break. So the "error" path is a happy accident.

In this challenge, every leading coefficient is coprime to `N`, the Euclidean algorithm runs to completion, and the resulting GCD is a clean monic linear polynomial `(x - m1)`. That is exactly what we want.

### 7. The Vulnerability in One Sentence

Two plaintexts related by a known linear function, encrypted with the same small public exponent `e` and the same `N`, give two polynomials that share the secret plaintext as a root — and polynomial GCD lifts it out in milliseconds.

---

## Solution — Step by Step

I worked out of `~/ctf/related-messages` on my Kali VM.

### Step 1 — Set Up the Working Directory

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/related-messages && cd ~/ctf/related-messages

┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ cp /media/sf_downloads/chall.py .
└─$ cp /media/sf_downloads/output.txt .
```

I always start by reading the source even when the flag is "obviously" recoverable — the code is the truth, the description is marketing.

### Step 2 — Sanity Check the Parameters

```
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ python3 -c "
e = 0x11
print('e       =', e)
print('e (hex) =', hex(e))
"
e       = 17
e (hex) = 0x11
```

`e = 17` confirms we are in the "small `e`" regime. Now check `N`:

```
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ python3 -c "
with open('output.txt') as f:
    lines = f.read().strip().split('\n')
c1, c2, delta, N = int(lines[0]), int(lines[1]), int(lines[2]), int(lines[3])
print('c1 bits :', c1.bit_length())
print('c2 bits :', c2.bit_length())
print('N bits  :', N.bit_length())
print('delta   :', delta)
"
c1 bits : 2045
c2 bits : 2046
N bits  : 2047
delta   : -3
```

`N` is 2047 bits (~2048), so `p` and `q` are both ~1024 bits as the source claims. Standard RSA-2048 size — completely safe to keep, we are not factoring `N`. The only thing we need is the small public exponent and the polynomial trick.

### Step 3 — Install `pycryptodome` (if Needed)

We need `long_to_bytes` for the final decode.

```
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ pip install pycryptodome
```

### Step 4 — Write the Franklin-Reiter Solver

I am not pulling in SageMath just for this — `pycryptodome` plus ~50 lines of plain Python does the job.

```
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ nano solve.py
```

Paste the following inside `nano`:

```python
#!/usr/bin/env python3
"""
picoCTF - Related Messages
Franklin-Reiter Related Message Attack

We are given two RSA ciphertexts of related messages:
    c1 = m1^e mod N
    c2 = m2^e mod N
with m2 = m1 + 3 (the script prints m1 - m2 = -3, so m2 = m1 + 3).
e is small (0x11 = 17) and the same N is reused.

The attack:
    Let x = m1.
    f(x) = x^e  - c1       (root: m1)
    g(x) = (x+3)^e - c2     (root: m1, because m2 = m1 + 3)
Both polynomials vanish at x = m1 (mod N), so their GCD over (Z/NZ)[x]
is the linear polynomial (x - m1). Reading off the constant term gives m1.
"""

from math import gcd, comb
from Crypto.Util.number import long_to_bytes

# ---- RSA parameters (read straight from output.txt) ----
c1 = 3486364849772584627692611749053367200656673358261596068549224442954489368512244047032432842601611650021333218776410522726164792063436874469202000304563253268152374424792827960027328885841727753251809392141585739745846369791063025294100126955644910200403110681150821499366083662061254649865214441429600114378725559898580136692467180690994656443588872905046189428367989340123522629103558929469463071363053880181844717260809141934586548192492448820075030490705363082025344843861901475648208157572346004443100461870519699021342998731173352225724445397168276113254405106732294978648428026500248591322675321980719576323749
c2 = 201982790559548563915678784397933493721879152787419243871599124287434576744055997870874349538398878336345269929647585648144070475012256331468688792105087899416655051702630953882466457932737483198442642588375981620937494661378586614008496182135571457352400128892078765628319466855732569272509655562943410536265866312968101366413636251672211633011159836642751480632253423529271185888171036917413867011031963618529122680143291205470937752671602494831117301480813590683791618751348224964277861127486155552153012612562009905595646626759034581358425916638671884927506025703373056113307665093346439014722219878575598308124
delta = -3          # this is Message - Message_fixed, so Message_fixed = Message + 3
N    = 17334845546772507565250479697360218105827285681719530148909779921509619103084219698006014339278818598859177686131922807448182102049966121282308256054696565796008642900453901629937223685292142986689576464581496406676552201407729209985216274086331582917892470955265888718120511814944341755263650688063926284195007148056359887333784052944201212155189546062807573959105963160320187551755272391293705288576724811668369745107148481856135696249862795476376097454818009481550162364943945249601744881676746859305855091288055082626399929893610275614840617858985993338556889612804266896309310999363054134373435198031731045253881
e    = 0x11          # 17

assert N.bit_length() in (2047, 2048)


# ---- Polynomial helpers over Z/NZ ----
# Polynomials are stored low-order first:  p[i] = coefficient of x^i.

def poly_trim(p, N):
    p = [c % N for c in p]
    while len(p) > 1 and p[-1] == 0:
        p.pop()
    return p

def poly_divmod(a, b, N):
    """Divide a by b in (Z/NZ)[x]. Returns (q, r)."""
    a, b = poly_trim(list(a), N), poly_trim(list(b), N)
    if len(b) == 1 and b[0] == 0:
        raise ZeroDivisionError("division by zero polynomial")
    if len(a) < len(b):
        return [0], a

    lead_b = b[-1] % N
    g = gcd(lead_b, N)
    if g != 1:
        # We accidentally stumbled onto a factor of N (e.g. p or q).
        raise ValueError(f"non-invertible leading coeff; gcd = {g}")

    inv_lead_b = pow(lead_b, -1, N)
    b_monic = [(c * inv_lead_b) % N for c in b]

    q = [0] * (len(a) - len(b) + 1)
    while len(a) >= len(b):
        lead_a = a[-1] % N
        deg_diff = len(a) - len(b)
        q[deg_diff] = (q[deg_diff] + lead_a) % N
        for i in range(len(b)):
            a[deg_diff + i] = (a[deg_diff + i] - lead_a * b_monic[i]) % N
        a.pop()
        a = poly_trim(a, N)

    return poly_trim(q, N), a

def poly_gcd(a, b, N):
    """GCD of a, b in (Z/NZ)[x], returned as a monic polynomial."""
    a, b = poly_trim(a, N), poly_trim(b, N)
    while not (len(b) == 1 and b[0] % N == 0):
        try:
            _, r = poly_divmod(a, b, N)
        except ValueError as exc:
            print(f"[!] Factor leaked during GCD: {exc}")
            return None
        a, b = b, r
    inv = pow(a[-1] % N, -1, N)
    return [(c * inv) % N for c in a]


# ---- Build f(x) = x^e - c1  and  g(x) = (x+3)^e - c2 ----

f = [0] * (e + 1)
f[e] = 1
f[0] = (-c1) % N
f = poly_trim(f, N)

# (x + 3)^e expanded via the binomial theorem, coefficients reduced mod N.
g = [0] * (e + 1)
for k in range(e + 1):
    g[k] = (comb(e, k) * pow(3, e - k, N)) % N
g[0] = (g[0] - c2) % N
g = poly_trim(g, N)

print("[*] Running Franklin-Reiter polynomial GCD ...")
h = poly_gcd(f, g, N)
print(f"[*] GCD polynomial (low->high): {h}")

# h should be the monic linear polynomial (x - m1).
if h is None or len(h) != 2 or h[1] != 1:
    raise SystemExit("[!] GCD did not return a monic linear polynomial.")

m1 = (-h[0]) % N
flag = long_to_bytes(m1 + 3)        # the corrected message (Message_fixed)
print(f"[+] Recovered flag: {flag.decode()}")
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 5 — Run the Solver

```
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ time python3 solve.py

[*] Running Franklin-Reiter polynomial GCD ...
[*] GCD polynomial (low->high): [17334845546772507565250479697360218105827285681719530148909779921509619103084219698006014339278818598859177686131922807448182102049966121282308256054696565796008642900453901629937223685292142986689576464581496406676552201407729209985216274086331582917892470955265888718120511814944341755263650688063926284195007148056359887333784052944201212155189546062807573959105963160320187551755272391293705288576724811668369745107148481856135696249862795476376097454818009481550162364943945249601744881676746859305855091288055082626399929893610275614840617858985993157918294843766363627457397471216401878877293277050524140638847, 1]
[+] Recovered flag: picoCTF{m3ssage_w1th_typ0}

real    0m0.092s
user    0m0.082s
sys     0m0.007s
```

Flag recovered in under 100 milliseconds. The GCD came out as a 2-coefficient polynomial `(x - m1)`, which is exactly the success criterion — the leading coefficient is `1` (monic), and the constant term is `-m1 mod N`.

### Step 6 — Verify by Re-Encrypting

Always confirm by feeding the recovered plaintext back into the original encryption and checking the ciphertext matches.

```
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ python3 -c "
from Crypto.Util.number import bytes_to_long, long_to_bytes

N = 17334845546772507565250479697360218105827285681719530148909779921509619103084219698006014339278818598859177686131922807448182102049966121282308256054696565796008642900453901629937223685292142986689576464581496406676552201407729209985216274086331582917892470955265888718120511814944341755263650688063926284195007148056359887333784052944201212155189546062807573959105963160320187551755272391293705288576724811668369745107148481856135696249862795476376097454818009481550162364943945249601744881676746859305855091288055082626399929893610275614840617858985993338556889612804266896309310999363054134373435198031731045253881
c1 = 3486364849772584627692611749053367200656673358261596068549224442954489368512244047032432842601611650021333218776410522726164792063436874469202000304563253268152374424792827960027328885841727753251809392141585739745846369791063025294100126955644910200403110681150821499366083662061254649865214441429600114378725559898580136692467180690994656443588872905046189428367989340123522629103558929469463071363053880181844717260809141934586548192492448820075030490705363082025344843861901475648208157572346004443100461870519699021342998731173352225724445397168276113254405106732294978648428026500248591322675321980719576323749
c2 = 201982790559548563915678784397933493721879152787419243871599124287434576744055997870874349538398878336345269929647585648144070475012256331468688792105087899416655051702630953882466457932737483198442642588375981620937494661378586614008496182135571457352400128892078765628319466855732569272509655562943410536265866312968101366413636251672211633011159836642751480632253423529271185888171036917413867011031963618529122680143291205470937752671602494831117301480813590683791618751348224964277861127486155552153012612562009905595646626759034581358425916638671884927506025703373056113307665093346439014722219878575598308124
e = 17

m1 = bytes_to_long(b'picoCTF{m3ssage_w1th_typ0z')   # Message (with typo)
m2 = bytes_to_long(b'picoCTF{m3ssage_w1th_typ0}')   # Message_fixed (corrected)
print('m1 - m2 =', m1 - m2)
print('c1 == m1^e mod N :', pow(m1, e, N) == c1)
print('c2 == m2^e mod N :', pow(m2, e, N) == c2)
"
m1 - m2 = -3
c1 == m1^e mod N : True
c2 == m2^e mod N : True
```

Both ciphertexts match. The recovered flag is the corrected message `m2 = m1 + 3`. The typo was the closing `}` of the flag getting mistyped as `z` (`}` is `0x7d`, `z` is `0x7a`, difference of exactly 3 — and since the last byte is the least significant byte of the integer, that single-byte difference is the entire `-3` reported by the script).

---

## What Happened Internally (Timeline)

| Step | What the program did | What I did |
|------|----------------------|------------|
| 1 | Author called `getPrime(1024)` twice to get `p` and `q`, then set `N = p*q` (~2048 bits). | — |
| 2 | Author picked `e = 0x11 = 17`, computed `d = inverse(e, phi)`. Standard RSA key generation. | Read `chall.py`, spotted `e = 17` and `print(Message - Message_fixed)` on line 15. |
| 3 | `Message` (typo) was encrypted to `c1`, then `Message_fixed` (corrected) was encrypted to `c2`, both with the same `(N, e)`. The difference `m1 - m2 = -3` and `N` were written to `output.txt`. | Parsed `output.txt` into `c1, c2, delta, N`. |
| 4 | — | Defined `f(x) = x^17 - c1` and `g(x) = (x+3)^17 - c2` in `(Z/NZ)[x]`. Both vanish at `x = m1`. |
| 5 | — | Ran the Euclidean algorithm to compute `gcd(f, g)` in `(Z/NZ)[x]`. Each step is at most 17-coefficient polynomial arithmetic mod N, with one modular inverse per division. |
| 6 | — | The GCD collapsed to a monic linear polynomial `(x - m1)`, meaning `f` and `g` shared exactly one root in `(Z/NZ)`. That root IS the original (typo'd) message. |
| 7 | — | Read off `m1 = -h[0] mod N`, added `3` to recover the corrected message, and decoded the bytes: `picoCTF{m3ssage_w1th_typ0}`. |

The whole attack is "exploit that two polynomials built from related plaintexts share a secret root" — no factoring, no LLL, no Sage.

---

## Alternative Method 1 — SageMath One-Liner

If you have SageMath installed, the polynomial GCD is a built-in operation and the whole solver collapses to a few lines.

```
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ sage
sage: R.<x> = PolynomialRing(Zmod(N))
sage: f = x^17 - c1
sage: g = (x + 3)^17 - c2
sage: h = gcd(f, g)
sage: m1 = -h.monic().coefficients()[0]
sage: print(long_to_bytes(m1 + 3))
picoCTF{m3ssage_w1th_typ0}
```

Sage hides the messy Euclidean-loop bookkeeping (`poly_trim`, `poly_divmod`, leading-coefficient inversion) behind its `gcd` implementation. The math is identical — this is the canonical Franklin-Reiter recipe.

---

## Alternative Method 2 — RsaCtfTool (Black-Box Version)

If you don't want to write any code, `RsaCtfTool` ships a built-in `--attack related_message` that automates the Franklin-Reiter attack (and several variants).

```
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ git clone https://github.com/RsaCtfTool/RsaCtfTool.git
┌──(zham㉿kali)-[~/ctf/related-messages]
└─$ cd RsaCtfTool && pip install -r requirements.txt
┌──(zham㉿kali)-[~/ctf/RsaCtfTool]
└─$ python3 RsaCtfTool.py \
    --n 17334845546772507565250479697360218105827285681719530148909779921509619103084219698006014339278818598859177686131922807448182102049966121282308256054696565796008642900453901629937223685292142986689576464581496406676552201407729209985216274086331582917892470955265888718120511814944341755263650688063926284195007148056359887333784052944201212155189546062807573959105963160320187551755272391293705288576724811668369745107148481856135696249862795476376097454818009481550162364943945249601744881676746859305855091288055082626399929893610275614840617858985993338556889612804266896309310999363054134373435198031731045253881 \
    --e 17 \
    --uncipherfile output.txt \
    --attack related_message
```

`RsaCtfTool` will detect the small `e` and the duplicated `(N, e)`, invoke Franklin-Reiter under the hood, and print the flag.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` / `nano` | Read `chall.py`, inspect `output.txt`, write the solver |
| Python 3 | Polynomial arithmetic, modular inverse, Euclidean GCD over `(Z/NZ)[x]` |
| `pycryptodome` (`Crypto.Util.number`) | `long_to_bytes` to print the recovered plaintext |
| SageMath (alternative 1) | One-liner `gcd` over `PolynomialRing(Zmod(N))` |
| `RsaCtfTool` (alternative 2) | Black-box Franklin-Reiter with `--attack related_message` |

---

## Key Takeaways

- **Reusing the same `(N, e)` across multiple plaintexts is not "double encryption."** RSA is deterministic for fixed keys, so reusing them leaks every relationship that holds between the plaintexts. Real-world systems use padding schemes (OAEP) precisely to avoid this.
- **Any known linear relationship between two RSA plaintexts is a full break**, as long as `e` is small enough that polynomial GCD is cheap. The relationship does not have to be addition — it can be `m2 = a*m1 + b` for any known `(a, b)`, and the attack generalises immediately.
- **Polynomial GCD over `(Z/NZ)[x]` is the right abstraction for "shared root of two modular polynomial equations."** Most RSA challenges that mention "two related messages" reduce to this exact construction, and the Euclidean algorithm is the workhorse.
- **Small `e` is fine for single-message RSA but lethal under related-plaintext attacks.** The author chose `e = 17` because it speeds up encryption — that is also the reason Franklin-Reiter breaks it in under a second.
- **`Z/NZ` is not a field when `N` is composite**, but polynomial GCD still works because the leading coefficients almost always end up coprime to `N`. The rare "non-invertible leading coefficient" failure mode is a feature, not a bug — it hands you a factor of `N` for free.
- **Always re-encrypt the recovered plaintext and check the ciphertext.** It costs nothing and catches off-by-one errors in `m1 - m2 = -3` vs `m2 - m1 = 3` (a classic CTF stumble).
- **`RsaCtfTool` is the lazy-but-correct move** when you would rather not reimplement the Euclidean loop. The `--attack related_message` mode is built on exactly the same math.

### Flag Wordplay Decode

The flag is `picoCTF{m3ssage_w1th_typ0}`. Read the inner word:

- `m3ssage` → "message" with the classic leet `3 → e` swap. The flag content is "message" — literally, an RSA-encrypted message.
- `_w1th_` → "with", with `1 → i` swap. The flag is "with" typos.
- `typ0` → "typo", with `0 → o` swap. The flag literally names the vulnerability: typos in plaintexts.
- `}` → the closing brace that was mistyped as `z` in the first message — `}` is `0x7d`, `z` is `0x7a`, exactly 3 apart, exactly the `delta` reported by the challenge script.

So the readable part of the flag is **"message with typo"** — a self-describing name for the exact vulnerability the author introduced by encrypting a typo'd flag, fixing it, and sending it again with the same key. The challenge title, the hint, and the flag all say the same thing in three different ways.
