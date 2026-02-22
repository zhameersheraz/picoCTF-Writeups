# EVEN RSA CAN BE BROKEN??? - picoCTF Writeup

**Challenge:** EVEN RSA CAN BE BROKEN???  
**Category:** Cryptography  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{tw0_1$_pr!m305af7255}`

---

## Description

This service provides you an encrypted flag. Can you decrypt it with just N & e?

Connect to the program with netcat:
```
nc verbal-sleep.picoctf.net 53376
```

The program's source code can be downloaded here.

---

## Hints

1. Try comparing N across multiple requests

---

## Solution

### Step 1: Connect to the Server

I used netcat to connect to the challenge server:
```bash
nc verbal-sleep.picoctf.net 53376
```

**Output:**
```
N: 20644215349775590552410650310412844225776854839619024783751476839963780745182385359503567602562583207209981352591054690116384779173366501792986214410988686
e: 65537
cyphertext: 8068674388761396575984820904752113663737384849556331681361005967261136763901401907329354327258945953275620968085975269255703202210614776699338982064442385
```

I received:
- **N** (modulus): The product of two prime numbers (p × q)
- **e** (public exponent): 65537
- **Ciphertext**: The encrypted flag

### Step 2: Understanding RSA

In RSA encryption:
- **Public Key**: (N, e)
- **Private Key**: (d)
- **Encryption**: c = m^e mod N
- **Decryption**: m = c^d mod N

To decrypt, I need to find **d**, which requires knowing the factors of N (p and q).

### Step 3: Factor N Using dCode RSA Tool

I went to https://dcode.fr/rsa-cipher and used their RSA decoder tool.

I entered:
- **N**: `20644215349775590552410650310412844225776854839619024783751476839963780745182385359503567602562583207209981352591054690116384779173366501792986214410988686`
- **e**: `65537`
- **Ciphertext**: `8068674388761396575984820904752113663737384849556331681361005967261136763901401907329354327258945953275620968085975269255703202210614776699338982064442385`

### Step 4: Tool Automatically Factored N

The dCode tool automatically factored N and found:
- **p** (Factor 1): A prime number
- **q** (Factor 2): Another prime number

The tool computed the private key **d** and decrypted the message.

### Step 5: Get the Flag

After clicking "CALCULATE/DECRYPT", the tool showed:
```
picoCTF{tw0_1$_pr!m305af7255}
```

The flag reads: "two is prime" (tw0_1$_pr!m305af7255)

---

## Why This Works

* **Weak RSA Implementation**: The N value is factorable
* **Small Prime Factors**: N was generated using weak/small prime numbers
* **Automated Tools**: Tools like dCode have databases of known prime factors
* If N can be factored into p and q, RSA is completely broken
* Once you have p and q, you can compute:
  - φ(N) = (p-1)(q-1)
  - d = e^(-1) mod φ(N)
  - Then decrypt: m = c^d mod N

---

## What is RSA?

**RSA** is a public-key cryptosystem used for secure data transmission:

**Key Generation:**
1. Choose two large prime numbers p and q
2. Calculate N = p × q
3. Calculate φ(N) = (p-1)(q-1)
4. Choose e (usually 65537)
5. Calculate d = e^(-1) mod φ(N)

**Why This Challenge is Vulnerable:**
- N uses weak/small primes that can be factored
- Once N is factored, the entire system is broken
- Real RSA uses 2048-4096 bit primes (impossible to factor with current technology)

---

## Alternative Method: Using RsaCtfTool
```bash
# Install RsaCtfTool
git clone https://github.com/RsaCtfTool/RsaCtfTool.git
cd RsaCtfTool
pip3 install -r requirements.txt

# Create a file with the ciphertext
echo "8068674388761396575984820904752113663737384849556331681361005967261136763901401907329354327258945953275620968085975269255703202210614776699338982064442385" > cipher.txt

# Decrypt
python3 RsaCtfTool.py -n 20644215349775590552410650310412844225776854839619024783751476839963780745182385359503567602562583207209981352591054690116384779173366501792986214410988686 -e 65537 --uncipher cipher.txt
```

---

## Terminal Commands
```bash
# Connect to the server
nc verbal-sleep.picoctf.net 53376

# Copy N, e, and ciphertext values

# Use online tool: https://dcode.fr/rsa-cipher
# Or use RsaCtfTool (automated RSA cracking tool)
```

---

## Flag

`picoCTF{tw0_1$_pr!m305af7255}`

---

## Tools Used

* **netcat (nc)** - Connect to challenge server
* **dCode RSA Cipher Tool** - https://dcode.fr/rsa-cipher
* **RsaCtfTool** (alternative) - Automated RSA attack tool

---

## Key Takeaways

* RSA security depends entirely on the difficulty of factoring N
* If N can be factored, RSA is completely broken
* Always use large prime numbers (2048+ bits) for real RSA
* Never use weak/small primes in production
* Tools like dCode and RsaCtfTool can factor weak RSA implementations
* The challenge name "EVEN RSA CAN BE BROKEN???" teaches that poorly implemented RSA is vulnerable
* The flag "two is prime" is a joke - 2 is actually the only even prime number!
* This demonstrates why key size matters in cryptography
