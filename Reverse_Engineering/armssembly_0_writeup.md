# ARMssembly 0 — picoCTF Writeup

**Challenge:** ARMssembly 0  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 40  
**Flag:** `picoCTF{9a9c8593}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> What integer does this program print?
>
> Flag format: `picoCTF{XXXXXXXX}` → (hex, lowercase, no `0x`, and 32 bits. ex. 5614267 would be `picoCTF{0055aabb}`)
>
> Use arguments a and b: 2593949075 and 2233560849
>
> File: `chall.s`

## Hints

> 1. Simple compare.

(Hint panel for this instance surfaces exactly one hint: "Simple compare." That is the whole clue — `chall.s` is a short ARM64 assembly listing whose job is to compare two numbers and return the larger. There is no encryption, no overflow trick, no anti-RE. Read the file, find the comparison, pick the bigger of the two values, convert to 32-bit hex.)

---

## Background Knowledge

`ARMssembly 0` is the first in picoCTF's ARM/assembly series. Unlike `bit-o-asm-*` (x86, where you just spot the one `mov`), this one is a small, complete ARM64 program. A few concepts make it readable in five minutes.

**1. The file is already a complete program, not just a snippet.**
`chall.s` is a `gcc`-emitted assembly file with a `main` function, a `func1` function, a `.rodata` section holding the `"Result: %ld\n"` format string, and the boilerplate every C program compiles into. Because the source `chall.c` is *not* given, you can either compile `chall.s` and run it (then read the output) or read the assembly directly. The latter is faster once you know the vocabulary.

**2. ARM64 (AArch64) registers, in one paragraph.**
ARM64 has 31 general-purpose 64-bit registers named `x0` through `x30`, plus a stack pointer `sp`, a zero register `xzr`, and a frame pointer `x29`. Each `xN` register has a 32-bit view called `wN` — writing to `w0` writes the low 32 bits of `x0` and zeroes the high 32 bits. The calling convention passes the first eight arguments in `x0`–`x7`; return values come back in `x0`. For 32-bit values, that means `w0` holds the first argument and the return value. `w0`, `w1` are the "function arguments 1 and 2, and the return value" for this challenge. `x29` is the frame pointer, `x30` is the link register (return address), and `x19` is a callee-saved register `main` uses to hold the first argument across a function call.

**3. The stack frame instructions.**
You will see a lot of `str` / `ldr` / `stp` / `ldp` / `add sp, sp, #N` / `sub sp, sp, #N` lines. These are just saving registers onto a scratch area on the stack so the function can use them. They are noise for the comparison. Same as x86's `push` / `pop` / `mov [rbp-0x4], edi` boilerplate in the `bit-o-asm-*` challenges — they allocate stack space, save callee-saved registers, and tear down at the end. They do not affect the result.

**4. The `bls` instruction — "branch if lower or same" (unsigned).**
This is the only non-boilerplate instruction in `func1`. `cmp w1, w0` sets the flags based on `w1 - w0`. `bls .L2` means "branch to `.L2` if `w1` is **lower than or equal to** `w0`, *interpreted as an unsigned 32-bit comparison*." In other words:

- If `w1 <= w0` (unsigned): jump to `.L2`.
- Otherwise: fall through (return `w1`).

The signed version of this instruction is `ble` ("branch if less or equal"); the unsigned version is `bls` (L for "lower"). For this challenge the inputs are both positive and fit comfortably in `int32`, so signed and unsigned give the same answer — but in general the distinction matters when a number is greater than `0x7FFFFFFF` and could be negative in a signed view.

**5. The function `func1` is a max().**
Once you see the `bls` plus the two paths (return `w1` on the fall-through, return `w0` on the branch target), the function collapses to "return the larger of the two unsigned 32-bit arguments." Same as a textbook `max(a, b)`. There is nothing else going on.

