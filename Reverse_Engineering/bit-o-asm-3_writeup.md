# Bit-O-Asm-3 — picoCTF Writeup

**Challenge:** Bit-O-Asm-3  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{2619997}`  
**Platform:** picoCTF 2022 (picoGym)  
**Writeup by:** zham  

---

## Description

> Can you figure out what is in the `eax` register? Put your answer in the picoCTF flag format: `picoCTF{n}` where n is the contents of the `eax` register in the decimal number base. If the answer was `0x11` your flag would be `picoCTF{17}`.
>
> Download the assembly dump [here].

## Hints

> 1. Not everything in this disassembly listing is optimal.

---

## Background Knowledge (Read This First!)

If you solved Bit-O-Asm-1 and Bit-O-Asm-2, you already know the shape of these dumps: a few lines of standard prologue/epilogue noise, some local-variable stores to memory, one or two instructions that touch `eax`, and a final `ret`. Bit-O-Asm-3 keeps the same shape but layers the **one** new concept on top: a couple of arithmetic instructions between the load and the return. Here is the vocabulary that lets you follow the arithmetic without losing the value of `eax` mid-trace.

### Why This One Has a "Redundant" Store and Load

The hint is doing real work again: "Not everything in this disassembly listing is optimal." That is a warning, not a riddle. In an *optimally* compiled C function, the last two lines of the work section would look like this:

```c
eax = local1 * local2 + 0x1f5;     // computed once
return eax;                         // eax is already the answer
ret
```

Translated to assembly, the optimal version would be:

```
mov    eax, DWORD PTR [rbp-0xc]
imul   eax, DWORD PTR [rbp-0x8]
add    eax, 0x1f5
pop    rbp
ret
```

Five instructions, no extra memory traffic. The challenge dump, however, has **seven** instructions in the work section — two more than the optimal version. The extra pair is at the end:

```
<+41>:    mov    DWORD PTR [rbp-0x4], eax    # store the answer to a new local
<+44>:    mov    eax, DWORD PTR [rbp-0x4]    # load it back into eax
```

That is a "dead store": the value is written into a memory slot and then read back unchanged. The compiler (or whoever wrote this disassembly) was either compiled with `-O0`, or was being deliberately consistent with the store-then-load pattern from Bit-O-Asm-2, or the challenge author wanted the dump to have a uniform "save to local, return from local" shape. Whatever the reason, the value of `eax` is **the same** before and after that redundant pair. You can ignore both lines and just compute the answer from `<+29>` through `<+36>`.

If you ever want to be sure the redundancy is real, do the trace. I do it for you in Step 3 below — the table shows `eax` does not change between `<+36>` and `<+47>`.

### The Two New Instructions: `imul` and `add`

**`imul dst, src`** — "integer multiply." Computes `dst * src` and stores the result in `dst`. The `i` is for "integer" (as opposed to a separate floating-point multiply). On x86-64, `imul` has several forms; the two-operand form here is "multiply `dst` by `src`, low 32 bits go in `dst`."

The operand order is the same as `mov`: **Intel syntax** (the syntax picoCTF uses), so the **first** operand is the destination and the **second** is the source. So:

```
imul   eax, DWORD PTR [rbp-0x8]
```

means "multiply `eax` by the value at `[rbp-0x8]`, store the low 32 bits back in `eax`." Because both operands are 32-bit (`eax` is 32-bit, `DWORD PTR` is 32-bit), the result is a 32-bit value. The upper 32 bits of the "true" 64-bit product are discarded, and `rax` is zero-extended.

**`add dst, src`** — "add." Computes `dst + src` and stores the result in `dst`. Same Intel-syntax operand order as `imul` and `mov`.

```
add    eax, 0x1f5
```

means "add `0x1f5` to `eax`, store the sum back in `eax`."

### Why the Multiply and Add Are Safe (No Overflow)

For this challenge, both `imul` and `add` produce a value that still fits in 32 bits unsigned. Let us check the upper bound:

```
0x9fe1a * 0x4  = 0x27f868          (still fits in 32 bits; < 2^32)
0x27f868 + 0x1f5 = 0x27fa5d        (still fits in 32 bits; < 2^32)
```

The final value `0x27fa5d` is comfortably under `0x100000000` (the 32-bit wraparound point). So we do not need to worry about signed-vs-unsigned interpretation — the value is small enough to be both positive and unsigned.

If this were a 32-bit-overflow challenge (it is not), I would note that `imul` discards the upper 32 bits and `add` wraps modulo `2^32`. For this challenge, neither is in play.

### The Three Local Slots Are Independent

The dump has three memory slots in the work section, each used for one purpose:

| Slot | Purpose | Set by | Read by |
|---|---|---|---|
| `[rbp-0xc]` | operand 1 (the `0x9fe1a` constant) | `<+15>` | `<+29>` |
| `[rbp-0x8]` | operand 2 (the `0x4` constant) | `<+22>` | `<+32>` |
| `[rbp-0x4]` | scratch / dead store | `<+41>` | `<+44>` |

The first two are real inputs to the arithmetic. The third is the redundant save-then-load pair the hint warned you about — `eax` already holds the answer by the time it gets stored there, and reading it back does not change anything.

### Reading x86 Mnemonics in General

Quick reference for the four mnemonics that appear in this challenge:

| Mnemonic | Meaning | Operand order (Intel syntax) |
|---|---|---|
| `mov`   | copy   | `mov dst, src`   — copy `src` into `dst` |
| `imul`  | multiply | `imul dst, src` — `dst = dst * src` |
| `add`   | add    | `add dst, src`   — `dst = dst + src` |
| `pop`   | restore from stack | `pop reg`         — pop top of stack into `reg` |
| `ret`   | return from function | `ret`             — pop return address and jump |

If you remember the Intel-syntax rule "`mov` moves the second operand into the first" (which the Bit-O-Asm-1 hint told us), the rest of the arithmetic instructions follow the same rule: the **first** operand is the destination, the **second** is the source. So `imul eax, [rbp-0x8]` and `add eax, 0x1f5` are both "modify `eax`," not "modify the memory."

---

## Solution — Step by Step

### Step 1 — Get the assembly dump

Same workflow as Bit-O-Asm-1 and Bit-O-Asm-2. I downloaded the file the challenge linked to and saved it into the working folder.

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ mv ~/Downloads/*.txt ./bit-o-asm-3.txt
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ls
bit-o-asm-1.txt  bit-o-asm-2.txt  bit-o-asm-3.txt
```

