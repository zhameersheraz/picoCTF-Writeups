# Small Trouble — picoCTF Writeup

**Challenge:** Small Trouble  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{sm4ll_d_3d2584a9}`  
**Platform:** picoCTF (2026)  
**Author (challenge):** Yahaya Meddy  
**Writeup by:** zham  

---

## Description

> Everything seems secure; strong numbers, familiar parameters but something small might ruin it all. Can you recover the message?
>
> Download the message and source code.

## Hints

> 1. This might be a job for Boneh-Durfee.

The challenge ships with two files:

- `code.py` — the script that generated the ciphertext
- `message.txt` — the public key `(n, e)` and the ciphertext `c`

`code.py`:

```python
from Crypto.Util.number import getPrime, inverse, bytes_to_long
import random

# Generate two large primes (1048 bits each)
p = getPrime(1048)
q = getPrime(1048)
n = p * q
phi = (p - 1) * (q - 1)

# compute d
d = getPrime(256)

# Compute the public exponent
e = inverse(d, phi)

# Encrypt a flag
flag = b'picoCTF{...}'
m = bytes_to_long(flag)
c = pow(m, e, n)
```

`message.txt`:

```
n = 3980993015101017140353277804369745949706344223334890812840413257071044175973139291856038905399468319530362030870405443017903655374929929825011733964330181337551061274726015170805648591339935699566483704533804413902772334194671185814032757795167276567492610548489766891166023546451221494011601527912413323143640008533569546329046772124896140545774468831882189458647605131344071766512469603284171712833770168967153998797122819825938323489525196597407482515674210085730485950171311356595462370875129472210846138025654237220573246971086221110031195278019873474587710322597773432501095235059971088078096926751790014797097704337356321889
e = 3336497038614663222541921459680551835187913731404732354361770350895041393777650313575506088859422849385970294213215931070595265411244105028409623278013975724125500479919877564207207562572189315593170541926598007670763000263224010236185517132229343161667668429997117301579363090706170019283552359847390040311521617091884426220652237247589452213130475528794167610900761281292390567153221748933849959961895636324991675428047481887984343193310278137379109243220192216924968789976201537337617841164156347418356710488273730184036786730194843318185762453423678066812440979743181541976510807538521163968859836855958692325875567630213728207
c = 1826031079095822845536526213992736069835634421976077836267143793181964583083024760950942723002771287666852728279522910274806690415793596687180742873080830698626932982997385432329275546871401690500831953978994557961462780510004840010773264058566952483483889805568428664825400505121671890223584721114411827697594512296341594773601537519080793749568472726239590091904052247534142381322885597999201255135897334420626115847929127670295300664641023915921412100348810694368790829410468324587518896440574662249893060377916803644769267529820246473511018540449718324002956264174312180181563530270721168751856698020681743829723342459520727632
```

The numbers look big and scary, but the third line of `code.py` is the smoking gun: `d = getPrime(256)`. A 256-bit private exponent paired with a ~2096-bit modulus. The hint calls it Boneh-Durfee territory, and it is, but Wiener's much simpler continued-fraction attack also handles this comfortably.

---

## Background Knowledge (Read This First!)

If you have never touched RSA or continued fractions before, read this section — it is the only theory you need for this challenge.

### 1. RSA in 30 Seconds

RSA is built on the fact that multiplying two huge primes is cheap, but factoring the product back is hard. The standard parameters are:

| Symbol | Meaning | Size in this challenge |
|---|---|---|
| `p`, `q` | Two large random primes | 1048 bits each |
| `n` | Public modulus, `p * q` | ~2096 bits |
| `phi` | Euler's totient, `(p-1)(q-1)` | ~2096 bits |
| `e` | Public exponent (encrypts) | ~2096 bits here |
| `d` | Private exponent (decrypts) | **256 bits** here |

Encryption is `c = m^e mod n`. Decryption is `m = c^d mod n`. The math only works because `e * d ≡ 1 (mod phi)`. RSA is secure when `n` is too large to factor AND `d` is too large to recover from `e`.

### 2. Why This Challenge is Doomed

The standard advice is to keep `d` roughly the same size as `n` (so `d ≈ n`). Here `d` is a 256-bit prime while `n` is ~2096 bits. The private exponent is wildly undersized.

There is a famous 1990 result by Wiener showing that whenever `d < n^0.25 / 3`, RSA can be broken in polynomial time. A 1998 improvement by Boneh and Durfee pushed the bound to `d < n^0.292`. In our case `d ≈ 2^256` and `n^0.25 ≈ 2^524`, so `d` is about `2^268` times smaller than Wiener's bound. Both attacks are overkill here, and Wiener is much easier to code.

### 3. Wiener's Attack — The Intuition

From `e * d ≡ 1 (mod phi)`:

```
e * d = 1 + k * phi     for some positive integer k
```

Since `phi ≈ n`, the fraction `e / n` is very close to `k / d`. That means `k / d` is a **convergent** of the continued fraction expansion of `e / n`. Convergents are exactly the "best rational approximations," and for any `e, n` they can be enumerated in `O(log n)` steps. As soon as we try a convergent `k / d` that satisfies the modular arithmetic, we have the private key.

For each candidate `(k, d)` from the convergents, we:

1. Check that `(e * d - 1)` is divisible by `k` (otherwise the modular equation fails).
2. Compute `phi = (e * d - 1) / k`.
3. Compute `s = p + q = n - phi + 1` (from `phi = n - (p+q) + 1`).
4. Check that `s^2 - 4n` is a non-negative perfect square (otherwise `p, q` aren't integers).
5. Recover `p, q` from `s` and the square root, then verify `p * q == n`.

### 4. The Boneh-Durfee Connection (Why the Hint Mentions It)

Wiener is a one-dimensional lattice problem: a 2×2 lattice whose short vector encodes the secret `(k, d)`. Boneh-Durfee generalises this to a **bivariate** polynomial `f(x, y) = x * (A + y) - 1 ≡ 0 (mod e)` where `A = (n+1)`, and looks for the small root `(k, -(p+q))`. The 2D LLL attack (Coppersmith-style) is what David Wong's `boneh_durfee.sage` script implements, and it is the canonical "go-to" tool when `d` is just a bit too big for Wiener. Here Wiener is more than enough, but Boneh-Durfee is the right hammer for the harder variants of this family.

### 5. The Vulnerability in One Sentence

`d` is so small that the secret fraction `k/d` shows up as a convergent of the public fraction `e/n`. Recovering `d` from a continued-fraction expansion is a classical polynomial-time algorithm.

---

## Solution — Step by Step

I worked out of `~/ctf/small-trouble` on my Kali VM.

### Step 1 — Set Up the Working Directory

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/small-trouble && cd ~/ctf/small-trouble

┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ cp /media/sf_downloads/code.py . && cp /media/sf_downloads/message.txt .
```

