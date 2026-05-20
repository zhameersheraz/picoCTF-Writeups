# Bytemancy 2 — picoCTF Writeup

**Challenge:** Bytemancy 2  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{3ff5_4_d4yz_99f7d4fe}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> Can you conjure the right bytes? The program's source code can be downloaded here.
> Connect to the program with netcat:
> `$ nc lonely-island.picoctf.net 53779`

**Hint 1:** `There's no way to print these bytes`  
**Hint 2:** `Use pwntools to send raw bytes over the network`

**Downloads:** `app.py`

---

## Background Knowledge (Read This First!)

### What does the program actually want?

When you connect, the server prompts:

```
Send me the HEX BYTE 0xFF 3 times, side-by-side, no space.
==> 
```

And the relevant check in `app.py` is:

```python
user_input = sys.stdin.buffer.readline().rstrip(b"\n")
if user_input == b"\xff\xff\xff":
    print(open("./flag.txt", "r").read())
```

It reads raw bytes from stdin and compares them directly against `b"\xff\xff\xff"` — three bytes each with the decimal value **255**.

### Why can't you just type it?

Your keyboard only produces ASCII characters. If you open a terminal and type `0xFF`, you are sending:

| What you type | Bytes actually sent |
|---|---|
| `0` | `0x30` |
| `x` | `0x78` |
| `F` | `0x46` |
| `F` | `0x46` |

That is four ASCII bytes — `0x30 0x78 0x46 0x46` — which is completely different from the single raw byte `0xFF` (decimal 255). There is no key on your keyboard that produces `0xFF` directly.

The hint confirms this: *"There's no way to print these bytes"* — meaning you cannot type them manually.

### What is pwntools?

**pwntools** is a Python library built for CTF challenges and binary exploitation. Among many other things, it lets you open a TCP connection to a remote server and send arbitrary raw bytes — including bytes like `0xFF` that are not typeable in a normal terminal.

```python
from pwn import *

conn = remote("host", port)  # opens TCP connection
conn.send(b"\xff\xff\xff\n") # sends exact raw bytes
```

---

## Reading the Source Code

```python
user_input = sys.stdin.buffer.readline().rstrip(b"\n")
if user_input == b"\xff\xff\xff":
    print(open("./flag.txt", "r").read())
    break
else:
    print("That wasn't it. I got: " + str(user_input))
```

Key observations:
- Uses `sys.stdin.buffer` — reads **raw bytes**, not decoded text
- Strips the trailing newline with `.rstrip(b"\n")` before comparing
- Requires exactly `b"\xff\xff\xff"` — three raw `0xFF` bytes, nothing else
- Prints the flag and exits on a correct match; loops and shows what it received otherwise

The `else` branch is useful for debugging — if you send the wrong thing, the server tells you exactly what bytes it received.

---

## Failed Attempt — Connecting with netcat

```
┌──(zham㉿kali)-[~]
└─$ nc lonely-island.picoctf.net 53779
⊹──────[ BYTEMANCY-2 ]──────⊹
☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐
Send me the HEX BYTE 0xFF 3 times, side-by-side, no space.
☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐
⊹─────────────⟡─────────────⊹
==> ^?^?^C
```

The `^?^?^C` shows attempts to backspace and eventually ctrl+c to quit. There is no way to input `0xFF` through a standard terminal — netcat alone cannot solve this.

---

## Solution 1 — pwntools (Intended Method)

### Step 1 — Create the script with nano

```
┌──(zham㉿kali)-[~]
└─$ nano solve.py
```

Inside nano, type the following script:

```python
from pwn import *

conn = remote("lonely-island.picoctf.net", 53779)

# Read up to the prompt so the buffer is clear
conn.recvuntil(b"==> ")

# Send three raw 0xFF bytes followed by a newline
conn.send(b"\xff\xff\xff\n")

# Print the server's response (the flag)
print(conn.recvline())
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

### Step 2 — Run the script

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
[+] Opening connection to lonely-island.picoctf.net on port 53779: Done
b'picoCTF{3ff5_4_d4yz_99f7d4fe}\n'
[*] Closed connection to lonely-island.picoctf.net port 53779
```

Flag retrieved on the first try.

---

## Solution 2 — printf + netcat (No Python Needed)

This is actually the fastest method — no script file needed at all. `printf` in bash can produce raw bytes using `\xNN` escape sequences, and piping it directly into netcat sends those bytes over the network.

```
┌──(zham㉿kali)-[~]
└─$ printf '\xff\xff\xff' | nc lonely-island.picoctf.net 53779
⊹──────[ BYTEMANCY-2 ]──────⊹
☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐

Send me the HEX BYTE 0xFF 3 times, side-by-side, no space.

☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐
⊹─────────────⟡─────────────⊹
==> picoCTF{3ff5_4_d4yz_99f7d4fe}
```

One line. No install required. `printf '\xff'` produces the actual byte `0xFF` — not the text — so three of them piped into nc sends exactly what the server expects.

---

## Solution 3 — Python socket (No pwntools Install)

If pwntools is not installed and you prefer Python over bash, the standard library `socket` module works just as well:

```
┌──(zham㉿kali)-[~]
└─$ nano solve_socket.py
```

```python
import socket

s = socket.socket()
s.connect(("lonely-island.picoctf.net", 53779))
s.recv(1024)                   # drain the banner
s.send(b"\xff\xff\xff\n")      # send raw bytes
print(s.recv(1024).decode())   # print the flag
s.close()
```

```
┌──(zham㉿kali)-[~]
└─$ python3 solve_socket.py
picoCTF{3ff5_4_d4yz_99f7d4fe}
```

---

## What Happened Internally

```
Timeline:
1. pwntools / printf opens a TCP connection to the server
2. Server prints the banner and "==> " prompt
3. recvuntil(b"==> ") drains everything up to the prompt
4. send(b"\xff\xff\xff\n") sends 4 bytes: 0xFF, 0xFF, 0xFF, 0x0A
5. Server reads with sys.stdin.buffer.readline(), strips the 0x0A newline
6. Remaining input b"\xff\xff\xff" matches the condition exactly
7. Server opens flag.txt and prints its contents
8. recvline() reads the flag back to us
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reading source code | Understand the exact byte comparison | Easy |
| netcat | Confirming the prompt (and failing to solve by hand) | Easy |
| `nano` | Writing the solve script | Easy |
| Python `pwntools` | Send raw non-typeable bytes over TCP | Easy |
| `printf \| nc` | One-liner bash alternative, no install needed | Easy |
| Python `socket` | Alternative if pwntools is unavailable | Easy |

---

## Key Takeaways

- **Raw bytes and ASCII text are not the same thing** — `0xFF` as characters is four ASCII bytes; `0xFF` as a raw byte is a single byte with value 255
- **`sys.stdin.buffer` reads raw bytes** — when a program uses this instead of `sys.stdin`, it is working at the byte level and text input will fail
- **`printf '\xNN' | nc` is often the fastest solve** for simple raw byte challenges — no script, no install, one line
- **pwntools is the standard for more complex challenges** — when you need conditional logic, loops, or multi-step interactions, a full script is cleaner than a bash pipe
- **CyberChef cannot send network traffic** — it is a data transformation tool, not a network tool
- The flag `3ff5_4_d4yz` is "0xFF's 4 days" — a nod to the `0xFF` byte at the heart of the challenge
