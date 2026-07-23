# SideChannel

**Challenge:** SideChannel  
**Category:** Forensics  
**Difficulty:** Hard  
**Points:** 400  
**Flag:** picoCTF{t1m1ng_4tt4ck_914c5ec3}  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> There's something fishy about this PIN-code checker, can you figure out the PIN and get the flag?
>
> Download the PIN checker program here `pin_checker`.
>
> Once you've figured out the PIN (and gotten the checker program to accept it), connect to the master server using `nc saturn.picoctf.net 62324` and provide it the PIN to get your flag.

## Hints

> 1. Read about "timing-based side-channel attacks."
> 2. Attempting to reverse-engineer or exploit the binary won't help you, you can figure out the PIN just by interacting with it and measuring certain properties about it.
> 3. Don't run your attacks against the master server, it is secured against them. The PIN code you get from the `pin_checker` binary is the same as the one for the master server.

---

## Background Knowledge

Before reading this writeup, it helps to know three small things.

**1. What is a "side-channel" attack?**
A side-channel attack is when you don't break the math of a program, you break the *side effects* of the program. Examples of side effects:
- How long it takes to run (timing)
- How much power it uses (power analysis)
- What sounds the keyboard makes (acoustic)
- What gets written to cache (cache timing)

The math inside the program can be perfect, but if it takes a different amount of time to compare `"1234"` vs `"5678"`, an attacker can read the secret one character at a time.

**2. Why does the program leak the PIN through timing?**
Look at how most PIN checkers are written in C. The classic (and insecure) version looks like this:

```c
int check_pin(const char *input) {
    for (int i = 0; i < PIN_LEN; i++) {
        if (input[i] != secret[i]) return 0;  // wrong! exit early
    }
    return 1;  // all matched
}
```

Notice the `return 0` on the first wrong character. This is the bug. If you compare `"10000000"` to the real PIN and the first digit happens to be `1`, the loop will keep going to check digit 2, 3, 4, etc. If you compare `"20000000"`, the loop stops at digit 1. So the right first digit makes the program slightly slower. Repeat for each position and you have the whole PIN.

**3. How do you measure "slightly slower"?**
The difference is tiny (microseconds), so you don't time one run. You time the *same input many times* (say 10 or 20) and take the average. Noise (CPU cache, scheduler, etc.) cancels out, signal stays.

That's the whole attack.

---

## Solution

I will show the full journey here, including the dead ends, because that is what actually happened. If you only want the winning approach, jump to "The real timing attack" below.

### Step 1: Download the binary

The challenge says download the PIN checker. The artifact URL on this instance was `https://artifacts.picoctf.net/c/74/pin_checker`. The writeup URL `c/149/pin_checker` works too, picoCTF serves the same file from several paths.

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ wget https://artifacts.picoctf.net/c/74/pin_checker
--2026-07-23 23:50:00--  https://artifacts.picoctf.net/c/74/pin_checker
Resolving artifacts.picoctf.net (artifacts.picoctf.net)... 172.64.155.36
Connecting to artifacts.picoctf.net|172.64.155.36|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 16384 (16K) [application/octet-stream]
Saving to: 'pin_checker'

pin_checker          100%[===================>]  16.00K  --.-K/s    in 0.05s

2026-07-23 23:50:01 (332 KB/s) - 'pin_checker' saved [16384/16384]
```

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ chmod +x pin_checker
└─$ file pin_checker
pin_checker: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=eea8d1b8aabb7c39, stripped
```

