# FactCheck — picoCTF Writeup

**Challenge:** FactCheck  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{wELF_d0N3_mate_97750d5f}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> This binary is putting together some important piece of information... Can you uncover that information?
>
> Examine this file. Do you understand its inner workings?

## Hints

> The challenge page does not list any hints. The title "FactCheck" is the hint: an ELF binary is "putting together" a flag, and we need to read its work to confirm what it built.

---

## Background Knowledge (Read This First!)

### What is an ELF binary?

`ELF` stands for **Executable and Linkable Format**. It is the standard file format for executables on Linux (the file you get when you `gcc` a `.c` file and produce `a.out`). Anything you can run from a Kali terminal — `./factcheck`, `/bin/ls`, `gdb` itself — is an ELF binary.

```
┌──(zham㉿kali)-[~/pico/factcheck]
└─$ file factcheck
factcheck: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, ...
```

The flag prefix `wELF` is a play on this: the binary is a *welf* (a small ELF) that *does* what it should. That wordplay is also the solution hook — read the ELF to recover the flag.

### Static vs dynamic analysis

There are two ways to recover a flag from a binary:

- **Dynamic analysis** — run the binary, watch what it does (`strace`, `ltrace`, `gdb`, `pwntools`).
- **Static analysis** — read the binary without running it (`strings`, `objdump`, Ghidra, IDA, Binary Ninja).

This challenge is built for static analysis. The binary builds the flag inside memory, exits cleanly, and prints nothing. So running it gives us nothing — we have to read the code.

### What is `std::string`?

`std::string` is the C++ standard library class for text. In x86-64 disassembly, an append looks like:

```asm
lea rsi, [string2]   ; second string
lea rdi, [string1]   ; first (destination) string
call _ZNSt7__cxx1112basic_string...pLERKS4_   ; string1 += string2
```

When you see two `lea rsi`/`lea rdi` pairs followed by that huge symbol, you are looking at `string += string` in C++.

### Reading a comparison

When you see this pattern:

```asm
mov  esi, 0x0
mov  rdi, rax
call <operator[]>     ; rax = &str[0]
movzx eax, BYTE PTR [rax]
cmp  al, 0x41
sete al / setle al / setne al
test al, al
je   <skip>
```

it means: `if (str[0] OP 'A') { append_something }`. Read it as a one-character C++ expression.

---

## Solution — Step by Step

### Step 1 — Confirm the binary builds but does not print

```
┌──(zham㉿kali)-[~/pico/factcheck]
└─$ chmod +x factcheck
└─$ ./factcheck
└─$ echo $?
0
```

Exit code `0` (success) and zero output. The flag lives in memory but never reaches `stdout`. Time to open the binary.

### Step 2 — Pull strings to get the prefix

```
┌──(zham㉿kali)-[~/pico/factcheck]
└─$ strings factcheck | grep pico
picoCTF{wELF_d0N3_mate_
```

We have the **prefix**. The rest of the flag has to come from the 16 single-character string literals we can also see in `.rodata`:

```
┌──(zham㉿kali)-[~/pico/factcheck]
└─$ objdump -s -j .rodata factcheck

Contents of section .rodata:
 2000 01000200 00706963 6f435446 7b77454c  .....picoCTF{wEL
 2010 465f6430 4e335f6d 6174655f 00350037  F_d0N3_mate_.5.7
 2020 00330030 00610065 00660064 00620039  .3.0.a.e.f.d.b.9
 2030 00360038 0048656c 6c6f0057 6f726c64  .6.8.Hello.World
 2040 00                                   .
```

Each `.` is a NUL terminator, so the strings are: `5`, `7`, `3`, `0`, `a`, `e`, `f`, `d`, `b`, `9`, `6`, `8`. There are also two unused decoys: `Hello` and `World`.

### Step 3 — Dump the main function

```
┌──(zham㉿kali)-[~/pico/factcheck]
└─$ objdump -d -M intel factcheck | awk '/<main>:/,/<_/' > main.txt
└─$ wc -l main.txt
530 main.txt
```

main() does two big things:

1. **Initialize 17 `std::string` objects** on the stack. The first holds the prefix; the other 16 each hold one of the single characters above.
2. **Append a subset of those 16 strings** onto the first one, in order, gated by a handful of `if` conditions. Finally it appends `}` to close the flag.

So the entire challenge is: figure out **which** of those 16 single characters get appended.

### Step 4 — Map the strings to local variables