**6. `main` is just `atoi` twice and a `printf`.**
`main` reads `argv[1]` and `argv[2]`, calls `atoi` on each, calls `func1`, and prints the result with `"Result: %ld\n"`. So the program prints `max(atoi(argv[1]), atoi(argv[2]))` as a signed long decimal. Our job is to compute that max, then format it as a 32-bit lowercase hex string with no `0x` prefix, zero-padded to 8 characters (per the flag-format example: `5614267 → 0055aabb`).

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The challenge ships one file: `chall.s`.

### 1. Move the file into a working directory and look at it

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ mkdir -p ~/armssembly-0 && cd ~/armssembly-0

┌──(zham㉿kali)-[~/armssembly-0]
└─$ cp ~/Downloads/chall.s .

┌──(zham㉿kali)-[~/armssembly-0]
└─$ wc -l chall.s
60 chall.s
```

60 lines. Two functions (`func1` and `main`), a `.rodata` section with the `"Result: %ld\n"` format string, and a `.note.GNU-stack` directive at the end. Skim the file to find the two functions:

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ grep -n 'func1\|main\|\.L[0-9]\|\.LC0\|bl ' chall.s
8:func1:
21:	bls	.L2
30:.L2:
33:.LC0:
36:main:
51:	bl	atoi
57:	bl	atoi
62:	bl	func1
71:	bl	printf
```

The interesting calls are `bls` (one — the comparison) and four `bl` (call to `atoi` twice, `func1` once, `printf` once). That is the entire control flow of the program.

### 2. Read `func1` — the comparison

`func1` is short. The `str` and `ldr` at the top save `w0` and `w1` to the stack so the function can use those registers freely without losing the arguments. After the saves, the function loads the *swapped* values back into `w1` and `w0` (purely so the next `cmp` line reads naturally), performs the unsigned `<=` comparison, and picks the larger.

```
func1:
    sub  sp, sp, #16
    str  w0, [sp, 12]              ; save arg1 to [sp+12]
    str  w1, [sp, 8]               ; save arg2 to [sp+8]
    ldr  w1, [sp, 12]              ; w1 = arg1
    ldr  w0, [sp, 8]               ; w0 = arg2
    cmp  w1, w0                    ; flags = arg1 - arg2
    bls  .L2                       ; if arg1 <= arg2 (unsigned), jump
    ldr  w0, [sp, 12]              ; else w0 = arg1  (arg1 was bigger)
    b    .L3                       ; jump to epilogue
.L2:
    ldr  w0, [sp, 8]               ; w0 = arg2  (arg2 was bigger or equal)
.L3:
    add  sp, sp, 16
    ret
```

Translate to C: `return (a <= b) ? b : a;` — that is a max. (Reading `bls` as "branch if arg1 is *lower or same* than arg2, take the arg2 path; otherwise the fall-through returns arg1.") So `func1(a, b) = max(a, b)` as an unsigned 32-bit integer.

### 3. Read `main` — the wrapper

`main` is the standard ARM64 prologue, the `argv` indexing, two `atoi` calls, one `func1` call, and one `printf`:

```
main:
    stp  x29, x30, [sp, -48]!      ; save frame pointer + return address
    add  x29, sp, 0                ; set up frame
    str  x19, [sp, 16]             ; save callee-saved x19
    str  w0, [x29, 44]             ; save argc
    str  x1, [x29, 32]             ; save argv
    ldr  x0, [x29, 32]             ; x0 = argv
    add  x0, x0, 8                 ; x0 = &argv[1]  (one pointer = 8 bytes)
    ldr  x0, [x0]                  ; x0 = argv[1]   (the first user string)
    bl   atoi                      ; w0 = atoi(argv[1])
    mov  w19, w0                   ; w19 = atoi(argv[1])  (cache in callee-saved)
    ldr  x0, [x29, 32]
    add  x0, x0, 16                ; x0 = &argv[2]
    ldr  x0, [x0]                  ; x0 = argv[2]   (the second user string)
    bl   atoi                      ; w0 = atoi(argv[2])
    mov  w1, w0                    ; w1 = atoi(argv[2])
    mov  w0, w19                   ; w0 = atoi(argv[1])
    bl   func1                     ; w0 = max(atoi(argv[1]), atoi(argv[2]))
    mov  w1, w0                    ; w1 = the max
    adrp x0, .LC0                  ; load .LC0 page
    add  x0, x0, :lo12:.LC0        ; load .LC0 page offset
    bl   printf                    ; printf("Result: %ld\n", max)
    mov  w0, 0                     ; return 0
    ldr  x19, [sp, 16]             ; restore x19
    ldp  x29, x30, [sp], 48        ; restore frame + return
    ret
```

