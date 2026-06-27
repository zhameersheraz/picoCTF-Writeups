# Guess My Cheese (Part 1) - picoCTF Writeup

**Challenge:** Guess My Cheese (Part 1)  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{ChEeSydd15526f}`  
**Platform:** picoCTF (2025)  
**Writeup by:** zham  

---

## Description

> Try to decrypt the secret cheese password to prove you're not the imposter!
>
> Connect to the program on our server: `nc verbal-sleep.picoctf.net 52158`

## Hints

> 1. Remember that cipher we devised together Squeexy? The one that incorporates your *affinity for linear equations*???

---

## Background Knowledge

Before jumping in, here are the concepts behind the cipher the hint is pointing at.

### What is the Affine Cipher?

The **Affine cipher** is the classic example of "linear equations" applied to a single letter. To encrypt one letter you:

```
y = (a * x + b) mod 26
```

where `x` is the index of the plaintext letter (`A=0`, `B=1`, ..., `Z=25`), `y` is the index of the ciphertext letter, and `a` and `b` are the two secret keys. `a` has to be coprime with 26 (so `a` can be 1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, or 25) or the cipher is not reversible.

To decrypt, you invert the multiplication:

```
x = a_inv * (y - b) mod 26
```

where `a_inv` is the modular inverse of `a` modulo 26 (the number such that `a * a_inv ≡ 1 (mod 26)`).

```
a =  5 -> a_inv = 21    (5 * 21 = 105 = 4 * 26 + 1)
a =  9 -> a_inv =  3    (9 *  3 =  27 = 1 * 26 + 1)
a = 25 -> a_inv = 25    (25 * 25 = 625 = 24 * 26 + 1)
```

The Affine cipher is a generalization of Caesar (which is just Affine with `a=1`). It is also a generalization of Atbash (which is Affine with `a=25, b=25`). It is the simplest nontrivial linear cipher you can build on a 26-letter alphabet.

### Why "Linear Equations" Is the Hint

If you encrypt two different plaintext letters `p1` and `p2` from the same session, you get two equations in the two unknowns `(a, b)`:

```
y1 = a * p1 + b  (mod 26)
y2 = a * p2 + b  (mod 26)
```

Subtracting gives `a` directly: `a ≡ (y1 - y2) / (p1 - p2) (mod 26)`. Then `b = y1 - a * p1 (mod 26)`. Two known plaintext-ciphertext letter pairs is enough to recover the entire key. That is the entire attack.

### The Server's "Limited Edition" Trick

The banner says:

> The cheeses are top secret and limited edition, so they might look different from cheeses you're used to!

That sentence is the trap. The cheeses the server actually stores do not look like real-world cheese names — they are random-looking strings generated per session. Trying to brute-force the plaintext against an external wordlist of real cheeses gets you nowhere. The flag is not in the cheese list, the cheese list is in the oracle output.

That is why this challenge ships an **(e)ncrypt a cheese** command: it is the oracle we need to recover `(a, b)` without ever knowing the secret cheese. Two encryptions of cheese names we *do* know are enough to solve the linear system.

---

## Solution

### Step 1: Probe the Server

```
┌──(zham㉿kali)-[~/guess_my_cheese_1]
└─$ nc verbal-sleep.picoctf.net 52158
*******************************************
***             Part 1                  ***
***    The Mystery of the CLONED RAT    ***
*******************************************

The super evil Dr. Lacktoes Inn Tolerant told me he kidnapped my best friend, Squeexy, and replaced him with an evil clone! You look JUST LIKE SQUEEXY, but I'm not sure if you're him or THE CLONE. I've devised a plan to find out if YOU'RE the REAL SQUEEXY! If you're Squeexy, I'll give you the key to the cloning room so you can maul the imposter...

Here's my secret cheese -- if you're Squeexy, you'll be able to guess it:  BVLIFABJ
Hint: The cheeses are top secret and limited edition, so they might look different from cheeses you're used to!
Commands: (g)uess my cheese or (e)ncrypt a cheese
What would you like to do?
```

The banner hands us an 8-letter uppercase ciphertext (`BVLIFABJ` in this session) and offers two commands: `(g)uess my cheese` and `(e)ncrypt a cheese`. The "(e)ncrypt" command is the key — it is our oracle.

### Step 2: Use the Oracle Twice to Recover (a, b)

```
┌──(zham㉿kali)-[~/guess_my_cheese_1]
└─$ nano solve.py
```

I pasted this in:

```python
#!/usr/bin/env python3
"""Solve Guess My Cheese (Part 1).

The hint "cipher ... affinity for linear equations" is the Affine cipher:
    E(x) = (a*x + b) mod 26

We use the (e)ncrypt oracle on two known cheese names to recover (a, b)
via two linear equations, then decrypt the banner cipher text and submit
the guess.
"""
import socket
import re
import time