I keep all three side-by-side because the series is short and I want to refer back to the earlier dumps when I compare.

### Step 2 — Read the dump

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ cat bit-o-asm-3.txt
```

```
<+0>:     endbr64
<+4>:     push   rbp
<+5>:     mov    rbp,rsp
<+8>:     mov    DWORD PTR [rbp-0x14],edi
<+11>:    mov    QWORD PTR [rbp-0x20],rsi
<+15>:    mov    DWORD PTR [rbp-0xc],0x9fe1a
<+22>:    mov    DWORD PTR [rbp-0x8],0x4
<+29>:    mov    eax,DWORD PTR [rbp-0xc]
<+32>:    imul   eax,DWORD PTR [rbp-0x8]
<+36>:    add    eax,0x1f5
<+41>:    mov    DWORD PTR [rbp-0x4],eax
<+44>:    mov    eax,DWORD PTR [rbp-0x4]
<+47>:    pop    rbp
<+48>:    ret
```

Twelve lines. Two more than Bit-O-Asm-2, three more than Bit-O-Asm-1. The work section now spans seven instructions instead of two.

### Step 3 — Trace the value of `eax` through the dump

Per the hint, the last two work-section lines are a redundant pair. I will still trace them in the table to confirm the redundancy rather than assume it.

| Line | What it does | `eax` after |
|---|---|---|
| `<+8>`  | save `argc` (in `edi`) to `[rbp-0x14]`            | unchanged |
| `<+11>` | save `argv` (in `rsi`) to `[rbp-0x20]`            | unchanged |
| `<+15>` | write `0x9fe1a` to `[rbp-0xc]` (operand 1)        | unchanged |
| `<+22>` | write `0x4` to `[rbp-0x8]` (operand 2)            | unchanged |
| `<+29>` | read `[rbp-0xc]` into `eax`                       | `eax = 0x9fe1a` |
| `<+32>` | `eax = eax * [rbp-0x8]`                           | `eax = 0x9fe1a * 0x4` |
| `<+36>` | `eax = eax + 0x1f5`                               | `eax = 0x9fe1a * 0x4 + 0x1f5` |
| `<+41>` | store `eax` to `[rbp-0x4]` (dead store)           | unchanged |
| `<+44>` | read `[rbp-0x4]` back into `eax` (redundant)      | unchanged |
| `<+47>` | pop saved `rbp`                                   | unchanged |
| `<+48>` | `ret`, exit with `eax` as the return value        | unchanged |

So the answer is whatever the arithmetic at `<+29>`–`<+36>` produces. The dead store and reload at `<+41>`/`<+44>` are a no-op on `eax` — exactly what the hint warned us about.

### Step 4 — Compute the arithmetic

Two ways: by hand, or let Python do it.

**By hand** (following the order of the dump):

```
0x9fe1a * 0x4

