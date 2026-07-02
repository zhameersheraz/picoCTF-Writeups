# Classic Crackme 0x100 — picoCTF Writeup

**Challenge:** Classic Crackme 0x100  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{s0lv3_angry_symb0ls_e1ad09b7}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> A classic Crackme. Find the password, get the flag!
>
> Crack the Binary file locally and recover the password. Use the same password on the server to get the flag!
>
> Access the server using `nc titan.picoctf.net 61068`

## Hints

> 1. Let the machine figure out the symbols!

---

## Background Knowledge (Read This First!)

### What is a crackme?

A *crackme* is a small program that takes a password (or a key) and either says `CORRECT` or `WRONG`. They are the standard training ground for reverse engineering — same shape as real software unlockers, license checks, and anti-tamper logic, just stripped down to a few hundred lines of code.

The approach is always the same:

1. Run the binary and watch what it does.
2. Open it in a disassembler (`objdump`, Ghidra, Binary Ninja, IDA) and read the password check.
3. Reconstruct the secret password, either by solving a check or by inverting the transformation.

### `%50s` in `scanf` — the input length limit

The binary reads the password with `scanf("%50s", buf)`. That means it accepts **at most 50 characters** (then a NUL). If the encoded password is 50 characters long, the input is also exactly 50. Knowing the bound up front prevents a lot of "off by one" bugs in your solver.

### `memcmp` and the "compare with a stored buffer" pattern

After the binary transforms the input, it calls `memcmp(input, encoded, len)`. This is the "compute a function of the input, then compare with a stored answer" pattern. The hint **"Let the machine figure out the symbols!"** is nudging you to just run the algorithm with the right inverse and read the answer — don't try to crack the transformation algebraically.

### Caesar / shift ciphers, modular arithmetic

The inner loop of this binary is essentially a *position-dependent Caesar cipher*: for each character at index `j`, it shifts the character by a fixed amount that depends only on `j`. The hint in the title (`0x100` = 256) and the modular arithmetic (`% 26` in the inner loop) is a dead giveaway that the answer is a Caesar-style cipher, not anything cryptographic.

---

## Solution — Step by Step

### Step 1 — Run the binary and confirm the behaviour

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ chmod +x crackme
└─$ echo "test" | ./crackme
Enter the secret password: FAILED!
```

It reads a password, prints either `SUCCESS! Here is your flag: ...` or `FAILED!`. The local binary prints a placeholder flag (`picoCTF{sample_flag}`); the real flag is on the server.

### Step 2 — Pull the visible strings and symbols

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ strings crackme | grep -iE 'flag|secret|password|success|failed|enter'
Enter the secret password:
%50s
picoCTF{sample_flag}
SUCCESS! Here is your flag: %s
FAILED!
```

`picoCTF{sample_flag}` is the local placeholder. `%50s` says the input is at most 50 characters.

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ nm crackme | grep -E "main|check|verify"
0000000000401176 T main
```

Just one interesting function. Time to read it.

### Step 3 — Disassemble `main`

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ objdump -d -M intel crackme | awk '/<main>:/,/<__libc_csu/' > main.txt
└─$ wc -l main.txt
155 main.txt
```

The function is short — only 155 lines. Here is the high-level flow I see in the disassembly:

1. Build a 51-character encoded string on the stack using six `movabs` QWORDs and one trailing `mov DWORD`. The result starts with `apijaczh...`.
2. `printf("Enter the secret password: "); scanf("%50s", input);` — read up to 50 chars.
3. Compute `len = strlen(encoded)` and store the constants `0x55, 0x33, 0x0f, 0x61`.
4. Run a **double loop**:
   - Outer `i` from 0 to 2 (three full passes).
   - Inner `j` from 0 to `len-1`.
   - On each pass, replace `input[j]` with `'a' + ((input[j] - 'a' + shift(j)) mod 26)`.
5. `memcmp(input, encoded, len)`. If equal, print the flag; else print `FAILED!`.

So we need to invert the transform on each character. Because the shift is the same on every outer iteration, the total per-character shift is just `3 * shift(j) mod 26`.

### Step 4 — Extract the encoded string

The encoded buffer is at `[rbp-0x60]`. Each `movabs` stores 8 bytes (little-endian):

```
401181: movabs rax, 0x687a63616a697061   -> "apijaczh"
40118b: movabs rdx, 0x676a796e6674677a   -> "zgtfnyjg"
40119d: movabs rax, 0x6d626a7271766472   -> "rdvqrjbm"
4011a7: movabs rdx, 0x7a636a6d63727563   -> "curcmjcz"
4011b9: movabs rax, 0x6c65646777627673   -> "svbwgdel"
4011c3: movabs rdx, 0x69796b6a78787876   -> "vxxxjkyi"
```

```
4011d5: mov DWORD PTR [rbp-0x31], 0x796769   -> "igy\0"
```

