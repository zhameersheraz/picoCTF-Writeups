# bytemancy 1 — picoCTF Writeup

**Challenge:** bytemancy 1  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{h0w_m4ny_e's???_7dbc095c}`  

---

## Description

> Can you conjure the right bytes? The program's source code can be downloaded [here].
>
> Connect to the program with netcat:
> `$ nc foggy-cliff.picoctf.net 50368`

**Hint 1:** `No copy-pasta, please - use Python!`

**Tags:** `netcat`, `python`, `ascii`, `socket`

---

## Background Knowledge (Read This First!)

### What is ASCII?

**ASCII (American Standard Code for Information Interchange)** is a character encoding standard that assigns a number to each character. For example:

- `A` = 65
- `a` = 97
- `e` = **101** ← the one we care about here

When the challenge says "Send me ASCII DECIMAL 101", it's asking you to send the **character** whose ASCII code is 101 — which is just the letter **`e`**.

In Python, you can convert between characters and their ASCII codes using:
```python
chr(101)   # returns 'e'
ord('e')   # returns 101
```

And in hex: `0x65` = 101 in decimal = `'e'` in ASCII. All three mean exactly the same thing.

### What is `\x65` in Python?

In Python strings, `\x65` is a **hex escape sequence** meaning "the character with hex value 65" — which is decimal 101, which is `'e'`. So:

```python
"\x65" == "e"  # True
```

This is how the source code `app.py` stores the expected answer — it doesn't say `"e"` directly, it uses `"\x65"` to be slightly obscure. But they're identical.

### What is `netcat` (`nc`)?

`nc` (netcat) is a tool that opens a raw TCP connection to a remote host and port, letting you send and receive text. Think of it like a very barebones browser — you connect to a server and type messages back and forth.

```bash
nc foggy-cliff.picoctf.net 50368
```

This opens a connection to the challenge server. But as the hint says — **no copy-pasta!** Manually pasting 1751 `e`'s won't work reliably, so we need a Python script.

### What is a socket in Python?

A **socket** is how Python opens a network connection. It's the programmable equivalent of running `nc`. You connect to a host and port, then send and receive raw bytes:

```python
import socket
s = socket.socket()
s.connect(('foggy-cliff.picoctf.net', 50368))
s.send(b'e' * 1751 + b'\n')
```

This lets us automate the whole interaction — connect, receive the prompt, send our answer, and get the flag back.

### What is `exiftool`?

`exiftool` reads file metadata — things like file type, size, encoding, and line count. It's not needed to solve the challenge, but it's good habit to run it on any downloaded file to understand what you're working with before opening it.

---

## Solution — Step by Step

### Step 1 — Navigate to the downloads folder

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

We move to the folder where `app.py` was downloaded.

### Step 2 — Inspect the source code with `exiftool`

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool app.py
ExifTool Version Number         : 13.50
File Name                       : app.py
Directory                       : .
File Size                       : 771 bytes
File Modification Date/Time     : 2026:04:17 13:32:03-04:00
File Access Date/Time           : 2026:04:17 13:32:03-04:00
File Inode Change Date/Time     : 2026:04:17 13:32:03-04:00
File Permissions                : -rwxrwx---
File Type                       : TXT
File Type Extension             : txt
MIME Type                       : text/plain
MIME Encoding                   : utf-8
Byte Order Mark                 : No
Newlines                        : Unix LF
Line Count                      : 21
Word Count                      : 49
```

Confirms it's a plain UTF-8 Python script — safe to open and read.

### Step 3 — Read and understand `app.py`

The source code the challenge provides (`app.py`) reveals exactly what the server expects:

```python
while(True):
  try:
    print('⊹──────[ BYTEMANCY-1 ]──────⊹')
    print("☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐")
    print()
    print('Send me ASCII DECIMAL 101 1751 times, side-by-side, no space.')
    print()
    print("☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐")
    print('⊹─────────────⟡─────────────⊹')
    user_input = input('==> ')
    if user_input == "\x65"*1751:
      print(open("./flag.txt", "r").read())
      break
    else:
      print("That wasn't it. I got: " + str(user_input))
  except Exception as e:
    print(e)
    break
```

Key line:
```python
if user_input == "\x65"*1751:
```

The server checks if your input is exactly `"\x65"` (i.e., `"e"`) repeated **1751 times** with no spaces. So the expected string is:

```
eeeeeeeeeeeeeeeeeee...  (1751 e's total)
```

### Step 4 — Probe the server with `nc` (and fail)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nc foggy-cliff.picoctf.net 50368
⊹──────[ BYTEMANCY-1 ]──────⊹
☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐

Send me ASCII DECIMAL 101 1751 times, side-by-side, no space.

☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐
⊹─────────────⟡─────────────⊹
==> ^C
```

We see the prompt but Ctrl+C out immediately — there's no way to reliably type or paste 1751 `e`'s by hand. Time to write a script.

### Step 5 — Write `solve.py` with `nano`

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano solve.py
```

Inside nano, type the following script:

```python
import socket

s = socket.socket()
s.connect(('foggy-cliff.picoctf.net', 50368))

# Receive and print the challenge banner
print(s.recv(4096).decode())

# Send 'e' (ASCII 101) exactly 1751 times, followed by a newline
s.send(b'\x65' * 1751 + b'\n')

# Receive and print the flag
print(s.recv(4096).decode())

s.close()
```

Save the file:
- Press **Ctrl+O** → confirm the filename → press **Enter**
- Press **Ctrl+X** to exit nano

Breaking down `solve.py`:
- `socket.socket()` — creates a TCP socket
- `s.connect(...)` — connects to the challenge server on port 50368
- `s.recv(4096).decode()` — receives up to 4096 bytes (the banner) and decodes it to text
- `b'\x65' * 1751 + b'\n'` — builds the payload: byte `0x65` (`e`) × 1751, plus a newline so the server processes the input
- Final `s.recv(4096)` — receives the server's response (the flag)

### Step 6 — Run the script and get the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 solve.py
⊹──────[ BYTEMANCY-1 ]──────⊹
☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐

Send me ASCII DECIMAL 101 1751 times, side-by-side, no space.

☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐☉⟊☽☈⟁⧋⟡☍⟐
⊹─────────────⟡─────────────⊹
==> 
picoCTF{h0w_m4ny_e's???_7dbc095c}
```

✅ Flag captured! 🎯

> **Fun detail:** The flag says `h0w_m4ny_e's???` — a cheeky nod to the challenge itself. The answer is 1751 e's, in case you were wondering.

---

## Alternative Methods

### Alternative 1 — One-liner with Python `-c`

You can skip writing a file entirely and run a one-liner straight from the terminal:

```bash
python3 -c "
import socket
s = socket.socket()
s.connect(('foggy-cliff.picoctf.net', 50368))
print(s.recv(4096).decode())
s.send(b'e'*1751+b'\n')
print(s.recv(4096).decode())
"
```

Quick and dirty — great for simple interactions like this one.

### Alternative 2 — Using `pwntools`

[pwntools](https://docs.pwntools.com/) is a popular CTF library that makes socket interactions even easier:

```python
from pwn import *

r = remote('foggy-cliff.picoctf.net', 50368)
r.recvuntil(b'==> ')
r.sendline(b'e' * 1751)
print(r.recvline().decode())
```

`recvuntil` waits for the prompt before sending, which is more robust than a plain `recv` with a fixed byte count.

### Alternative 3 — Bash here-string (works but fragile)

In theory, you could pipe the string directly:

```bash
python3 -c "print('e'*1751)" | nc foggy-cliff.picoctf.net 50368
```

This might not work reliably because `nc` can close the connection before the server processes the input. The Python socket approach is cleaner and more dependable.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `exiftool` | Inspect the downloaded source file's metadata | ⭐ Easy |
| `nc` | Probe the server to understand the challenge prompt | ⭐ Easy |
| `nano` | Write the solve script | ⭐ Easy |
| `python3` + `socket` | Automate the connection and send the correct payload | ⭐ Easy |

---

## Key Takeaways

- **Always read the source code** — `app.py` told us exactly what the server expected. The answer was right there: `"\x65" * 1751`
- **`\x65` is just `e`** — hex escape sequences look cryptic but they're just another way to write the same character. ASCII 101 = 0x65 = `'e'`
- **The hint says it all** — "No copy-pasta, use Python!" is a direct nudge toward the socket script approach
- **`socket` is your best friend** for simple netcat-style challenges when you need to automate input — it gives you full control over what you send and receive
- **Byte strings matter** — remember to use `b'\x65'` (bytes) not `'\x65'` (str) when sending over a socket, and append `b'\n'` so the server registers your input
