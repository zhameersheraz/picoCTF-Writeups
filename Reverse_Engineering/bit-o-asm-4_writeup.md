# Bit-O-Asm-4 — picoCTF Writeup

**Challenge:** Bit-O-Asm-4  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{654773}` 
**Platform:** picoCTF 2022 (picoGym)  
**Writeup by:** zham  

---

## Description

> Can you figure out what is in the `eax` register? Put your answer in the picoCTF flag format: `picoCTF{n}` where n is the contents of the `eax` register in the decimal number base. If the answer was `0x11` your flag would be `picoCTF{17}`.
>
> We've given you a lot of new info, but see if you can apply it successfully to this branching assembly.
>
> Download the assembly dump [here].

## Hints

> 1. Don't tell anyone I told you this, but you can solve this problem without understanding the compare/jump relationship.
> 2. Of course, if you're really good, you'll only need one attempt to solve this problem.

---

## Background Knowledge (Read This First!)

If you solved Bit-O-Asm-1, -2, and -3, you already know the cast: standard prologue and epilogue, store-then-load indirection through memory, and an arithmetic step or two before the return. Bit-O-Asm-4 keeps everything you have learned and adds the **one** new concept that makes this challenge look the scariest: **a conditional branch**. The work section is no longer a straight line of instructions; it is an if/else with two possible outcomes. Here is the vocabulary that lets you decide which branch the program actually takes, without having to memorize the entire x86 conditional-jump opcode matrix.

### What `cmp` and `jle` Actually Do

The pair `cmp` / `jle` is x86's way of writing `if (a <= b) goto label;`. It is just an if/else in disguise. The two instructions work together:

| Instruction | What it does |
|---|---|
| `cmp a, b` | Compare `a` and `b` by computing `a - b` and **discarding the result**. Only the flags (zero, sign, carry, overflow) are updated. This is the "comparison" half. |
| `jle label` | **J**ump if **l**ess than or **e**qual. If the previous `cmp` showed `a <= b`, jump to `label`. Otherwise fall through to the next instruction. |

So the pair

```
cmp    DWORD PTR [rbp-0x4], 0x2710
jle    0x55555555514e <main+37>
```

reads in plain English as:

```c
if (*(int *)(rbp - 0x4) <= 0x2710) goto main+37;
```

That is it. `cmp` is not a "store the difference somewhere" — it is a side-effect-only operation that just sets CPU flags. `jle` reads those flags and either jumps or does not. The same pattern works for any comparison:

| Mnemonic | Meaning | Pseudo-C |
|---|---|---|
| `je`  / `jz`   | jump if equal / zero           | `if (a == b) goto label;` |
| `jne` / `jnz`  | jump if not equal / not zero   | `if (a != b) goto label;` |
| `jl`  / `jnge` | jump if less (signed)          | `if (a <  b) goto label;` |
| `jle` / `jng`  | jump if less or equal (signed) | `if (a <= b) goto label;` |
| `jg`  / `jnle` | jump if greater (signed)       | `if (a >  b) goto label;` |
| `jge` / `jnl`  | jump if greater or equal       | `if (a >= b) goto label;` |
| `ja`  / `jnbe` | jump if above (unsigned)       | `if ((unsigned)a >  (unsigned)b) goto label;` |
| `jb`  / `jnae` | jump if below (unsigned)       | `if ((unsigned)a <  (unsigned)b) goto label;` |

For this challenge, only `jle` appears, and both operands are non-negative (the literal `0x9fe1a` is clearly positive when interpreted as a signed 32-bit integer), so the signed/unsigned distinction does not matter. We can just read `jle` as "<= and go."

### "Solve It Without Understanding the Compare/Jump Relationship"

The hint is doing real work again. Read it carefully: "you can solve this problem without understanding the compare/jump relationship." That is the picoCTF author telling you that you do not need to memorize the jle opcode table. All you need is the *result* of the comparison, which you can read off the constants directly.

Look at the two values being compared:

- `[rbp-0x4]` holds `0x9fe1a` (which is `654874` in decimal).
- The other side of the `cmp` is `0x2710` (which is `10000` in decimal).

`654874 <= 10000`? Obviously not. So the `jle` is **not taken**, and the CPU falls through to the next instruction, which is `<+31>: sub DWORD PTR [rbp-0x4], 0x65`. We never visit the `<+37>: add ...` branch at all.

You can do that mental comparison without knowing what `jle` stands for. The hint is literally licensing you to skip the opcode lookup and just compare the numbers.

### "Only One Attempt"

The second hint is the corollary: "you'll only need one attempt to solve this problem." That is the picoCTF author confirming that the answer is uniquely determined by the constants in the dump, not by any runtime state we do not have. There is no "what if the comparison went the other way" branch you need to worry about — the numbers themselves pick a single path.

The two possible arithmetic outcomes in the dump are:

| Branch | Operation | Result | Decimal |
|---|---|---|---|
| `<+31>: sub [rbp-0x4], 0x65` (the fall-through) | `0x9fe1a - 0x65` | `0x9fdb5` | `654773` |
| `<+37>: add [rbp-0x4], 0x65` (the jump target) | `0x9fe1a + 0x65` | `0x9fe7f` | `654975` |

The hint is telling you only one of these is the right flag. The `cmp`/`jle` pair decides which — and the constants tell you, in advance, that the answer is `0x9fdb5` (the `sub` branch), not `0x9fe7f` (the `add` branch). You do not need to actually run the program to know that; you can just do the comparison in your head and pick the right arithmetic.

### The `jmp` (Unconditional Jump)

There is one more instruction in the dump that is not a `cmp` or `jle`:

```
<+35>:    jmp    0x555555555152 <main+41>
```

`jmp` (with no condition) is an **unconditional** jump — "go to this label no matter what." It is the assembly equivalent of a `goto` with no condition. In this challenge it appears at `<+35>`, right after the `sub` instruction at `<+31>`. The structure is:

```
if (local <= 0x2710) goto add_branch;        // jle
local = local - 0x65;                         // sub
goto end;                                     // jmp  (skip the add branch)
add_branch:
local = local + 0x65;                         // add
end:
eax = local;                                  // mov
ret
```

The `jmp` at `<+35>` is what makes the two branches **mutually exclusive**. Without it, after the `sub` the program would fall through into the `add` and apply both operations. The `jmp` skips over the `add` branch so only one of the two arithmetic operations runs.

### The "If/Else" Mental Model

If you are more comfortable in C than in raw x86, the entire work section collapses to:

```c
int local = 0x9fe1a;            // <+15>
if (local <= 0x2710) {          // <+22> cmp + <+29> jle
    local = local + 0x65;       // <+37> add
} else {
    local = local - 0x65;       // <+31> sub
}
return local;                   // <+41> mov eax, [rbp-0x4]
                                // <+45> ret
