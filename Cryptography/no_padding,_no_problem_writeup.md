# No Padding, No Problem — picoCTF Writeup

**Challenge:** No Padding, No Problem  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 90  
**Flag:** `picoCTF{m4yb3_Th0se_m3s54g3s_4r3_difurrent_298df64b}`  
**Platform:** picoCTF (2021)  
**Writeup by:** zham  

---

## Description

> Oracles can be your best friend, they will decrypt anything, except the flag's ciphertext. How will you break it?
>
> Connect with `nc wily-courier.picoctf.net 53704`.

The oracle is a service you can talk to over the network. It gives you the RSA public key `(n, e)` and the flag's ciphertext, but it refuses to decrypt that exact ciphertext. It will, however, happily decrypt anything else you send it. So the question is: how do we get it to spill the flag without ever directly asking for it?

## Hints

> 1. What can you do with a different pair of ciphertext and plaintext? What if it is not so different after all...

This is the heart of the challenge. Hint 1 is asking: if you send a *different* ciphertext to the oracle and get a *different* plaintext back, can that different plaintext still help you recover the original? The answer is yes, because of a property of textbook RSA that we will explore below.

---

## Background Knowledge (Read This First!)

### 1. RSA, the Short Version

RSA is an asymmetric cipher. The public key is a pair `(n, e)` where `n` is a huge product of two primes and `e` is usually 65537. The private key `d` lets you reverse the operation.

- Encrypt a message `m`: `c = m^e mod n`
- Decrypt a ciphertext `c`: `m = c^d mod n`

For this challenge we only need the public key `(n, e)` and the ciphertext `c`. We do not know the private key `d`, and we do not need it.

### 2. The Multiplicative Homomorphism (This Is the Whole Trick)

Textbook RSA has a magical property:

```
E(m1) * E(m2)  ==  E(m1 * m2)   (mod n)
```

That is, if we multiply two ciphertexts together, the resulting ciphertext decrypts to the product of the two plaintexts. This is what makes textbook RSA *malleable* — given a ciphertext for `m`, you can construct a ciphertext for `m * r` for any `r` you like, without ever decrypting anything.

The math, in one line:

```
c1 = m1^e mod n
c2 = m2^e mod n
c1 * c2 mod n = (m1 * m2)^e mod n = E(m1 * m2)
```

We do not need `d` to build a ciphertext for `m * r`. We just pick an `r`, compute `r^e mod n`, and multiply the original ciphertext by it. The oracle cannot tell the new ciphertext is related to the one it is protecting.

### 3. Why "No Padding" Matters

Real-world RSA is always used with a padding scheme like OAEP or PKCS#1 v1.5. Padding is not just decoration — it specifically exists to destroy this multiplicative property. A padded message becomes a randomized, structured value, so multiplying two ciphertexts does not produce a ciphertext for a meaningful product. Without padding, textbook RSA is homomorphic, and homomorphic encryption under chosen-ciphertext attacks is broken.

That is what the title is telling you: "No Padding, No Problem" — the problem is that there is no padding.

### 4. The Attack (RSA Blinding)

The attack is three steps:

1. Pick any `r` you like (commonly `r = 2`).
2. Send the oracle `c' = c * r^e mod n`. The oracle happily decrypts this to `m' = m * r mod n`, since it does not realize `c'` is just a blinded version of the protected ciphertext.
3. Recover the original message by multiplying by the modular inverse of `r`: `m = m' * r^(-1) mod n`.
4. Convert the integer `m` to bytes — that is the flag.

We never asked the oracle for the protected ciphertext, so it never refused us. We asked for a *different* ciphertext that, by the homomorphism, decrypts to a *different* plaintext — but a plaintext that is mathematically equivalent (one is `r` times the other mod `n`).

---

## Solution — Step by Step

### Step 1 — Connect to the Oracle and Read the Public Parameters

```
┌──(zham㉿kali)-[~/picoctf/no-padding]
└─$ nc wily-courier.picoctf.net 53704
Welcome to the Padding Oracle Challenge
This oracle will take anything you give it and decrypt using RSA. It will not accept the ciphertext with the secret message... Good Luck!


n: 136238540102260303821270268531178140125577314705197939235419598313872468535711958698996314989518526939490892088736333005588072277729714284822616632691759044087069567246300175983027115756512653493646080776598711439529047415082888353741320305458445641371552343193685581917152812911662099572715217572204683711069
e: 65537
ciphertext: 112606110457865472161958224621633745961138735001253162292002126403763451349804876598117503869085336165514398806411549274887587381497411552405569458589225523626723541955836230739970091622232174519911242555236424500598232371479739734571594278782913524931991196490799284460853308021831072236635486408473163589070


Give me ciphertext to decrypt:
```

We get `n`, `e`, and the flag's ciphertext `c`.

### Step 2 — Write the Exploit

