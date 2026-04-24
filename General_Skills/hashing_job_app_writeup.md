# HashingJobApp — picoCTF Writeup

**Challenge:** HashingJobApp  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{4ppl1c4710n_r3c31v3d_3eb82b73}`  

---

## Description

> If you want to hash with the best, beat this test!
> `nc saturn.picoctf.net 64790`

**Hint 1:** `You can use a commandline tool or web app to hash text`

**Tags:** `hashing`, `nc`, `shell`, `Python`

---

## Background Knowledge (Read This First!)

### What is MD5?

**MD5** (Message Digest Algorithm 5) is a hash function that takes any input and produces a fixed 32-character hex string. Like SHA-256, it's a one-way function — you can't reverse it. It's fast to compute, which is why this challenge uses it as a speed test.

Example: `'leeches'` → `1719980de08011881ae8f13c90baaa92`

### What is `echo -n` + `md5sum`?

On Linux, you can hash any string from the terminal:

```bash
echo -n 'sometext' | md5sum
```

The `-n` flag is **critical** — without it, `echo` adds a newline character `\n` at the end of the string, which changes the hash entirely. Always use `-n` when hashing strings.

### Why can't you solve this manually?

The server has a **very short timer**. By the time you:
1. Read the word
2. Switch to another terminal
3. Run `echo -n 'word' | md5sum`
4. Copy the hash
5. Switch back and paste it

...the timer has already expired. The server also asks **multiple rounds** in a row. This forces you to automate the solution with a script.

### What is a socket in Python?

A **socket** is a connection between two computers over a network. Python's `socket` library lets you open a TCP connection (like `nc` does), send and receive data, and automate the interaction — all in code. This is the same technique used throughout CTF challenges involving netcat.

### ⚠️ Two Important Notes

**Note 1 — The timer is strict**  
Do not try to answer manually. Even copy-pasting is too slow. Use the script.

**Note 2 — Use `nano` to save the script**  
Pasting multi-line Python into the terminal directly causes issues with indentation. Use `nano solve.py`, paste the code, then `Ctrl+X` → `Y` → `Enter` to save.

---

## Solution — Step by Step

### Step 1 — Try manually first (to understand what's happening)

```
┌──(zham㉿kali)-[~]
└─$ nc saturn.picoctf.net 64790
Please md5 hash the text between quotes, excluding the quotes: 'leeches'
Answer:
Time's up. Press Ctrl-C to disconnect.
```

The timer expires before you can hash and paste manually. Automation is required.

### Step 2 — Create the solve script

```
┌──(zham㉿kali)-[~]
└─$ nano solve.py
```

Paste this code:

```python
import socket, hashlib

s = socket.socket()
s.connect(('saturn.picoctf.net', 64790))
s.settimeout(10)

while True:
    try:
        data = s.recv(1024).decode()
        print(data)
        if 'picoCTF' in data:
            break
        if "'" in data:
            word = data.split("'")[1]
            hash = hashlib.md5(word.encode()).hexdigest()
            print('Sending:', hash)
            s.send((hash + '\n').encode())
    except:
        break
```

Press `Ctrl+X` → `Y` → `Enter` to save.

### Step 3 — Run the script

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
Please md5 hash the text between quotes, excluding the quotes: 'a morgue'
Answer:
Sending: 1c5d1684ae8cd2f62a070044e5fc40c7
Correct.
Please md5 hash the text between quotes, excluding the quotes: 'assembly lines'
Answer:
Sending: f8ad2642937fc973fb27821a6c047e7e
Correct.
Please md5 hash the text between quotes, excluding the quotes: 'jelly beans'
Answer:
Sending: cb7d86ece76c57eac5ed18420ca67ea0
Correct.
picoCTF{4ppl1c4710n_r3c31v3d_3eb82b73}
```

✅ Got the flag! 🎯

---

## How the Script Works

```python
word = data.split("'")[1]
```
Splits the server response on the `'` character and takes index `[1]` — the word between the quotes.

```python
hash = hashlib.md5(word.encode()).hexdigest()
```
Hashes the word using MD5 and returns the 32-character hex digest.

```python
s.send((hash + '\n').encode())
```
Sends the hash back to the server with a newline (equivalent to pressing Enter).

```python
if 'picoCTF' in data:
    break
```
Stops the loop once the flag appears in the server response.

---

## Alternative Method — One-liner manual hash (too slow but good to know)

```bash
echo -n 'word' | md5sum
```

This is how you would hash a single word manually. It works fine for understanding MD5, but is too slow for this challenge due to the timer.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` | Connect to the challenge server initially | ⭐ Easy |
| Python `socket` | Automate the TCP connection | ⭐⭐ Medium |
| Python `hashlib.md5` | Hash each word instantly | ⭐ Easy |
| `nano` | Save the multi-line script | ⭐ Easy |
| `echo -n \| md5sum` (optional) | Hash a word manually for testing | ⭐ Easy |

---

## Key Takeaways

- **When a netcat challenge has a timer, automate it** — Python's `socket` library replicates `nc` and responds in milliseconds
- **Always use `echo -n`** when hashing strings — the `-n` flag prevents a trailing newline from changing the hash
- **`hashlib.md5(word.encode()).hexdigest()`** is the standard Python one-liner for MD5 hashing
- **`data.split("'")[1]`** is a clean way to extract text between quotes — a useful pattern for parsing CTF server output
- The flag `4ppl1c4710n_r3c31v3d` → "application received" — you passed the hashing job application!
