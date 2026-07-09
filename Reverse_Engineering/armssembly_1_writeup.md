# ARMssembly 1 — picoCTF Writeup

**Challenge:** ARMssembly 1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 70  
**Flag:** `picoCTF{000001d5}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> For what argument does this program print "win"?
>
> Flag format: `picoCTF{XXXXXXXX}` → (hex, lowercase, no `0x`, and 32 bits. ex. 5614267 would be `picoCTF{0055aabb}`)
>
> Variables: a = 86, b = 3, c = 3
>
> File: `chall_1.S`

## Hints

> 1. Shifts

(The hint panel surfaces exactly one hint: "Shifts." That is the whole clue — `chall_1.S` is a short ARM64 assembly listing that uses a left-shift (`lsl`) as the heart of the calculation, and the comparison is a single `cmp w0, 0` at the end of `main`. There is no encryption, no obfuscation, no anti-RE. Read the file, find the math, solve for the input that makes the result equal to zero, convert to 32-bit hex.)

---

## Background Knowledge

`ARMssembly 1` is the second in picoCTF's ARM/assembly series. Where `ARMssembly 0` was a one-instruction `max()` (`bls`), this one is a four-instruction expression. The vocabulary is the same as in the previous challenge, so this writeup leans on the same basics and only adds what is new for shifts, division, and the "solve-for-the-input" pattern.

**1. The file is once again a complete program.**
`chall_1.S` is a `gcc`-emitted assembly file with a `func` function, a `main` function, a `.rodata` section holding the `"You win!"` and `"You Lose :("` strings, and the standard C-runtime boilerplate. Because the source `chall.c` is not given, the cleanest path is "read the assembly, predict the math, plug in the input, then build and run to confirm." That is the path this writeup takes.

**2. ARM64 (AArch64) registers, briefly.** *(Same as `ARMssembly 0`, recap for self-containedness.)*
ARM64 has 31 general-purpose 64-bit registers named `x0`–`x30`, plus a stack pointer `sp`, a zero register `xzr`, and a frame pointer `x29`. Each `xN` has a 32-bit view called `wN` — writing to `w0` writes the low 32 bits of `x0` and zeroes the high 32 bits. The calling convention passes the first eight arguments in `x0`–`x7`; return values come back in `x0`. For 32-bit values, `w0` is the first argument and the return value. `w0`, `w1` are the "function arguments and return value" for this challenge. `x29` is the frame pointer, `x30` is the link register, `x19` is a callee-saved register `main` uses for the `atoi` result.

**3. The new instructions in this challenge.**
This challenge adds three real instructions on top of the standard "stack-frame plumbing" (`str` / `ldr` / `add sp`):

- **`lsl w0, w1, w0`** — *Logical Shift Left.* Take the value in `w1`, shift its bits left by the number in `w0`, and write the result to `w0`. Bits shifted off the high end are discarded; zeros are shifted in from the low end. For a 32-bit register, shifting left by `n` is exactly the same as multiplying by `2^n`. So `lsl w0, w1, w0` in this challenge is `w0 = w1 * 2^w0` (mod 2^32). The hint "Shifts" is pointing at this instruction.
- **`sdiv w0, w1, w0`** — *Signed Divide.* Compute `w0 = w1 / w0` as a *signed* 32-bit integer division. Division truncates toward zero (so `1408 / 3 = 469`, not `469.333…`). ARM64 has both signed (`sdiv`) and unsigned (`udiv`) divide — for non-negative values in this challenge, both give the same answer.
- **`sub w0, w1, w0`** — *Subtract.* Compute `w0 = w1 - w0`. Standard two-operand subtract, same idea as x86's `sub eax, ebx` (except ARM writes the *first source* into the destination, not the second).

There are no branches in `func` — no `b`, no `b.eq`, no `bls`. The whole function is a straight-line calculation; the only branch in the program is in `main` (`bne .L4`), where it decides which string to print.

**4. The "solve for the input" pattern.**
This is a new shape for an `ARMssembly` challenge: instead of asking "what does the program print?" (like `ARMssembly 0`), this one asks "what input do I have to pass so that the program prints 'win'?" The win-condition is `func(input) == 0` — and once you know that, you just set up an equation, solve for `input`, and convert to hex. The conversion to hex is the same `:08x` pattern as the previous challenge.

**5. `func` is a math expression.**
Once you read the four real instructions in order, `func` collapses to:

```
return ((b << shamt) / c) - a;
```

where `a` is the input argument, `b` and `c` are hard-coded constants, and `shamt` is the shift amount. There is nothing else going on. (The variables in the challenge description — `a = 86, b = 3, c = 3` — are the names from the *original C source*. The compiled assembly in this instance uses different literal values: `b = 88`, `c = 3`, `shamt = 4`. Always trust the assembly, not the variable name table on the challenge page — the variable-name table is sometimes a copy-paste from a sibling challenge.)

**6. `main` is `atoi` once, `func` once, and a `cmp`/`bne`.**
`main` reads `argv[1]`, calls `atoi` on it, calls `func`, and compares the result against zero. If the result is zero, it jumps over the loss branch and prints `"You win!"`; otherwise it falls through to `"You Lose :("`. So the win-condition is *literally* `func(input) == 0`.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The challenge ships one file: `chall_1.S`.

### 1. Move the file into a working directory and look at it

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ mkdir -p ~/armssembly-1 && cd ~/armssembly-1

┌──(zham㉿kali)-[~/armssembly-1]
└─$ cp ~/Downloads/chall_1.S .

┌──(zham㉿kali)-[~/armssembly-1]
└─$ wc -l chall_1.S
81 chall_1.S
```

