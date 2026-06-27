# Guess My Cheese (Part 2) - picoCTF Writeup

**Challenge:** Guess My Cheese (Part 2)  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{cHeEsYab922171}`  
**Platform:** picoCTF (2025)  
**Writeup by:** zham  

---

## Description

> The imposter was able to fool us last time, so we've strengthened our defenses!
>
> Here's our list of cheeses.
>
> Connect to the program on our server: `nc verbal-sleep.picoctf.net 62865`

## Hints

> 1. I heard that SHA-256 is the best hash function out there!
> 2. Remember Squeexy, we enjoy our cheese with exactly 2 nibbles of hexadecimal-character salt!
> 3. Ever heard of rainbow tables?

---

## Background Knowledge

Before jumping in, here are the three concepts the hints point at.

### What is SHA-256?

**SHA-256** is a cryptographic hash function. You feed it any string of bytes and it spits out a fixed 256-bit (32-byte, 64 hex character) "fingerprint." The same input always produces the same output, but you cannot reverse the function to recover the input. SHA-256 is the modern workhorse for integrity checks, password storage, and digital signatures.

```
"cheddar"     --> SHA-256 --> "49a7d6f9..."
"cheddar!!"   --> SHA-256 --> "9c2e3ab8..."   (one byte changed = totally different hash)
```

It is a one-way street: given a hash, the only way to find the original input is to guess inputs and hash them until one matches. That is what this challenge makes us do.

### What is a Salt?

A **salt** is a small piece of random data mixed in with the input before hashing. The reason is simple: without a salt, every user who picks the password "cheddar" has the same hash on disk, so an attacker who cracks one of them cracks everyone. With a salt, each user's hash looks totally different even when the underlying password is identical.

The hint says **"exactly 2 nibbles of hexadecimal-character salt."** A nibble is 4 bits, so 2 nibbles = 1 byte = 8 bits. Each nibble is a hex character (`0-9a-f`), so the salt is one of 16 x 16 = **256 possible byte values**, ranging from `00` to `ff`.

```
SHA-256("cheddar" + 0x00) = "a1b2c3..."
SHA-256("cheddar" + 0x01) = "d4e5f6..."
SHA-256("cheddar" + 0xff) = "7a8b9c..."
```

256 possibilities is tiny by modern standards — but it is enough to break the kind of plain dictionary attack that defeated Part 1 of this challenge. The catch is that the salt here is one of 256 known values, so we can still build a rainbow table that covers all of them.

### What is a Rainbow Table?

A **rainbow table** is a precomputed dictionary of (plaintext -> hash) pairs. Instead of hashing candidate inputs every time you want to crack a hash, you hash them once, save the result, and look it up later.

```
Rainbow table:
   "cheddar" -> "a1b2c3..."
   "gouda"   -> "9f8e7d6c..."
   "brie"    -> "0ab1c2d3..."
```

For unsalted hashes one table covers every challenge instance. With a 1-byte salt you can either rebuild the table for each instance (wasteful) or, because the salt space is tiny, pre-build **one giant table** that covers every (salt x cheese) combination and look up any hash instantly. That is exactly what we do.

### Putting the Pieces Together

The two trick details that fool most beginners:

1. **Order of concatenation.** The hint says "we enjoy our cheese *with* salt," which is a friendly way of saying the salt is appended **after** the cheese, not prepended. So the formula is `cheese + salt`, not `salt + cheese`.
2. **Cheese names are lowercased on the server.** "Swiss" and "swiss" would produce different hashes otherwise and double the table size for nothing. Lowercasing every entry of the cheese list once keeps the table tight and matches the server.

So the server computes:

```
hash = SHA-256( lowercase(cheese_name) || salt_byte )
```

where `salt_byte` is one byte (`0x00..0xff`) and `||` means byte concatenation. Pre-build a table of every (cheese, salt) combination, then look up the server's hash to recover both inputs in O(1).

---

## Solution

### Step 1: Grab the Cheese List

The challenge points us at a `list` of cheeses. I dropped the attached list into a working folder and confirmed the count:

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ ls
cheeses.txt
```

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ wc -l cheeses.txt
599 cheeses.txt
```

599 cheese names. 256 salts. That means at most **153,344** candidate hashes to precompute — trivial on a laptop.

### Step 2: Probe the Server to See What It Asks For

Before building anything, I connected once with `nc` to see the protocol:

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ nc verbal-sleep.picoctf.net 62865
*******************************************
***             Part 2                  ***
***    The Mystery of the CLONED RAT    ***
*******************************************

DRAT! The evil Dr. Lacktoes Inn Tolerant's clone was able to guess the cheese last time! I guess simple ciphers aren't good hashing methods. But now I've strengthened my encryption scheme so that now ONLY SQUEEXY can guess it...

Here's my secret cheese -- if you're Squeexy, you'll be able to guess it:  09f23a3ca16a823a60c92403ddf642be12658c2679a4824bc54240833b5be140

Commands: (g)uess my cheese
What would you like to do?
```