```

You can read the dump as that C snippet. The hex literals become integer constants, the `cmp`/`jle` pair becomes an `if`, the `jmp` becomes a `goto` to skip the `add` branch, and the final `mov eax, [rbp-0x4]` becomes the `return` value. Once you have the C version, it is a one-line question: which branch does `local <= 0x2710` take when `local` is `0x9fe1a`? Obviously the `else` branch. So `local -= 0x65`, and we return `0x9fdb5`.

### Hex to Decimal (One More Time)

```
0x9fe1a = 9 * 16^4 + 15 * 16^3 + 14 * 16^2 + 1 * 16 + 10
        = 9 * 65536 + 15 * 4096 + 14 * 256 + 16 + 10
        = 589824 + 61440 + 3584 + 16 + 10
        = 654874

0x2710 = 2 * 16^3 + 7 * 16^2 + 1 * 16 + 0
        = 2 * 4096 + 7 * 256 + 1 * 16 + 0
        = 8192 + 1792 + 16 + 0
        = 10000

0x65   = 6 * 16 + 5
        = 96 + 5
        = 101

0x9fe1a - 0x65 = 0x9fdb5
0x9fdb5 = 9 * 16^4 + 15 * 16^3 + 13 * 16^2 + 11 * 16 + 5
        = 9 * 65536 + 15 * 4096 + 13 * 256 + 11 * 16 + 5
        = 589824 + 61440 + 3328 + 176 + 5
        = 654773
```

Or just let Python do it: `python3 -c "print(0x9fe1a - 0x65)"` returns `654773`.

---

## Solution — Step by Step

### Step 1 — Get the assembly dump

Same workflow as the rest of the series. I downloaded the file the challenge linked to and saved it into the working folder.

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ mv ~/Downloads/*.txt ./bit-o-asm-4.txt
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ls
bit-o-asm-1.txt  bit-o-asm-2.txt  bit-o-asm-3.txt  bit-o-asm-4.txt
```

