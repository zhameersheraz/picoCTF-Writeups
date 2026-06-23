# Quantum Scrambler — picoCTF Writeup

**Challenge:** Quantum Scrambler  
**Category:** Reverse Engineering  
**Difficulty:** Medium   
**Points:** 200   
**Flag:** `picoCTF{python_is_weird0288d140}`  
**Platform:** picoCTF  
**Writeup by:** zham   

---

## Description

> We invented a new cypher that uses "quantum entanglement" to encode the flag. Do you have what it takes to decode it?
>
> Connect to the program with netcat:
>
> ```
> nc verbal-sleep.picoctf.net 51442
> ```
>
> The program's source code can be downloaded [here](https://artifacts.picoctf.net/c/445/quantum_scrambler.py).

## Hints

> 1. Run `eval` on the cypher to interpret it as a python object
> 2. Print the outer list one object per line
> 3. Feed in a known plaintext through the scrambler

---

## Background Knowledge (Read This First!)

### What is a list "cipher"?

This challenge is a small piece of Python that takes a string (the flag), wraps each character in a list as its hex code, then runs a custom `scramble()` function on the list. The result is a deeply nested Python list-of-lists that the server prints with `print(cypher)`. When you `print()` a Python list, you are looking at its `repr`, which is also valid Python source code — meaning the output is itself a Python expression you can `eval` to get the original list back as a real object.

That is exactly what hint 1 means: copy the printed output, wrap it in `eval(...)`, and you have the structure as data, not as text. From there you can walk it programmatically.

### Why the nested lists look scary

The scrambler takes a list `[c0, c1, c2, c3, ...]` (each `cN` is a one-character list) and repeatedly does two things:
- Move the element at `i-1` to the end of the element at `i-2`
- Append a snapshot of the first `i-2` elements to `i-1`

So every character ends up duplicated all over the place, inside history snapshots inside other history snapshots. The "quantum" name is a joke — the flag is still in there, just buried under recursive references.

### Why the hints 2 and 3 work together

Hint 2 says "print the outer list one object per line". If you just `eval` the blob and then print each top-level element, you can see the hex strings clearly, but you still don't know which position each one was originally at. Hint 3 fills in that gap: run the same `scramble()` function on a known plaintext of the same length and you can map "this leaf at this path in the result came from original position N". Then walk the real cypher the same way and read off the original order.

---

## Files Provided

- **`quantum_scrambler.py`** — the Python source for the scrambler (and the server-side program that runs it)

### Source Code

```python
import sys

def exit():
  sys.exit(0)

def scramble(L):
  A = L
  i = 2
  while (i < len(A)):
    A[i-2] += A.pop(i-1)
    A[i-1].append(A[:i-2])
    i += 1
    
  return L

def get_flag():
  flag = open('flag.txt', 'r').read()
  flag = flag.strip()
  hex_flag = []
  for c in flag:
    hex_flag.append([str(hex(ord(c)))])

  return hex_flag

def main():
  flag = get_flag()
  cypher = scramble(flag)
  print(cypher)

if __name__ == '__main__':
  main()
```

Key observations:
- The flag is wrapped character-by-character as `[[hex(ord(c0))], [hex(ord(c1))], ...]`
- `scramble()` mutates the list in place and returns it
- The server just prints the result; we receive the same `print(cypher)` text on the socket

### What the scrambler does, step by step

For input `L = [[a], [b], [c], [d], [e], ...]` (one-element lists):

```
i = 2
while i < len(L):
    L[i-2] += L.pop(i-1)         # pop the list at i-1, extend L[i-2] with its contents
    L[i-1].append(L[:i-2])       # snapshot the first i-2 elements, append to L[i-1]
    i += 1
```

Each iteration:
- The character at original position `i-1` is moved into the list at `i-2`
- A reference to the first `i-2` lists is then glued onto `L[i-1]`

So the original list shrinks by one element per iteration, but each surviving element grows bigger and more nested.

### Tracing a 5-character example

For `L = [[a], [b], [c], [d], [e]]`:

- i=2: `L[0] += L.pop(1)` → `L = [[a,b], [c], [d], [e]]`. Then `L[1].append(L[:0])` → `L = [[a,b], [c, []], [d], [e]]`. i=3.
- i=3: `L[1] += L.pop(2)` → `L = [[a,b], [c,[],d], [e]]`. Then `L[2].append(L[:1])` → `L = [[a,b], [c,[],d], [e, [[a,b]]]]`. i=4.
- i=4: `4 < 3` is false → stop.

Final: `[[a,b], [c,[],d], [e, [[a,b]]]]`. Original positions 0..4 are still in there:
- `a` is at result[0][0]
- `b` is at result[0][1]
- `c` is at result[1][0]
- `d` is at result[1][2]
- `e` is at result[2][0]