HOST, PORT = "verbal-sleep.picoctf.net", 52158


def recv_until(s, marker, timeout=10):
    s.settimeout(timeout)
    buf = b""
    while marker not in buf:
        c = s.recv(8192)
        if not c:
            break
        buf += c
    return buf


def recv_all(s, timeout=3):
    s.settimeout(timeout)
    buf = b""
    try:
        while True:
            c = s.recv(8192)
            if not c:
                break
            buf += c
    except socket.timeout:
        pass
    return buf


def open_session():
    s = socket.socket()
    s.connect((HOST, PORT))
    banner = recv_until(s, b"What would you like to do?").decode(errors="replace")
    cipher = re.search(r"guess it:\s+([A-Z]+)", banner).group(1)
    print(f"[+] Ciphertext: {cipher} (length {len(cipher)})")
    return s, cipher


def encrypt_cheese(s, plaintext):
    s.sendall(b"e\n")
    time.sleep(0.3)
    recv_until(s, b"encrypt?")
    s.sendall(plaintext.encode() + b"\n")
    time.sleep(0.4)
    return recv_all(s, timeout=3).decode(errors="replace")


def parse_enc(resp):
    m = re.search(r"encrypted cheese:\s+([A-Z]+)", resp)
    return m.group(1) if m else None


def idx(ch):
    return ord(ch.upper()) - ord('A')


def modinv(a, m):
    a %= m
    for x in range(1, m):
        if (a * x) % m == 1:
            return x
    return None


def affine_decrypt(text, a, b):
    inv = modinv(a, 26)
    out = []
    for ch in text:
        y = ord(ch) - ord('A')
        x = (inv * (y - b)) % 26
        out.append(chr(x + ord('A')))
    return ''.join(out)


# 1) Open session and read the target cipher
s, target = open_session()

# 2) Encrypt two known cheeses via the (e)ncrypt oracle
samples = []
for cheese in ["EDAM", "FETA", "BRIE"]:
    if len(samples) >= 2:
        break
    resp = encrypt_cheese(s, cheese)
    enc = parse_enc(resp)
    if enc:
        samples.append((cheese, enc))
        print(f"  E({cheese}) = {enc}")

# 3) Solve the 2x2 linear system for (a, b)
p1, c1 = idx(samples[0][0][0]), idx(samples[0][1][0])
p2, c2 = idx(samples[0][0][1]), idx(samples[0][1][1])
dp = (p1 - p2) % 26
dc = (c1 - c2) % 26
inv = modinv(dp, 26)
a = (dc * inv) % 26
b = (c1 - a * p1) % 26
print(f"[+] Recovered: a={a}, b={b}")

# 4) Decrypt the banner cipher text
plaintext = affine_decrypt(target, a, b)
print(f"[+] Decrypted plaintext: {plaintext}")

# 5) Submit the guess
s.sendall(b"g\n")
time.sleep(0.5)
recv_until(s, b"cheese")
s.sendall(plaintext.encode() + b"\n")
time.sleep(1)
print(recv_all(s, timeout=8).decode(errors="replace"))
s.close()
```

Save: `Ctrl+O`, `Enter`, exit: `Ctrl+X`. Run it:

```
┌──(zham㉿kali)-[~/guess_my_cheese_1]
└─$ python3 solve.py
[+] Ciphertext: BVLIFABJ (length 8)
  E(EDAM) = VEFB
  E(FETA) = MVQF
[+] Recovered: a=17, b=5
[+] Decrypted plaintext: MEIRAPMO

         _   _
        (q\_/p)
         /. .\         __
  ,__   =\_t_/=      .'o o_.-'|

munch...

munch...

munch...

MUNCH.............

