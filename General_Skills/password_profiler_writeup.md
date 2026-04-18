# Password Profiler — picoCTF Writeup

**Challenge:** Password Profiler  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{Aj_15901990}`  

---

## Description

> We intercepted a suspicious file from a system, but instead of the password itself, it only contains its SHA-1 hash. Using OSINT techniques, you are provided with personal details about the target. Your task is to leverage this information to generate a custom password list and recover the original password by matching its hash.

**Downloads:** `userinfo.txt`, `hash.txt`, `check_password.py`  
**Hint 1:** `CUPP is a Python tool for generating custom wordlists from personal data.`

---

## Background Knowledge (Read This First!)

### What is Password Profiling?

In real-world attacks and CTFs alike, people often base their passwords on personal information — names, birthdays, nicknames, and combinations of these. **Password profiling** is the technique of using known personal details about a target to generate a tailored wordlist of likely passwords, rather than brute-forcing millions of random combinations.

### What is CUPP?

**CUPP** (Common User Passwords Profiler) is a tool that takes personal information about a target and automatically generates a wordlist of likely passwords. It combines names, dates, nicknames, and appends common suffixes like special characters, numbers, and leet speak (e.g. `a → 4`, `e → 3`).

For example, given the name `Alice` and birthdate `15071990`, CUPP might generate:
```
Alice1990
alice15071990
4l1c3!
AJ1990@
...
```

### What is SHA-1?

**SHA-1** is a hash function — it takes any input and produces a fixed 40-character hex string. Like SHA-256, it's a one-way function: you can't reverse it to get the original password. The only way to crack it is to hash many candidate passwords and compare each result against the target hash.

### What is a Wordlist Attack?

Instead of trying every possible combination of characters (pure brute force), a **wordlist attack** hashes each word in a pre-built list and checks if it matches the target. If the real password is somewhere in the list, it will be found. CUPP makes this practical by generating a smart, targeted list based on what we know about the target.

### ⚠️ Important Note

CUPP on Kali may show `SyntaxWarning` messages when it runs — these are harmless and don't affect the output. The wordlist will still be generated correctly.

---

## Files Provided

### `userinfo.txt`
```
First Name: Alice
Surname: Johnson
Nickname: AJ
Birthdate: 15-07-1990
Partner's Name: Bob
Child's Name: Charlie
```

### `hash.txt`
```
968c2349040273dd57dc4be7e238c5ac200ceac5
```
This is the SHA-1 hash of Alice's password. We need to find what password produces this exact hash.

### `check_password.py`
```python
import hashlib

HASH_FILE = "hash.txt"
WORDLIST_FILE = "passwords.txt"

def load_hash():
    with open(HASH_FILE, "r") as f:
        return f.read().strip()

def crack_password(target_hash):
    with open(WORDLIST_FILE, "r", encoding="utf-8", errors="ignore") as f:
        for password in f:
            password = password.strip()
            if hashlib.sha1(password.encode()).hexdigest() == target_hash:
                return password
    return None

if __name__ == "__main__":
    target_hash = load_hash()
    result = crack_password(target_hash)
    if result:
        print(f"Password found: picoCTF{{{result}}}")
    else:
        print("No match found.")
```

This script reads every word from `passwords.txt`, hashes it with SHA-1, and compares it to the target hash. When it finds a match, it prints the flag.

---

## Solution — Step by Step

### Step 1 — Install CUPP

```
┌──(zham㉿kali)-[~]
└─$ sudo apt install cupp -y
```

### Step 2 — Run CUPP in interactive mode

```
┌──(zham㉿kali)-[~]
└─$ cupp -i
```

Fill in the prompts using the information from `userinfo.txt`. For anything not provided, just press Enter to skip:

```
> First Name: Alice
> Surname: Johnson
> Nickname: AJ
> Birthdate (DDMMYYYY): 15071990

> Partners) name: Bob
> Partners) nickname: (Enter)
> Partners) birthdate (DDMMYYYY): (Enter)

> Child's name: Charlie
> Child's nickname: (Enter)
> Child's birthdate (DDMMYYYY): (Enter)

> Pet's name: (Enter)
> Company name: (Enter)

> Do you want to add some key words about the victim? Y/[N]: N
> Do you want to add special chars at the end of words? Y/[N]: y
> Do you want to add some random numbers at the end of words? Y/[N]: y
> Leet mode? (i.e. leet = 1337) Y/[N]: y
```

CUPP generates the wordlist and saves it:

```
[+] Saving dictionary to alice.txt, counting 28128 words.
```

### Step 3 — Rename the wordlist

The script looks for a file named `passwords.txt`, so rename it:

```
┌──(zham㉿kali)-[~]
└─$ mv alice.txt passwords.txt
```

### Step 4 — Copy the challenge files to the same folder

Make sure `hash.txt` and `check_password.py` are in the same directory as `passwords.txt`:

```
┌──(zham㉿kali)-[~]
└─$ cp /media/sf_downloads/hash.txt .
┌──(zham㉿kali)-[~]
└─$ cp /media/sf_downloads/check_password.py .
```

The `.` means "copy to the current folder."

### Step 5 — Run the cracking script

```
┌──(zham㉿kali)-[~]
└─$ python3 check_password.py
Password found: picoCTF{Aj_15901990}
```

✅ Got the flag! 🎯

---

## Breaking Down the Password

The cracked password is `Aj_15901990`. Let's see where each part came from:

| Part | Source |
|------|--------|
| `Aj` | Alice's **nickname** (AJ), mixed case |
| `_` | Special character appended by CUPP |
| `1590` | Part of the **birthdate** (15-07-1990) rearranged |
| `1990` | **Birth year** |

This is a classic example of how people construct passwords from personal details — and why profiled wordlists are so effective at cracking them.

---

## Alternative Methods

### Alternative 1 — Hash the wordlist manually with Python

If you don't want to use `check_password.py`, you can do the same thing in a one-liner:

```bash
while read p; do
    echo -n "$p" | sha1sum | grep -q "968c2349040273dd57dc4be7e238c5ac200ceac5" && echo "Found: $p"
done < passwords.txt
```

### Alternative 2 — Use hashcat

**Hashcat** is a professional password cracking tool that can use GPU acceleration for much faster cracking. For SHA-1 (mode 100):

```bash
hashcat -m 100 hash.txt passwords.txt
```

This is overkill for a small wordlist, but useful to know for larger challenges.

### Alternative 3 — Use john the ripper

**John the Ripper** is another classic cracking tool available on Kali:

```bash
john --format=raw-sha1 --wordlist=passwords.txt hash.txt
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `cupp -i` | Generate a profiled wordlist from personal info | ⭐ Easy |
| `check_password.py` | Hash each word and compare to target SHA-1 | ⭐ Easy |
| `hashcat` (optional) | GPU-accelerated wordlist attack | ⭐⭐ Medium |
| `john` (optional) | Alternative wordlist cracker | ⭐⭐ Medium |

---

## Key Takeaways

- **People use personal details as passwords** — names, birthdays, and nicknames are the most common building blocks
- **CUPP automates profiled wordlist generation** — given basic OSINT data, it creates thousands of likely password variations automatically
- **SHA-1 is not a secure password storage method** — it's fast to compute, which makes it fast to crack; modern systems should use bcrypt or Argon2
- **A targeted wordlist beats brute force** — 28,128 profiled guesses found the password; a random brute force of the same length would take billions of attempts
- **OSINT + wordlist = powerful combination** — knowing even basic personal details about a target dramatically narrows the search space