The `DWORD` overwrites the last `'i'` of `vxxxjkyi` (which is already `'i'`) and writes `'g', 'y', '\0'` to the next three byte slots. So the final 50-character encoded password is:

```
apijaczhzgtfnyjgrdvqrjbmcurcmjczsvbwgdelvxxxjkyigy
```

(8 + 8 + 8 + 8 + 8 + 8 + 2 = 50 characters, with a NUL terminator at position 50.)

### Step 5 — Reverse the shift function

The hot loop body is a chain of bitwise ANDs, shifts, and additions. After unwrapping the compiler's division-by-255 trick, the logic collapses to:

```python
def shift(j):
    m = j % 255
    t  = (m & 0x55) + ((m >> 1) & 0x55)
    t2 = (t & 0x33) + ((t >> 2) & 0x33)
    return (t2 & 0x0f) + ((t2 >> 4) & 0x0f)
```

The encryption per character is `c = 'a' + ((c - 'a' + shift(j)) mod 26)`. Inverting:

```python
def decrypt_char(c, s):
    return chr(ord('a') + (ord(c) - ord('a') - s) % 26)
```

Because the outer loop runs three times and `shift(j)` does not depend on the outer counter, the total per-character shift is `3 * shift(j) mod 26`.

### Step 6 — Solve in Python

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ nano solve.py
```

Paste this:

```python
#!/usr/bin/env python3
"""Solve Classic Crackme 0x100 by reversing the inner shift transform."""
encoded = "apijaczhzgtfnyjgrdvqrjbmcurcmjczsvbwgdelvxxxjkyigy"

def shift_value(j):
    m  = j % 255
    t  = (m & 0x55) + ((m >> 1) & 0x55)
    t2 = (t & 0x33) + ((t >> 2) & 0x33)
    return (t2 & 0x0f) + ((t2 >> 4) & 0x0f)

def decrypt_char(c, s):
    return chr(ord('a') + (ord(c) - ord('a') - s) % 26)

candidate = "".join(
    decrypt_char(c, 3 * shift_value(j)) for j, c in enumerate(encoded)
)
print(candidate)
```

Save: Ctrl+O, Enter, Ctrl+X.

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ python3 solve.py
amfdxwtywanwhpauoxphlasawliqdxqkppvnauvzpoolaymtap
```

### Step 7 — Verify locally

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ echo "amfdxwtywanwhpauoxphlasawliqdxqkppvnauvzpoolaymtap" | ./crackme
Enter the secret password: SUCCESS! Here is your flag: picoCTF{sample_flag}
```

Locally we get the placeholder. To get the real flag we have to send the password to the server.

### Step 8 — Get the real flag from the server

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ echo "amfdxwtywanwhpauoxphlasawliqdxqkppvnauvzpoolaymtap" | nc titan.picoctf.net 61068
Enter the secret password: SUCCESS! Here is your flag: picoCTF{s0lv3_angry_symb0ls_e1ad09b7}
```

Got the flag.

---

## Alternative Solve — Brute-Force with Python

If you do not want to read the assembly at all, the same algorithm can be brute-forced: for each position `j` of the encoded string, try every letter `a..z` as the plaintext and see which one, after three Caesar shifts of `shift(j)`, lands on the encoded character. There are only 26 possibilities per position, so the whole thing runs instantly.

```
┌──(zham㉿kali)-[~/pico/crackme100]
└─$ cat > brute.py <<'EOF'
#!/usr/bin/env python3
encoded = "apijaczhzgtfnyjgrdvqrjbmcurcmjczsvbwgdelvxxxjkyigy"

def shift_value(j):
    m  = j % 255
    t  = (m & 0x55) + ((m >> 1) & 0x55)
    t2 = (t & 0x33) + ((t >> 2) & 0x33)
    return (t2 & 0x0f) + ((t2 >> 4) & 0x0f)

def encrypt_char(c, s):
    return chr(ord('a') + (ord(c) - ord('a') + s) % 26)

out = []
for j, target in enumerate(encoded):
    s = 3 * shift_value(j)
    for guess in [chr(ord('a') + k) for k in range(26)]:
        if encrypt_char(guess, s) == target:
            out.append(guess)
            break
print("".join(out))
EOF
└─$ python3 brute.py
amfdxwtywanwhpauoxphlasawliqdxqkppvnauvzpoolaymtap
```

Same answer. This is the "let the machine figure out the symbols!" approach the hint recommends — no algebra, just 26 trials per position.

### Alternative Solve — Patching the Binary

If you wanted to skip the math entirely, you could patch the binary to print `encoded` at the start of `main` and skip the `memcmp` check, but that would just give you the encoded buffer — which is what we already extracted from the disassembly.

A more useful patch is to add a `printf` of the input *before* the `memcmp`, so the binary tells you what the input is. That does not help here either, but is a useful technique for "guess and check" crackmes where the verification is opaque.

---

## What Happened Internally

Here is the timeline of what `main` does, in plain English.

