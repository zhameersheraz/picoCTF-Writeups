# EVEN RSA CAN BE BROKEN??? — picoCTF Writeup

**Challenge:** EVEN RSA CAN BE BROKEN???  
**Category:** Cryptography  
**Difficulty:** Easy  
**Flag:** `picoCTF{tw0_1$_pr!m305af7255}`  

---

## Description

> This service provides you an encrypted flag. Can you decrypt it with just N & e?
> Connect to the program with netcat:
> `nc verbal-sleep.picoctf.net 53376`

**Hint shown in challenge:** `Try comparing N across multiple requests`

---

## Background Knowledge (Read This First!)

### What is RSA?

RSA is a public-key encryption system. Key generation:

1. Choose two large prime numbers **p** and **q**
2. Calculate **N = p × q**
3. Calculate **φ(N) = (p-1)(q-1)**
4. Choose **e** (usually 65537)
5. Calculate **d = e⁻¹ mod φ(N)** (private key)

Encryption: `c = m^e mod N`
Decryption: `m = c^d mod N`

### Why is this Challenge Vulnerable?

RSA security depends entirely on the difficulty of **factoring N**. If N is generated using weak or small prime numbers, it can be factored — and once factored, RSA is completely broken.

Real RSA uses 2048–4096 bit primes (impossible to factor with current technology). This challenge uses weak primes that can be factored instantly.

---

## Solution — Step by Step

### Step 1 — Connect to the Server

```
┌──(zham㉿kali)-[~]
└─$ nc verbal-sleep.picoctf.net 53376
N: 20644215349775590552410650310412844225776854839619024783751476839963780745182385359503567602562583207209981352591054690116384779173366501792986214410988686
e: 65537
cyphertext: 8068674388761396575984820904752113663737384849556331681361005967261136763901401907329354327258945953275620968085975269255703202210614776699338982064442385
```

### Step 2 — Factor N Using dCode RSA Tool

I went to https://dcode.fr/rsa-cipher and entered:

- **N:** `20644215349775590552410650310412844225776854839619024783751476839963780745182385359503567602562583207209981352591054690116384779173366501792986214410988686`
- **e:** `65537`
- **Ciphertext:** `8068674388761396575984820904752113663737384849556331681361005967261136763901401907329354327258945953275620968085975269255703202210614776699338982064442385`

### Step 3 — Get the Flag

The dCode tool automatically factored N, computed the private key d, and decrypted the message:

```
picoCTF{tw0_1$_pr!m305af7255}
```

Got the flag! 🎯

---

## Alternative Method — RsaCtfTool

```
┌──(zham㉿kali)-[~]
└─$ git clone https://github.com/RsaCtfTool/RsaCtfTool.git

┌──(zham㉿kali)-[~]
└─$ cd RsaCtfTool

┌──(zham㉿kali)-[~/RsaCtfTool]
└─$ pip3 install -r requirements.txt

┌──(zham㉿kali)-[~/RsaCtfTool]
└─$ echo "806867438876..." > cipher.txt

┌──(zham㉿kali)-[~/RsaCtfTool]
└─$ python3 RsaCtfTool.py -n 20644215... -e 65537 --uncipher cipher.txt
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` (netcat) | Connect to challenge server |
| dCode RSA Cipher Tool | Factor N and decrypt the flag |
| RsaCtfTool (alternative) | Automated RSA attack tool |

---

## Key Takeaways

- RSA security depends entirely on the difficulty of factoring N
- If N can be factored, RSA is completely broken
- Always use large prime numbers (2048+ bits) for real RSA
- The flag "two is prime" is a joke — 2 is actually the only even prime number!
- dcode.fr is a great go-to site for CTF crypto challenges