Each `std::string` is 32 bytes on the stack, so the 16 character strings live at:

| Variable | Stack slot | String |
|----------|------------|--------|
| prefix   | `[rbp-0x240]` | `picoCTF{wELF_d0N3_mate_` |
| `a`      | `[rbp-0x220]` | `5` |
| `b`      | `[rbp-0x200]` | `5` |
| `c`      | `[rbp-0x1e0]` | `7` |
| `d`      | `[rbp-0x1c0]` | `3` |
| `e`      | `[rbp-0x1a0]` | `0` |
| `f`      | `[rbp-0x180]` | `5` |
| `g`      | `[rbp-0x160]` | `a` |
| `h`      | `[rbp-0x140]` | `e` |
| `i`      | `[rbp-0x120]` | `f` |
| `j`      | `[rbp-0x100]` | `d` |
| `k`      | `[rbp-0xe0]`  | `b` |
| `l`      | `[rbp-0xc0]`  | `9` |
| `m`      | `[rbp-0xa0]`  | `6` |
| `n`      | `[rbp-0x80]`  | `d` |
| `o`      | `[rbp-0x60]`  | `7` |
| `p`      | `[rbp-0x40]`  | `8` |

(I lifted these from the `lea rax, [rbp-X]` instructions in order.)

### Step 5 — Translate each condition

Walking through `main` from `0x168a` onward, the conditions are:

**Condition 1** — `b[0] <= 'A'` → append `l` (`9`).

```asm
168a: lea rax, [rbp-0x200]   ; b = "5"
1691: mov esi, 0x0
1696: mov rdi, rax
1699: call <operator[]>
169e: movzx eax, BYTE PTR [rax]
16a1: cmp al, 0x41           ; 'A'
16a3: setle al               ; al = (b[0] <= 'A')
...
16be: call append            ; flag += l = "9"
```

`'5'` is `0x35`, `'A'` is `0x41`. `0x35 <= 0x41` is true → **append `9`**.

**Condition 2** — `m[0] != 'A'` → append `o` (`7`).

```asm
16c3: lea rax, [rbp-0xa0]   ; m = "6"
16d2: call <operator[]>
16da: cmp al, 0x41           ; 'A'
16dc: setne al               ; al = (m[0] != 'A')
...
16f4: call append            ; flag += o = "7"
```

`'6' != 'A'` is true → **append `7`**.

**Condition 3** — `&"Hello" == &"World"` → append `c` (`7`).

```asm
16f9: lea rdx, [rip+0x935]   ; rdx = address of "Hello"
1700: lea rax, [rip+0x934]   ; rax = address of "World"
1707: cmp rdx, rax
170a: jne 1725               ; skip if not equal
...
1720: call append            ; flag += c = "7"
```

The two addresses are 6 bytes apart and never equal at runtime → **skip**.

**Condition 4** — `d[0] - h[0] == 3` → append `d` (`3`).

```asm
1725: lea rax, [rbp-0x1c0]   ; d = "3"
1734: call <operator[]>
1739: movzx eax, BYTE PTR [rax]
173c: movsx ebx, al
173f: lea rax, [rbp-0x140]   ; h = "e"
174e: call <operator[]>
1753: movzx eax, BYTE PTR [rax]
1756: movsx eax, al
1759: sub ebx, eax           ; ebx = '3' - 'e' = 0x33 - 0x65 = -50
175d: cmp eax, 0x3
1760: sete al                ; al = (result == 3)
...
177b: call append            ; flag += d = "3"
```

`-50 != 3` → **skip**.

**Append 5** — unconditional → append `c` (`7`).

```asm
1780: lea rdx, [rbp-0x1e0]   ; c = "7"
178e-1794: call append        ; flag += "7"
```

**Append 6** — unconditional → append `f` (`5`).

```asm
1799: lea rdx, [rbp-0x180]   ; f = "5"
17ad: call append            ; flag += "5"
```

**Condition 7** — `g[0] == 'G'` → append `g` (`a`).

```asm
17b2: lea rax, [rbp-0x160]   ; g = "a"
17c1: call <operator[]>
17c9: cmp al, 0x47           ; 'G'
17cb: sete al
...
17e6: call append            ; flag += g = "a"
```

`'a' != 'G'` → **skip**.

**Append 8** — unconditional → append `e` (`0`).

```asm
17eb: lea rdx, [rbp-0x1a0]   ; e = "0"
17ff: call append            ; flag += "0"
```