I wrote a small Python script that talks to the oracle over a raw socket (no `pwntools` needed), performs the blinding, and recovers the plaintext.

```
┌──(zham㉿kali)-[~/picoctf/no-padding]
└─$ nano solve.py
```

```python
#!/usr/bin/env python3
"""
RSA blinding attack on a textbook RSA decryption oracle.

The oracle refuses to decrypt the flag ciphertext c directly, but will decrypt
any other ciphertext. Textbook RSA is multiplicatively homomorphic:

    E(m1) * E(m2)  ==  E(m1 * m2)   (mod n)

So we pick an arbitrary r, send c' = c * r^e mod n. The oracle decrypts that
to m' = m * r mod n. We then recover m = m' * r^-1 mod n and read the flag.
"""
import socket
import re


HOST = "wily-courier.picoctf.net"
PORT = 53704


def recv_until(sock, marker, timeout=5):
    sock.settimeout(timeout)
    buf = b""
    while marker not in buf:
        chunk = sock.recv(4096)
        if not chunk:
            break
        buf += chunk
    return buf.decode(errors="replace")


def main():
    s = socket.socket()
    s.connect((HOST, PORT))

    banner = recv_until(s, b"Give me ciphertext to decrypt:")
    print(banner)

    # Parse n, e, and the flag ciphertext.
    n = int(re.search(r"n:\s*(\d+)", banner).group(1))
    e = int(re.search(r"e:\s*(\d+)", banner).group(1))
    c = int(re.search(r"ciphertext:\s*(\d+)", banner).group(1))
    print(f"\n[+] n = {n}")
    print(f"[+] e = {e}")
    print(f"[+] c = {c}")

    # Blinding factor (any r coprime with n works; 2 is fine).
    r = 2
    blinded = (c * pow(r, e, n)) % n
    print(f"[+] r = {r}")
    print(f"[+] c' = c * r^e mod n = {blinded}")

    # Send the blinded ciphertext to the oracle.
    s.sendall(str(blinded).encode() + b"\n")
    response = recv_until(s, b"Give me ciphertext to decrypt:", timeout=10)
    print("\n[oracle response]")
    print(response)

    m_blinded = int(re.search(r"Here you go:\s*(\d+)", response).group(1))
    print(f"[+] m' (oracle returned) = {m_blinded}")

    # Unblind: m = m' * r^-1 mod n
    r_inv = pow(r, -1, n)
    m = (m_blinded * r_inv) % n
    print(f"[+] m = m' * r^-1 mod n = {m}")

    # Convert the integer plaintext to bytes (the flag).
    flag_bytes = m.to_bytes((m.bit_length() + 7) // 8, "big")
    flag = flag_bytes.decode(errors="replace")
    print(f"\n[+] Recovered plaintext bytes: {flag_bytes!r}")
    print(f"[+] Flag: {flag}")

    s.close()
    return flag


if __name__ == "__main__":
    main()
```

Save and exit:

```
^O   (WriteOut)
File Name to Write: solve.py -> press Enter
^X   (Exit)
```

### Step 3 — Run the Exploit

```
┌──(zham㉿kali)-[~/picoctf/no-padding]
└─$ python3 solve.py
Welcome to the Padding Oracle Challenge
This oracle will take anything you give it and decrypt using RSA. It will not accept the ciphertext with the secret message... Good Luck!


n: 136238540102260303821270268531178140125577314705197939235419598313872468535711958698996314989518526939490892088736333005588072277729714284822616632691759044087069567246300175983027115756512653493646080776598711439529047415082888353741320305458445641371552343193685581917152812911662099572715217572204683711069
e: 65537
ciphertext: 112606110457865472161958224621633745961138735001253162292002126403763451349804876598117503869085336165514398806411549274887587381497411552405569458589225523626723541955836230739970091622232174519911242555236424500598232371479739734571594278782913524931991196490799284460853308021831072236635486408473163589070


Give me ciphertext to decrypt: 

[+] n = 136238540102260303821270268531178140125577314705197939235419598313872468535711958698996314989518526939490892088736333005588072277729714284822616632691759044087069567246300175983027115756512653493646080776598711439529047415082888353741320305458445641371552343193685581917152812911662099572715217572204683711069
[+] e = 65537
[+] c = 112606110457865472161958224621633745961138735001253162292002126403763451349804876598117503869085336165514398806411549274887587381497411552405569458589225523626723541955836230739970091622232174519911242555236424500598232371479739734571594278782913524931991196490799284460853308021831072236635486408473163589070
[+] r = 2
[+] c' = c * r^e mod n = 89205808983735687486720357657946352434496455980808958632860608558725633930921662448345586392627040482543534389040138493980984523449763330954289442350169398365156024800201415830210890048725939882994742621377218633691783915575978992401063132855180319465311491538285228947831918665654925607129437636464080421665

[oracle response]
Here you go: 148620815460275220210409788604137413155822463411854375299452784868712097032909495944869257747184256740036863621988704596837626
Give me ciphertext to decrypt: 
[+] m' (oracle returned) = 148620815460275220210409788604137413155822463411854375299452784868712097032909495944869257747184256740036863621988704596837626
[+] m = m' * r^-1 mod n = 74310407730137610105204894302068706577911231705927187649726392434356048516454747972434628873592128370018431810994352298418813

[+] Recovered plaintext bytes: b'picoCTF{m4yb3_Th0se_m3s54g3s_4r3_difurrent_298df64b}'
[+] Flag: picoCTF{m4yb3_Th0se_m3s54g3s_4r3_difurrent_298df64b}
```

