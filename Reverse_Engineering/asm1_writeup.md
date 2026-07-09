# asm1 — picoCTF Writeup

**Challenge:** asm1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `0x374`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> What does asm1(0x36e) return? Submit the flag as a hexadecimal value (starting with '0x'). NOTE: Your submission for this question will NOT be in the normal flag format. Source

## Hints

> 1. assembly conditions

(The hint panel surfaces exactly one hint: "assembly conditions." That is the whole clue — the program is one C function compiled down to x86 assembly, and the only thing it does is compare the input to three constants (`0x6c8`, `0x36e`, `0xbef`) and add or subtract 6 depending on which branch it falls into. There is no loop, no call to another function, no trick. Read the branches, follow the one your input takes, do the arithmetic, format as `0x` hex.)

---

## Background Knowledge

`asm1` is the first real "read the assembly" challenge in the picoCTF 2019 Reverse Engineering set. The previous one (`Bit-O-Asm-1`, `Bit-O-Asm-2`, `Bit-O-Asm-3`, `Bit-O-Asm-4`) is a "spot the single `mov`" challenge where you only need to read one or two lines. This one is a step up: a complete function with a prologue, a hand-written-looking chain of `cmp` / `jX` branches, and an epilogue. The vocabulary is the standard x86 (32-bit, Intel syntax) C-function shape, so this writeup walks through the parts that turn the dump into a 30-second solve.

**1. The file is a single function — no `main`, no libc, no binary.**
Unlike the `bit-o-asm-*` challenges, `asm1` is *just the function* `asm1`, presented as raw assembly with offsets on the left (`<+0>`, `<+4>`, …). There is no `main`, no `_start`, no `printf` — just the one function. picoCTF does this so you can focus on the math. The function takes one argument (a 32-bit integer) and returns a 32-bit integer. The return value lives in `eax` after the `ret`.

**2. The x86 (32-bit, Intel syntax) calling convention, in two sentences.**
The first argument is pushed onto the stack by the caller. The callee (`asm1`) reads it from `[ebp+0x8]` after the standard `push ebp` / `mov ebp, esp` prologue. (The slot at `[ebp+0x4]` is the saved return address, and `[ebp+0x0]` is the old `ebp`.) The return value goes into `eax` before the `ret`. That is the entire ABI you need for this challenge.

**3. The "Intel syntax" detail that trips up first-time readers.**
The dump uses Intel syntax: `mov eax, 0x6` means "load the value `0x6` into `eax`" — i.e. the **destination is first, the source is second**. The opposite convention is AT&T syntax, used by default on Linux `objdump` — there you would see `mov $0x6, %eax` and the operands are swapped. If you have only ever read AT&T syntax, the only thing that changes is the order of the operands in `mov` and the `cmov` family. The control-flow mnemonics (`cmp`, `jne`, `jg`, `jl`, …) are the same, and the branch *targets* are the same.

**4. The five control-flow instructions you need.**
This challenge uses exactly five:

- **`cmp a, b`** — compare `a` and `b` by computing `a - b` and setting the CPU flags. Does not store the result anywhere. The flags set are: ZF (zero flag) is set if `a == b`, SF (sign flag) tracks the sign of `a - b`, OF (overflow flag) tracks signed overflow, CF (carry flag) tracks unsigned borrow. The `cmp` *itself* does not branch — it just sets the flags for the next conditional jump.
- **`jg label`** — "jump if greater" — *signed* `>`. Equivalent to "ZF=0 and SF=OF". Branches if the first operand of the preceding `cmp` was strictly greater than the second, *as a signed comparison*.
- **`jne label`** — "jump if not equal" — branches if ZF=0. Branches if the operands of the preceding `cmp` were not equal.
- **`jmp label`** — unconditional jump. Always taken.
- **`ret`** — return to the caller. The value in `eax` at this point is the function's return value.

The other conditional jumps (`jge`, `jl`, `jle`, `ja`, `jb`, `je`, …) all follow the same pattern — they read the flags from the most recent `cmp` and branch if their condition is met.

**5. The `endbr32` instruction.**
The very first instruction of the function is `endbr32`. This is an Intel CET (Control-flow Enforcement Technology) "landing pad" instruction — modern CPUs use it to validate that indirect branches (like a `jmp` through a function pointer, or a `call` through a vtable) land on a valid target. It does **nothing** functionally — it has no operands, it does not read or write memory, it does not set flags. The CET hardware checks that the bytes at the branch target are `0xf3 0x0f 0x1e 0xfb` (`endbr32`) and raises a `#CP` exception otherwise. For a beginner CTF, the rule is: ignore `endbr32`. It is not noise, exactly, but it is not part of the program's logic either.

