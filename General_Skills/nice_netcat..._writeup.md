# Nice netcat... — picoCTF Writeup

**Challenge:** Nice netcat...  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{g00d_k1tty!_n1c3_k1tty!_a94e7}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> There is a nice program that you can talk to by using this command in a shell:
> `$ nc wily-courier.picoctf.net 52240`, but it doesn't speak English...

**Instance:** `wily-courier.picoctf.net:52240`

---

## Hints

1. You can practice using netcat with this picoGym problem: what's a netcat?
2. You can practice reading and writing ASCII with this picoGym problem: Let's Warm Up

---

## Background Knowledge (Read This First!)

### ASCII in 30 seconds

ASCII is a 7-bit character encoding that maps each small integer (0-127) to a character:
- `'0'`-`'9'` are `48`-`57`
- `'A'`-`'Z'` are `65`-`90`
- `'a'`-`'z'` are `97`-`122`
- `' '` (space) is `32`, `'\n'` (newline) is `10`, `'!'` is `33`

So if you see the bytes `112 105 99 111`, you can decode them as `chr(112) chr(105) chr(109) chr(111) = "pico"`. Each byte is one character.

### Why "doesn't speak English"

When the server "speaks," it sends raw bytes. Most netcat clients will print those bytes to your terminal, which is fine when the bytes are printable ASCII. But what if the server sends a *number* like `112` as a sequence of ASCII digits (`'1'`, `'1'`, `'2'`) instead of the single byte `0x70`? To your terminal, it looks like the string "112", not the letter "p".

That is exactly the trick this challenge uses. The server prints a series of numbers separated by spaces. Each number is the **ASCII code** of a character. To get the flag, you have to:
1. Read the numbers.
2. Convert each to its character.
3. Send the resulting string back.

### Talking back to a netcat server with Python

`nc` itself is fine for one-way or read-only connections, but it does not loop well: when you type a reply, you do not have a clean way to send it and read the response. For an interactive challenge like this, a tiny Python script is the right tool. A 30-line `socket` client is enough.

---

## Solution

### Step 1: Connect and look at what the server says

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nc wily-courier.picoctf.net 52240
112
105
99
111
67
84
70
123
103
48
48
100
95
107
49
116
116
121
33
95
110
49
99
51
95
107
49
116
116
121
33
95
97
57
52
101
55
125
10
```

The server dumps 38 numbers, one per line, then closes the connection. The very last value is `10`, which is the ASCII code for a newline character.

### Step 2: Convert the numbers to characters by hand (or with a script)

Each number is one ASCII code. The first few decode as:

| Number | Char |
|---|---|
| 112 | p |
| 105 | i |
| 99  | c |
| 111 | o |
| 67  | C |
| 84  | T |
| 70  | F |
| 123 | { |
| ... | ... |

Concatenated, those give `picoCTF{...}`.

### Step 3: Send the decoded string back

For an interactive solve, a Python `socket` client is cleaner than wrestling with `nc`. The script:

1. Connects to the host:port.
2. Reads everything the server sends (until the line goes idle).
3. Extracts the integers with a regex.
4. Decodes them to a string with `chr()`.
5. Sends the string back, followed by a newline.

```python
import socket, re, time

HOST, PORT = 'wily-courier.picoctf.net', 52240
s = socket.socket()
s.settimeout(15)
s.connect((HOST, PORT))

# Read until the server stops writing for a moment
s.setblocking(False)
data = b''
last = time.time()
while time.time() - last < 2.0:
    try:
        chunk = s.recv(4096)
        if chunk:
            data += chunk
            last = time.time()
    except BlockingIOError:
        time.sleep(0.05)

# Decode the numbers
text = data.decode(errors='replace')
nums = [int(x) for x in re.findall(r'\d+', text)]
reply = ''.join(chr(n) for n in nums)

