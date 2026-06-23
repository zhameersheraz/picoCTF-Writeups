# Perplexed — picoCTF Writeup

**Challenge:** Perplexed  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{0n3_bi7_4t_a_7im3}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> Download the binary here.

The binary asks for a password and replies either "Correct!! :D" or "Wrong :(". That is the whole game.

## Hints

> The challenge page for this one does not list any hints. We have to reverse it the hard way.

---

## Background Knowledge (Read This First!)

If anything below sounds unfamiliar, take ten minutes to skim it. The whole challenge is built on these three ideas.

### Bits and bytes

A byte is eight bits, numbered `7 6 5 4 3 2 1 0` from most significant to least significant. For example, the ASCII letter `'p'` is the byte `0x70 = 0b01110000`. Its bits are:

```
bit 7 6 5 4 3 2 1 0
     0 1 1 1 0 0 0 0
```

To test a specific bit, you AND the byte with a "mask" that has a single `1` in that position. To grab just bit `n`, write `(byte >> n) & 1`.

```python
>>> 'p' = 0x70
>>> (0x70 >> 7) & 1   # bit 7
0
>>> (0x70 >> 6) & 1   # bit 6
1
>>> (0x70 >> 0) & 1   # bit 0
0
```

### x86-64 calling convention (just enough)

When a C function is called on Linux x86-64, the first integer argument goes into `rdi`, the second into `rsi`, the third into `rdx`, and so on. The return value comes back in `rax`. Local variables live at negative offsets from `rbp` (e.g. `[rbp-0x50]`).

### `setg` and "bit extraction" in assembly

`test eax, eax; setg cl` sets `cl = 1` if `eax` is strictly greater than zero (signed), otherwise `cl = 0`. Combined with `and eax, mask`, this is just a fancy way to read one bit:

```asm
and  eax, 0x40     ; eax = byte & 0b01000000
test eax, eax
setg cl            ; cl = (byte bit 6) ? 1 : 0
```

That is what `check()` does over and over.

---

## Solution — Step by Step

### Step 1 — Run the binary and confirm behaviour

```
┌──(zham㉿kali)-[~/pico/perplexed]
└─$ chmod +x perplexed
└─$ echo "test" | ./perplexed
Enter the password: Wrong :(
```

So we need to find the magic string.

### Step 2 — Pull out the strings

```
┌──(zham㉿kali)-[~/pico/perplexed]
└─$ strings perplexed | grep -E 'Correct|Wrong|password'
Enter the password:
Wrong :(
Correct!! :D
```

Just two interesting functions: `main` and `check`. Let us look at them.

### Step 3 — Disassemble `check()`

```
┌──(zham㉿kali)-[~/pico/perplexed]
└─$ objdump -d -M intel perplexed | awk '/<check>:/,/^$/'
```

The function starts by checking `strlen(input) == 0x1b`. **0x1b = 27**, so the password is 26 characters plus the newline that `fgets` always appends.

Then it stuffs three giant 8-byte constants into a stack buffer at `[rbp-0x50]`:

```asm
movabs rax, 0x617b2375f81ea7e1   ; 8 bytes at [rbp-0x50]
movabs rdx, 0xd269df5b5afc9db9   ; 8 bytes at [rbp-0x48]
movabs rax, 0xf467edf4ed1bfed2   ; 8 bytes at [rbp-0x41]
```

In little-endian, the 23-byte buffer is:

```
e1 a7 1e f8 75 23 7b 61   b9 9d fc 5a 5b df 69 d2   fe 1b ed f4 ed 67 f4
```

This is the "expected" byte string the password is compared against.

### Step 4 — Decode the loop

The hot loop is two nested `for`s:

```asm
; outer: i = 0 .. 22 (cmp 0x16)
; inner: loop_count = 0 .. 7
;   j is a separate counter, starts at 0 for i=0, then carries over

mov eax, 7
sub eax, [rbp-loop_count]          ; expected_mask = 1 << (7 - loop_count)
mov eax, 7
sub eax, [rbp-j]                   ; input_mask     = 1 << (7 - j)
...
; compare expected[i] & expected_mask  with  input[k] & input_mask
; if the two setg bits differ, return 1 (wrong)
; j++; if j == 8 then j = 0 and k++
```