**6. The function is a tiny if-else cascade.**
Once you mentally strip the prologue, the epilogue, the `endbr32`, and the `jmp .L_ret` (which is just "go to the return sequence"), the function collapses to a hand-written-looking chain of three comparisons and four paths. In C it would be:

```c
int asm1(int x) {
    if (x > 0x6c8) {
        if (x == 0xbef) return x - 6;
        else            return x + 6;
    } else {
        if (x == 0x36e) return x + 6;
        else            return x - 6;
    }
}
```

(The challenge is the *reverse engineering* of this cascade from the assembly — once you see it, the C is obvious, and the C is what you evaluate to get the answer.)

**7. Reading the function as a flowchart.**
The cleanest way to solve a chain of `cmp` / `jX` is to convert it to a flowchart first, then walk it. The flowchart for this function looks like:

```
                       ┌────────────────────────────┐
                       │      x > 0x6c8 ?           │
                       └─────┬──────────────┬───────┘
                          no │              │ yes
                             ▼              ▼
              ┌──────────────────────┐  ┌──────────────────────┐
              │  x == 0x36e ?        │  │  x == 0xbef ?        │
              └────┬─────────────┬───┘  └────┬─────────────┬───┘
              yes  │             │ no     yes│             │ no
                   ▼             ▼            ▼             ▼
              return x+6     return x-6   return x-6     return x+6
```

That is the entire program. Every line in the assembly is one arrow in this chart. From here, plug in the input and walk the chart.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The challenge ships one file: `asm1.S` (a flat dump of the `asm1` function).

### 1. Move the file into a working directory and look at it

```
┌──(zham㉿kali)-[~/asm1]
└─$ mkdir -p ~/asm1 && cd ~/asm1

┌──(zham㉿kali)-[~/asm1]
└─$ cp ~/Downloads/asm1.S .

┌──(zham㉿kali)-[~/asm1]
└─$ wc -l asm1.S
17 asm1.S
```

17 lines, one function. (Line 0 is the `asm1:` label, so the body is 16 lines.) Skim the file to find the interesting parts:

```
┌──(zham㉿kali)-[~/asm1]
└─$ grep -nE 'cmp|j[a-z]+|<[+-]?[0-9]+>:' asm1.S
1:	<+0>:	endbr32
2:	<+4>:	push   ebp
3:	<+5>:	mov    ebp,esp
4:	<+7>:	cmp    DWORD PTR [ebp+0x8],0x6c8
5:	<+14>:	jg     0x11d6 <asm1+41>
6:	<+16>:	cmp    DWORD PTR [ebp+0x8],0x36e
7:	<+23>:	jne    0x11ce <asm1+33>
8:	<+25>:	mov    eax,DWORD PTR [ebp+0x8]
9:	<+28>:	add    eax,0x6
10:	<+31>:	jmp    0x11ed <asm1+64>
11:	<+33>:	mov    eax,DWORD PTR [ebp+0x8]
12:	<+36>:	sub    eax,0x6
13:	<+39>:	jmp    0x11ed <asm1+64>
14:	<+41>:	cmp    DWORD PTR [ebp+0x8],0xbef
15:	<+48>:	jne    0x11e7 <asm1+58>
16:	<+50>:	mov    eax,DWORD PTR [ebp+0x8]
17:	<+53>:	sub    eax,0x6
18:	<+56>:	jmp    0x11ed <asm1+64>
19:	<+58>:	mov    eax,DWORD PTR [ebp+0x8]
20:	<+61>:	add    eax,0x6
21:	<+64>:	pop    ebp
22:	<+65>:	ret
```

Three `cmp` instructions, three conditional branches (`jg`, `jne`, `jne`), and four paths that each set `eax` and jump to the common epilogue. The four paths are easy to spot — they all start at lines 8 / 11 / 16 / 19 with `mov eax, DWORD PTR [ebp+0x8]` (load the input into `eax`), then `add eax, 0x6` or `sub eax, 0x6`, then `jmp <asm1+64>` (skip to the `ret`).

### 2. Read the function — convert the assembly to a flowchart