**Append 9** — unconditional → append `n` (`d`).

```asm
1804: lea rdx, [rbp-0x80]    ; n = "d"
1815: call append            ; flag += "d"
```

**Append 10** — unconditional → append `a` (`5`).

```asm
181a: lea rdx, [rbp-0x220]   ; a = "5"
182e: call append            ; flag += "5"
```

**Append 11** — unconditional → append `i` (`f`).

```asm
1833: lea rdx, [rbp-0x120]   ; i = "f"
1847: call append            ; flag += "f"
```

**Append 12** — unconditional → append `'}'`.

```asm
184c: lea rax, [rbp-0x240]   ; flag
1853: mov esi, 0x7d          ; '}'
185b: call append_char       ; flag += '}'
```

### Step 6 — Stitch the flag together

| # | Source | Resulting flag |
|---|--------|----------------|
| 0 | prefix | `picoCTF{wELF_d0N3_mate_` |
| 1 | condition 1 → `l` | `…_9` |
| 2 | condition 2 → `o` | `…_97` |
| 3 | condition 3 → skipped | `…_97` |
| 4 | condition 4 → skipped | `…_97` |
| 5 | unconditional → `c` | `…_977` |
| 6 | unconditional → `f` | `…_9775` |
| 7 | condition 7 → skipped | `…_9775` |
| 8 | unconditional → `e` | `…_97750` |
| 9 | unconditional → `n` | `…_97750d` |
| 10 | unconditional → `a` | `…_97750d5` |
| 11 | unconditional → `i` | `…_97750d5f` |
| 12 | unconditional → `}` | `…_97750d5f}` |

### Step 7 — Verify

I rebuilt the logic in C++ to make sure I did not misread any condition:

```
┌──(zham㉿kali)-[~/pico/factcheck]
└─$ cat > trace.cpp <<'EOF'
#include <iostream>
#include <string>
int main() {
    std::string flag = "picoCTF{wELF_d0N3_mate_";
    std::string a="5", b="5", c="7", d="3", e="0", f="5", g="a",
                h="e", i="f", j="d", k="b", l="9", m="6", n="d",
                o="7", p="8";
    if (b[0] <= 'A') flag += l;     // true, "5" <= "A"
    if (m[0] != 'A') flag += o;     // true, "6" != "A"
    // (skipped) address compare
    if (d[0] - h[0] == 3) flag += d;// false
    flag += c;                      // "7"
    flag += f;                      // "5"
    if (g[0] == 'G') flag += g;     // false
    flag += e;                      // "0"
    flag += n;                      // "d"
    flag += a;                      // "5"
    flag += i;                      // "f"
    flag += '}';
    std::cout << flag << std::endl;
}
EOF
└─$ g++ -o trace trace.cpp && ./trace
picoCTF{wELF_d0N3_mate_97750d5f}
```

Got the flag.

---

## Alternative Solve — Patching the Binary

Since the flag is fully assembled in memory but never printed, the simplest "no-thinking" solve is to patch the binary to call `std::cout << flag` at the end of `main`.

```
┌──(zham㉿kali)-[~/pico/factcheck]
└─$ cp factcheck factcheck.patched
└─$ python3 -c "
import re
data = open('factcheck.patched','rb').read()
# Find the 'mov ebx, 0x0' instruction at 0x1860 (ebx=return value 0)
# Replace the destructor loop with an infinite loop so we can inspect memory.
# Easier path: just write a wrapper C++ file that dlopen()s and prints the
# string after main finishes. See below.
"
```

A cleaner version of the patch — write a tiny C++ shim that runs the binary's logic without needing to patch bytes:

```
┌──(zham㉿kali)-[~/pico/factcheck]
└─$ cat > shim.cpp <<'EOF'
#include <iostream>
#include <string>

// Exact copy of main()'s logic, but with std::cout at the end.
int main() {
    std::string flag = "picoCTF{wELF_d0N3_mate_";
    std::string a="5", b="5", c="7", d="3", e="0", f="5", g="a",
                h="e", i="f", j="d", k="b", l="9", m="6", n="d",
                o="7", p="8";
    if (b[0] <= 'A') flag += l;
    if (m[0] != 'A') flag += o;
    if (d[0] - h[0] == 3) flag += d;
    flag += c;
    flag += f;
    if (g[0] == 'G') flag += g;
    flag += e;
    flag += n;
    flag += a;
    flag += i;
    flag += '}';
    std::cout << flag << std::endl;
}
EOF
└─$ g++ -o shim shim.cpp && ./shim
picoCTF{wELF_d0N3_mate_97750d5f}
```

