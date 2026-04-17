# Shared Secrets — picoCTF Writeup

**Challenge:** Shared Secrets  
**Category:** Cryptography  
**Difficulty:** Easy  
**Flag:** `picoCTF{dh_s3cr3t_1bcf19a9}`  

---

## Description

> A message was encrypted using a shared secret... but it looks like one side of the exchange leaked something. Can you piece together the secret and get the flag?
> Download the **message** and source **code**.

**Hint 1:** `What do you get if you combine a public key with a known private one?`

**Attachments:** `message.txt`, `encryption.py`

---

## Background Knowledge (Read This First!)

### What is Diffie-Hellman Key Exchange?

**Diffie-Hellman (DH)** is a method that lets two parties establish a **shared secret** over a public channel — without ever directly sending the secret itself. Here's how it works:

1. Both parties agree on two public values: a generator `g` and a large prime `p`
2. The **server** picks a secret number `a`, computes `A = g^a mod p`, and sends `A` publicly
3. The **client** picks a secret number `b`, computes `B = g^b mod p`, and sends `B` publicly
4. The server computes `shared = B^a mod p`
5. The client computes `shared = A^b mod p`
6. Both arrive at the **same shared secret** — because `g^(a*b) mod p = g^(b*a) mod p`

The security of DH relies on the **discrete logarithm problem** — it's easy to compute `A = g^a mod p`, but computationally infeasible to reverse it and recover `a` from `A` alone.

### What went wrong here?

The source code shows the client's private key `b` written directly into `message.txt` — the same file that gets shared as the challenge attachment. That one leak breaks everything: anyone with `b` and the server's public key `A` can compute `shared = A^b mod p` themselves.

### What is XOR encryption?

The flag is encrypted with a simple XOR:
```python
enc = bytes([x ^ (shared % 256) for x in flag])
```
Only **one byte** of the shared secret is used as the key: `shared % 256` (the last byte). XOR is its own inverse — applying it again with the same key recovers the original.

### ⚠️ Two Important Notes

**Note 1 — No special libraries needed**
Python's built-in `pow(A, b, p)` handles modular exponentiation efficiently, even with 1000-bit numbers. No `pycryptodome` or anything extra required.

**Note 2 — Save as a .py file**
The numbers in this challenge are enormous (300+ digits). Typing them into a one-liner is error-prone — save a script file instead.

---

## Solution — Step by Step

### Step 1 — Read the source code

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings encryption.py
```

```python
from Crypto.Util.number import getPrime
from random import randint
# Public parameters
g = 2
p = getPrime(1048)
# Server's secret
a = randint(2, p-2)
A = pow(g, a, p)
# Client secret
b = '???'
B = pow(g, b, p)
# Shared key
shared = pow(A, b, p)
# Encrypt flag
flag = b"picoCTF{...}"
enc = bytes([x ^ (shared % 256) for x in flag])
# Write challenge info
with open("file.txt", "w") as f:
    f.write(f"g = {g}\n")
    f.write(f"p = {p}\n")
    f.write(f"A = {A}\n")
    f.write(f"b = {b} \n")
    f.write(f"enc = {enc.hex()}\n")
```

Notice that `b` — the **client's private key** — is written to the output file. That is the vulnerability.

### Step 2 — Read the message file

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings message.txt
g = 2
p = 200548979693513758965253670921542122741347490011547718415928880623135536115617609273814473922906178118197100483745066360055145517219160768349976194504131009423076298323344508653906037653621528705868308522403513247646424920776933876599283326344568815472913995125085641956939303716076049196728284427490861164089212867 9
A = 152657432412512996125973230367283020556700242189613701755857153552921567531613376444437865452970759483618898239447203454086466024297763602933904798172389537552456363010030065520028378546904934459571424445840603928297430232679035815326467242406853271535271785882882920855778355282090387018752542250796962152257612852 9
b = 509160177730197944921155430683057403251740842734170287832867942540738055861009944443738638587125553292842157324977185671585742050562430030882589659241602138008956713608431696199507406174765901469913371166897127354209889596259884458927157926821014407437019772163790129553017138635328607276432132415865673419672072309
enc = 2a333935190e1c213e320529693928692e056b38393c6b633b6327
```

All the values needed are right here — including `b`, the private key that was never supposed to be public.

### Step 3 — Write the solve script

Save this as `solve.py`:

```python
p = 2005489796935137589652536709215421227413474900115477184159288806231355361156176092738144739229061781181971004837450663600551455172191607683499761945041310094230762983233445086539060376536215287058683085224035132476464249207769338765992833263445688154729139951250856419569393037160760491967282844274908611640892128679
A = 1526574324125129961259732303672830205567002421896137017558571535529215675316133764444378654529707594836188982394472034540864660242977636029339047981723895375524563630100300655200283785469049344595714244458406039282974302326790358153264672424068532715352717858828829208557783552820903870187525422507969621522576128529
b = 509160177730197944921155430683057403251740842734170287832867942540738055861009944443738638587125553292842157324977185671585742050562430030882589659241602138008956713608431696199507406174765901469913371166897127354209889596259884458927157926821014407437019772163790129553017138635328607276432132415865673419672072309
enc_hex = '2a333935190e1c213e320529693928692e056b38393c6b633b6327'

shared = pow(A, b, p)           # Compute the shared secret: A^b mod p
key = shared % 256              # Only the last byte is used as the XOR key
enc_bytes = bytes.fromhex(enc_hex)
flag = bytes([x ^ key for x in enc_bytes])
print(flag.decode())
```

### Step 4 — Run it

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 solve.py
picoCTF{dh_s3cr3t_1bcf19a9}
```

✅ Got the flag! 🎯

---

## Alternative Methods

### Alternative 1 — One-liner in terminal

If you don't want a script file, paste the full one-liner (make sure to use the full numbers, not truncated ones):

```bash
python3 -c "
p = 20054897...679
A = 15265743...529
b = 50916017...309
enc_hex = '2a333935190e1c213e320529693928692e056b38393c6b633b6327'
shared = pow(A, b, p)
flag = bytes([x ^ (shared % 256) for x in bytes.fromhex(enc_hex)])
print(flag.decode())
"
```

> ⚠️ The `...` are placeholders — you must paste the **full numbers** from `message.txt` or Python will throw a `SyntaxError`.

### Alternative 2 — Interactive Python shell

For a step-by-step approach where you can inspect each value:

```bash
python3
>>> p = 200548979...
>>> A = 152657432...
>>> b = 509160177...
>>> shared = pow(A, b, p)
>>> key = shared % 256
>>> print(key)
90
>>> enc = bytes.fromhex('2a333935190e1c213e320529693928692e056b38393c6b633b6327')
>>> print(bytes([x ^ key for x in enc]).decode())
picoCTF{dh_s3cr3t_1bcf19a9}
```

This is useful for learning — you can print `shared`, `key`, and intermediate values to understand what each step produces.

### Alternative 3 — CyberChef (partial)

CyberChef can handle the XOR decryption step once you know the key byte:

1. Input: `2a333935190e1c213e320529693928692e056b38393c6b633b6327`
2. Recipe: **From Hex** → **XOR** (key = `5a` in hex, which is decimal 90)
3. Output: `picoCTF{dh_s3cr3t_1bcf19a9}`

You still need Python (or a big-number calculator) to compute `shared = pow(A, b, p)` and derive the key byte first.

---

## Why Did This Work?

The challenge title says it all — **Shared** Secrets. The whole point of Diffie-Hellman is that neither party ever reveals their private key. The moment `b` was written to an output file and handed to the attacker, the exchange was fully broken.

The flag name confirms the lesson: `dh_s3cr3t` → "DH secret." A secret that isn't secret is just a number.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `strings` | Read plaintext values from message.txt | ⭐ Easy |
| Python `pow(A, b, p)` | Modular exponentiation for shared secret | ⭐⭐ Medium |
| Python XOR loop | Decrypt flag bytes | ⭐ Easy |
| CyberChef (optional) | XOR decryption in browser GUI | ⭐ Easy |

---

## Key Takeaways

- **Never write private keys to output files** — once `b` was in `message.txt`, the entire DH exchange was worthless
- **Diffie-Hellman is only secure if both private keys stay private** — leaking either one immediately exposes the shared secret
- **Weak key derivation** — using only `shared % 256` as the XOR key means only 256 possible keys even if you didn't have `b`; a brute-force loop over all 256 values would also crack this
- **XOR with a single byte is not encryption** — it is trivially reversible and should never be used alone for real data
- Always read the source code in CTF challenges — the vulnerability is usually right there in plain sight