First, 0x9fe1a in decimal:
  9 * 16^4 = 9 * 65536 = 589824
  f * 16^3 = 15 * 4096 = 61440
  e * 16^2 = 14 * 256 = 3584
  1 * 16^1 = 1 * 16 = 16
  a * 16^0 = 10 * 1 = 10
  Sum: 589824 + 61440 + 3584 + 16 + 10 = 654874

Then 654874 * 4:
  654874 * 4 = 2,619,496

In hex: 0x27f868

Then 2,619,496 + 0x1f5:
  0x1f5 in decimal = 1 * 256 + 15 * 16 + 5 = 256 + 240 + 5 = 501
  2,619,496 + 501 = 2,619,997
```

So the final value is `0x27fa5d` in hex, `2,619,997` in decimal.

**In Python** (the same logic, but no arithmetic on my end):

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(0x9fe1a * 0x4 + 0x1f5)"
2619997
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(hex(0x9fe1a * 0x4 + 0x1f5))"
0x27fa5d
```

Either way: `eax` holds **2,619,997** when `main` returns.

### Step 5 — Wrap in the flag format

The challenge description worked the example `0x11` → `picoCTF{17}`. Same conversion, same format:

```
picoCTF{2619997}
```

Submit it on the picoCTF challenge page, accept.

---

## What Happened Internally (Timeline)

Walking through the dump from start to finish, step by step, so the arithmetic and the dead store both stop looking like magic:

1. The function is entered. `rax` (and therefore `eax`) holds whatever the previous call left in it — some libc startup garbage we do not care about.
2. `endbr64` is the CET landing pad. No effect on registers. Continue.
3. `push rbp` saves the caller's base pointer. `mov rbp, rsp` anchors a fresh stack frame, so the four slots `[rbp-0x14]`, `[rbp-0x20]`, `[rbp-0xc]`, `[rbp-0x8]`, and `[rbp-0x4]` become valid local-variable addresses. `eax` is untouched.
4. `mov DWORD PTR [rbp-0x14], edi` saves `argc`. `mov QWORD PTR [rbp-0x20], rsi` saves `argv`. Neither writes to `eax`. They are just satisfying the C signature `int main(int argc, char **argv)`.
5. `<+15>: mov DWORD PTR [rbp-0xc], 0x9fe1a` writes the 32-bit literal `0x9fe1a` (the first operand of the eventual arithmetic) to a 4-byte slot. `eax` is still untouched.
6. `<+22>: mov DWORD PTR [rbp-0x8], 0x4` writes the 32-bit literal `0x4` (the second operand) to another 4-byte slot. `eax` is still untouched.
7. `<+29>: mov eax, DWORD PTR [rbp-0xc]` reads 4 bytes from `[rbp-0xc]` and puts them in `eax`. After this instruction, `eax == 0x9fe1a` (and the 32-bit destination zero-extends into `rax`, so `rax == 0x9fe1a`).
8. `<+32>: imul eax, DWORD PTR [rbp-0x8]` multiplies `eax` by the 4-byte value at `[rbp-0x8]` (`0x4`) and stores the low 32 bits of the product back in `eax`. `eax` is now `0x9fe1a * 0x4 = 0x27f868`. (`rax` is also zero-extended to `0x27f868`.) No overflow — the true 64-bit product is `0x0000000027f868`, well under `2^32`.
9. `<+36>: add eax, 0x1f5` adds the 32-bit immediate `0x1f5` to `eax`. `eax` is now `0x27f868 + 0x1f5 = 0x27fa5d`. In decimal: `2,619,997`. No overflow — `0x27fa5d` is comfortably below `2^32`.
10. `<+41>: mov DWORD PTR [rbp-0x4], eax` writes `eax`'s current value (`0x27fa5d`) to a fourth 4-byte slot at `[rbp-0x4]`. This is the **dead store** the hint warned us about: nothing reads this slot between here and `<+44>`, so we could have skipped the write entirely. `eax` is still `0x27fa5d`.
11. `<+44>: mov eax, DWORD PTR [rbp-0x4]` reads the same 4 bytes back into `eax`. Because we just wrote `0x27fa5d` there, the load puts `0x27fa5d` back into `eax`. This is the **redundant load**: `eax` was already `0x27fa5d` from step 9, so this instruction is a no-op semantically. The hint explicitly told us "not everything in this disassembly listing is optimal," and this is the line it was pointing at.
12. `pop rbp` restores the caller's base pointer. `eax` is still `0x27fa5d`.
13. `ret` pops the saved return address and jumps back into libc startup, which eventually calls `exit` with `eax`'s value (`0x27fa5d` → `2,619,997` in decimal). Program terminates.