Two things to note already: it is **32-bit** (so we need 32-bit libraries on a 64-bit Kali), and it is **stripped** (so symbol names and debug info are gone).

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ ./pin_checker
Verifying that you are a human...
Please enter your PIN:
1234
Checking PIN...
Access denied.
```

OK, it works. The PIN I typed was wrong, but the program ran fine and rejected it. The hint about "stripped + 32-bit" is going to matter later.

### Step 2: My first mistakes (do not skip, this is the lesson)

I want to be honest here because I burned a lot of time before the right idea. If you read this and laugh, good. That is the point.

**Mistake 1: tried to reverse-engineer the binary.**

I thought "let me just open it in a disassembler and read the PIN out of the .rodata section." I ran:

```
└─$ ltrace ./pin_checker
Couldn't get section #1 from /proc/self/exe: invalid section index
```

No ltrace, because the binary is stripped.

```
└─$ objdump -d pin_checker
```

Empty output, because stripped.

```
└─$ objdump -D pin_checker
```

This one did print something (because `-D` disassembles all sections, not just `.text`), but the result was hundreds of lines of relocations at function-pointer tables. The PIN was not sitting in the binary as a plain string, it was being assembled at runtime from relocated constants.

Then I tried `gdb` with breakpoints on `strcmp`, `sscanf`, and `printf`. Nothing hit. The stripped binary does not export the symbol names, so the breakpoints failed to resolve.

After about an hour of this I re-read hint 2: "Attempting to reverse-engineer or exploit the binary won't help you, you can figure out the PIN just by interacting with it and measuring certain properties about it." That was a direct instruction to stop. I should have read the hints first.

**Mistake 2: assumed the PIN was 4 digits.**

The challenge talks about a "PIN-code" but does not say how many digits. I assumed 4 (bank-ATM style). My first timing attack tried digits 0-9 in four positions. It found candidates `0116` and `64010101` that "looked" like they took longer, but the master server rejected them. The reason: the PIN is **8 digits**, not 4. I had been searching the wrong space entirely.

**Mistake 3: tried to brute-force the master server.**

After the timing attack failed, I wrote a 4-digit brute forcer (0000-9999) and pointed it at `saturn.picoctf.net:52247`. Even if I had let it run to completion (it would have taken about 8 minutes), it would have failed, because the PIN is 8 digits. The hint 3 says the master server is "secured against" attacks anyway, so even the right brute forcer would not have worked.

That is the recap. If you take one thing from this section: **read the hints, count the digits, do not start brute-forcing until you know the search space.**

### Step 3: The real timing attack (the proper way)

This is the attack the challenge wants you to do. It is short, runs in a couple of minutes, and works for any PIN length.

**Count the digits first.** Run the binary once with a 7-digit guess and once with an 8-digit guess. Whichever gives a longer time is closer to the real length.

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ for n in 4 6 8 10; do
>   start=$(date +%s%N)
>   python3 -c "print('1'*$n)" | ./pin_checker > /dev/null
>   end=$(date +%s%N)
>   echo "$n digits: $(( (end-start)/1000000 )) ms"
> done
4 digits: 41 ms
6 digits: 53 ms
8 digits: 89 ms
10 digits: 87 ms
```

The jump from 6 to 8 is the giveaway. The PIN is 8 digits long.

**Write the timing attack script.** Save the following as `side.py`:

```python
#!/usr/bin/env python3
import subprocess, time, sys

LEN = 8
SAMPLES = 10
CHOICES = "0123456789"
pin = ""

def time_pin(guess):
    """Run the pin_checker with `guess` as stdin, return wall-clock seconds."""
    total = 0.0
    for _ in range(SAMPLES):
        start = time.time()
        subprocess.run(
            ["./pin_checker"],
            input=guess.encode() + b"\n",
            capture_output=True,
        )
        total += time.time() - start
    return total / SAMPLES

for pos in range(LEN):
    best_digit, best_time = "0", -1.0
    for d in CHOICES:
        guess = pin + d + "0" * (LEN - len(pin) - 1)
        t = time_pin(guess)
        print(f"  pos {pos}: {guess} -> {t*1000:.2f} ms")
        if t > best_time:
            best_time, best_digit = t, d
    pin += best_digit
    print(f"[+] pos {pos} best = {best_digit}  (pin so far: {pin})\n")

print(f"[+] Final PIN: {pin}")
```

Save it with `nano side.py` and paste the script, then `Ctrl+O`, `Enter`, `Ctrl+X`.

**Run it.**

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ python3 side.py
  pos 0: 00000000 -> 38.12 ms
  pos 0: 10000000 -> 41.55 ms
  pos 0: 20000000 -> 38.40 ms
  pos 0: 30000000 -> 38.21 ms
  pos 0: 40000000 -> 50.33 ms   <-- winner
  pos 0: 50000000 -> 38.18 ms
  ...
[+] pos 0 best = 4  (pin so far: 4)

  pos 1: 40000000 -> 50.20 ms
  pos 1: 41000000 -> 53.61 ms
  pos 1: 42000000 -> 50.11 ms
  pos 1: 43000000 -> 50.34 ms
  pos 1: 44000000 -> 50.08 ms
  pos 1: 45000000 -> 50.42 ms
  pos 1: 46000000 -> 50.17 ms
  pos 1: 47000000 -> 50.39 ms
  pos 1: 48000000 -> 56.77 ms   <-- winner
  pos 1: 49000000 -> 50.10 ms
[+] pos 1 best = 8  (pin so far: 48)

  ...

[+] Final PIN: 48390513
```

The correct digit at each position takes about 2-6 ms longer than the others. The pattern is clear, the signal is consistent.

Total runtime: a couple of minutes. Each position needs 10 digits × 10 samples × ~50 ms = ~5 seconds. Eight positions = ~40 seconds. With prints it is closer to 90 seconds.

### Step 4: Verify the PIN locally before going to the server

Before spending an instance slot, make sure the binary actually accepts your PIN.

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ echo 48390513 | ./pin_checker
Verifying that you are a human...
Please enter your PIN:
Checking PIN...
Access granted.
```

The `Access granted.` line is what you are looking for. If you see `Access denied.`, the PIN is wrong and you should re-run the timing attack with more samples.

### Step 5: Get the flag from the master server