The oracle happily decrypted our blinded ciphertext. We divided by `r = 2` (mod n) and got the original plaintext.

### Step 4 — Submit

```
picoCTF{m4yb3_Th0se_m3s54g3s_4r3_difurrent_298df64b}
```

Submitted, accepted.

---

## What Happened Internally (Timeline)

1. **Connected to the oracle.** It gave us the public key `(n, e)` and the protected ciphertext `c`.
2. **Picked a blinding factor** `r = 2` (any small number coprime with `n` works).
3. **Built a blinded ciphertext** `c' = c * r^e mod n`. Mathematically, this is `E(m * r)`.
4. **Sent `c'` to the oracle.** It checked that `c' != c` (so the guard passed) and decrypted it to `m' = m * r mod n`.
5. **Recovered `m`** by multiplying `m'` by `r^(-1) mod n`, the modular inverse of `r`.
6. **Converted the integer `m` to bytes** to read the flag string.

---

## Alternative Method — One-Liner with `printf | nc`

If you do not want a Python script, you can do the math in Python and pipe the blinded ciphertext to `nc`. This is the quick-and-dirty version using the `subprocess` module to drive `nc`:

```
┌──(zham㉿kali)-[~/picoctf/no-padding]
└─$ python3 -c "
import socket, re
s = socket.socket(); s.connect(('wily-courier.picoctf.net', 53704))
s.settimeout(5); buf = b''
while b'Give me ciphertext' not in buf: buf += s.recv(4096)
b = buf.decode()
n = int(re.search(r'n: (\d+)', b).group(1))
e = int(re.search(r'e: (\d+)', b).group(1))
c = int(re.search(r'ciphertext: (\d+)', b).group(1))
r = 2
s.sendall(str(c * pow(r, e, n) % n).encode() + b'\n')
buf = b''
while b'Give me ciphertext' not in buf: buf += s.recv(4096)
mb = int(re.search(r'Here you go: (\d+)', buf.decode()).group(1))
m = mb * pow(r, -1, n) % n
print(bytes.fromhex(hex(m)[2:]).decode())
"
picoCTF{m4yb3_Th0se_m3s54g3s_4r3_difurrent_298df64b}
```

If your Kali has `pwntools` installed, the equivalent one-liner with `pwntools` is even shorter — but the raw `socket` approach above works on a fresh install with no extra dependencies.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` / raw socket | Talk to the oracle over TCP |
| Python 3 `pow()` | Fast modular exponentiation for `r^e mod n` and `r^(-1) mod n` |
| Python 3 `int.to_bytes()` | Convert the recovered integer back to the flag string |
| `regex` (`re`) | Parse `n`, `e`, `c`, and the oracle's reply |
| `nano` | Edit the solver script in the terminal |

---

## Key Takeaways

- The flag wordplay: **`m4yb3_Th0se_m3s54g3s_4r3_difurrent`** in leetspeak is "maybe those messages are different". The pun is that the *ciphertexts* we send to the oracle are different, but the *plaintexts* are not so different — they are related by a multiplicative factor of `r`. The challenge title **No Padding, No Problem** is the punchline: textbook RSA without padding is multiplicatively homomorphic, and homomorphic + chosen-ciphertext oracle = broken.
- Textbook RSA is *malleable*: given `E(m)`, anyone can build `E(m * r)` for any `r` they choose. Padding schemes like OAEP exist specifically to kill this property by randomizing and structuring the plaintext before encryption.
- The general attack pattern is called a **blinding attack** or **chosen-ciphertext attack on unpadded RSA**. It is the textbook example of why you should never use raw RSA in production.
- The modular inverse `r^(-1) mod n` is computed with Python's built-in `pow(r, -1, n)` (Python 3.8+). Older Python needs the `sympy` library or a hand-rolled extended-Euclidean implementation.
- picoCTF regenerates this challenge every time you open it, so `n`, `c`, and the hex suffix in the flag change between sessions. The attack pattern stays the same; only the numbers change.

**Flag:** `picoCTF{m4yb3_Th0se_m3s54g3s_4r3_difurrent_298df64b}`