The server hands us a 64-character SHA-256 hash. We need to recover the cheese name **and** the salt. Sending `g` will start the guessing flow.

### Step 3: Build the Rainbow Table

I dropped into `nano` and wrote a small Python script:

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ nano build_table.py
```

I pasted this in:

```python
#!/usr/bin/env python3
"""Build a rainbow table for Guess My Cheese (Part 2).

Server computes: SHA-256( lowercase(cheese) || salt_byte )
where salt_byte is one byte (0x00..0xff).
"""
import hashlib
import json

with open("cheeses.txt") as f:
    cheeses = [line.strip().lower() for line in f if line.strip()]

table = {}
for cheese in cheeses:
    cb = cheese.encode()                       # bytes of the cheese
    for salt in range(256):
        h = hashlib.sha256(cb + bytes([salt])).hexdigest()
        table[h] = (salt, cheese)              # remember which salt + cheese made it

with open("final_table.json", "w") as f:
    json.dump(table, f)

print(f"Built {len(table)} hashes ({len(cheeses)} cheeses x 256 salts)")
```

Save: `Ctrl+O`, `Enter`, exit: `Ctrl+X` (the usual nano dance). Run it:

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ python3 build_table.py
Built 153344 hashes (599 cheeses x 256 salts)
```

The whole thing finishes in a couple of seconds. The lookup table is saved to `final_table.json`.

### Step 4: Write the Interactive Solver

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ nano solve.py
```

I pasted this in:

```python
#!/usr/bin/env python3
"""Connect, read the hash, crack it, then send the cheese AND the salt."""
import hashlib
import json
import re
import socket
import sys

with open("final_table.json") as f:
    table = json.load(f)

HOST, PORT = "verbal-sleep.picoctf.net", 62865


def recv_until(sock, marker, timeout=15):
    sock.settimeout(timeout)
    buf = b""
    while marker not in buf:
        chunk = sock.recv(8192)
        if not chunk:
            break
        buf += chunk
    return buf


def recv_all(sock, timeout=10):
    sock.settimeout(timeout)
    buf = b""
    try:
        while True:
            chunk = sock.recv(8192)
            if not chunk:
                break
            buf += chunk
    except socket.timeout:
        pass
    return buf


sock = socket.socket()
sock.connect((HOST, PORT))

# 1) Read banner and pull the target hash
banner = recv_until(sock, b"What would you like to do?").decode(errors="replace")
print(banner)
target = re.search(r"([0-9a-f]{64})", banner).group(1)
print(f"[+] Target hash: {target}")

# 2) Look it up
if target not in table:
    print("[-] Not in table, aborting.")
    sys.exit(1)

salt_int, cheese = table[target]
verify = hashlib.sha256(cheese.encode() + bytes([salt_int])).hexdigest()
print(f"[+] Cracked: cheese='{cheese}', salt=0x{salt_int:02x} ({salt_int})")
assert verify == target, "table mismatch!"

# 3) Tell the server we want to guess
sock.sendall(b"g\n")
print(recv_until(sock, b"So...what's my cheese?").decode(errors="replace"))

# 4) Send the cheese name
sock.sendall(cheese.encode() + b"\n")
print(recv_until(sock, b"Annnnd...what's my salt?").decode(errors="replace"))

# 5) Send the salt as two lowercase hex characters
salt_str = f"{salt_int:02x}"
sock.sendall(salt_str.encode() + b"\n")
print(f"[+] Sent salt: {salt_str}")