In C this is:

```c
int main(int argc, char **argv) {
    int a = atoi(argv[1]);
    int b = atoi(argv[2]);
    printf("Result: %ld\n", (long)func1(a, b));
    return 0;
}
```

Or shorter: `printf("Result: %ld\n", max(atoi(argv[1]), atoi(argv[2])));`.

### 4. Pick the larger of the two arguments

The challenge gives us `a = 2593949075` and `b = 2233560849`. No need for the binary yet:

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ python3 -c "print(max(2593949075, 2233560849))"
2593949075
```

So the program will print `Result: 2593949075`. The flag is the 32-bit lowercase hex of that number.

### 5. Convert the integer to 32-bit hex

The flag format example in the challenge says `5614267 → 0055aabb`. That is *zero-padded to 8 hex digits*, lowercase, no `0x` prefix. The Python idiom is `:08x`:

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ python3 -c "print(2593949075, '->', format(2593949075, '08x'))"
2593949075 -> 9a9c8593
```

So the hex is `9a9c8593` (already 8 digits, no padding needed in this case), and the flag is:

```
picoCTF{9a9c8593}
```

---

## Alternative Solves

**A. Skip the reading — build and run the binary.**
The cleanest sanity-check is to compile `chall.s` and run it. ARM64 cross-toolchain + QEMU is a one-shot install on Kali:

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ sudo apt install -y gcc-aarch64-linux-gnu qemu-user qemu-user-static
```

Compile statically so QEMU does not need the target dynamic linker (much faster, and works even on a clean container):

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ aarch64-linux-gnu-gcc -static -o chall chall.s

┌──(zham㉿kali)-[~/armssembly-0]
└─$ file chall
chall: ELF 64-bit LSB executable, ARM aarch64, version 1 (GNU/Linux), statically linked, ...

┌──(zham㉿kali)-[~/armssembly-0]
└─$ qemu-aarch64-static ./chall 2593949075 2233560849
Result: 2593949075
```

Same answer. Now convert to hex (same step as in the main solve):

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ python3 -c "print(format(2593949075, '08x'))"
9a9c8593
```

Flag: `picoCTF{9a9c8593}`. This path is the right move if you do not yet trust your reading of the assembly, or if the next challenge in the series (`ARMssembly 1`, `2`, `3`) is too long to read by hand.

**B. Skip ARM entirely — replicate `func1` in Python.**
`func1` is `max(a, b)` with the comparison done as unsigned 32-bit. In Python integers are arbitrary precision, so to mimic the unsigned 32-bit behavior on a value that could exceed `0x7FFFFFFF`, mask with `0xFFFFFFFF` before comparing:

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ python3 -c "
def func1(a, b):
    a &= 0xFFFFFFFF
    b &= 0xFFFFFFFF
    return a if a > b else b

import sys
a = int(sys.argv[1]) & 0xFFFFFFFF
b = int(sys.argv[2]) & 0xFFFFFFFF
m = func1(a, b)
print('Result:', m)
print('Flag:   picoCTF{' + format(m, '08x') + '}')
" 2593949075 2233560849
Result: 2593949075
Flag:   picoCTF{9a9c8593}
```

This re-implementation is overkill for the values in this challenge (both fit in `int32` so the mask is a no-op), but the same template generalizes to the harder ARM challenges in the series where the program is doing real bit-twiddling (`and`, `orr`, `eor`, `lsl`, `lsr`, `asr`) and the inputs are sometimes chosen to exceed `0x7FFFFFFF` on purpose.