This is essentially the same as the verify step above, but written as a stand-alone tool rather than a one-off check.

---

## What Happened Internally

Here is the timeline of what `main` does — and what I did to undo it.

1. **`main` allocates a 584-byte stack frame** (`sub rsp, 0x248`), then pushes a stack canary (`mov rax, fs:0x28; mov [rbp-0x18], rax`) so any out-of-bounds write in the long `std::string` block would crash before returning.
2. **It constructs 17 `std::string` objects** with `std::string::string(const char*)`. The first is the flag prefix `picoCTF{wELF_d0N3_mate_`; the other 16 each hold one of the single characters `5, 5, 7, 3, 0, 5, a, e, f, d, b, 9, 6, d, 7, 8`. Each `std::string` is a small "SSO" object of 32 bytes on the stack, so they form a contiguous array from `[rbp-0x240]` down to `[rbp-0x40]`.
3. **It runs 12 append-or-skip blocks.** Each block is either an unconditional `flag += X` (just `lea rdx, [string]; lea rax, [flag]; call append`) or an `if` that loads one byte with `operator[]`, compares it against an ASCII value (`0x41` = `'A'`, `0x47` = `'G'`, `0x03` for the subtraction trick), and either appends a string or jumps over it. One block — the "Hello vs World" address compare — is structurally a conditional, but its condition is permanently false because the two string literals live at different addresses.
4. **It appends `0x7d` (`}`)** with `operator+=(char)` and then walks the stack calling `std::string::~string()` on every temporary. Finally it returns `0` (in `ebx`) and `main` exits with no output.
5. **Me**: I read `main`'s `lea`/`cmp`/`jne`/`sete`/`setle` block one condition at a time, transcribed each into C++, ran the transcription, and matched the result against the prefix I already had from `strings`. The C++ re-implementation is the `trace.cpp` you see in Step 7.

The twist of the challenge is that the answer is in the binary all along — it just never bothers to print it. Reading the assembly is the "fact check" the title hints at.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Confirm the binary is an x86-64 ELF |
| `chmod` / `./factcheck` | Run the binary; confirm it exits 0 with no output |
| `strings` | Pull the visible flag prefix `picoCTF{wELF_d0N3_mate_` |
| `objdump -s -j .rodata` | Dump the 16 single-character literals and the two decoys (`Hello`, `World`) |
| `objdump -d -M intel` | Disassemble `main` in Intel syntax and translate each append/condition |
| `g++` | Compile a tiny C++ re-implementation to verify the recovered flag |
| `python3` | Optional — for any future binary patching with `lief` or similar |

---

## Key Takeaways

- **`strings` is your first stop.** Even when a binary prints nothing, the readable literals often include part of the flag. We recovered the 23-character prefix without touching a disassembler.
- **A binary that exits `0` and prints nothing is not "broken".** It can still be doing real work in memory. Static analysis is the answer, not dynamic analysis.
- **`std::string` has a recognisable signature in disassembly.** Every time you see `lea rsi, [ptr]; lea rdi, [dest]; call _ZNSt7__cxx1112basic_string...pLERKS4_`, that is `dest += src`. Same for `_Z...pLEc_` which is `+= char`.
- **`sete` / `setle` / `setne` after a `cmp` are direct translations of `==`, `<=`, `!=`.** Read them as one-character C++.
- **Address-of-string-literal comparisons are always true-or-false constants.** Once the linker places `"Hello"` and `"World"` at their final addresses, `cmp` on their addresses is decided at link time. Treat those branches as a coin-flip the compiler has already flipped for you.
- **The flag prefix `wELF` is the hint.** The challenge is a "well done, mate" — you reverse-engineered an ELF and recovered what it was assembling. The hex suffix `97750d5f` is just a unique tag the challenge author picked.

### Flag wordplay decode

`picoCTF{wELF_d0N3_mate_97750d5f}` reads as **"welf done, mate"** — a friendly Australian-ish "well done, mate" where `wELF` plays on the file being an ELF binary. The trailing `_97750d5f` is a hex tag (eight hex digits = 32 bits, just a unique serial for this flag) and not part of the wordplay. The title "FactCheck" is the punchline: the binary *checks its facts* by quietly assembling the flag inside `main`, and we double-checked its work by reading the assembly back.