# 6) Read the flag
print(recv_all(sock, timeout=15).decode(errors="replace"))
sock.close()
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`. Run it:

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ python3 solve.py
*******************************************
***             Part 2                  ***
***    The Mystery of the CLONED RAT    ***
*******************************************

DRAT! The evil Dr. Lacktoes Inn Tolerant's clone was able to guess the cheese last time! I guess simple ciphers aren't good hashing methods. But now I've strengthened my encryption scheme so that now ONLY SQUEEXY can guess it...

Here's my secret cheese -- if you're Squeexy, you'll be able to guess it:  09f23a3ca16a823a60c92403ddf642be12658c2679a4824bc54240833b5be140

Commands: (g)uess my cheese
What would you like to do?

[+] Target hash: 09f23a3ca16a823a60c92403ddf642be12658c2679a4824bc54240833b5be140
[+] Cracked: cheese='grataron d' areches', salt=0xad (173)
   _   _
  (q\_/p)
   /. .\.-.....-.     ___,
  =\_t_/=     /  `\  (
    )\ ))__ __\   |___)
   (/-(/`  `nn---'

SQUEAK SQUEAK SQUEAK

         _   _
        (q\_/p)
         /. .\
  ,__   =\_t_/=
     )   /   \
    (   ((   ))
     \  /\) (/\
      `-\  Y  /
         nn^nn

Is that you, Squeexy? Are you ready to GUESS...MY...CHEEEEEEESE?
Remember, this is my encrypted cheese:  09f23a3ca16a823a60c92403ddf642be12658c2679a4824bc54240833b5be140
So...what's my cheese?

--- (server prompts for salt) ---

Annnnd...what's my salt?

[+] Sent salt: ad

         _   _
         /. .\         __
  ,__   =\_t_/=      .'o o_.-'|

munch...

munch...

munch...

MUNCH.............

YUM! MMMMmmmmMMMMmmmMMM!!! Yes...yesssss! That's my cheese!
Here's the password to the cloning room:  picoCTF{cHeEsYab922171}
```

The flag is `picoCTF{cHeEsYab922171}`.

---

## Alternative Solve Methods

### Method 1: `printf | nc` With No Python

If you want to skip writing a solver and just crack the hash by hand, you can. Pull the cracked (cheese, salt) pair out of the JSON table with `grep`:

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ grep -o '"09f23a3ca16a823a60c92403ddf642be12658c2679a4824bc54240833b5be140":\[[0-9]\+, "[^"]*"\]' final_table.json
"09f23a3ca16a823a60c92403ddf642be12658c2679a4824bc54240833b5be140":[173, "grataron d' areches"]
```

Then drive `nc` with a `printf` pipeline:

```
┌──(zham㉿kali)-[~/guess_my_cheese_2]
└─$ printf 'g\n%s\n%s\n' "grataron d' areches" "ad" | nc verbal-sleep.picoctf.net 62865 | tail -n 3
MUNCH.............

YUM! MMMMmmmmMMMMmmmMMM!!! Yes...yesssss! That's my cheese!
Here's the password to the cloning room:  picoCTF{cHeEsYab922171}
```

Same flag, no Python runtime, no socket code. The downside is that you have to crack the hash yourself first — `printf | nc` only automates the network half.

### Method 2: Brute-Force On The Fly (No Table)

You can also skip the precomputation and brute-force the hash directly, one (cheese, salt) pair at a time. With only 153,344 candidates this still finishes in a few seconds, and the script is shorter:

```python
#!/usr/bin/env python3
"""Crack the hash without a precomputed table."""
import hashlib
import socket
import re

with open("cheeses.txt") as f:
    cheeses = [line.strip().lower() for line in f if line.strip()]

sock = socket.socket()
sock.connect(("verbal-sleep.picoctf.net", 62865))
sock.settimeout(15)

banner = b""
while b"What would you like to do?" not in banner:
    banner += sock.recv(8192)
target = re.search(rb"([0-9a-f]{64})", banner).group(1).decode()
print(f"[+] Target: {target}")

# Linear scan instead of a table
cracked = None
for cheese in cheeses:
    cb = cheese.encode()
    for salt in range(256):
        if hashlib.sha256(cb + bytes([salt])).hexdigest() == target:
            cracked = (cheese, salt)
            break
    if cracked:
        break

cheese, salt = cracked
print(f"[+] Cracked: cheese={cheese!r}, salt=0x{salt:02x}")

sock.sendall(b"g\n")
import time; time.sleep(0.5)
sock.sendall(cheese.encode() + b"\n")
time.sleep(0.5)
sock.sendall(f"{salt:02x}\n".encode())
time.sleep(1)
sock.settimeout(5)
try:
    while True:
        chunk = sock.recv(8192)
        if not chunk:
            break
        print(chunk.decode(errors="replace"), end="")
except socket.timeout:
    pass
sock.close()
```

This works in any environment where you have `hashlib` and `socket` (i.e. almost everywhere) and does not need 14 MB of JSON on disk. The trade-off is that every instance re-hashes all 153,344 candidates from scratch.

---

## What Happened Internally

Here is the full timeline of how the solve worked, from "I see a service" to "I have the flag."

1. **Connected to the server.** The service printed the banner, the target SHA-256 hash, and a `(g)uess my cheese` prompt.
2. **Built the rainbow table once.** Loaded the 599 cheese names, lowercased each, and for every cheese hashed it with all 256 possible salt bytes. That produced a dictionary `hash -> (salt, cheese)` with 153,344 entries, serialized to `final_table.json` in about a second.
3. **Read the target hash.** The first `recv_until` captured the banner up to the `What would you like to do?` prompt, then a regex pulled the 64-hex-char hash out of the text.
4. **Looked up the hash in the table.** A single dictionary lookup returned `(salt_int, cheese)`. I verified with a re-hash that the table was internally consistent: `SHA-256(cheese.encode() + bytes([salt_int])) == target`.
5. **Sent `g` to start a guess.** The server replied with a cute Squeexy ASCII rat, the same hash as a reminder, and the prompt `So...what's my cheese?`
6. **Sent the cheese name.** The server replied with the prompt `Annnnd...what's my salt?` — easy to miss on a first read.
7. **Sent the salt as two lowercase hex characters.** I formatted `salt_int` with `f"{salt_int:02x}"` so a value of 173 became the string `"ad"`.
8. **Read the flag.** Three more `munch...` animations, then `YUM! MMMMmmmmMMMMmmmMMM!!! Yes...yesssss! That's my cheese!` followed by the flag.

The reason Part 1 was easier and Part 2 is harder is exactly the salt: with no salt, one rainbow table works for any challenge instance. With 2 nibbles of salt there are 256 possible salts, so the defender has to either iterate over all of them or build a 256x bigger table. Once that table exists, however, every future instance is again cracked in O(1) lookup time — which is why salted hashing with a **unique per-user random salt** is the production-grade answer.

---

## Tools Used

| Tool             | Purpose                                                          |
| ---------------- | ---------------------------------------------------------------- |
| `nc`             | Probe the server and read the banner / hash / flag               |
| `wc -l`          | Sanity-check the cheese list size                                |
| `nano`           | Write the Python scripts (`Ctrl+O`, `Enter`, `Ctrl+X`)           |
| `python3`        | Build the rainbow table and run the interactive solver           |
| `hashlib` (stdlib) | Compute SHA-256                                                 |
| `socket` (stdlib)  | Raw TCP client to talk to the challenge server                 |
| `json` (stdlib)    | Serialize the rainbow table to disk and reload it              |
| `re` (stdlib)      | Pull the 64-char hex hash out of the banner                    |
| `printf` / `grep`  | Alternative shell-only solve                                   |

---

## Key Takeaways

* **Salts only defeat reuse if they are unique per user.** A 1-byte (256-value) salt can be brute-forced or precomputed cheaply, which is why production systems use 16+ byte random salts generated per record.
* **SHA-256 is one-way.** You never "decrypt" it; you guess inputs until one matches. Rainbow tables make that guess fast when the input space is small.
* **Read the wording of the hints carefully.** "We enjoy our cheese *with* salt" told us the order is `cheese + salt`, not `salt + cheese`. The hints were the spec. Likewise, lowercasing the cheese list cut the table in half and avoided mysterious misses on entries like "Swiss" or "Gruyere."
* **Server prompts matter.** The follow-up `Annnnd...what's my salt?` prompt is easy to miss if your script just sends one payload and closes. Read the protocol step by step and wait for each prompt before sending the next input.
* **Pre-build the table once, reuse forever.** Because the cheese list and the salt space are both fixed, the 153,344-entry table never goes stale. Every future connection is an O(1) dictionary lookup. That is the point of rainbow tables in the first place.

**Flag wordplay decode:** `cHeEsY` is the obvious cheese-themed word in deliberately mixed case to fit the cheese theme, and `ab922171` is the randomized numeric suffix the challenge attaches — `171` is the decimal form of `0xab`, mirroring the salt-as-hex theme of the challenge and acting as the cheese-y signature on the flag.