**C. Use GDB on the cross-compiled binary.**
If you have aarch64 GDB and prefer interactive debugging:

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ sudo apt install -y gdb-multiarch
```

Start the binary under QEMU paused for a debugger, then attach:

```
┌──(zham㉿kali)-[~/armssembly-0]
└─$ qemu-aarch64-static -g 1234 ./chall 2593949075 2233560849 &
[1] 1234

┌──(zham㉿kali)-[~/armssembly-0]
└─$ gdb-multiarch -ex 'set architecture aarch64' \
                  -ex 'file chall' \
                  -ex 'target remote :1234' \
                  -ex 'b *func1+24' \
                  -ex 'c' \
                  -ex 'info registers w0 w1 pc' \
                  -ex 'si' \
                  -ex 'info registers pc' \
                  -ex 'finish' \
                  -ex 'print/x $w0' chall
```

`func1+24` is the `b.ls` (a.k.a. `bls`) conditional-branch instruction — breakpoint there, `c` to run, and the registers window shows `w0 = 2233560849` and `w1 = 2593949075` (the function puts its second argument in `w0` and the first in `w1` after the swap, but the values are the same two numbers). `si` steps one instruction; the PC will jump to `0x4006f0` (the fall-through) because `b.ls` was *not* taken. `finish` runs the rest of `func1` and prints the return value. `print/x $w0` shows `0x9a9c8593` — the flag in hex form. Useful when the next challenge in the series hides the comparison inside a longer bit-manipulation pipeline.

---

## What Happened Internally

1. Linux / QEMU loaded the ELF and called `_start`, which set up `argc` in `w0` and `argv` in `x1` and then called `main`.
2. `main` saved the frame pointer (`x29`) and link register (`x30`) on the stack with `stp x29, x30, [sp, -48]!`. The `!` means "pre-index" — the stack pointer is decremented *before* the store, and the new `sp` is written back. The 48 bytes give us room to save the `argc` and `argv` values plus the callee-saved register `x19`.
3. `add x29, sp, 0` set up the frame pointer so locals can be addressed as `[x29+44]`, `[x29+32]`, etc. (This is the ARM64 equivalent of x86's `mov rbp, rsp`.)
4. `str w0, [x29, 44]` saved `argc`, and `str x1, [x29, 32]` saved `argv`.
5. `ldr x0, [x29, 32]` reloaded `argv`, then `add x0, x0, 8` advanced by 8 bytes (one pointer width) to point at `argv[1]`. `ldr x0, [x0]` dereferenced that pointer to get the `char *` for the first user argument. `bl atoi` converted it to `int` and returned it in `w0`. The result (`2593949075`) was moved to `w19` for safekeeping — `x19` is callee-saved, so the upcoming `atoi` call will preserve it.
6. The same dance happened for `argv[2]`: reload `argv`, add 16 (to point at `argv[2]`), dereference, `bl atoi`, and the result (`2233560849`) lands in `w0`. Then `mov w1, w0` and `mov w0, w19` set up the argument registers `x0` and `x1` (well, `w0` and `w1` for 32-bit calls) for `func1`.
7. `bl func1` jumped to `func1`. The function:
   - Allocated 16 bytes of stack with `sub sp, sp, #16`.
   - Saved `w0` (the larger of the two would be returned in `w0`; we stash a copy at `[sp+12]`) and `w1` (stashed at `[sp+8]`).
   - Re-loaded `w1 ← [sp+12]` (the first arg) and `w0 ← [sp+8]` (the second arg). This swap is purely cosmetic — it puts the operands in the order the `cmp` line wants, but the original values are still in the stack slots, so the branch targets can pick either one to return.
   - `cmp w1, w0` set the flags based on `w1 - w0` (i.e., `arg1 - arg2`).
   - `bls .L2` checked "is `w1` lower than or equal to `w0` (unsigned)?" Since `2593949075 > 2233560849`, the answer was no. The branch was *not* taken.
   - The fall-through loaded `w0 ← [sp+12]` (the saved first arg, `2593949075`) and jumped to `.L3`.
   - `.L3` deallocated the stack with `add sp, sp, #16` and returned. The return value `2593949075` was in `w0`.