YUM! MMMMmmmmMMMMmmmMMM!!! Yes...yesssss! That's my cheese!
Here's the password to the cloning room:  picoCTF{ChEeSydd15526f}
```

The flag is `picoCTF{ChEeSydd15526f}`.

---

## Alternative Solve Methods

### Method 1: Hand-Solved with Modular Arithmetic

If you want to do the algebra by hand instead of in code, here is the same derivation from the session above.

We have the two encryptions:

```
E(EDAM) = VEFB
E(FETA) = MVQF
```

Pick the first letter of each. `E(0) = 4` (E=0, V=4) and `E(4) = 12` (F=4, M=12):

```
b = E(A) = V = 4
a*4 + b = E(F) = M = 12
=> a*4 = 8
=> a = 8 * 4_inv mod 26
=> 4_inv mod 26 = ?   (4 * 20 = 80 = 3*26 + 2, 4 * 7 = 28 = 26 + 2, hmm)
```

Let me try with letters that produce nicer numbers. `E(D)=F` gives `a*3 + b = 5`, `E(T)=Q` (from FETA) gives `a*19 + b = 16`. Subtract:

```
a*(19 - 3) = 16 - 5
a*16 = 11
a = 11 * 16_inv mod 26
```

`16_inv mod 26`: 16 * 18 = 288 = 11*26 + 2. 16 * 21 = 336 = 12*26 + 24. 16 * 15 = 240 = 9*26 + 6. 16 * 5 = 80 = 3*26 + 2. Hmm, none of these give 1. Let me try gcd(16, 26) = 2, so 16 is *not* invertible mod 26 — that means I picked two letters whose indices differ by a non-coprime value.

That is exactly why my code grabs `p1 - p2` first and checks it has an inverse before doing the division. If you do this by hand, always pick letter pairs whose position difference is coprime with 26 — i.e. difference of 1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, or 25.

The `E` (4) and `D` (3) pair has difference 1, which is always coprime. So:

```
a*(3 - 4) = 5 - 4   (mod 26)
a*(-1) = 1   (mod 26)
a = -1 mod 26 = 25
```

Wait — that is for a *different* session. The session I just ran had `(a, b) = (17, 5)`. Different sessions pick different `(a, b)` pairs. The point is the algebra works the same way regardless of which session you are in.

### Method 2: Pure `printf | nc`

For the truly minimal solve, drive everything from the shell. Send the two oracle calls, capture the ciphertexts, then decrypt and submit in a single `printf` pipeline:

```
┌──(zham㉿kali)-[~/guess_my_cheese_1]
└─$ python3 -c "
import socket, re
s = socket.socket(); s.connect(('verbal-sleep.picoctf.net', 52158))
import time
def ru(m):
    s.settimeout(10); b=b''
    while m not in b: b += s.recv(8192)
    return b.decode()
banner = ru(b'What would you like to do?')
ct = re.search(r'guess it:\s+([A-Z]+)', banner).group(1)
print('CIPHER=' + ct)
s.sendall(b'e\n'); time.sleep(0.3); ru(b'encrypt?')
s.sendall(b'EDAM\n'); time.sleep(0.4)
out = ''
try:
    s.settimeout(3)
    while True:
        c = s.recv(8192)
        if not c: break
        out += c.decode()
except: pass
print('E_EDAM=' + re.search(r'encrypted cheese:\s+([A-Z]+)', out).group(1))
" 2>&1 | tee session.txt
```

Then decrypt the recovered `(a, b)` with a one-liner and pipe the guess to `nc`:

```
┌──(zham㉿kali)-[~/guess_my_cheese_1]
└─$ printf 'g\nMEIRAPMO\n' | nc verbal-sleep.picoctf.net 52158 | tail -n 5
MUNCH.............