In English: the function takes 23 "expected" bytes and walks over them bit by bit. For every bit of every expected byte, it grabs the matching bit of the input — **but it does not grab bits in the order you would expect**. The `j` counter carries state across outer iterations and gets bumped one extra step whenever it would have been zero. That offset shifts the input bits by one position with every 8 expected bits.

So the bits are matched like this:

| outer `i` (`m = i mod 8`) | input byte `i` bits used | input byte `i+1` bits used |
|---------------------------|--------------------------|----------------------------|
| 0                         | 6,5,4,3,2,1,0            | 6                          |
| 1                         | 5,4,3,2,1,0              | 6,5                        |
| 2                         | 4,3,2,1,0                | 6,5,4                      |
| 3                         | 3,2,1,0                  | 6,5,4,3                    |
| 4                         | 2,1,0                    | 6,5,4,3,2                  |
| 5                         | 1,0                      | 6,5,4,3,2,1                |
| 6                         | 0                        | 6,5,4,3,2,1,0              |
| 7                         | (nothing — input byte `i` is filled by `i-1`) | |
| 8 (= m=0)                 | 6,5,4,3,2,1,0            | 6                          |
| ...                       | repeats with period 8    |                            |

`bit 7` of every input byte is **never read**. That is normal ASCII — the high bit of every printable character is `0`.

### Step 5 — Write a Python simulator (the easy way)

Instead of doing the algebra by hand, I just ported the inner loop to Python and let it tell me the constraints. Each constraint says "input byte `X`, bit `Y`, must equal value `V`".

```
┌──(zham㉿kali)-[~/pico/perplexed]
└─$ nano solve.py
```

Paste this:

```python
#!/usr/bin/env python3
"""
Perplexed solver. Port the inner loop of check() to Python and let it
emit a list of (input_byte_idx, bit_idx, value) constraints. Then assign
each bit accordingly. Bit 7 of every input byte is unconstrained (ASCII),
so we just leave it as 0.
"""
import struct

# Expected buffer pulled from the three movabs immediates
q1, q2, q3 = 0x617b2375f81ea7e1, 0xd269df5b5afc9db9, 0xf467edf4ed1bfed2
expected = bytearray(23)
expected[0:8]   = struct.pack("<Q", q1)
expected[8:16]  = struct.pack("<Q", q2)
expected[15:23] = struct.pack("<Q", q3)

constraints = []

i = j = k = loop_count = 0
while i <= 0x16:
    loop_count = 0
    while loop_count <= 7:
        if j == 0:
            j += 1                 # the special "skip j=0" rule
        e_bit = (expected[i] >> (7 - loop_count)) & 1
        i_bit = 7 - j              # bit position inside input[k]
        constraints.append((k, i_bit, e_bit))
        j += 1
        if j == 8:
            j = 0
            k += 1
        loop_count += 1
    i += 1

# Apply every constraint. If two constraints disagree, the binary is
# unsatisfiable and we have a problem.
bits = {}
for byte_idx, bit_idx, val in constraints:
    key = (byte_idx, bit_idx)
    if key in bits and bits[key] != val:
        raise SystemExit(f"Inconsistent at {key}: {bits[key]} vs {val}")
    bits[key] = val

# Build the candidate (bit 7 free -> 0).
max_byte = max(b for (b, _) in bits)
candidate = bytearray()
for b in range(max_byte + 1):
    byte = 0
    for bit in range(7):
        if bits.get((b, bit)):
            byte |= 1 << bit
    candidate.append(byte)

print(f"Recovered password ({len(candidate)} bytes):")
print(candidate.decode('latin-1'))
```

Save: Ctrl+O, Enter, Ctrl+X.

```
┌──(zham㉿kali)-[~/pico/perplexed]
└─$ python3 solve.py
Recovered password (27 bytes):
picoCTF{0n3_bi7_4t_a_7im3}
```

(The 27th byte is the newline that `fgets` appends — bit 5 of byte 26 is constrained by the algorithm, but the rest of that byte is irrelevant because the password check stops at the end of the buffer.)

### Step 6 — Confirm against the binary

```
┌──(zham㉿kali)-[~/pico/perplexed]
└─$ echo "picoCTF{0n3_bi7_4t_a_7im3}" | ./perplexed
Enter the password: Correct!! :D
```

Got the flag.

---

## Alternative Solve — Angr (symbolic execution)

If you do not want to manually port the loop, **angr** will brute-force the input for you. The function is short, the loop bounds are tight, and angr solves it in well under a second:

