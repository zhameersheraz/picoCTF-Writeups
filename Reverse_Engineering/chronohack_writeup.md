# Chronohack — picoCTF Writeup

**Challenge:** Chronohack  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{UseSecure#$_Random@j3n3r@T0rs13909228}`  
**Platform:** picoCTF 2025  
**Writeup by:** zham  

---

## Description

> Can you guess the exact token and unlock the hidden flag?
>
> Our school relies on tokens to authenticate students. Unfortunately, someone leaked an important file for token generation. Guess the token to get the flag.
>
> The access is granted through `nc verbal-sleep.picoctf.net 52304`.

## Hints

> 1. https://www.epochconverter.com/
> 2. https://learn.snyk.io/lesson/insecure-randomness/
> 3. Time tokens generation
> 4. Generate tokens for a range of seed values very close to the target time

---

## Background Knowledge (Read This First!)

### What is `random.seed()`?

Python's `random` module produces a deterministic stream of "random" numbers. The whole stream is generated from a single starting value called the **seed**. If you know the seed, you can reproduce the exact same sequence on your own machine.

```python
import random
random.seed(42)
print(random.choice("ABCDE"))  # always prints the same letter
```

That is great for repeatable tests, but it is terrible for security. If the attacker can guess the seed, the attacker can predict every "random" value the program will ever produce.

### Why Seeding With the Time is a Disaster

The leaked source code does this:

```python
def get_random(length):
    random.seed(int(time.time() * 1000))   # seed = current time, in milliseconds
    ...
```

`int(time.time() * 1000)` is the current Unix time in **milliseconds**. The attacker just needs to:

1. Note the current time (publicly available, same on every machine that synchronises with NTP).
2. Try that exact value as the seed.
3. Regenerate the token locally and submit it.

The program treats the wall clock as a secret, but the wall clock is the opposite of a secret. This is the textbook example of **insecure randomness**.

### The Million-Millisecond Trick

`time.time()` returns a float. The integer part keeps growing forever, so the seed space is huge. But the server only picks a seed from a tiny window around the moment the connection is established. The attacker only has to scan a small range of millisecond values (say, a few hundred) and one of them will match.

### The Hint Sites Confirm It

- `epochconverter.com` — converts Unix timestamps to human-readable dates, so you can see exactly what the seed value looks like.
- `learn.snyk.io/lesson/insecure-randomness` — explains why using `time()` (or any predictable value) as an RNG seed is unsafe in production.
- "Time tokens generation" — measure when the server starts up.
- "Generate tokens for a range of seed values" — once you know the rough time, brute-force the rest by trying seeds in a small window.

---

## Solution — Step by Step

### Step 1 — Read the Leaked Source

I opened the leaked Python file the challenge gave me and spotted the vulnerable line straight away:

```python
import random
import time

def get_random(length):
    alphabet = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    random.seed(int(time.time() * 1000))  # seeding with current time
    s = ""
    for i in range(length):
        s += random.choice(alphabet)
    return s
```

Two things jump out:

- The seed is the **current time in milliseconds**, which the attacker can measure.
- The token is 20 characters long, drawn from a 62-character alphabet, generated with the standard `random.choice`. Reproducing it locally is trivial once we know the seed.

The other 50 guesses the program allows per connection is more than enough, because a single connection spans only a few tens of milliseconds.

### Step 2 — Sanity-Check the Reproducer Locally

Before touching the server, I verified that feeding different millisecond seeds really does produce different tokens, so I know I can brute-force a small window:

```
┌──(zham㉿kali)-[~/picoctf/chronohack]
└─$ python3 -c "
import random, time
alpha = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
now_ms = int(time.time() * 1000)
for d in (-2,-1,0,1,2):
    random.seed(now_ms + d)
    print(d, ''.join(random.choice(alpha) for _ in range(20)))"
-2 9TzR4dQ9m0o1FjLi0Q0j
-1 1XKuC0Qi6u4w8O0y8W2B
0 4YrR5rM9c2s5Vt2k1E3a
1 8N0f3kL0e7d4Cq6h9P2z
2 6B1g7pP2a9s1Wn8e5R4x
```

Every millisecond produces a completely different token. Good — the brute-force window just needs to land on the right millisecond.

### Step 3 — Probe the Server to Measure Latency

I connected once with a tiny script that records the round-trip time, just to know how much buffer to add to my brute-force window:

```
┌──(zham㉿kali)-[~/picoctf/chronohack]
└─$ python3 probe.py
connect: 18.7 ms, banner-received: 47.3 ms
--- server says ---
Welcome to the token generation challenge!
Can you guess the token?

Enter your guess for the token (or exit):
```

The banner arrives about 47 ms after I call `connect()`. The server's `get_random()` is called inside `main()` right after the prints, so the server's seed is roughly `my_connect_time + a few ms`. A window of about 100 ms around my pre-connect timestamp is plenty.

### Step 4 — Brute-Force the Seed

The plan:

1. Record the current time in milliseconds.
2. Open one connection.
3. For each offset in a small range, compute the token the server would generate for that seed.
4. Send all of them and watch for `Congratulations`.

I first tried a window centred on the reference time, but the server's seed lands a little later because of the connect-time overhead. So I tried a slightly skewed window that covers `[-30, +69]` and used one guess per millisecond (each guess is its own connection attempt, so the loop stays in the right time range).

```
┌──(zham㉿kali)-[~/picoctf/chronohack]
└─$ nano exploit.py
```

In `nano` I pasted the script and saved with `Ctrl+O`, `Enter`, `Ctrl+X`:

```python
#!/usr/bin/env python3
import socket, time, random, re, sys

HOST = "verbal-sleep.picoctf.net"
PORT = 52304
ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

def token_for(seed_ms):
    random.seed(seed_ms)
    return "".join(random.choice(ALPHABET) for _ in range(20))

def attempt(seed_ms, token):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(4)
    try:
        s.connect((HOST, PORT))
        s.recv(4096)              # banner
        s.sendall((token + "\n").encode())
        s.settimeout(3)
        resp = s.recv(4096).decode(errors="replace")
        s.close()
        if "Congratulations" in resp or "picoCTF{" in resp:
            return resp
    except Exception as e:
        print("err:", e)
    return None

if __name__ == "__main__":
    offsets = list(range(-30, 70))    # 100 attempts
    for off in offsets:
        t_ref = int(time.time() * 1000)
        seed = t_ref + off
        tok = token_for(seed)
        r = attempt(seed, tok)
        mark = " <-- WIN" if r else ""
        print(f"  off={off:+4d}  seed={seed}  tok={tok}{mark}")
        if r:
            print("\n=== SERVER RESPONSE ===")
            print(r)
            m = re.search(r"picoCTF\{[^}]+\}", r)
            if m: print(f"\nFLAG: {m.group(0)}")
            sys.exit(0)
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`.

Run it:

```
┌──(zham㉿kali)-[~/picoctf/chronohack]
└─$ python3 exploit.py
  off= -30  seed=1782225280822  tok=DweB0OoCEWF0vzNooXzf
  off= -29  seed=1782225280885  tok=4WhA6D0cRUgEYKb9m7L9
  ...
  off= +30  seed=1782225284402  tok=mMWo1TxklagQjr8XuW1W
  off= +31  seed=1782225284454  tok=7dEOm9Yv7YVZPy6A65qp
  off= +32  seed=1782225284516  tok=rcCBJ3TEoBoKFkGOvcAC <-- WIN

=== SERVER RESPONSE ===
Congratulations! You found the correct token.
picoCTF{UseSecure#$_Random@j3n3r@T0rs13909228}

FLAG: picoCTF{UseSecure#$_Random@j3n3r@T0rs13909228}
```

Got the flag on offset `+32`. That tells me the server's seed was about 32 ms after my pre-connect timestamp, which lines up nicely with the ~18 ms TCP connect plus the few milliseconds the server spends printing the banner before calling `get_random()`.

---

## Alternative — One Connection, 50 Guesses

If you would rather only open one connection (and the server's seed lands inside the first 50 guesses), here is a tighter version that sends all 50 tokens in a single session:

```python
#!/usr/bin/env python3
import socket, time, random

ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

def token_for(seed_ms):
    random.seed(seed_ms)
    return "".join(random.choice(ALPHABET) for _ in range(20))

s = socket.socket(); s.settimeout(5)
s.connect(("verbal-sleep.picoctf.net", 52304))
s.recv(4096)                         # banner

centre = int(time.time() * 1000)
for d in range(-25, 25):             # 50 guesses
    tok = token_for(centre + d)
    s.sendall((tok + "\n").encode())
    data = s.recv(4096).decode(errors="replace")
    if "Congratulations" in data:
        print("FLAG:", data.strip())
        break
s.close()
```

```
┌──(zham㉿kali)-[~/picoctf/chronohack]
└─$ python3 exploit_one.py
FLAG: Congratulations! You found the correct token.
picoCTF{UseSecure#$_Random@j3n3r@T0rs13909228}
```

Faster, but you only get one shot — if your 50 ms window misses the seed, you have to reconnect.

---

## Alternative — pwntools

For a cleaner, more "CTF-canonical" implementation, pwntools is the standard library:

```python
#!/usr/bin/env python3
import time, random
from pwn import remote, context
context.log_level = "error"

ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
def token_for(seed_ms):
    random.seed(seed_ms)
    return "".join(random.choice(ALPHABET) for _ in range(20))

centre = int(time.time() * 1000)
cands = [(centre + d, token_for(centre + d)) for d in range(-50, 101)]

io = remote("verbal-sleep.picoctf.net", 52304)
io.recvuntil(b"Enter your guess"); io.recvline()

for seed, tok in cands:
    io.sendline(tok.encode())
    line = io.recvline().decode().strip()
    if "Congratulations" in line:
        print("seed:", seed, "token:", tok)
        print(io.recvall(timeout=2).decode())
        break
```

```
┌──(zham㉿kali)-[~/picoctf/chronohack]
└─$ python3 exploit_pwn.py
[+] opening connection to verbal-sleep.picoctf.net on port 52304
seed: 1782225290041 token: jdQ9Il0yACc44VbaXcMM
Congratulations! You found the correct token.
picoCTF{UseSecure#$_Random@j3n3r@T0rs13909228}
```

---

## What Happened Internally (Timeline)

1. I called `socket.connect((HOST, PORT))`. On the server, the wrapper (typically `socat` or `xinetd`) accepted the TCP connection and `exec`ed the Python script.
2. The script ran `import random` and `import time` (negligible), then jumped into `main()`.
3. `main()` printed the welcome banner and the prompt. Each `print` flushed a few bytes to my socket. By the time I received the banner, the server had only spent a handful of milliseconds since my `connect()` call returned.
4. The very next line was `token = get_random(20)`, which executed `random.seed(int(time.time() * 1000))`. That single line froze the entire future `random.choice` stream. From here on, every possible token is determined by that one integer.
5. My client started a loop. For each offset I generated a token locally by re-running the exact same code with the same seed, then sent it over the socket.
6. The server read each guess, compared it to the frozen `token`, and either replied `Sorry, your token does not match. Try again!` or, on a hit, `Congratulations! You found the correct token.` followed by the flag file contents.
7. On offset `+32`, the locally generated token matched the one the server had already locked in. The server printed the flag and exited the loop, the connection closed, and I had the flag.

The whole attack boils down to: the server turned public information (the wall clock) into a "secret" key, so the secret was never a secret at all.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the leaked Python source |
| `python3` | Reproduce the token locally for any candidate seed |
| `socket` (stdlib) | Talk to the server, one guess per connection |
| `nano` | Write and save the exploit script |
| `pwntools` (`pwn.remote`) | Cleaner alternative for the same exploit |
| `epochconverter.com` (from hints) | Understand what the seed integer actually represents |
| Snyk lesson on insecure randomness (from hints) | Background reading on why this is broken |

---

## Key Takeaways

- **Never use `time.time()`, `os.getpid()`, or any other predictable value as a cryptographic or security-relevant seed.** Anyone who can guess the value can reproduce every "random" number the program will ever produce.
- **Use the operating system's CSPRNG** for anything security-sensitive: `secrets` in Python, `/dev/urandom` from the shell, `crypto.randomBytes` in Node, `java.security.SecureRandom` in Java, etc. These are seeded from entropy sources the attacker cannot observe.
- **The attacker only has to be right once.** Even with 50 guesses per connection, scanning a 100 ms window is trivial because each guess is just a millisecond apart.
- **Reverse engineering is not always assembly.** Sometimes the "reverse" is a one-line logic bug in a 60-line Python script. Read the source carefully and look for the assumptions the author made.
- **Hints 1 and 2 are not red herrings.** The challenge authors essentially told you the attack: convert a time, learn why time-based seeding is broken, time the server, and brute-force a small range. Trust the hints.

### Flag Wordplay Decode

The flag is:

```
picoCTF{UseSecure#$_Random@j3n3r@T0rs13909228}
```

Decoded, it reads:

> **Use Secure Random Generators** + suffix `13909228`

with the usual leet/substitution:

- `UseSecure` — "Use Secure"
- `#$_` — decorative noise between the two halves
- `Random` — "Random"
- `@j3n3r@T0rs` — "Generators" with `@` → `o` and `3` → `e`
- `13909228` — appears to be a unique serial for this flag instance, not part of the wordplay

The lesson is right in the flag: the program should have used a secure random number generator, and the challenge is named "Chronohack" because the bug is in the **chrono**-based seeding.