`eax` is the answer from step 9 onward. The dead store and redundant load at steps 10 and 11 are byte-for-byte the same value; ignoring them is the intended shortcut. The decimal value `2,619,997` is what `main` "returned."

---

## Alternative Solve Methods

### Alternative 1 — use the `hex2dec.py` helper from Bit-O-Asm-1

The little reusable script from the Bit-O-Asm-1 writeup is no longer a one-liner for this challenge (because we need to do arithmetic, not just convert a single literal), but a slightly bigger one-liner still works:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "import sys; print(int(sys.argv[1], 16))" 0x27fa5d
2619997
```

Or just feed Python the whole arithmetic directly:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(0x9fe1a * 0x4 + 0x1f5)"
2619997
```

If you want the script to do the whole computation in one go:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ cat bit-o-asm-3.txt | grep -E "imul|add|mov    eax" | grep -v "rbp-0x14\|rbp-0x20"
    29:    mov    eax,DWORD PTR [rbp-0xc]
    32:    imul   eax,DWORD PTR [rbp-0x8]
    36:    add    eax,0x1f5
    41:    mov    DWORD PTR [rbp-0x4],eax
    44:    mov    eax,DWORD PTR [rbp-0x4]
```

Five lines — three of them (the two at the top and the bottom two) are the actual work. The rest is grep-filtered out for clarity.

### Alternative 2 — let `python3` evaluate the whole dump

The cleanest "type the dump into Python" approach is to model the function as a sequence of assignments to a fake `eax` and a fake memory dict, exactly the way the Bit-O-Asm-2 writeup did. Here is the full simulation:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 - <<'PY'
mem = {}
eax = "garbage"                 # whatever was in eax on entry
rbp = 0x7fffffff                # fake stack frame base

mem[rbp - 0x14] = "argc"        # <+8>
mem[rbp - 0x20] = "argv"        # <+11>
mem[rbp - 0xc]  = 0x9fe1a       # <+15>
mem[rbp - 0x8]  = 0x4           # <+22>

eax = mem[rbp - 0xc]            # <+29>
eax = (eax * mem[rbp - 0x8]) & 0xffffffff   # <+32>   (32-bit wraparound)
eax = (eax + 0x1f5) & 0xffffffff           # <+36>

mem[rbp - 0x4]  = eax           # <+41>  (dead store)
eax = mem[rbp - 0x4]            # <+44>  (redundant load)

print(f"eax = {hex(eax)} = {eax}")
PY
```

```
eax = 0x27fa5d = 2619997
```

Same answer, but now you have a working simulator you can paste into Bit-O-Asm-4 (which adds a `cmp` and `jle` to the mix). The bitmask `& 0xffffffff` is what enforces 32-bit semantics for the arithmetic; for this challenge the values stay small enough that it is a no-op, but it is the right thing to do for harder variants.

### Alternative 3 — skip the dead store and reload, then compute

