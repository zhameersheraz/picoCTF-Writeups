# bytemancy 3 — picoCTF Writeup

**Challenge:** bytemancy 3  
**Category:** General Skills  
**Difficulty:** Medium  
**Flag:** `picoCTF{0bjdump_m4g1c_e6e2de85}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  

---

## Description

> Can you conjure the right bytes? The program's source code can be downloaded here and the compiled spellbook binary can be downloaded here.
> Connect to the program with netcat:
> `$ nc green-hill.picoctf.net 50110`

**Hint 1:** `objdump -t spellbook reveals the symbol table.`  
**Hint 2:** `Send the addresses as 4 raw bytes in little-endian order.`  
**Hint 3:** `pwnlib.util.packing.p32() simplifies crafting the payloads.`

**Downloads:** `app.py`, `spellbook`

---

## Background Knowledge (Read This First!)

### What does the server do?

Reading `app.py` reveals the full challenge logic:

```python
SPELLBOOK_FUNCTIONS = [
    "ember_sigil",
    "glyph_conflux",
    "astral_spark",
    "binding_word",
]

selections = random.sample(SPELLBOOK_FUNCTIONS, QUESTION_COUNT)

for idx, symbol in enumerate(selections, 1):
    target_addr = elf.symbols[symbol]
    expected_bytes = p32(target_addr)
    # reads 4 bytes from user and compares to expected_bytes
```

For each round the server picks 3 random functions from the 4 in `SPELLBOOK_FUNCTIONS`, looks up their addresses in the binary, and expects you to send the correct **4-byte little-endian** address for each one. Get all 3 right and the flag is printed.

### What is a symbol table?

A compiled binary stores a **symbol table** — a list of function names and their memory addresses. Tools like `objdump -t` and pwntools' `ELF.symbols` can read this table without running the binary.

### What is little-endian?

On x86 systems, multi-byte numbers are stored **least significant byte first**. For example, address `0x8049176` in little-endian 4 bytes is:

```
76 91 04 08
```

This is what `p32()` from pwntools does automatically.

### What is `p32()`?

`p32(n)` converts a 32-bit integer to its 4-byte little-endian representation:

```python
from pwn import p32
p32(0x8049176)  # → b'v\x91\x04\x08'
```

This is exactly the raw bytes the server expects.

---

## Reading the Source Code

The key lines from `app.py`:

```python
target_addr = elf.symbols[symbol]    # look up address from binary
expected_bytes = p32(target_addr)    # convert to 4-byte little-endian
user_bytes = read_exact_bytes(len(expected_bytes))  # read 4 bytes from user
if user_bytes != expected_bytes:     # compare
    success = False
```

Since we have the same binary (`spellbook`) that the server uses, we can look up all 4 function addresses locally and send the correct bytes instantly.

---

## Solution — Step by Step

### Step 1 — Extract all function addresses from the binary

Using pwntools' `ELF` class to read the symbol table:

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "
from pwn import ELF, p32
elf = ELF('/media/sf_downloads/spellbook', checksec=False)
funcs = ['ember_sigil','glyph_conflux','astral_spark','binding_word']
for f in funcs:
    addr = elf.symbols[f]
    print(f'{f}: {hex(addr)} -> {p32(addr)}')
"
ember_sigil:  0x8049176 -> b'v\x91\x04\x08'
glyph_conflux: 0x804919a -> b'\x9a\x91\x04\x08'
astral_spark:  0x80491c1 -> b'\xc1\x91\x04\x08'
binding_word:  0x80491e3 -> b'\xe3\x91\x04\x08'
```

### Step 2 — Automate with pwntools remote

Since the server picks 3 random functions each round and we need to respond instantly with raw bytes, we use pwntools to connect and automate:

```python
from pwn import *

elf = ELF('/media/sf_downloads/spellbook', checksec=False)
funcs = ['ember_sigil','glyph_conflux','astral_spark','binding_word']
addrs = {f: p32(elf.symbols[f]) for f in funcs}

r = remote('green-hill.picoctf.net', 50110)

while True:
    line = r.recvline().decode(errors='ignore').strip()
    print(line)
    if 'procedure' in line and "'" in line:
        name = line.split("'")[1]
        r.recvuntil(b'==> ')
        r.send(addrs[name])
    if 'picoCTF' in line:
        print('FLAG:', line)
        break
```

### Step 3 — Run and get the flag

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
[+] Opening connection to green-hill.picoctf.net on port 50110: Done
[1/3] Send the 4-byte little-endian address for procedure 'binding_word'.
[2/3] Send the 4-byte little-endian address for procedure 'glyph_conflux'.
[3/3] Send the 4-byte little-endian address for procedure 'ember_sigil'.
picoCTF{0bjdump_m4g1c_e6e2de85}
FLAG: picoCTF{0bjdump_m4g1c_e6e2de85}
```

---

## Alternative Method — objdump manually

The hint says `objdump -t spellbook reveals the symbol table`. You can find the addresses manually:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ objdump -t spellbook | grep -E "ember_sigil|glyph_conflux|astral_spark|binding_word"
08049176 g     F .text  0000001e ember_sigil
0804919a g     F .text  00000027 glyph_conflux
080491c1 g     F .text  00000022 astral_spark
080491e3 g     F .text  0000001d binding_word
```

Then manually convert each address to little-endian bytes and send them. Automation is faster but this shows the underlying mechanism.

---

## Address Table

| Function | Address | Little-endian bytes |
|----------|---------|-------------------|
| `ember_sigil` | `0x08049176` | `76 91 04 08` |
| `glyph_conflux` | `0x0804919a` | `9a 91 04 08` |
| `astral_spark` | `0x080491c1` | `c1 91 04 08` |
| `binding_word` | `0x080491e3` | `e3 91 04 08` |

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reading `app.py` | Understand exactly what the server expects | Easy |
| `pwntools ELF` | Parse the binary and extract symbol addresses | Medium |
| `p32()` | Convert addresses to 4-byte little-endian format | Easy |
| `pwntools remote` | Automate the netcat interaction | Medium |
| `objdump -t` (optional) | Manually inspect the symbol table | Easy |

---

## Key Takeaways

- **Always read the source code first** — `app.py` revealed exactly what bytes were expected, making this a straightforward extraction problem
- **`ELF.symbols`** in pwntools reads any compiled binary's symbol table without running it — a fundamental reverse engineering technique
- **Little-endian matters** — sending the address bytes in the wrong order would fail. `p32()` handles this automatically for 32-bit values
- **Automation beats manual interaction** — the server sends raw bytes and picks random functions each round; pwntools handles both cleanly
- The flag `0bjdump_m4g1c` is "objdump magic" — the hint and the flag both point to `objdump -t` as the intended approach