I always start by reading the source even when the flag is "obviously" recoverable — the code is the truth, the description is marketing.

### Step 2 — Confirm the Vulnerability

```
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ grep -n "d =" code.py

d = getPrime(256)
```

Yep, `d` is a 256-bit prime. Quick sanity check on the Wiener's bound:

```
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ python3 -c "
n = 3980993015101017140353277804369745949706344223334890812840413257071044175973139291856038905399468319530362030870405443017903655374929929825011733964330181337551061274726015170805648591339935699566483704533804413902772334194671185814032757795167276567492610548489766891166023546451221494011601527912413323143640008533569546329046772124896140545774468831882189458647605131344071766512469603284171712833770168967153998797122819825938323489525196597407482515674210085730485950171311356595462370875129472210846138025654237220573246971086221110031195278019873474587710322597773432501095235059971088078096926751790014797097704337356321889
print('n bits :', n.bit_length())
print('n^0.25 : 2^', n.bit_length() // 4)
"
n bits : 2096
n^0.25 : 2^ 524
```

Wiener's bound is `2^524 / 3 ≈ 2^522`. We have a `d` of `2^256` bits. That is a factor-of-`2^266` safety margin in our favour. Wiener will hit in a few convergents.

### Step 3 — Install `pycryptodome` (if Needed)

```
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ pip install pycryptodome
```

### Step 4 — Write the Wiener Solver

Pure Python. No LLL library needed for the 1D case.

```
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ nano solve_wiener.py
```

Paste the following inside `nano`:

```python
#!/usr/bin/env python3
"""
Small Trouble - picoCTF solver (Wiener's continued-fraction attack).
d is a 256-bit prime, n is ~2096 bits. Wiener's bound is d < n^0.25 / 3,
so enumerating the convergents of e/n recovers d in milliseconds.
"""

from math import isqrt
from Crypto.Util.number import long_to_bytes


# ---- challenge parameters --------------------------------------------------
n = 3980993015101017140353277804369745949706344223334890812840413257071044175973139291856038905399468319530362030870405443017903655374929929825011733964330181337551061274726015170805648591339935699566483704533804413902772334194671185814032757795167276567492610548489766891166023546451221494011601527912413323143640008533569546329046772124896140545774468831882189458647605131344071766512469603284171712833770168967153998797122819825938323489525196597407482515674210085730485950171311356595462370875129472210846138025654237220573246971086221110031195278019873474587710322597773432501095235059971088078096926751790014797097704337356321889

e = 3336497038614663222541921459680551835187913731404732354361770350895041393777650313575506088859422849385970294213215931070595265411244105028409623278013975724125500479919877564207207562572189315593170541926598007670763000263224010236185517132229343161667668429997117301579363090706170019283552359847390040311521617091884426220652237247589452213130475528794167610900761281292390567153221748933849959961895636324991675428047481887984343193310278137379109243220192216924968789976201537337617841164156347418356710488273730184036786730194843318185762453423678066812440979743181541976510807538521163968859836855958692325875567630213728207

c = 1826031079095822845536526213992736069835634421976077836267143793181964583083024760950942723002771287666852728279522910274806690415793596687180742873080830698626932982997385432329275546871401690500831953978994557961462780510004840010773264058566952483483889805568428664825400505121671890223584721114411827697594512296341594773601537519080793749568472726239590091904052247534142381322885597999201255135897334420626115847929127670295300664641023915921412100348810694368790829410468324587518896440574662249893060377916803644769267529820246473511018540449718324002956264174312180181563530270721168751856698020681743829723342459520727632


# ---- continued-fraction utilities -----------------------------------------
def cf_expansion(num, den):
    while den:
        q = num // den
        yield q
        num, den = den, num - q * den


def convergents(cf):
    h_prev, k_prev = 0, 1
    h_curr, k_curr = 1, 0
    for a in cf:
        h_next = a * h_curr + h_prev
        k_next = a * k_curr + k_prev
        yield h_next, k_next
        h_prev, k_prev = h_curr, k_curr
        h_curr, k_curr = h_next, k_next


# ---- Wiener's attack -------------------------------------------------------
def wiener(e, n):
    for k, d in convergents(cf_expansion(e, n)):
        if k == 0:
            continue
        if (e * d - 1) % k != 0:
            continue
        phi = (e * d - 1) // k
        s = n - phi + 1
        disc = s * s - 4 * n
        if disc < 0:
            continue
        sq = isqrt(disc)
        if sq * sq != disc:
            continue
        p = (s + sq) // 2
        q = (s - sq) // 2
        if p * q == n:
            return d, p, q
    return None


if __name__ == "__main__":
    print("[*] Running Wiener's attack ...")
    d, p, q = wiener(e, n)
    print(f"[+] d     = {d}")
    print(f"[+] d bit = {d.bit_length()}")
    print(f"[+] p bit = {p.bit_length()}")
    print(f"[+] q bit = {q.bit_length()}")
    m = pow(c, d, n)
    print(f"[+] flag  = {long_to_bytes(m).decode()}")
```

Save and exit: `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 5 — Run the Solver

```
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ time python3 solve_wiener.py

[*] Running Wiener's attack ...
[+] d     = 70665447137626744270376541365651056764139589257356990807994269489091258603567
[+] d bit = 256
[+] p bit = 1048
[+] q bit = 1048
[+] flag  = picoCTF{sm4ll_d_3d2584a9}