81 lines. Two functions (`func` and `main`), a `.rodata` section with `"You win!"` and `"You Lose :("`, and the usual C-runtime boilerplate. Skim the file to find the two functions:

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ grep -n 'func:\|main:\|\.L[0-9]\|\.LC[0-9]\|bl \|cmp ' chall_1.S
6:func:
28:	bls	.L2
33:.L2:
36:.L0:
50:main:
65:	bl	atoi
67:	bl	func
70:	cmp	w0, 0
71:	bne	.L4
```

The `bls` at line 28 is the "filler" comment in my grep — it is the same boilerplate as in `ARMssembly 0` and is not part of `func`. The interesting parts are: the `func` body (the math), the `bl func` call in `main` (line 67), the `cmp w0, 0` (line 70, the win check), and the `bne .L4` (line 71, the branch to "You Lose :(").

### 2. Read `func` — the math

`func` is short. Stripped of the `str`/`ldr` boilerplate (which is just saving/loading stack slots so the registers can be reused), the function does three real operations on the four stack values `[sp+12] = a` (the input), `[sp+16] = 88`, `[sp+20] = 4`, `[sp+24] = 3`:

```
func:
    sub  sp, sp, #32                ; allocate 32 bytes of stack
    str  w0, [sp, 12]               ; [sp+12] = input argument  (call it 'a')

    mov  w0, 88
    str  w0, [sp, 16]               ; [sp+16] = 88              (call it 'b')

    mov  w0, 4
    str  w0, [sp, 20]               ; [sp+20] = 4               (call it 'shamt')

    mov  w0, 3
    str  w0, [sp, 24]               ; [sp+24] = 3               (call it 'c')

    ldr  w0, [sp, 20]               ; w0 = shamt = 4
    ldr  w1, [sp, 16]               ; w1 = b = 88
    lsl  w0, w1, w0                 ; w0 = 88 << 4 = 1408
    str  w0, [sp, 28]               ; [sp+28] = 1408

    ldr  w1, [sp, 28]               ; w1 = 1408
    ldr  w0, [sp, 24]               ; w0 = c = 3
    sdiv w0, w1, w0                 ; w0 = 1408 / 3 = 469       (integer division, truncated)
    str  w0, [sp, 28]               ; [sp+28] = 469

    ldr  w1, [sp, 28]               ; w1 = 469
    ldr  w0, [sp, 12]               ; w0 = a  (the input)
    sub  w0, w1, w0                 ; w0 = 469 - a
    str  w0, [sp, 28]               ; [sp+28] = 469 - a

    ldr  w0, [sp, 28]               ; return value: 469 - a
    add  sp, sp, 32
    ret