1. **`main` allocates a 160-byte stack frame** (`sub rsp, 0xa0`) and writes a 50-character encoded password plus a NUL onto the stack using six `movabs` QWORDs and one `mov DWORD`. The result at `[rbp-0x60]` is the string `apijaczhzgtfnyjgrdvqrjbmcurcmjczsvbwgdelvxxxjkyigy`.
2. **`setvbuf(stdout, NULL, _IONBF, 0)`** disables stdout buffering so the prompt shows up before the `scanf` blocks. This is the kind of thing GDB users never see but CTF binaries do all the time.
3. **`printf("Enter the secret password: ")`** then **`scanf("%50s", input)`** read up to 50 characters into `[rbp-0xa0]`.
4. **`len = strlen(encoded)`** (50) is stored at `[rbp-0xc]`. Four constants `0x55, 0x33, 0x0f, 0x61` are stored at `[rbp-0x10..rbp-0x19]`. The `'a'` (`0x61`) is what makes the inner loop behave like a Caesar shift.
5. **Outer loop** `i = 0..2` (`cmp [rbp-0x4], 0x2; jle back`). Each iteration transforms the entire input buffer.
6. **Inner loop** `j = 0..len-1` (`cmp j, len; jl back`). For each `j`:
   - `m = j % 255` (the compiler's division-by-255 trick with magic constant `0xffffffff80808081`).
   - `t = (m & 0x55) + ((m >> 1) & 0x55)` — a bit-shuffling step.
   - `t2 = (t & 0x33) + ((t >> 2) & 0x33)` — another shuffle.
   - `s = (t2 & 0x0f) + ((t2 >> 4) & 0x0f)` — collapse to a small integer.
   - `input[j] = 'a' + ((input[j] - 'a' + s) mod 26)` — Caesar shift by `s`.
7. After the three outer passes, **`memcmp(input, encoded, len)`** checks whether the transformed input equals the encoded buffer. The flag is printed on success; `FAILED!` on failure.
8. **Me**: I read the assembly, recognised the inner loop as a position-dependent Caesar cipher, extracted the encoded string from the `movabs` immediates, computed the per-position shifts in Python, and inverted the cipher to recover the password `amfdxwtywanwhpauoxphlasawliqdxqkppvnauvzpoolaymtap`. Then I sent it to `titan.picoctf.net:61068` and got the real flag.

The whole thing collapses once you see the inner loop is a Caesar shift. The bit-twiddling on the index `j` is just obfuscation; the modular arithmetic and the `0x61` constant are the dead giveaways.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` / `chmod` / `./crackme` | Run the binary; confirm the prompt and the placeholder flag |
| `strings` | Pull `%50s`, `picoCTF{sample_flag}`, the success / fail messages |
| `nm` | Confirm there is only one interesting function: `main` |
| `objdump -d -M intel` | Disassemble `main` in Intel syntax; find the encoded string, the loop bounds, and the shift function |
| `python3` | Re-derive the shift function, invert the cipher, recover the password |
| `nc` (netcat) | Send the recovered password to `titan.picoctf.net 61068` and read the real flag |

---

## Key Takeaways

- **The hint "let the machine figure out the symbols"** is a direct instruction: write a Python script that inverts the transformation instead of trying to do the algebra by hand. The 26-letter alphabet is so small that brute force or a direct inversion runs in microseconds.
- **`movabs rax, IMM64` is how the compiler builds a string literal into a stack buffer.** The 8-byte little-endian value in the immediate is the raw bytes of the string. Decode it once and you never have to guess what the encoded password looks like.
- **A `DWORD` write to the same slot as the last `QWORD` overwrites the tail.** When the trailing `mov DWORD` lands on the last byte of the previous `QWORD`, you have to merge them carefully — that one-character difference is exactly the kind of off-by-one that costs 20 minutes of debugging.
- **A position-dependent Caesar cipher is a tiny OTP variant.** Each position has its own shift, but the shifts are *fixed* and *known* (they come from the disassembly). Once you know the shifts, recovery is `c = 'a' + (c - 'a' - shift) mod 26`.
- **The local binary can be a red herring.** Many crackmes ship a sample flag locally and only return the real flag from the server. Always connect to the server once you have a candidate password.
- **The compiler's `imul ... 0xffffffff80808081` is division by 255.** When you see this constant in disassembly, immediately suspect the surrounding code is computing `x / 255` or `x % 255` and the next several instructions are the rest of that pattern.

### Flag wordplay decode

`picoCTF{s0lv3_angry_symb0ls_e1ad09b7}` reads as **"solve angry symbols"** in l33t-speak: `s0lv3` for `solve`, `angry` is literal (the disassembly looks like a wall of `imul`/`shr`/`and`/`.` noise), `symb0ls` for `symbols`. The trailing `e1ad09b7` is an 8-hex-digit serial (32 bits) that makes every flag unique. The flag is a wink at the fact that what looks like angry symbol soup in the disassembly is actually a tiny, mechanical Caesar-shift transform — once you name the pattern, the angry symbols solve themselves.