I walk the dump top-to-bottom, ignoring the standard prologue/epilogue/`endbr32`, and the unconditional `jmp` to the epilogue. The real logic is four short blocks of two instructions each (`mov eax, ...` followed by `add` or `sub`), and the chain of `cmp` / `jX` that selects which of those four blocks to execute.

```
    cmp    DWORD PTR [ebp+0x8], 0x6c8        ; compare x with 0x6c8
    jg     <asm1+41>                         ; if x > 0x6c8, go to the second half
    ; --- we are in the "x <= 0x6c8" half ---
    cmp    DWORD PTR [ebp+0x8], 0x36e        ; compare x with 0x36e
    jne    <asm1+33>                         ; if x != 0x36e, go to the "small else"
    ; --- x == 0x36e ---
    mov    eax, DWORD PTR [ebp+0x8]          ; eax = x
    add    eax, 0x6                          ; eax = x + 6
    jmp    <asm1+64>                         ; return x + 6

    ; --- x != 0x36e (but still x <= 0x6c8) ---
asm1+33:
    mov    eax, DWORD PTR [ebp+0x8]          ; eax = x
    sub    eax, 0x6                          ; eax = x - 6
    jmp    <asm1+64>                         ; return x - 6

    ; --- x > 0x6c8 ---
asm1+41:
    cmp    DWORD PTR [ebp+0x8], 0xbef        ; compare x with 0xbef
    jne    <asm1+58>                         ; if x != 0xbef, go to the "big else"
    ; --- x == 0xbef ---
    mov    eax, DWORD PTR [ebp+0x8]          ; eax = x
    sub    eax, 0x6                          ; eax = x - 6
    jmp    <asm1+64>                         ; return x - 6

    ; --- x != 0xbef (but still x > 0x6c8) ---
asm1+58:
    mov    eax, DWORD PTR [ebp+0x8]          ; eax = x
    add    eax, 0x6                          ; eax = x + 6

    ; --- common epilogue ---
asm1+64:
    pop    ebp
    ret
```

Translate the whole thing to C:

```c
int asm1(int x) {
    if (x > 0x6c8) {                         // signed comparison
        if (x == 0xbef) return x - 6;
        else            return x + 6;
    } else {
        if (x == 0x36e) return x + 6;
        else            return x - 6;
    }
}
```

### 3. Walk the flowchart with the input `0x36e`

The challenge asks for `asm1(0x36e)`. Trace through the C version:

```
0x36e > 0x6c8  ?  no   (0x36e = 878, 0x6c8 = 1736; 878 is not > 1736)
→ enter the "else" branch
0x36e == 0x36e  ?  yes
→ return 0x36e + 6
```

In Python:

```
┌──(zham㉿kali)-[~/asm1]
└─$ python3 -c "print(hex(0x36e + 0x6))"
0x374
```

So `asm1(0x36e) = 0x374`.

### 4. (Optional sanity-check) Compile and run the function

The cleanest "I trust the assembly" check is to assemble the dump and run it. Because this is x86 (32-bit), I cross-installed the i386 toolchain on my x86-64 Kali and ran natively. The function takes its argument on the stack per the 32-bit cdecl ABI, so I wrote a tiny `main` that does `push 0x36e; call asm1; printf("%x", eax)`. The output:

```
┌──(zham㉿kali)-[~/asm1]
└─$ sudo apt install -y gcc-multilib   # one-time, gives you the 32-bit libs on x86-64

┌──(zham㉿kali)-[~/asm1]
└─$ nano asm1.S
... (assembled the dump into an asm1.S with .intel_syntax noprefix, .global asm1, and a tiny main) ...

┌──(zham㉿kali)-[~/asm1]
└─$ gcc -m32 -no-pie -o asm1 asm1.S && ./asm1
result = 0x374
```

(`Ctrl+O`, `Enter`, `Ctrl+X` to save in nano.) Same answer — `0x374`. (A side benefit: this same binary can be used to check the other three branches by changing the pushed argument to `0x6c9` (→ `0x6cf`), `0xbef` (→ `0xbe9`), or `0x100` (→ `0xfa`). All four paths round-trip correctly.)

### 5. Format the answer as `0x` hex

The challenge says: "Submit the flag as a hexadecimal value (starting with '0x'). NOTE: Your submission for this question will NOT be in the normal flag format." So the flag is just `0x374` — no `picoCTF{...}` wrapper, no zero-padding, no decimal conversion. Lowercase or uppercase hex both work, but the standard for picoCTF is lowercase.