The general rule: the original character at position `k` is still reachable in the result, at a position that depends on how many iterations ran.

---

## Solution — Step by Step

### Step 1 — Pull the cypher text from the server

I just opened a raw TCP socket, read everything the server sent, and saved it. Python's `print(cypher)` dumps a valid Python expression, so I can `eval` it back into a list.

```
┌──(zham㉿kali)-[~/ctf/quantum-scrambler]
└─$ nc verbal-sleep.picoctf.net 51442 > cypher.txt
^C
```

The file starts and ends like this (heavily truncated in the middle):

```
[['0x70', '0x69'], ['0x63', [], '0x6f'], ['0x43', [['0x70', '0x69']], '0x54'], ... , ['0x7d']]
```

Hint 1 says to `eval` it. Let me do that and look at the top-level structure.

### Step 2 — Eval the cypher and check the outer list

```
┌──(zham㉿kali)-[~/ctf/quantum-scrambler]
└─$ python3 -c "
import ast
cypher = ast.literal_eval(open('cypher.txt').read().strip())
print('outer list length:', len(cypher))
print('first entry:', cypher[0])
print('last entry :', cypher[-1])
"
outer list length: 17
first entry: ['0x70', '0x69']
last entry : ['0x7d']
```

The last entry is `['0x7d']`, which decodes to `}` (`0x7d` is the ASCII code for `}`). So the flag ends with `}` — good, that matches the picoCTF flag format.

The outer list has length 17. The next step is to figure out the original length.

### Step 3 — Figure out the original flag length

I'll feed several known-length inputs through `scramble` and check the outer list length. The remote payload's outer length is 17, so the original length is whatever I fed in.

```
┌──(zham㉿kali)-[~/ctf/quantum-scrambler]
└─$ python3 -c "
def scramble(L):
    A = L; i = 2
    while i < len(A):
        A[i-2] += A.pop(i-1)
        A[i-1].append(A[:i-2])
        i += 1
    return L

for n in [5, 6, 8, 16, 17, 32, 33]:
    inp = [[k] for k in range(n)]
    scramble(inp)
    print(f'n={n} -> outer length = {len(inp)}')
"
n=5 -> outer length = 3
n=6 -> outer length = 4
n=8 -> outer length = 5
n=16 -> outer length = 9
n=17 -> outer length = 9
n=32 -> outer length = 17   <-- matches the remote!
n=33 -> outer length = 17
```

`n=32` produces an outer list of length 17 — that's the remote. So the flag is **32 characters** long. With `picoCTF{...}` (8 + 1 = 9) wrapping, the inside is 23 characters.

### Step 4 — Build a decoder using a known plaintext

Hint 3 says to feed a known plaintext through the scrambler. I will use this trick: instead of feeding characters, I feed **integer placeholders** `[0], [1], [2], ...`. After scrambling, the structure is the same, but each "char" is now an integer index. Then I walk the result and pair each integer with the hex string at the **same path** in the real remote payload.

```
┌──(zham㉿kali)-[~/ctf/quantum-scrambler]
└─$ nano decode.py
```

```python
#!/usr/bin/env python3
"""
Quantum Scrambler decoder.

Strategy:
  1. eval the remote cypher text into a real Python list `cypher`.
  2. Build a parallel structure `order` by running scramble() on
     [[0], [1], ..., [n-1]] where n is the original flag length.
     After scrambling, the structure is identical to cypher's, but
     each "char" position is now an integer index.
  3. Walk `order` and `cypher` in parallel. The first time we see
     integer k in `order`, record the hex string at the same path
     in `cypher`. That hex string is the k-th character of the flag.
"""
import socket
import ast

HOST = "verbal-sleep.picoctf.net"
PORT = 51442
ORIGINAL_LEN = 32   # derived in Step 3


def recv_all(sock, timeout=10.0):
    sock.settimeout(timeout)
    buf = b""
    try:
        while True:
            chunk = sock.recv(65536)
            if not chunk:
                break
            buf += chunk
    except socket.timeout:
        pass
    return buf


def scramble(L):
    A = L
    i = 2
    while i < len(A):
        A[i-2] += A.pop(i-1)
        A[i-1].append(A[:i-2])
        i += 1
    return L


def main():
    s = socket.create_connection((HOST, PORT))
    raw = recv_all(s)
    s.close()

    cypher = ast.literal_eval(raw.decode().strip())
    print(f"[+] outer length: {len(cypher)}")

    # Build the parallel integer-keyed structure
    order = [[k] for k in range(ORIGINAL_LEN)]
    scramble(order)

    # Walk both in parallel, recording the first hex string seen for each index
    flag_chars = {}

    def walk(o, r):
        if isinstance(o, int):
            flag_chars.setdefault(o, r)
            return
        if isinstance(o, list):
            for a, b in zip(o, r):
                walk(a, b)

    for e, rem in zip(order, cypher):
        walk(e, rem)

    missing = [k for k in range(ORIGINAL_LEN) if k not in flag_chars]
    if missing:
        print(f"[!] missing positions: {missing}")
        return 1

    hex_str = "".join(flag_chars[k] for k in range(ORIGINAL_LEN))
    flag = "".join(chr(int(hex_str[i:i + 4], 16))
                   for i in range(0, len(hex_str), 4))
    print(f"[+] flag: {flag}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 5 — Run the decoder

```
┌──(zham㉿kali)-[~/ctf/quantum-scrambler]
└─$ python3 decode.py
[+] outer length: 17
[+] flag: picoCTF{python_is_weird0288d140}
```

The flag is `picoCTF{python_is_weird0288d140}`.

### Alternative solve — solve the math, no socket needed

The only reason the script connects to the server is to grab a fresh cypher. You can also solve this by hand: paste the printed output into a `.txt` file, then run the decoder against the file. The decoding logic is purely local.

```
┌──(zham㉿kali)-[~/ctf/quantum-scrambler]
└─$ nano decode_file.py
```

```python
#!/usr/bin/env python3
import ast
ORIGINAL_LEN = 32