real    0m0.026s
user    0m0.014s
sys     0m0.005s
```

Flag recovered in 26 milliseconds. The `d` we recovered is exactly 256 bits long, matching `getPrime(256)`. Both primes factor out at 1048 bits, exactly as the source claims.

### Step 6 — Verify by Hand

```
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ python3 -c "
n = 3980993015101017140353277804369745949706344223334890812840413257071044175973139291856038905399468319530362030870405443017903655374929929825011733964330181337551061274726015170805648591339935699566483704533804413902772334194671185814032757795167276567492610548489766891166023546451221494011601527912413323143640008533569546329046772124896140545774468831882189458647605131344071766512469603284171712833770168967153998797122819825938323489525196597407482515674210085730485950171311356595462370875129472210846138025654237220573246971086221110031195278019873474587710322597773432501095235059971088078096926751790014797097704337356321889
e = 3336497038614663222541921459680551835187913731404732354361770350895041393777650313575506088859422849385970294213215931070595265411244105028409623278013975724125500479919877564207207562572189315593170541926598007670763000263224010236185517132229343161667668429997117301579363090706170019283552359847390040311521617091884426220652237247589452213130475528794167610900761281292390567153221748933849959961895636324991675428047481887984343193310278137379109243220192216924968789976201537337617841164156347418356710488273730184036786730194843318185762453423678066812440979743181541976510807538521163968859836855958692325875567630213728207
c = 1826031079095822845536526213992736069835634421976077836267143793181964583083024760950942723002771287666852728279522910274806690415793596687180742873080830698626932982997385432329275546871401690500831953978994557961462780510004840010773264058566952483483889805568428664825400505121671890223584721114411827697594512296341594773601537519080793749568472726239590091904052247534142381322885597999201255135897334420626115847929127670295300664641023915921412100348810694368790829410468324587518896440574662249893060377916803644769267529820246473511018540449718324002956264174312180181563530270721168751856698020681743829723342459520727632
d = 70665447137626744270376541365651056764139589257356990807994269489091258603567
from Crypto.Util.number import long_to_bytes
m = pow(c, d, n)
print(long_to_bytes(m).decode())
"
picoCTF{sm4ll_d_3d2584a9}
```

Decryption is deterministic and matches. Done.

---

## What Happened Internally (Timeline)

| Step | What the program did | What I did |
|------|----------------------|------------|
| 1 | Author called `getPrime(1048)` twice to get `p` and `q`, then set `n = p*q` (~2096 bits). | — |
| 2 | Author called `getPrime(256)` for `d`, then `e = inverse(d, phi)`. Keys were now public `(n, e)` and private `d`. | Read `code.py`, spotted `d = getPrime(256)` — Wiener's attack is in play. |
| 3 | Plaintext `m` was encrypted: `c = m^e mod n`. The triple `(n, e, c)` was written to `message.txt`. | Inspected `message.txt`, confirmed `n` and `e` are huge but `d` is small. |
| 4 | — | Generated the continued fraction of `e/n`. The convergents are candidate `k/d` pairs. |
| 5 | — | For each convergent, computed `phi = (e*d - 1)/k`, then `s = n - phi + 1`, then checked whether `s^2 - 4n` is a perfect square. |
| 6 | — | Convergent #2 hit: `d = 70665…567`, `p` and `q` factor out, `c^d mod n` decrypts to `picoCTF{sm4ll_d_3d2584a9}`. |

The whole attack is "guess the private exponent by trying convergents of a public fraction" — polynomial time, no factoring, no LLL, no Sage.

---

## Alternative Method 1 — Boneh-Durfee in Sage (What the Hint Actually Says)

When `d` is too big for Wiener but still smaller than `n^0.292`, the right tool is the **Boneh-Durfee attack**. It is a bivariate lattice (Coppersmith-style) reduction that recovers the small root of `f(x, y) = x * (A + y) - 1 ≡ 0 (mod e)` where `A = (n+1)`. The canonical implementation is David Wong's `boneh_durfee.sage` from his `crypto-attacks` repo:

```
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ git clone https://github.com/jvdsn/crypto-attacks.git
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ cd crypto-attacks && sage attacks/rsa/boneh_durfee.sage
```

Drop the `n`, `e` from `message.txt` into the script's parameters, run `sage`, and it will grind through the lattice reduction until the flag appears. It is the canonical "go-to" tool for the harder variant of this challenge (where `d ≈ n^0.29`), and it is the reason the hint points at Boneh-Durfee even though Wiener already wins here.

---

## Alternative Method 2 — RsaCtfTool One-Liner

If you do not want to write any code at all, `RsaCtfTool` automates the entire family of small-`d` attacks behind one flag:

```
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ git clone https://github.com/RsaCtfTool/RsaCtfTool.git
┌──(zham㉿kali)-[~/ctf/small-trouble]
└─$ cd RsaCtfTool && pip install -r requirements.txt
┌──(zham㉿kali)-[~/ctf/RsaCtfTool]
└─$ python3 RsaCtfTool.py \
    --n 3980993015101017140353277804369745949706344223334890812840413257071044175973139291856038905399468319530362030870405443017903655374929929825011733964330181337551061274726015170805648591339935699566483704533804413902772334194671185814032757795167276567492610548489766891166023546451221494011601527912413323143640008533569546329046772124896140545774468831882189458647605131344071766512469603284171712833770168967153998797122819825938323489525196597407482515674210085730485950171311356595462370875129472210846138025654237220573246971086221110031195278019873474587710322597773432501095235059971088078096926751790014797097704337356321889 \
    --e 3336497038614663222541921459680551835187913731404732354361770350895041393777650313575506088859422849385970294213215931070595265411244105028409623278013975724125500479919877564207207562572189315593170541926598007670763000263224010236185517132229343161667668429997117301579363090706170019283552359847390040311521617091884426220652237247589452213130475528794167610900761281292390567153221748933849959961895636324991675428047481887984343193310278137379109243220192216924968789976201537337617841164156347418356710488273730184036786730194843318185762453423678066812440979743181541976510807538521163968859836855958692325875567630213728207 \
    --uncipherfile message.txt \
    --attack wiener