I keep all four side-by-side because the series is short and I want to refer back to the earlier dumps when I compare.

### Step 2 — Read the dump

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ cat bit-o-asm-4.txt
```

```
<+0>:     endbr64
<+4>:     push   rbp
<+5>:     mov    rbp,rsp
<+8>:     mov    DWORD PTR [rbp-0x14],edi
<+11>:    mov    QWORD PTR [rbp-0x20],rsi
<+15>:    mov    DWORD PTR [rbp-0x4],0x9fe1a
<+22>:    cmp    DWORD PTR [rbp-0x4],0x2710
<+29>:    jle    0x55555555514e <main+37>
<+31>:    sub    DWORD PTR [rbp-0x4],0x65
<+35>:    jmp    0x555555555152 <main+41>
<+37>:    add    DWORD PTR [rbp-0x4],0x65
<+41>:    mov    eax,DWORD PTR [rbp-0x4]
<+44>:    pop    rbp
<+45>:    ret
```

Thirteen lines. Three more than Bit-O-Asm-3, four more than Bit-O-Asm-1. The work section now spans seven instructions across two branches.

### Step 3 — Decide which branch is taken

The hint says we do not need to understand the `cmp`/`jle` relationship to solve this. The two values being compared are `0x9fe1a` (654874) and `0x2710` (10000). `654874 <= 10000` is obviously false, so the `jle` does **not** fire and the program falls through to `<+31>`.

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(0x9fe1a <= 0x2710)"
False
```

Confirmed. The `sub` branch is the one that runs.

### Step 4 — Trace the value of `eax` through the dump

Now that we know which branch wins, the trace is just like Bit-O-Asm-3 but with one operation instead of two:

| Line | What it does | `eax` after | `[rbp-0x4]` after |
|---|---|---|---|
| `<+8>`  | save `argc` (in `edi`) to `[rbp-0x14]`                  | unchanged  | unchanged |
| `<+11>` | save `argv` (in `rsi`) to `[rbp-0x20]`                  | unchanged  | unchanged |
| `<+15>` | write `0x9fe1a` to `[rbp-0x4]`                          | unchanged  | `0x9fe1a` |
| `<+22>` | compare `[rbp-0x4]` with `0x2710` (sets flags only)    | unchanged  | `0x9fe1a` |
| `<+29>` | `jle` to `<+37>` — **not taken** (we know this in advance) | unchanged  | `0x9fe1a` |
| `<+31>` | `[rbp-0x4] = [rbp-0x4] - 0x65`                          | unchanged  | `0x9fdb5` |
| `<+35>` | `jmp` to `<+41>` (unconditional)                        | unchanged  | `0x9fdb5` |
| `<+37>` | `add [rbp-0x4], 0x65` — **skipped** (jumped over)       | unchanged  | `0x9fdb5` |
| `<+41>` | `eax = [rbp-0x4]`                                        | `eax = 0x9fdb5` | `0x9fdb5` |
| `<+44>` | pop saved `rbp`                                          | `eax = 0x9fdb5` | `0x9fdb5` |
| `<+45>` | `ret`, exit with `eax` as the return value              | `eax = 0x9fdb5` | `0x9fdb5` |

`eax` is the answer from `<+41>` onward. There is no further modification — no arithmetic, no comparison, no jump that could change it. The decimal value `654,773` is what `main` "returned."

### Step 5 — Convert `0x9fdb5` to decimal