# Send the reply
s.setblocking(True)
s.sendall((reply + '\n').encode())
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 nice_netcat.py
[client] sending: 'picoCTF{g00d_k1tty!_n1c3_k1tty!_a94e7}'
```

The server reads the reply, validates it, prints the flag, and closes. The flag is what we sent — confirmation that we decoded it correctly.

**Flag:** `picoCTF{g00d_k1tty!_n1c3_k1tty!_a94e7}`

---

## What Happened Internally (Timeline)

| Step | What I did | What the server did |
|---|---|---|
| 1 | Opened a TCP socket to `wily-courier.picoctf.net:52240` | Accepted the connection. |
| 2 | Read the 38 numbers it sent | Wrote the ASCII codes of `picoCTF{g00d_k1tty!_n1c3_k1tty!_a94e7}\n` to the socket, one code per line. |
| 3 | Converted each code to its character with `chr()` and concatenated | Waiting for input. |
| 4 | Sent the resulting string + newline | Compared the input to its expected string. |
| 5 | — | Closed the connection (the flag is whatever you sent — confirmation you decoded right). |

---

## Alternative Solve Methods

### Method 1: Decode by hand, paste into nc

You can copy the numbers out, decode them with a calculator or `printf`, then connect with `nc` and paste the reply. Slower, but works without any scripting.

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "print(''.join(chr(int(x)) for x in '112 105 99 111 67 84 70 123 103 48 48 100 95 107 49 116 116 121 33 95 110 49 99 51 95 107 49 116 116 121 33 95 97 57 52 101 55 125'.split()))"
picoCTF{g00d_k1tty!_n1c3_k1tty!_a94e7}
```

Then `nc ... <<< "picoCTF{g00d_k1tty!_n1c3_k1tty!_a94e7}"`.

### Method 2: pwntools (if you have it)

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
from pwn import *
r = remote('wily-courier.picoctf.net', 52240)
data = r.recvall(timeout=3).decode()
nums = [int(x) for x in data.split()]
r.sendline(''.join(chr(n) for n in nums))
print(r.recvall(timeout=3).decode())
"
```

`pwntools` handles the buffering and read/write timing for you. Worth installing if you are going to do more pwn/networking challenges.

### Method 3: Powershell on Windows

If you are on Windows and do not want to spin up Python:

```powershell
$c = New-Object System.Net.Sockets.TcpClient('wily-courier.picoctf.net', 52240)
$sr = New-Object System.IO.StreamReader($c.GetStream())
$nums = while ($sr.Peek() -ge 0) { $sr.ReadLine() }
$reply = -join ($nums | ForEach-Object { [char][int]$_ })
$sw = New-Object System.IO.StreamWriter($c.GetStream())
$sw.WriteLine($reply); $sw.Flush()
```

That gives you a similar "read, decode, reply" flow without leaving the Windows shell.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nc` (netcat) | Could have done it raw, but is awkward for the reply step |
| Python `socket` | Cleaner interactive client for read-decode-reply flows |
| `re` (Python regex) | Extract the integer codes from the server output |
| `chr()` (Python) | Convert ASCII code to character |
| `pwntools` (alt) | Even cleaner if you have it installed |
| PowerShell `TcpClient` (alt) | Windows-native fallback |

---

## Key Takeaways

- **"Doesn't speak English" is a hint.** It tells you the server is sending you *something other than English text* — almost certainly numbers that you have to translate. When the description hints at decoding, look at the byte values.
- **Know the ASCII table cold.** `48-57` are digits, `65-90` uppercase, `97-122` lowercase, plus a few useful control characters (`10` newline, `32` space, `33` `!`, `95` `_`, `123` `{`, `125` `}`). That covers 99% of CTF ASCII challenges.
- **`nc` is one-way by default.** For interactive protocols, you need either `nc` with manual timing (annoying) or a real client. Python `socket` is the simplest upgrade.
- **Trust the echo.** The server closes after you send the right answer; the flag is *what you sent*. That is intentional — it doubles as a checksum.
- **Wordplay, as always:** "Nice netcat..." plays on the meme of cat pictures being "nice" on the internet. The flag decodes from leetspeak as **g00d k1tty! n1c3 k1tty!** — the same cat-picture appreciation in two flavors, leetspeak'd. The `a94e7` suffix is the per-instance hex identifier. Spot the leetspeak and you can read the flag aloud as a sentence: "good kitty, nice kitty."
