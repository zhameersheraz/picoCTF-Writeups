# hashcrack — picoCTF Writeup

**Challenge:** hashcrack  
**Category:** Cryptography  
**Difficulty:** Easy  
**Flag:** `picoCTF{UseStr0nG_h@shEs_&PaSswDs!_5b836723}`  

---

## Description

> A company stored a secret message on a server which got breached due to the admin using weakly hashed passwords. Can you gain access to the secret stored within the server?
> Access the server using `nc verbal-sleep.picoctf.net 49490`

**Hint shown in challenge:** `Understanding hashes is very crucial. Read more here.`

---

## Background Knowledge (Read This First!)

### What are Password Hashes?

A hash is a **one-way function** — you can hash a password, but you cannot reverse it directly. However, if the password is weak and common, it will exist in a pre-computed database of hashes called **rainbow tables**.

### Common Hash Types

| Hash | Length | Example |
|------|--------|---------|
| MD5 | 32 chars | `482c811da5d5b4bc6d497ffa98491e38` |
| SHA-1 | 40 chars | `b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3` |
| SHA-256 | 64 chars | `916e8c4f79b25028...` |

### What is CrackStation?

CrackStation (https://crackstation.net/) has a massive database of pre-computed hashes. If a password is weak and common, it will be cracked instantly.

---

## Solution — Step by Step

### Step 1 — Connect to the Server

```
┌──(zham㉿kali)-[~]
└─$ nc verbal-sleep.picoctf.net 49490
Welcome!! Looking For the Secret?
We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash:
```

### Step 2 — Crack the First Hash (MD5)

Pasted `482c811da5d5b4bc6d497ffa98491e38` into CrackStation.

- **Type:** MD5
- **Password:** `password123`

```
Enter the password for identified hash: password123
Correct! You've cracked the MD5 hash with no secret found!
Flag is yet to be revealed!! Crack this hash: b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3
```

### Step 3 — Crack the Second Hash (SHA-1)

Pasted `b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3` into CrackStation.

- **Type:** SHA-1
- **Password:** `letmein`

```
Enter the password for the identified hash: letmein
Correct! You've cracked the SHA-1 hash with no secret found!
Almost there!! Crack this hash: 916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745
```

### Step 4 — Crack the Third Hash (SHA-256)

Pasted `916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745` into CrackStation.

- **Type:** SHA-256
- **Password:** `qwerty098`

```
Enter the password for the identified hash: qwerty098
Correct! You've cracked the SHA-256 hash with a secret found.
The flag is: picoCTF{UseStr0nG_h@shEs_&PaSswDs!_5b836723}
```

Got the flag! 🎯

---

## Alternative Method — hashcat

```
┌──(zham㉿kali)-[~]
└─$ echo "482c811da5d5b4bc6d497ffa98491e38" > hash.txt

┌──(zham㉿kali)-[~]
└─$ hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
# -m 0    = MD5
# -m 100  = SHA-1
# -m 1400 = SHA-256
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` (netcat) | Connect to the server |
| CrackStation | Online hash cracker |
| `hashcat` (alternative) | Command-line hash cracker |

---

## Key Takeaways

- Hashing ≠ Encryption — Hashes are one-way, encryption is two-way
- Weak passwords are vulnerable regardless of hash algorithm
- Rainbow tables make cracking common passwords instant
- Always **add salt** (random data before hashing) to prevent rainbow table attacks
- Common weak passwords: `password123`, `letmein`, `qwerty`, `123456`, `admin`