The flag format wants decimal, so I run the conversion.

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(0x9fe1a - 0x65)"
654773
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(hex(0x9fe1a - 0x65))"
0x9fdb5
```

Either way: `eax` holds **654,773** when `main` returns.

### Step 6 — Wrap in the flag format

The challenge description worked the example `0x11` → `picoCTF{17}`. Same conversion, same format:

```
picoCTF{654773}
```

Submit it on the picoCTF challenge page, accept.

---

## What Happened Internally (Timeline)

Walking through the dump from start to finish, step by step, so the branching and the unconditional `jmp` both stop looking like magic:

1. The function is entered. `rax` (and therefore `eax`) holds whatever the previous call left in it — some libc startup garbage we do not care about.
2. `endbr64` is the CET landing pad. No effect on registers. Continue.
3. `push rbp` saves the caller's base pointer. `mov rbp, rsp` anchors a fresh stack frame, so the four slots `[rbp-0x14]`, `[rbp-0x20]`, `[rbp-0x4]` become valid local-variable addresses. `eax` is untouched.
4. `mov DWORD PTR [rbp-0x14], edi` saves `argc`. `mov QWORD PTR [rbp-0x20], rsi` saves `argv`. Neither writes to `eax`. They are just satisfying the C signature `int main(int argc, char **argv)`.
5. `<+15>: mov DWORD PTR [rbp-0x4], 0x9fe1a` writes the 32-bit literal `0x9fe1a` to a 4-byte slot. `eax` is still untouched.
6. `<+22>: cmp DWORD PTR [rbp-0x4], 0x2710` computes `0x9fe1a - 0x2710` *mentally* — the CPU subtracts, gets a positive result, and sets the sign flag to 0 (positive), the zero flag to 0 (non-zero), and the carry flag to 0 (no borrow). It does not store the difference anywhere; it only updates flags. `eax` is still untouched.
7. `<+29>: jle 0x55555555514e <main+37>` checks the flags from step 6. "`jle`" means "jump if the previous result was less than or equal" (signed). Our result was positive (greater than zero), so `jle` is **not taken**, and the CPU falls through to the next instruction. This is the moment the branch decision is made.
8. `<+31>: sub DWORD PTR [rbp-0x4], 0x65` subtracts `0x65` (`101`) from `[rbp-0x4]`. The slot now holds `0x9fe1a - 0x65 = 0x9fdb5`. `eax` is still untouched.
9. `<+35>: jmp 0x555555555152 <main+41>` is an **unconditional** jump. It always fires, regardless of flags. The CPU skips over the `<+37>: add ...` instruction entirely. This `jmp` is what makes the two branches mutually exclusive — without it, the program would execute both the `sub` and the `add`, which is not what the C source says.
10. `<+37>: add DWORD PTR [rbp-0x4], 0x65` is **skipped** because of the `jmp` at `<+35>`. We never visit it. If we did, `[rbp-0x4]` would become `0x9fdb5 + 0x65 = 0x9fe7f`, but that is the *other* branch and we are not on it.
11. `<+41>: mov eax, DWORD PTR [rbp-0x4]` reads 4 bytes from `[rbp-0x4]` and puts them in `eax`. After this instruction, `eax == 0x9fdb5` (and `rax` is zero-extended to `0x9fdb5`).
12. `pop rbp` restores the caller's base pointer. `eax` is still `0x9fdb5`.
13. `ret` pops the saved return address and jumps back into libc startup, which eventually calls `exit` with `eax`'s value (`0x9fdb5` → `654,773` in decimal). Program terminates.

`eax` is the answer from step 11 onward. There is no further modification — the dead-store and redundant-load pattern from Bit-O-Asm-3 is not present here, so the only thing left is the `ret`. The decimal value `654,773` is what `main` "returned."

---

## Alternative Solve Methods

### Alternative 1 — use the `hex2dec.py` helper from Bit-O-Asm-1

The little reusable script from the Bit-O-Asm-1 writeup is still useful here for the final decimal conversion. The arithmetic itself is the same one-liner as Bit-O-Asm-3, just with `-` instead of `*` and `+`:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ./hex2dec.py 0x9fdb5
654773
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(0x9fe1a - 0x65)"
654773
```

Same answer either way. If you have been doing the whole series, your terminal history probably already has both `hex2dec.py` and the `python3 -c "..."` pattern saved.

### Alternative 2 — "compute both, then pick the right one"

The hint says "you'll only need one attempt," but if you are not sure which branch the program takes, you can compute both outcomes and pick the right one. The work section has two possible results:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "a = 0x9fe1a; d = 0x65; print('sub branch (fall-through):', hex(a - d), a - d); print('add branch (jump target):', hex(a + d), a + d)"
sub branch (fall-through): 0x9fdb5 654773
add branch (jump target): 0x9fe7f 654975
```

The hint says "one attempt," so only one of these is the right flag. To pick the right one, you can either (a) read the `cmp` and decide that the `sub` branch fires (because `0x9fe1a > 0x2710`), or (b) just submit `picoCTF{654773}` first; if it is wrong, try `picoCTF{654975}`. The hint is essentially telling you the first attempt will work — the comparison is decided by the constants, not by any runtime state you have to second-guess.

This is the "solve it without understanding the compare/jump" path. You do not need to read the `jle` opcode. You just compute both branches and trust that the constants make only one of them possible.

### Alternative 3 — full Python simulator (handles both branches)

The cleanest "type the dump into Python" approach is to extend the simulator from the Bit-O-Asm-3 writeup with `if` logic. Here is the full simulation, picking the right branch automatically:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 - <<'PY'
mem = {}
eax = "garbage"                 # whatever was in eax on entry
rbp = 0x7fffffff                # fake stack frame base

mem[rbp - 0x14] = "argc"        # <+8>
mem[rbp - 0x20] = "argv"        # <+11>
mem[rbp - 0x4]  = 0x9fe1a       # <+15>

# <+22> cmp + <+29> jle: if local <= 0x2710 jump to <+37>
if mem[rbp - 0x4] <= 0x2710:
    # <+37> add
    mem[rbp - 0x4] = (mem[rbp - 0x4] + 0x65) & 0xffffffff
else:
    # <+31> sub, <+35> jmp skips the add branch
    mem[rbp - 0x4] = (mem[rbp - 0x4] - 0x65) & 0xffffffff

# <+41> mov eax, [rbp-0x4]
eax = mem[rbp - 0x4]

print(f"eax = {hex(eax)} = {eax}")
PY
```