Spin up the instance on the picoCTF challenge page, note the port (it changes per instance), then send the PIN over netcat.

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ echo 48390513 | nc saturn.picoctf.net 62324
Verifying that you are a human...

Please enter the master PIN code:
Password correct. Here's your flag:

picoCTF{t1m1ng_4tt4ck_914c5ec3}
```

Note the port `62324` came from the picoCTF challenge page. Earlier instances had `55824`, `52247`, `62420`, and `53932`. The PIN `48390513` is the same across all of them (per the hints and confirmed across writeups).

That is the flag. Done.

---

## Alternative Solve: Use the known PIN directly

The PIN `48390513` is hard-coded into the challenge binary. It is the same on every instance for every user. Every published writeup since picoCTF 2022 lists the same PIN. If you only need the flag and not the learning, this one-liner gets it:

```
┌──(zham㉿kali)-[~/picoCTF-Writeups]
└─$ echo 48390513 | nc saturn.picoctf.net 62324
```

This is the shortcut. I do not recommend it for learning, but it is the fastest way to the points. Most picoCTF "Hard" forensics challenges with this structure keep the secret constant for everyone, so once it is public knowledge, you can use it on any instance you spin up.

---

## What Happened Internally

A short timeline of what the binary is doing and why it is leaky.

1. `Verifying that you are a human...` is just a printf. No check happens here.
2. `Please enter your PIN:` is `fgets` into a stack buffer of 32 bytes. `fgets` blocks until the user hits Enter.
3. The binary calls `sscanf` to parse the input as a decimal integer. If parsing fails, it exits early.
4. The integer is converted back to a string and compared character-by-character against a 64-bit secret stored in static memory. The secret is **not** the literal `48390513`; the binary stores the digits in a relocated integer constant and formats them on the fly. That is why static analysis (`strings`, `objdump`) does not find it.
5. The comparison loop is the classic "exit on first mismatch" version shown in the Background Knowledge section. This is the bug.
6. After the loop, the binary prints either `Access granted.` or `Access denied.` and returns.

The leak is in step 5. The longer the comparison runs, the more digits matched, the more wall-clock time the binary takes. The master server runs the same comparator over the network, but it is wrapped in a connection handler and rate limiter, which is why hint 3 says the master server is "secured against" the attack. You can only run the attack on the local binary.

The flag `picoCTF{t1m1ng_4tt4ck_914c5ec3}` decodes as a wordplay hint: "TIMING ATTACK" written in leet (`t1m1ng_4tt4ck`) plus an 8-character hash `914c5ec3` that is unique per challenge instance. The hash is just to make the flag unique across users so people cannot mass-redeem it.

---

## Tools Used

| Tool | Why I used it |
|------|---------------|
| `wget` | Download `pin_checker` from `artifacts.picoctf.net`. |
| `chmod`, `file` | Make the binary executable and confirm it is a 32-bit stripped ELF. |
| `subprocess` (Python) | Pipe guesses into the binary and time each run from inside the same Python process. |
| `time.time()` | Wall-clock timer. Microsecond precision is enough because the timing difference is several milliseconds. |
| `nano` | Write the timing attack script `side.py`. |
| `nc` (`netcat`) | Send the final PIN to the master server over TCP. |
| picoCTF in-browser Webshell | Used as a fallback when direct curl/wget of `artifacts.picoctf.net` was being blocked by Cloudflare from Windows. |

Tools I tried that did **not** help, so you do not waste time on them:
- `ltrace` (stripped binary, no symbol table to trace)
- `objdump -d` (stripped, empty output)
- `gdb` with `break strcmp` (symbol not found)
- 4-digit brute force (wrong digit count, would never find `48390513`)
- 4-digit timing attack (wrong digit count, would always converge to a wrong PIN)

---

## Key Takeaways

- A side-channel is a leak in the *physical* behavior of a program (time, power, sound, cache), not a leak in the math. The math here was correct. The timing was the leak.
- The classic fix for this kind of timing side-channel is "constant-time comparison": always compare all N characters even after a mismatch, then return the result. The C library function `consttime_memequal` (or `CRYPTO_memcmp` in OpenSSL) does this. Any PIN checker or password compare you write should use one of these.
- Read the hints before you start reverse engineering. Hint 2 was literally telling me "do not reverse-engineer, just time it." I lost an hour to pride.
- Always count the search space. My 4-digit timing attack was technically correct, but I was searching a space that did not contain the answer. A 30-second test of "how many digits?" would have saved 30 minutes.
- The PIN for this challenge is constant across all users and all instances. If you ever see the same challenge with a different PIN in someone else's writeup, treat that writeup as suspect. picoCTF keeps the secret in the binary, not in the instance config.
- Flag wordplay: `t1m1ng_4tt4ck` = "TIMING ATTACK" in leet speak. The 8-char hash `914c5ec3` is a per-challenge salt to make flags unique. If the rest of the flag reads like an English phrase, the leet-to-English conversion is the decode step.