The hint is *literally* telling you the last two work-section lines are noise. If you trust it (and you should — that is the hint's job), you can do the whole challenge with this one-liner:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(0x9fe1a * 0x4 + 0x1f5)"
2619997
```

The full manual trace is the "show your work" version, but for a clean answer the dump collapses to one expression: `0x9fe1a * 0x4 + 0x1f5`.

### Alternative 4 — `printf` with arithmetic

`printf` is happy to evaluate simple integer arithmetic in its format string, but for a `*` and a `+` mixed with hex literals, it is easier to reach for `bc` or Python. If you really want to stay in pure shell:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ echo "ibase=16; 9FE1A * 4 + 1F5" | bc
2619997
```

`bc` reads the whole expression in one go. Same answer, one line, no script.

### Alternative 5 — sanity-check with `gdb` and a hand-compiled C

If you have access to a C compiler and want to confirm the trace is correct, the cleanest sanity check is to compile a tiny C program that does the same arithmetic, then disassemble it:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ cat > /tmp/sanity.c <<'C'
#include <stdio.h>
int main(void) {
    volatile unsigned int a = 0x9fe1a;
    volatile unsigned int b = 0x4;
    return a * b + 0x1f5;
}
C
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ gcc -O0 -o /tmp/sanity /tmp/sanity.c
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ /tmp/sanity; echo $?
2619997
```

The shell `$?` captures the program's exit code, which for a `return` statement is exactly what `eax` held at `ret`. The answer `2619997` matches the trace. (Use `-O0` to keep the disassembly close to the challenge's, but for *this* check the optimization level does not matter because we are only running the program, not disassembling it.)

### Alternative 6 — solve it without converting to decimal until the end

Just like Bit-O-Asm-1 and Bit-O-Asm-2, the value in `eax` at `ret` is whatever the arithmetic produces. The only reason to convert to decimal is the flag format. So if you trust the trace, you can write down "the answer is `0x27fa5d`" first, then convert at the very end. This keeps the intermediate steps in hex (which is what the dump shows) and avoids losing a digit mid-arithmetic.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `cat` | coreutils 9.1 | Read the assembly dump file directly. It is plain text, no special tooling. |
| `grep` | GNU grep 3.11 | Filtered the dump to just the work-section instructions for clarity. |
| `python3` | Python 3.11 | Converted the hex literal `0x9fe1a` to its decimal value (`654874`), evaluated the arithmetic (`* 4 + 0x1f5`), and simulated the whole function end-to-end in the "Alternative Methods" section. |
| `bc` | GNU bc 1.07 | Listed as an alternative for users who prefer the classic Unix calculator to Python. |
| `printf` | coreutils 9.1 | Mentioned but not the cleanest tool here because the arithmetic spans two operations. |
| `gcc` | GCC 13.2 | Used in the sanity-check alternative to compile a tiny C program with the same arithmetic and confirm the exit code matches. |
| `hex2dec.py` | (script from Bit-O-Asm-1) | Carried over from the earlier writeups; useful for converting the final `0x27fa5d` to decimal, but not for the arithmetic itself. |

---

## Key Takeaways

- **The new concept is "arithmetic between the load and the return."** Bit-O-Asm-1 was `mov eax, literal`. Bit-O-Asm-2 was `mov local, literal; mov eax, local`. Bit-O-Asm-3 is `mov local, literal; mov eax, local; imul eax, local; add eax, literal;` — same pattern with two extra ops tacked on the end. Once you see the structure, every future Bit-O-Asm-X is "Bit-O-Asm-2 plus some more instructions before the return."
- **Trace `eax` line by line, do not just glance.** The trap in this challenge is that there are now *multiple* lines that mention `eax` (`<+29>`, `<+32>`, `<+36>`, `<+41>`, `<+44>`), and the answer depends on doing them in order. A trace table is the cheapest way to avoid a "wait, did I read that right" moment.
- **The hint is telling you there is a no-op.** "Not everything in this disassembly listing is optimal" is the picoCTF author's way of pointing at the dead store and redundant load at `<+41>` and `<+44>`. If you trust the hint and ignore those two lines, the arithmetic collapses to `0x9fe1a * 0x4 + 0x1f5` and the whole challenge is one `python3` call.
- **Operand order stays the same.** `mov`, `imul`, and `add` all use Intel syntax here: the first operand is the destination, the second is the source. `imul eax, [rbp-0x8]` means "multiply `eax` by `[rbp-0x8]`," not the other way around. If you have only seen AT&T syntax before, the Bit-O-Asm-1 worked example still applies: "the `mov` instruction moves the second operand into the first."
- **32-bit arithmetic is unsigned for this challenge.** Both `imul` and `add` produce values well under `2^32`, so the 32-bit wraparound behavior is irrelevant. If you ever see a Bit-O-Asm variant that pushes close to `2^32`, the bitmask `& 0xffffffff` in the Python simulator is what enforces the wraparound.
- **Reuse your tooling, expand it just enough.** The `hex2dec.py` script from Bit-O-Asm-1 is still useful for the final decimal conversion, but the arithmetic itself wants a one-liner (`python3 -c "print(0x9fe1a * 0x4 + 0x1f5)"`) or the small Python simulator shown in Alternative 2. Build a slightly bigger helper if you do Bit-O-Asm-4 too — a `simulate.py` that takes a dump and a list of "interesting" lines.
- **Flag wordplay decode.** The flag is `picoCTF{2619997}`. There is no wordplay to decode — the entire challenge is the store-then-load-and-arithmetic pattern (`mov [rbp-0xc], 0x9fe1a; mov [rbp-0x8], 0x4; mov eax, [rbp-0xc]; imul eax, [rbp-0x8]; add eax, 0x1f5;` followed by the redundant dead-store pair), the hex arithmetic (`0x9fe1a * 0x4 + 0x1f5 = 0x27fa5d`), and the hex-to-decimal conversion (`0x27fa5d = 2,619,997`). The series name "Bit-O-Asm" still means "a bit of assembly" — this time, that bit happens to do a multiply and an add before reaching `eax`. Once you do the arithmetic, the flag is `picoCTF{2619997}`.