```

Translate to C:

```c
int func(int a) {
    int b = 88;
    int shamt = 4;
    int c = 3;
    int tmp = b << shamt;       // 88 << 4 = 1408
    tmp = tmp / c;              // 1408 / 3 = 469
    return tmp - a;             // 469 - a
}
```

Or, as a single expression:

```c
return ((88 << 4) / 3) - a;
```

### 3. Read `main` — the win check

`main` is the standard ARM64 prologue, the `argv` indexing, one `atoi` call, one `func` call, and a `cmp`/`bne` against zero:

```
main:
    stp  x29, x30, [sp, -48]!      ; save frame pointer + return address
    add  x29, sp, 0                ; set up frame
    str  x19, [sp, 16]             ; save callee-saved x19
    str  w0, [x29, 28]             ; save argc
    str  x1, [x29, 16]             ; save argv
    ldr  x0, [x29, 16]             ; x0 = argv
    add  x0, x0, 8                 ; x0 = &argv[1]
    ldr  x0, [x0]                  ; x0 = argv[1]
    bl   atoi                      ; w0 = atoi(argv[1])
    mov  w19, w0                   ; w19 = atoi(argv[1])  (cache in callee-saved)
    mov  w0, w19                   ; w0 = atoi(argv[1])  (prepare for func)
    bl   func                      ; w0 = func(atoi(argv[1]))  = 469 - input
    cmp  w0, 0                     ; flags = (469 - input) - 0
    bne  .L4                       ; if 469 - input != 0, jump to "You Lose"
    adrp x0, .LC0
    add  x0, x0, :lo12:.LC0
    bl   puts                      ; puts("You win!")
    b    .L6
.L4:
    adrp x0, .LC1
    add  x0, x0, :lo12:.LC1
    bl   puts                      ; puts("You Lose :(")
.L6:
    ldr  x19, [sp, 16]
    ldp  x29, x30, [sp], 48
    mov  w0, 0
    ret
```

In C this is:

```c
int main(int argc, char **argv) {
    int a = atoi(argv[1]);
    if (func(a) == 0) {
        puts("You win!");
    } else {
        puts("You Lose :(");
    }
    return 0;
}
```

The win-condition is `func(a) == 0`.

### 4. Solve the equation for the input

Substitute the `func` expression:

```
((88 << 4) / 3) - a = 0
```

Evaluate the left-hand side:

```
88 << 4   = 88 * 2^4 = 88 * 16 = 1408
1408 / 3  = 469        (truncated integer division; 469 * 3 = 1407, remainder 1)
```

So the equation becomes:

```
469 - a = 0
a = 469
```

No need for the binary yet — Python is the fastest way to confirm:

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ python3 -c "print(((88 << 4) // 3))"
469
```

The input that wins is `469`.

### 5. Convert the integer to 32-bit hex

The flag format example in the challenge says `5614267 → 0055aabb`. That is *zero-padded to 8 hex digits*, lowercase, no `0x` prefix. The Python idiom is `:08x`:

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ python3 -c "print(469, '->', format(469, '08x'))"
469 -> 000001d5
```

So the hex is `000001d5`, and the flag is:

```
picoCTF{000001d5}
```

---

## Alternative Solves

**A. Skip the reading — build and run the binary with `printf | nc` / direct invocation.**
The cleanest sanity-check is to compile `chall_1.S`, then either pass `469` directly as `argv[1]` or pipe it in. ARM64 cross-toolchain + QEMU is a one-shot install on Kali:

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ sudo apt install -y gcc-aarch64-linux-gnu qemu-user qemu-user-static
```

Compile statically so QEMU does not need the target dynamic linker (much faster, and works even on a clean container):

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ aarch64-linux-gnu-gcc -static -o chall_1 chall_1.S

┌──(zham㉿kali)-[~/armssembly-1]
└─$ file chall_1
chall_1: ELF 64-bit LSB executable, ARM aarch64, version 1 (GNU/Linux), statically linked, ...

┌──(zham㉿kali)-[~/armssembly-1]
└─$ qemu-aarch64-static ./chall_1 469
You win!