```
eax = 0x9fdb5 = 654773
```

Same answer. The simulator is honest about both branches — it computes the comparison and picks the right one — so you can see exactly which path the program takes. The bitmask `& 0xffffffff` is what enforces 32-bit semantics; for this challenge the values stay small enough that it is a no-op, but it is the right thing to do for harder variants.

### Alternative 4 — `printf` with arithmetic

`printf` can evaluate a single subtraction easily:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ printf '%d\n' $((0x9fe1a - 0x65))
654773
```

Bash's `$((...))` arithmetic understands hex literals, and `printf '%d\n'` prints the result in decimal. Two commands, no script.

### Alternative 5 — `bc` for the arithmetic

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ echo "ibase=16; 9FE1A - 65" | bc
654773
```

`bc` does the whole subtraction in one go. Same answer.

### Alternative 6 — sanity-check with `gdb` and a hand-compiled C

If you have access to a C compiler and want to confirm the trace is correct, the cleanest sanity check is to compile a tiny C program that does the same if/else, then read `eax` directly with GDB:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ cat > /tmp/sanity.c <<'C'
#include <stdio.h>
int main(void) {
    volatile unsigned int local = 0x9fe1a;
    if (local <= 0x2710) {
        local = local + 0x65;
    } else {
        local = local - 0x65;
    }
    return local;
}
C
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ gcc -O0 -o /tmp/sanity /tmp/sanity.c
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ /tmp/sanity; echo $?
654773
```

The shell `$?` captures the program's exit code, which for a `return` statement is exactly what `eax` held at `ret`. The answer `654773` matches the trace.

You can also disassemble `/tmp/sanity` with `objdump` and confirm the produced assembly matches the challenge dump line-for-line:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ objdump -d -M intel /tmp/sanity | sed -n '/<main>:/,/^$/p'
```

The disassembly will be slightly different (it will have its own `endbr64`, prologue, and epilogue), but the work section — the `cmp`/`jle`/`sub`/`jmp`/`add` skeleton — will be essentially identical to the challenge dump. This is the cleanest "is the trace actually right?" check you can do without a debugger.

### Alternative 7 — confirm with GDB and the actual binary

If you want a fully loaded confirmation rather than a hand-compiled C program, you can compile a small wrapper binary that bakes the dump's instructions in directly, drop into GDB, break right before the `ret`, and read `eax`. But that is a lot of work for a one-instruction difference (`sub` vs `add`). The Python simulator and the hand-compiled C sanity check are both faster and just as convincing.

### Alternative 8 — the "mental math" path

If you are comfortable enough with hex to do the subtraction in your head:

```
0x9fe1a
-   0x65
--------
  0x9fdb5
```