```python
import angr, claripy

p = angr.Project("./perplexed", auto_load_libs=False)
state = p.factory.entry_state(
    args=["./perplexed"],
    stdin=angr.SimFileStream(name="stdin", content=claripy.BVS("pw", 8 * 27)),
)
simgr = p.factory.simulation_manager(state)
simgr.explore(find=lambda s: b"Correct" in s.posix.dumps(1))
print(simgr.found[0].posix.dumps(0))
```

That prints the same flag. Use whichever approach you prefer; the Python port above is what I would actually run on a CTF box because angr is heavy to install.

---

## What Happened Internally

Here is the timeline of what the binary does — and what I did to undo it.

1. **`main` allocates a 272-byte buffer** (`sub rsp, 0x110`), zeros it out, prints `"Enter the password: "`, reads up to 256 bytes with `fgets`, and calls `check(buffer)`. If `check` returns `1`, it prints `"Correct!! :D"`, otherwise `"Wrong :("`.
2. **`check` reads `strlen(input)`** and demands it is exactly `27` (26 printable characters + the newline `fgets` appends).
3. **`check` writes 23 bytes of "expected" data** to its stack: three `movabs` instructions place `0x617b2375f81ea7e1`, `0xd269df5b5afc9db9`, and `0xf467edf4ed1bfed2` at `[rbp-0x50]` in little-endian. That gives the constant byte sequence `e1 a7 1e f8 75 23 7b 61 b9 9d fc 5a 5b df 69 d2 fe 1b ed f4 ed 67 f4`.
4. **`check` runs a double loop**: outer `i` from 0 to 22, inner `loop_count` from 0 to 7. For each `(i, loop_count)` it extracts `expected[i] >> (7-loop_count) & 1` and compares it to `input[k] >> (7-j) & 1` where `j` is a 1..7 state variable that carries across outer iterations and gets bumped one extra step whenever it would have wrapped to `0`. If any bit differs, `check` returns `1` (wrong).
5. **`check` returns `0`** if every bit matched and we walked off the end of the loop. `main` then prints `"Wrong :("` because `check` returning `0` means failure.
6. **Me**: I read the disassembly, ported the bit-extraction logic to Python, ran it, and got `picoCTF{0n3_bi7_4t_a_7im3}`. Fed that into the binary — `Correct!! :D`.

The fun twist is that the input bits get "shifted" by one position with every 8 expected bits. That is why a naive `expected[i] == input[i]` does not work — the bytes of the password are not equal to the bytes of the expected buffer; they are a permuted view of the bits of the expected buffer.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `strings` | Confirm the only two interesting strings (`Correct` / `Wrong`) and find `main` / `check` |
| `objdump -d -M intel` | Disassemble `check` and read the three `movabs` immediates |
| `python3` (stdlib) | Port the inner loop, collect bit constraints, recover the password |
| `echo … \| ./perplexed` | Verify the recovered password against the live binary |
| `angr` (optional) | Drop-in symbolic execution if you would rather skip the manual port |

---

## Key Takeaways

- **The check is bit-by-bit, not byte-by-byte.** Even though it looks like the function compares 23 bytes against the password, the inner loop pulls each bit out separately and re-aligns the input bits so they straddle two bytes. That is the trick of the challenge — and the source of the flag wordplay.
- **`setg` is just "non-zero"** when used after `and` with a positive mask. Treat it as a one-bit comparator and the disassembly reads itself.
- **`fgets` includes the trailing newline** in its return, which is why `strlen` is `27` even though the password is 26 characters.
- **Bit 7 of every input byte is irrelevant.** ASCII characters all have bit 7 = 0. If you ever see an algorithm that ignores the high bit, it is almost certainly dealing with text.
- **Always port the binary's logic to a scriptable language.** Pulling the immediates out and feeding them through a faithful re-implementation of the loop is faster, less error-prone, and easier to debug than trying to do the algebra on paper.

### Flag wordplay decode

`picoCTF{0n3_bi7_4t_a_7im3}` is "**one bit at a time**" written in l33t-speak: `0n3` for `one`, `bi7` for `bit`, `4t` for `at`, `a` stays, `7im3` for `time`. The phrase is a literal description of the algorithm in `check` — the function walks the password one bit at a time and compares each bit to a corresponding bit of the 23-byte expected buffer (with a clever bit-shift trick that hides the bytes from a casual `strcmp`-style reverse engineer).