8. Back in `main`, `mov w1, w0` moved the result into the `w1` register — the second argument to `printf`. `adrp x0, .LC0` + `add x0, x0, :lo12:.LC0` formed the address of the `"Result: %ld\n"` string. `bl printf` wrote the output to stdout. `%ld` interprets the second argument as a `long` and prints it as a signed decimal — but `2593949075` is positive in both signed and unsigned views, so it prints as `2593949075`.
9. `main` set `w0 = 0` (the C `return 0`), restored the callee-saved registers, and `ret`-ed.
10. Linux / QEMU received the exit code `0` and the process ended. The final visible output was `Result: 2593949075`, which converts to `0x9a9c8593`, which becomes the flag `picoCTF{9a9c8593}`.

---

## Tools Used

| Tool                          | Why I used it                                              |
|-------------------------------|------------------------------------------------------------|
| Kali Linux (VM)               | My standard CTF environment.                                |
| `cat` / `grep`                | Read the assembly, jumped to the comparison and the `bl` calls. |
| `wc -l`                       | Confirmed the file is 60 lines (small, fully readable by hand). |
| `python3`                     | Computed `max(...)`, formatted the hex, and ran the alternative-solve replica of `func1`. |
| `gcc-aarch64-linux-gnu`       | Cross-compiled `chall.s` into an ARM64 ELF for the build-and-run path. |
| `qemu-user-static`            | Ran the AArch64 binary on my x86-64 host without needing real ARM hardware. |
| `file`                        | Confirmed the cross-compile produced an `aarch64` ELF.       |
| `gdb-multiarch` (alternative) | Optional path for inspecting `w0`/`w1` interactively.       |

---

## Key Takeaways

- ARM64's first eight arguments go in `x0`–`x7`; the return value comes back in `x0`. For 32-bit values, that means `w0`/`w1` and the return is in `w0`. The vast majority of "what does this assembly do?" questions on this platform collapse to "what is in `w0` at the `ret`?".
- The `cmp` family on ARM64 is the same idea as x86 — set flags, then a conditional branch. The mnemonic is the part that tells you signed vs. unsigned: `bge` / `blt` / `ble` / `bgt` are signed; `bhs` / `blo` / `bhi` / `bls` are unsigned. Spotting which one was emitted tells you the data type: `bls` ↔ unsigned, `ble` ↔ signed. For a function that is doing `max(a, b)`, that distinction changes the answer when one of the inputs is above `0x7FFFFFFF`.
- AArch64's `adrp` + `add :lo12:` pair is the modern way to load a symbol address — it is the equivalent of a position-independent `lea` on x86. You do not need to follow the math, just know that "this is how the format string gets loaded for `printf`."
- Cross-compiling + `qemu-user-static` is the standard "I am on x86 Kali, I need to run an ARM binary" recipe. The combination of `gcc-aarch64-linux-gnu` (cross compiler) and `qemu-user-static` (binary translator) lets you exercise *any* aarch64 challenge from the comfort of your existing VM. The `-static` flag in the compile step is what lets QEMU run it without an ARM dynamic linker.
- The flag wordplay: there is no cute leet for this one — `9a9c8593` is just the 32-bit hex of `2593949075`. (`python3 -c "print(hex(2593949075))"` prints `0x9a9c8593`; in memory the four bytes of the integer would be `9a 9c 85 93` in big-endian or `93 85 9c 9a` in little-endian, but the *value* and the `printf("%x")` form are the same on both.) The lesson of the flag is the format itself: 8 hex digits, lowercase, no `0x` prefix, zero-padded. That is the same `0x%08x` C idiom you will see in a thousand `printf` calls — once you internalise it, the flag-format parser in your head works automatically.