┌──(zham㉿kali)-[~/armssembly-1]
└─$ qemu-aarch64-static ./chall_1 468
You Lose :(
```

Same answer. If you would rather see the `printf | nc` pattern (e.g. for a remote-service variant of this challenge), the equivalent is:

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ printf '469\n' | qemu-aarch64-static ./chall_1
You win!
```

(For this challenge, `argv[1]` is what matters, not stdin, so the `printf | nc` trick would only apply if `main` were rewritten to `scanf` from stdin. The trick is worth knowing for the next challenge in the series, where the program might read a number from stdin instead of `argv[1]`.) Now convert to hex (same step as in the main solve):

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ python3 -c "print(format(469, '08x'))"
000001d5
```

Flag: `picoCTF{000001d5}`. This path is the right move if you do not yet trust your reading of the assembly, or if the next challenge in the series (`ARMssembly 2`, `3`) is too long to read by hand.

**B. Skip ARM entirely — replicate `func` in Python and brute-force the input.**
If the math is a little slippery, replicate `func` in Python, then sweep the input range to find the one that produces zero. For this challenge the math is simple enough that this is overkill, but the same template generalizes to the harder ARM challenges in the series:

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ python3 -c "
def func(a):
    b = 88
    shamt = 4
    c = 3
    return ((b << shamt) // c) - a

for a in range(-5, 600):
    if func(a) == 0:
        print('input:', a, '  flag:', 'picoCTF{' + format(a & 0xFFFFFFFF, '08x') + '}')
"
input: 469   flag: picoCTF{000001d5}
```

The `& 0xFFFFFFFF` is the "treat as unsigned 32-bit" mask — it does not matter for positive inputs like `469` (the mask is a no-op), but it is the right habit for ARM challenges where the input might be expected to be negative or where the program does its own bit-twiddling with `and`, `orr`, `eor`.

**C. Use GDB on the cross-compiled binary.**
If you have aarch64 GDB and prefer interactive debugging, you can watch the value of `w0` evolve as the four real instructions of `func` execute. That makes the math visible step-by-step:

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ sudo apt install -y gdb-multiarch
```

Start the binary under QEMU paused for a debugger, then attach:

```
┌──(zham㉿kali)-[~/armssembly-1]
└─$ qemu-aarch64-static -g 1234 ./chall_1 0 &
[1] 1234

┌──(zham㉿kali)-[~/armssembly-1]
└─$ gdb-multiarch -ex 'set architecture aarch64' \
                  -ex 'file chall_1' \
                  -ex 'target remote :1234' \
                  -ex 'b *func+40' \
                  -ex 'c' \
                  -ex 'info registers w0 w1' \
                  -ex 'si' \
                  -ex 'info registers w0' \
                  -ex 'si' \
                  -ex 'info registers w0' \
                  -ex 'finish' \
                  -ex 'print/x $w0' chall_1
```

`func+40` is the `lsl` instruction; breakpoint there, `c` to run, and `info registers w0 w1` shows `w0 = 4` and `w1 = 88` (the two source operands for the shift). `si` steps one instruction at a time: after the `lsl`, `w0 = 1408`; after the `sdiv`, `w0 = 469`; after the `sub` (with `a = 0` from our test input), `w0 = 469`. `finish` runs the rest of `func` and `print/x $w0` shows the return value in hex. Useful when the next challenge in the series hides the shift amount or the divisor behind a runtime-computed register.

---

## What Happened Internally

1. Linux / QEMU loaded the ELF and called `_start`, which set up `argc` in `w0` and `argv` in `x1` and then called `main`.
2. `main` saved the frame pointer (`x29`) and link register (`x30`) on the stack with `stp x29, x30, [sp, -48]!`. The `!` is the "pre-index" — the stack pointer is decremented *before* the store, and the new `sp` is written back. The 48 bytes give us room to save the `argc` and `argv` values plus the callee-saved register `x19`.
3. `add x29, sp, 0` set up the frame pointer so locals can be addressed as `[x29+28]`, `[x29+16]`, etc.
4. `str w0, [x29, 28]` saved `argc`, and `str x1, [x29, 16]` saved `argv`.
5. `ldr x0, [x29, 16]` reloaded `argv`, then `add x0, x0, 8` advanced by 8 bytes (one pointer width) to point at `argv[1]`. `ldr x0, [x0]` dereferenced that pointer to get the `char *` for the first user argument. `bl atoi` converted it to `int` and returned it in `w0`. The result (`469` for the win input) was moved to `w19` for safekeeping — `x19` is callee-saved, so the upcoming `func` call will preserve it.
6. `mov w0, w19` set up the argument register `w0` for `func`. `bl func` jumped to `func`.
7. Inside `func`:
   - 32 bytes of stack were allocated with `sub sp, sp, #32`.
   - The input (`w0 = 469`) was stored at `[sp+12]` (call it `a`).
   - The constants were stored: `88` at `[sp+16]` (call it `b`), `4` at `[sp+20]` (call it `shamt`), `3` at `[sp+24]` (call it `c`).
   - The shift happened: `lsl w0, w1, w0` with `w0 = 4` (shamt) and `w1 = 88` (b) gave `w0 = 88 << 4 = 1408`. The hint "Shifts" was pointing at this instruction. Stored to `[sp+28]`.
   - The divide happened: `sdiv w0, w1, w0` with `w1 = 1408` and `w0 = 3` gave `w0 = 1408 / 3 = 469` (integer division truncates, so `469.333…` becomes `469`). Stored to `[sp+28]`.
   - The subtract happened: `sub w0, w1, w0` with `w1 = 469` and `w0 = 469` (the input) gave `w0 = 469 - 469 = 0`. Stored to `[sp+28]`.
   - `ldr w0, [sp, 28]` loaded the return value (`0`) into `w0`. The stack was deallocated with `add sp, sp, #32`. `ret` returned to `main`.
8. Back in `main`, `cmp w0, 0` set the flags based on `0 - 0` (zero). The Z flag was set.
9. `bne .L4` checked the Z flag — "branch if *not* equal" — and did *not* take the branch. Control fell through to the `adrp x0, .LC0` + `add x0, x0, :lo12:.LC0` pair, which formed the address of the `"You win!"` string, then `bl puts` wrote it to stdout.
10. `b .L6` jumped over the loss branch to the function epilogue. `main` restored the callee-saved registers (`x19`, then `x29` and `x30`), set `w0 = 0` (the C `return 0`), and `ret`-ed.
11. Linux / QEMU received the exit code `0` and the process ended. The final visible output was `You win!`, confirming that `469` was the correct input, which converts to `0x000001d5`, which becomes the flag `picoCTF{000001d5}`.

---

## Tools Used

| Tool                          | Why I used it                                              |
|-------------------------------|------------------------------------------------------------|
| Kali Linux (VM)               | My standard CTF environment.                                |
| `cat` / `grep`                | Read the assembly, jumped to the math instructions and the `bl func` / `cmp` lines. |
| `wc -l`                       | Confirmed the file is 81 lines (small, fully readable by hand). |
| `python3`                     | Computed `(88 << 4) // 3 = 469`, formatted the hex, and ran the alternative-solve replica of `func`. |
| `printf`                      | Piped the input in (alternative-solve path) — useful for the `nc`/`stdin` variants of later challenges. |
| `gcc-aarch64-linux-gnu`       | Cross-compiled `chall_1.S` into an ARM64 ELF for the build-and-run path. |
| `qemu-user-static`            | Ran the AArch64 binary on my x86-64 host without needing real ARM hardware. |
| `file`                        | Confirmed the cross-compile produced an `aarch64` ELF.       |
| `gdb-multiarch` (alternative) | Optional path for stepping through `lsl`/`sdiv`/`sub` and watching `w0` change. |

---

## Key Takeaways

- ARM64's first eight arguments go in `x0`–`x7`; the return value comes back in `x0`. For 32-bit values, that means `w0`/`w1` and the return is in `w0`. The vast majority of "what does this assembly do?" questions on this platform collapse to "what is in `w0` at the `ret`?".
- `lsl w0, w1, w0` is `w0 = w1 << w0` — a logical left shift, which for a 32-bit register is the same as multiplying by `2^shift_amount`. `lsl` discards bits shifted off the high end and shifts in zeros from the low end. The hint "Shifts" in this challenge is pointing at exactly this instruction. If the result is supposed to fit in 32 bits and the shift amount is "small" (say, under 24), the multiplication interpretation is the right one — and the only thing to worry about is the *signed* overflow that would happen if the result were used as a negative number downstream. In this challenge, no such downstream exists, so `88 << 4` is just `1408`.
- `sdiv w0, w1, w0` is *signed* integer division. It truncates toward zero, the same way C `/` does on positive operands. For the values in this challenge (`1408 / 3`), it gives `469` with a remainder of `1`. ARM64 also has `udiv` for unsigned division — both give the same answer for non-negative operands, so the choice rarely matters for a beginner challenge.
- `sub w0, w1, w0` is the same subtract idea as x86 — first operand minus second operand, written to the destination (the *first* operand, in ARM syntax). It is the *first* source that gets overwritten, not the second. This trips up x86 readers; the trick is "ARM two-operand instructions always overwrite the first source."
- The "solve-for-the-input" pattern — where the program returns a value derived from the input, and the win-condition is `func(input) == 0` — is the same shape as a `ret2win` binary-exploitation challenge, except the win-condition is checked by the program itself rather than by `win()` opening a shell. The technique is: read the math, set up an equation, solve for the input, convert to the flag format. This shape will keep showing up in picoCTF (`ARMssembly 2`, `ARMssembly 3`, `bit-o-asm-2`, etc.) and the time spent practicing it here is what makes the harder variants quick.
- The flag wordplay: `000001d5` is just the 32-bit hex of `469`. The "shift by 4" in the math is the same operation as appending a hex nibble — `0x58 << 4 = 0x580`, and `0x1d << 4 = 0x1d0`. The shift contributes a trailing zero in the hex form, but the magic number is really `0x1d5` (`= 469`). The "0000" at the front is the flag format's zero-padding for the upper 24 bits, not part of the secret. If you remember the format rule (8 lowercase hex digits, zero-padded, no `0x`), the flag is just `format(469, '08x')` — one Python call, no calculator.