def scramble(L):
    A = L; i = 2
    while i < len(A):
        A[i-2] += A.pop(i-1)
        A[i-1].append(A[:i-2])
        i += 1
    return L

cypher = ast.literal_eval(open("cypher.txt").read().strip())
order = [[k] for k in range(ORIGINAL_LEN)]
scramble(order)

flag_chars = {}
def walk(o, r):
    if isinstance(o, int):
        flag_chars.setdefault(o, r)
        return
    if isinstance(o, list):
        for a, b in zip(o, r):
            walk(a, b)
for e, rem in zip(order, cypher):
    walk(e, rem)

hex_str = "".join(flag_chars[k] for k in range(ORIGINAL_LEN))
print("".join(chr(int(hex_str[i:i + 4], 16))
              for i in range(0, len(hex_str), 4)))
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/ctf/quantum-scrambler]
└─$ python3 decode_file.py
picoCTF{python_is_weird0288d140}
```

This is the version I would use if the server went down after I had already saved the cypher text.

---

## What Happened Internally

1. The server opens `flag.txt`, strips the trailing newline, and wraps each character as a one-element list: `[hex(ord(c))]`. For a 32-character flag, `get_flag()` returns a 32-element list of single-element lists.
2. `scramble()` runs on that list. Each iteration pops one list off the front of the moving window, appends it to the previous one, and stores a reference to the past onto the new last list. After 16 iterations, the list has shrunk from 32 to 17 top-level entries, but each surviving entry contains a recursive mess of references to the originals.
3. `print(cypher)` dumps the whole structure using Python's `repr`, which is also valid Python syntax. That is why `eval` (or `ast.literal_eval`) round-trips it back to the original list.
4. The script's `scramble` mutates `L` in place — `L` and `A` are the same object. That detail does not matter for the solve, but it explains why the same list is both the input, the working buffer, and the return value.
5. On our end, we feed placeholder integers `[0]..[n-1]` through the **same** `scramble` and get a parallel structure where each leaf is an integer instead of a hex string. Walking both structures in lockstep lets us map every original position to the hex string sitting at the same path in the real cypher.
6. Once we have hex strings for positions `0..31`, we convert each `0xNN` to its ASCII character and concatenate. The result is the flag.

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `nc` | Grab the printed cypher from the server | Easy |
| `python3` (stdlib only) | `eval` the cypher, run the parallel scramble, walk the structure, decode the flag | Medium |
| `ast.literal_eval` | Safely parse the server's printed list (safer than `eval`) | Easy |
| `nano` | Edit the decoder script | Easy |

---

## Key Takeaways

- **Any `print()` of a Python object is a round-trippable Python expression.** That is why `eval` (or the safer `ast.literal_eval`) is the first move whenever a server hands you a printed structure.
- **Index placeholders beat character placeholders for structural analysis.** Feeding integers through a function gives you a "position map" you can walk against the real output. Hint 3 is the standard version of this trick.
- **The outer list length is a function of the input length.** For this scrambler, `n=32 -> outer=17` and `n=33 -> outer=17`, so you can pin down the input length by counting the outer entries.
- **`setdefault` keeps the first hit and ignores repeats.** Because every character is referenced many times in the scrambled structure, the walk will visit the same integer many times. We only care about the first one, since the hex string is the same everywhere.
- The flag decodes as `python_is_weird0288d140` — the "python is weird" half is the joke (a self-referential list-of-lists cipher written in Python is exactly the kind of weird Python thing), and `0288d140` is a 32-bit hex tail that you would not get any meaning out of. The whole point of the flag is the wordplay at the start.