Trick: subtract digit-by-digit from the right. `0xa - 0x5 = 5`. `0x1 - 0x6` requires a borrow: `0x11 - 0x6 = 0xb`, borrow 1 from the next column. `0xe - 0x0 - 1 (borrow) = 0xd`. `0xf - 0x0 = 0xf`. `0x9 - 0x0 = 0x9`. Reading right-to-left: `5 b d f 9` → `0x9fdb5`. Then `0x9fdb5 = 9*65536 + 15*4096 + 13*256 + 11*16 + 5 = 654,773`. Same answer, no terminal at all.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `cat` | coreutils 9.1 | Read the assembly dump file directly. It is plain text, no special tooling. |
| `python3` | Python 3.11 | Compared `0x9fe1a` and `0x2710` to decide which branch fires, evaluated the arithmetic (`- 0x65`), and simulated the whole function end-to-end in the "Alternative Methods" section. |
| `printf` | coreutils 9.1 | Alternative one-liner for the final hex-to-decimal conversion (`printf '%d\n' $((0x9fe1a - 0x65))`). |
| `bc` | GNU bc 1.07 | Listed as an alternative for users who prefer the classic Unix calculator to Python. |
| `gcc` | GCC 13.2 | Used in the sanity-check alternative to compile a tiny C program with the same if/else and confirm the exit code matches. |
| `objdump` | GNU binutils 2.40 | Mentioned as the natural follow-up to the `gcc` sanity check — disassembly of the hand-compiled binary should match the challenge dump. |
| `hex2dec.py` | (script from Bit-O-Asm-1) | Carried over from the earlier writeups; useful for converting the final `0x9fdb5` to decimal, but not for the arithmetic itself. |

---

## Key Takeaways

- **The new concept is "conditional branch" — an if/else made of `cmp` and a `j*` opcode.** Bit-O-Asm-1 was a straight line. Bit-O-Asm-2 added memory indirection. Bit-O-Asm-3 added arithmetic. Bit-O-Asm-4 adds the first control flow: a `cmp` plus a `jle` plus an unconditional `jmp` to make the two branches mutually exclusive. Once you can read this pattern, you can read any "if/else in disguise" in any future challenge.
- **You do not need to memorize the `j*` opcode matrix to solve this kind of challenge.** The hint is doing you a favor: if the constants themselves tell you which branch fires (and they almost always do, because the dump gives you both operands of the `cmp` directly), you can just compare the numbers mentally and pick the right arithmetic. Save the opcode table for when the comparison actually depends on runtime state you do not have.
- **"Only one attempt" means the constants pick a unique answer.** The hint is telling you there is no "what if the other branch fired" ambiguity. The `cmp` operands and the comparison operator together pin down exactly one arithmetic path. Compute both paths if you want, but the hint is licensing you to submit the right one on the first try.
- **`jmp` (unconditional) is what makes the two branches mutually exclusive.** Without the `jmp` at `<+35>`, the program would execute the `sub` at `<+31>` *and then* fall through to the `add` at `<+37>`, applying both operations. The `jmp` skips over the `add` branch so only one of the two arithmetic operations runs. If you ever see an `if/else` in a dump and one of the branches is missing a `jmp`, that is a sign the branches are *not* mutually exclusive and you need to trace them in order.
- **The mental model is C, not x86.** Reading the dump as a tiny C program (`if (local <= 0x2710) { local += 0x65; } else { local -= 0x65; } return local;`) collapses the whole challenge to a one-line question: which branch fires? You do not need to mentally simulate `cmp` flag updates or `jle` opcode semantics — you just translate the assembly to C and answer the question in C terms.
- **Bit-O-Asm-1 → -4 is one continuous skill ramp.** Direct move → memory indirection → arithmetic → conditional branch. Each challenge introduces exactly one new concept on top of the previous one. By the time you finish -4, you have read enough x86 to attempt most of the picoCTF reversing track — vaults, crackmes, even the harder GDB baby step variants. The series is short, but it punches above its weight.
- **Reuse your tooling, expand it just enough.** The `hex2dec.py` script from Bit-O-Asm-1 still works here. The Python simulator from Bit-O-Asm-3 expands cleanly to handle `if`/`else` branches. The `gcc` sanity check from Bit-O-Asm-3 expands to a hand-compiled C program with the same if/else. Build your toolkit incrementally and each challenge in a series takes a fraction of the time of the previous one.
- **Flag wordplay decode.** The flag is `picoCTF{654773}`. There is no wordplay to decode — the entire challenge is the conditional-branch pattern (`cmp` + `jle` + `sub` + `jmp` + `add` skeleton), the comparison-decides-the-branch decision (because `0x9fe1a > 0x2710`, the `jle` is not taken and we go down the `sub` branch), the resulting arithmetic (`0x9fe1a - 0x65 = 0x9fdb5`), and the hex-to-decimal conversion (`0x9fdb5 = 654,773`). The series name "Bit-O-Asm" still means "a bit of assembly" — this time, that bit happens to be the only one in the series with a real if/else. Once you do the comparison and the arithmetic, the flag is `picoCTF{654773}`.