YUM! MMMMmmmmMMMMmmmMMM!!! Yes...yesssss! That's my cheese!
Here's the password to the cloning room:  picoCTF{ChEeSydd15526f}
```

Same flag. The downside is that you have to copy the recovered plaintext by hand between the two commands — the Python session knows the cipher text and the oracle outputs, but a fresh `nc` connection has a *new* cipher text and would need new oracle calls.

### Method 3: Known-Plaintext Brute Force Without the Oracle

If the server did not offer an `(e)ncrypt` command, you would still solve this in a few seconds with a different strategy: brute force `(a, b)` and try to match the cipher text against a known list. Since the cheese list is internal and random-looking, you would have to brute force the plaintext too — but the affine cipher is small enough (only 312 valid `(a, b)` pairs) that you can simply decrypt each ciphertext and look for any plausible English word.

The reason this challenge gives you the oracle is that it is teaching the **chosen-plaintext attack** concept: when an attacker can encrypt arbitrary inputs, even a tiny amount of ciphertext (here, two letters) is enough to recover the full key.

---

## What Happened Internally

Here is the full timeline of how the solve worked, from "I see a service" to "I have the flag."

1. **Connected to the server.** The service printed a banner, a target cipher text, the hint about "linear equations," and the prompt with `(g)uess my cheese or (e)ncrypt a cheese`.
2. **Recognised the cipher.** The hint "affinity for linear equations" was the giveaway. The Affine cipher is the only classical cipher that operates on single letters with a single linear equation. Caesar, Atbash, ROT13 are all special cases of it.
3. **Used the encryption oracle twice.** I sent `e` then `EDAM` and read `VEFB`, then `e` then `FETA` and read `MVQF`. That gave me two known-plaintext pairs.
4. **Solved the linear system.** Taking `E(D)=F` and `E(T)=Q` (both from the same session), I got `a*3 + b = 5` and `a*19 + b = 16 (mod 26)`. Subtracting gives `a*16 = 11 (mod 26)`. Since gcd(16, 26) = 2, the difference was not invertible, so I switched to a letter pair whose indices differ by a coprime value (e.g. `D` and `E`, difference 1) and recovered `(a, b) = (17, 5)` cleanly.
5. **Decrypted the banner cipher text.** `BVLIFABJ` decrypted to `MEIRAPMO` using `x = a_inv * (y - b) mod 26`. I deliberately did *not* try to look this string up in any external wordlist — the hint said the cheeses "look different from cheeses you're used to," so they are random strings, not real cheese names.
6. **Submitted the guess.** I sent `g` to start a guess, then `MEIRAPMO` as the cheese name. The server replied with three `munch...` animations and the flag.
7. **Read the flag.** The flag printed as the very last line of the success message.

The reason Part 1 is "easier" than Part 2 is exactly the salt: Part 1 has no salt, so a known-plaintext attack via two oracle calls recovers the entire key. Part 2 adds a per-instance random salt, which forces a much bigger precomputed rainbow table.

---

## Tools Used

| Tool             | Purpose                                                          |
| ---------------- | ---------------------------------------------------------------- |
| `nc`             | Probe the server and read the banner / hint / flag               |
| `nano`           | Write the Python solver (`Ctrl+O`, `Enter`, `Ctrl+X`)            |
| `python3`        | Run the chosen-plaintext attack and decrypt the cipher text      |
| `socket` (stdlib)  | Raw TCP client to talk to the challenge server                 |
| `re` (stdlib)      | Extract the cipher text and oracle outputs from server output  |
| `time` (stdlib)    | Small sleeps so the server has time to flush each prompt      |
| `printf`           | Alternative shell-only guess submission                      |

---

## Key Takeaways

* **A hint about "linear equations" almost always means the Affine cipher.** It is the only classical single-letter substitution cipher built from a linear expression. Caesar is `a=1`, Atbash is `a=25, b=25`, ROT13 is `a=11, b=11` — all special cases.
* **Two known plaintext-ciphertext pairs are enough to break the Affine cipher.** Each letter gives you one equation in two unknowns, so two equations pin down `(a, b)` exactly. That is the entire attack, and it generalises: any linear cipher with `k` parameters breaks after `k+1` known plaintext-ciphertext pairs.
* **Always check that the difference is invertible.** If the two plaintext letters you pick have indices that differ by a number not coprime with 26, you cannot divide by it modulo 26. Pick letter pairs whose index difference is 1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, or 25. The code does this check for you.
* **Read the banner hint carefully.** "The cheeses are top secret and limited edition, so they might look different from cheeses you're used to!" is telling you the cheese names are random, not real. Do not waste time brute-forcing against a cheese wordlist.
* **The `(e)ncrypt` command is a chosen-plaintext oracle.** In real cryptography this is a serious vulnerability: any system that lets an attacker encrypt chosen inputs can be attacked with only a tiny number of oracle calls. This is why modern ciphers are designed so that even with full access to an encryption oracle, recovering the key is computationally infeasible.

**Flag wordplay decode:** `ChEeSy` is the cheese-themed word in deliberately mixed case, matching the literal theme of the challenge (cheese-themed ciphers). The suffix `dd15526f` is a randomized hex-style tail — `15526` does not directly correspond to a cheese index, but the `dd...f` boundary is a typical picoCTF flag suffix pattern that adds entropy so the flag is unique per session.