```

`RsaCtfTool` will try Wiener's continued-fraction attack, Boneh-Durfee, and several other small-`d` heuristics, and print the flag as soon as any of them succeed.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` / `grep` | Inspect `code.py` and confirm `d = getPrime(256)` |
| `nano` | Edit the Wiener solver script |
| Python 3 | Glued the continued-fraction math and modular arithmetic together |
| `pycryptodome` (`Crypto.Util.number`) | `long_to_bytes` to print the decrypted flag |
| `sage` + `boneh_durfee.sage` (alternative 1) | The full Boneh-Durfee attack for slightly larger `d` |
| `RsaCtfTool` (alternative 2) | One-shot automated RSA attack picker |

---

## Key Takeaways

- **In RSA, the size of `d` matters as much as the size of `n`.** A 256-bit `d` next to a 2096-bit `n` is an instant break. Never pick a tiny `d` for "performance" reasons.
- **Wiener's bound: `d < n^0.25 / 3`** is enough to break RSA in polynomial time. Memorise the bound and check it on every RSA challenge you see. If `d` is small enough, no factoring is required.
- **Continued fractions are the secret weapon for small `d`.** The convergents of `e/n` enumerate every plausible `(k, d)` pair in `O(log n)` steps. The math is ancient, the code is ten lines.
- **Wiener IS the 1D case of Boneh-Durfee.** Whenever you see "Boneh-Durfee" in a hint, try Wiener first — it is faster, simpler, and covers most "small `d`" challenges in practice. Reach for Boneh-Durfee when `d` is between `n^0.25` and `n^0.292`.
- **Verify your decrypted plaintext.** The whole attack rests on `phi = (e*d - 1) / k` being an integer AND `s^2 - 4n` being a perfect square. Both checks are cheap and let you reject false convergents instantly.
- **`RsaCtfTool` is the lazy-but-correct move** when you do not want to reimplement the math. It bundles Wiener, Boneh-Durfee, and a dozen other attacks behind a single CLI.

### Flag Wordplay Decode

The flag is `picoCTF{sm4ll_d_3d2584a9}`. Read the inner word:

- `sm4ll` → "small" with the classic leet `4 → a` swap. The punchline: the entire challenge boils down to a small `d`, and the flag tells you so out loud.
- `_d_` → literally "d" — the variable name from the RSA key generation.
- `3d2584a9` → a 32-bit hex nonce used to keep the flag unique across the picoCTF instance. It is the per-flag salt, nothing more.

So the readable part of the flag is **"small d"** — the vulnerability, stated plainly, exactly like the challenge title "Small Trouble" suggests.