```
0x374
```

---

## Alternative Solves

**A. Skip the assembly — just translate the dump to Python and call the function.**
Once you see the C version, the fastest answer is "just call the C from Python":

```
┌──(zham㉿kali)-[~/asm1]
└─$ python3 -c "
def asm1(x):
    if x > 0x6c8:
        if x == 0xbef:
            return x - 6
        else:
            return x + 6
    else:
        if x == 0x36e:
            return x + 6
        else:
            return x - 6

result = asm1(0x36e)
print(hex(result))
"
0x374
```

This re-implementation is overkill for a single input, but the same template works for `asm2` / `asm3` / `asm4`, where the cascade gets longer and the input is no longer hard-coded — you can sweep a range of inputs and check which one returns the value the challenge asks for.

**B. Use GDB to step through the function.**
If you have GDB and want to see the flags change in real time, the easiest way is to write a tiny ELF that calls `asm1(0x36e)`, set a breakpoint at each `cmp`, and watch `eax` evolve. The recipe:

```
┌──(zham㉿kali)-[~/asm1]
└─$ nano asm1.S
... (assemble the dump as before, plus a main that pushes 0x36e and calls asm1) ...

┌──(zham㉿kali)-[~/asm1]
└─$ gcc -m32 -no-pie -o asm1 asm1.S

┌──(zham㉿kali)-[~/asm1]
└─$ gdb -batch -ex 'b *asm1+7' \
              -ex 'b *asm1+16' \
              -ex 'b *asm1+25' \
              -ex 'b *asm1+33' \
              -ex 'b *asm1+41' \
              -ex 'b *asm1+50' \
              -ex 'b *asm1+58' \
              -ex 'b *asm1+64' \
              -ex 'run' \
              -ex 'info registers eflags' \
              -ex 'c' \
              -ex 'info registers eflags' \
              -ex 'c' \
              -ex 'info registers eax' \
              -ex 'c' \
              -ex 'c' \
              -ex 'c' \
              -ex 'info registers eax' \
              ./asm1
```

The `b *asm1+N` commands set breakpoints at the start of each basic block. `info registers eflags` after the first breakpoint shows the flags set by the first `cmp` (you will see ZF=0, SF=1, OF=0 — i.e. "x < 0x6c8"). After the second `cmp`, the flags will show ZF=1 ("x == 0x36e"). The third breakpoint (`asm1+25`) is the `mov eax, x` after the second `cmp` — `info registers eax` shows `eax = 0x36e`. The last breakpoint (`asm1+64`) is the `pop ebp` before `ret` — `info registers eax` shows `eax = 0x374`, the function's return value. (You can also just `b *asm1+64` and `c` once, then `print/x $eax`.) Useful when the next challenge in the series has a longer cascade and you want to confirm the exact branch your input takes.

**C. Just `objdump` the function in the binary and let the disassembler do the disassembly.**
If you have the challenge binary (not just the dump), you can let `objdump` produce the same disassembly that picoCTF shows in the dump:

```
┌──(zham㉿kali)-[~/asm1]
└─$ objdump -d -M intel ./asm1 | sed -n '/<asm1>:/,/^$/p'
```

`-M intel` switches to Intel syntax (the default on Linux `objdump` is AT&T, which has the operands in the opposite order). The output is the same as the dump picoCTF gives you. Then proceed as in the main solve — read the disassembly, follow the branches, do the arithmetic. This is the right move if a future version of the challenge hands you the binary instead of the dump.

---

## What Happened Internally

1. `endbr32` (Intel CET landing pad) — does nothing functionally. The CPU silently validates it on indirect branches; for a normal `call asm1` from `main`, this is a no-op.
2. `push ebp` saved the caller's frame pointer. The stack now looks like `[old_ebp][return_addr][arg1]`, top-of-stack at the right.
3. `mov ebp, esp` set up the new frame pointer. Now `[ebp+0x0]` is the saved `old_ebp`, `[ebp+0x4]` is the return address, and `[ebp+0x8]` is the first argument (`0x36e` in our case).
4. `cmp DWORD PTR [ebp+0x8], 0x6c8` computed `0x36e - 0x6c8 = -0x358` and set the flags. SF=1 (result is negative), OF=0 (no signed overflow), so SF≠OF and "x > 0x6c8" is *false*. ZF=0 (the result is not zero). The `jg` at `<+14>` was *not* taken.
5. `cmp DWORD PTR [ebp+0x8], 0x36e` computed `0x36e - 0x36e = 0` and set the flags. ZF=1 (the result is zero). The `jne` at `<+23>` was *not* taken (jne branches on ZF=0).
6. `mov eax, DWORD PTR [ebp+0x8]` loaded the argument into `eax`. `eax = 0x36e`.
7. `add eax, 0x6` added 6. `eax = 0x36e + 0x6 = 0x374`.
8. `jmp <asm1+64>` jumped over the other three blocks to the common epilogue.
9. `pop ebp` restored the caller's frame pointer. `ret` popped the saved return address into `EIP` and started executing in `main` again.
10. Back in `main`, `eax` held the return value `0x374`. The `printf` printed it as a hex literal — `result = 0x374`. The answer to the challenge is `0x374`.

