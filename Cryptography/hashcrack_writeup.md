# hashcrack - picoCTF Writeup

**Challenge:** hashcrack  
**Category:** Cryptography  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{UseStr0nG_h@shEs_&PaSswDs!_5b836723}`

---

## Description

A company stored a secret message on a server which got breached due to the admin using weakly hashed passwords.

Can you gain access to the secret stored within the server?

Access the server using `nc verbal-sleep.picoctf.net 49490`

---

## Hints

1. Understanding hashes is very crucial. Read more here.

---

## Solution

### Step 1: Connect to the Server

I used netcat to connect to the challenge server:
```bash
nc verbal-sleep.picoctf.net 49490
```

**Output:**
```
Welcome!! Looking For the Secret?
We have identified a hash: 482c811da5d5b4bc6d497ffa98491e38
Enter the password for identified hash:
```

The server gave me the first hash to crack.

### Step 2: Crack the First Hash (MD5)

I went to **CrackStation** (https://crackstation.net/) and pasted the first hash:

**Hash:** `482c811da5d5b4bc6d497ffa98491e38`

**CrackStation Result:**
- **Type:** MD5
- **Password:** `password123`

I entered the password:
```
Enter the password for identified hash: password123
Correct! You've cracked the MD5 hash with no secret found!
Flag is yet to be revealed!! Crack this hash: b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3
```

### Step 3: Crack the Second Hash (SHA-1)

The server gave me a second hash to crack.

**Hash:** `b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3`

I went back to CrackStation and pasted this hash:

**CrackStation Result:**
- **Type:** SHA-1
- **Password:** `letmein`

I entered the password:
```
Enter the password for the identified hash: letmein
Correct! You've cracked the SHA-1 hash with no secret found!
Almost there!! Crack this hash: 916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745
```

### Step 4: Crack the Third Hash (SHA-256)

The server gave me a final hash to crack.

**Hash:** `916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745`

I went back to CrackStation and pasted this hash:

**CrackStation Result:**
- **Type:** SHA-256
- **Password:** `qwerty098`

I entered the password:
```
Enter the password for the identified hash: qwerty098
Correct! You've cracked the SHA-256 hash with a secret found. 
The flag is: picoCTF{UseStr0nG_h@shEs_&PaSswDs!_5b836723}
```

Got the flag! 🎯

---

## Why This Works

* **Weak passwords** like "password123", "letmein", and "qwerty098" are commonly used
* **CrackStation** has a massive database of pre-computed hashes (rainbow tables)
* Even though different hash algorithms were used (MD5, SHA-1, SHA-256), the passwords were **weak enough** to be in the database
* **Hashing doesn't equal security** - weak passwords can be cracked regardless of the hash algorithm
* This demonstrates why **strong, unique passwords** are essential

---

## What are Password Hashes?

**Hashing** is a one-way function that converts a password into a fixed-length string:

**Common Hash Types:**
- **MD5** - 32 characters (128-bit) - Example: `482c811da5d5b4bc6d497ffa98491e38`
- **SHA-1** - 40 characters (160-bit) - Example: `b7a875fc1ea228b9061041b7cec4bd3c52ab3ce3`
- **SHA-256** - 64 characters (256-bit) - Example: `916e8c4f79b25028c9e467f1eb8eee6d6bbdff965f9928310ad30a8d88697745`

**How Hash Cracking Works:**
1. **Rainbow Tables** - Pre-computed tables of hash → password mappings
2. **Dictionary Attacks** - Hashing common passwords and comparing
3. **Brute Force** - Trying all possible combinations (very slow)

**Why Weak Passwords Fail:**
- Common passwords are in rainbow table databases
- "password123" is immediately crackable
- Even with strong hashing (SHA-256), weak passwords = no security

---

## Alternative Method: Using hashcat or John the Ripper

### Method 1: hashcat (Command Line)
```bash
# Save hash to file
echo "482c811da5d5b4bc6d497ffa98491e38" > hash.txt

# Crack using rockyou.txt wordlist
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
# -m 0 = MD5
# -m 100 = SHA-1
# -m 1400 = SHA-256
```

### Method 2: John the Ripper
```bash
# Save hash with format
echo "password123:482c811da5d5b4bc6d497ffa98491e38" > hash.txt

# Crack
john --format=raw-md5 hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

### Method 3: CrackStation (Easiest - What I Used)

Just paste the hash on https://crackstation.net/ and get instant results!

---

## Terminal Commands
```bash
# Connect to the challenge server
nc verbal-sleep.picoctf.net 49490

# Round 1: MD5 hash
# Input: password123

# Round 2: SHA-1 hash
# Input: letmein

# Round 3: SHA-256 hash
# Input: qwerty098

# Receive flag!
```

---

## Flag

`picoCTF{UseStr0nG_h@shEs_&PaSswDs!_5b836723}`

---

## Tools Used

* **netcat (nc)** - Connect to the server
* **CrackStation** - Online hash cracker (https://crackstation.net/)
* **Alternative:** hashcat, John the Ripper, rainbow tables

---

## Key Takeaways

* **Hashing ≠ Encryption** - Hashes are one-way, encryption is two-way
* **Weak passwords are vulnerable** regardless of hash algorithm
* **Rainbow tables** make cracking common passwords instant
* **Use strong passwords** - Long, random, unique
* **Add salt** - Random data added to passwords before hashing prevents rainbow table attacks
* Common weak passwords: password123, letmein, qwerty, 123456, admin
* This is why websites require: uppercase, lowercase, numbers, symbols, minimum length
* The flag message reinforces this: "Use Strong Hashes & Passwords!"