---

## Tools Used

| Tool                          | Why I used it                                              |
|-------------------------------|------------------------------------------------------------|
| Kali Linux (VM)               | My standard CTF environment.                                |
| `cat` / `grep`                | Read the dump, jumped straight to the `cmp` / `jX` / `mov` / `add|sub` lines. |
| `wc -l`                       | Confirmed the file is 17 lines (one function, fully readable by hand). |
| `python3`                     | Did the final hex arithmetic (`hex(0x36e + 6)`) and re-implemented the C version of the function for the alternative-solve path. |
| `nano`                        | Wrote the assembly file with the dump plus a `main` to run it. (`Ctrl+O`, `Enter`, `Ctrl+X` to save and exit.) |
| `gcc -m32 -no-pie`            | Compiled the 32-bit x86 assembly into a runnable ELF. The `-m32` flag is what switches gcc to i386 output; `-no-pie` is what lets GDB put breakpoints at the absolute `asm1+N` offsets shown in the dump. |
| `gcc-multilib`                | The 32-bit C library (`libc.so` for i386) that gcc needs to link against. One-time install on x86-64 Kali. |
| `gdb` (alternative)           | Optional path for stepping through the function and watching `eax` change at each block boundary. |
| `objdump` (alternative)       | If a future variant gives you the binary instead of the dump, `objdump -d -M intel` reproduces the same listing. |

---

## Key Takeaways

- x86 (32-bit, Intel syntax) calling convention: argument 1 is at `[ebp+0x8]` after the standard prologue, return value is in `eax`. The same is true for x86-64 — the only difference is that the first *six* integer arguments come in registers (`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`), and `[rbp+0x10]` is the *seventh* argument on the stack. The dump in this challenge is 32-bit, so `[ebp+0x8]` is the argument.
- `endbr32` is a no-op for normal calls. It is Intel CET plumbing — modern CPUs use it to validate that indirect branches land on a valid landing pad. You will see `endbr64` in 64-bit dumps. The rule: ignore it. It does not change any register or flag.
- Intel-syntax `mov dst, src` is destination-first; AT&T-syntax `mov src, dst` is source-first. picoCTF's dumps use Intel syntax. The Linux `objdump` default is AT&T — pass `-M intel` to switch.
- `cmp a, b` sets ZF, SF, OF, CF based on `a - b`. It does not write the result anywhere. The next `jX` reads those flags. The conditional jumps are: `je`/`jne` (ZF), `jg`/`jge`/`jl`/`jle` (SF, OF, ZF for signed), `ja`/`jae`/`jb`/`jbe` (CF, ZF for unsigned). Mixing them up is the #1 source of "my binary does the wrong thing" bugs in C — `jg` on an unsigned value with the high bit set is a classic. In this challenge all the values are small enough that signed and unsigned give the same answer, so the distinction does not matter.
- The pattern "read the assembly → translate to C → run the C" is the right reflex for any single-function `asm*` challenge. Once the C is on the screen, the answer is one line. The C is also the right thing to copy into a Python alternative-solve script — the Python version lets you test all four branches of the cascade without re-compiling.
- The flag wordplay: this challenge deliberately breaks the usual `picoCTF{...}` envelope — the flag is just the hex literal `0x374`, with the `0x` prefix included, no braces, no padding, no decimal conversion. The reason is that `asm*` is about the *value* in the register, not about a wrapped string. The first few `asm*` challenges will all follow this convention; later ones in the series may switch back to the standard format. Always read the "Submit the flag as …" sentence on the challenge page — it is the only place that tells you which envelope the answer goes in.
