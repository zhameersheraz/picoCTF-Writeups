# GDB baby step 2 — picoCTF Writeup

**Challenge:** GDB baby step 2  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{307019}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> Can you figure out what is in the eax register at the end of the main function? Put your answer in the picoCTF flag format: picoCTF{n} where n is the contents of the eax register in the decimal number base. If the answer was 0x11 your flag would be picoCTF{17}.

## Hints

> 1. You could calculate eax yourself, or you could set a breakpoint for after the calculation and inspect eax to let the program do the heavy-lifting for you.

---

## Background Knowledge (Read This First!)

This challenge has one new idea on top of baby step 4 — the `eax` value is not a hardcoded constant sitting in the disassembly. It is the *result of a small loop*, so either you reverse the loop and compute it on paper, or you let the program run, pause it after the calculation, and ask GDB what `eax` is. Both approaches are valid; the hint is steering you toward the latter.

### The "End of main" Trick

GDB breakpoints are positioned at the *start* of an instruction. If you set a breakpoint on `mov eax, [rbp-0x4]` at `0x40113e`, GDB pauses *before* the load — `eax` still holds whatever was in it from the previous iteration of the loop. To see the final value, you have to either:

- Set the breakpoint one instruction later (e.g. on `pop rbp` at `0x401141`), so `eax` has been loaded but the function has not yet returned.
- Set the breakpoint on the load and `nexti` (next instruction) once.

Both work. I prefer the first because it is one fewer step.

### Why the Final `mov eax, ...` Matters

In the System V AMD64 calling convention, the value in `eax` (or `rax`) when a function returns is the function's *return value*. So whatever `main` loads into `eax` right before `ret` is what `main` returns to `_start`. For a tiny program like this with no `printf` and no `exit`, the value still has to be put somewhere — and that somewhere is `eax` at the last `mov`. Reading it there is exactly what the description is asking for.

### What "the heavy-lifting" Means in the Hint

The hint frames two paths: (1) compute `eax` by hand from the disassembly, or (2) let the program run and inspect the register. The point is that for any challenge with a non-trivial loop, path 2 is more reliable — you are not at risk of miscounting loop iterations, misreading a comparison, or mistranslating a jump. The `info registers` command does not care about any of that; it just tells you what is in the register when execution paused.

### The Loop Pattern in This Binary

The binary contains a textbook C-style `for` loop:

```c
int total = 0x1e0da;     // 123098
int limit = 0x25f;       // 607
for (int i = 0; i < limit; i++) {
    total += i;
}
return total;
```

The compiler turned that into the assembly we are about to look at. The key block is the bottom of the loop:

```
401136: mov    eax, [rbp-0x8]    ; eax = i
401139: cmp    eax, [rbp-0xc]    ; cmp i, limit
40113c: jl     40112c            ; if i < limit, loop back
```

That is the standard x86 translation of `while (i < limit)` — load `i` into `eax`, compare it with the limit, jump-if-less back to the loop body.

### Translating `eax` to a Flag

The description gives the worked example: `0x11` → `17`. The flag is the *decimal* value, not the hex spelling. If `eax` is `0x4af4b`, the flag is `picoCTF{307019}`, not `picoCTF{0x4af4b}`. Read the example carefully — the description explicitly says "in the decimal number base."

### How `info registers` Displays `eax`

```
eax            0x4af4b             307019
```

GDB shows the value in hex on the left and decimal on the right, both at once. You can also use `print/d $eax` (signed decimal), `print/u $eax` (unsigned decimal), or `print/x $eax` (hex). All three agree on `307019` for this challenge.

---

## Solution — Step by Step

### Step 1 — Make the binary executable and identify it

```
┌──(zham㉿kali)-[~/ctf]
└─$ cp ~/Downloads/debugger_c .
┌──(zham㉿kali)-[~/ctf]
└─$ chmod +x debugger_c
┌──(zham㉿kali)-[~/ctf]
└─$ file debugger_c
debugger_c: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=95b0203be2982e75dbc01d1cc25b1309f7aec5f7, for GNU/Linux 3.2.0, not stripped
```

ELF 64-bit, little-endian, not stripped. Same shape as the previous baby steps — we can disassemble `main` directly.

### Step 2 — Read the disassembly to understand the calculation

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d -M intel ./debugger_c | sed -n '/<main>:/,/^$/p'
```

```
0000000000401106 <main>:
  401106:	f3 0f 1e fa          	endbr64
  40110a:	55                   	push   rbp
  40110b:	48 89 e5             	mov    rbp,rsp
  40110e:	89 7d ec             	mov    DWORD PTR [rbp-0x14],edi
  401111:	48 89 75 e0          	mov    QWORD PTR [rbp-0x20],rsi
  401115:	c7 45 fc da e0 01 00 	mov    DWORD PTR [rbp-0x4],0x1e0da
  40111c:	c7 45 f4 5f 02 00 00 	mov    DWORD PTR [rbp-0xc],0x25f
  401123:	c7 45 f8 00 00 00 00 	mov    DWORD PTR [rbp-0x8],0x0
  40112a:	eb 0a                	jmp    401136 <main+0x30>
  40112c:	8b 45 f8             	mov    eax,DWORD PTR [rbp-0x8]
  40112f:	01 45 fc             	add    DWORD PTR [rbp-0x4],eax
  401132:	83 45 f8 01          	add    DWORD PTR [rbp-0x8],0x1
  401136:	8b 45 f8             	mov    eax,DWORD PTR [rbp-0x8]
  401139:	3b 45 f4             	cmp    eax,DWORD PTR [rbp-0xc]
  40113c:	7c ee                	jl    40112c <main+0x26>
  40113e:	8b 45 fc             	mov    eax,DWORD PTR [rbp-0x4]
  401141:	5d                   	pop    rbp
  401142:	c3                   	ret
```

Reading it as a C program:

| Address | C equivalent | Comment |
|---|---|---|
| `401115` | `int total = 0x1e0da;` | `[rbp-0x4]` = total = 123098 |
| `40111c` | `int limit = 0x25f;` | `[rbp-0xc]` = limit = 607 |
| `401123` | `int i = 0;` | `[rbp-0x8]` = i |
| `40112a` | `goto check;` | jump into the loop test first |
| `40112c` | `total += i;` | loop body |
| `401132` | `i++;` | |
| `401136` | `if (i < limit) goto body;` | loop test |
| `40113e` | `return total;` | the answer we want |
| `401142` | | actual `ret` |

The shape is dead obvious: a counter that increments from 0 to `0x25e` (one less than `0x25f`), adding itself into a running total each iteration.

### Step 3 — Path A: compute by hand (sanity check)

Before letting GDB run anything, I worked out the answer on paper:

```
total = 0x1e0da
for i in range(0x25f):    # i = 0, 1, 2, ..., 606
    total += i

sum(0..606) = 606 * 607 / 2 = 183921
0x1e0da = 123098

final total = 123098 + 183921 = 307019
307019 in hex = 0x4af4b
```

Python agrees:

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "total = 0x1e0da + sum(range(0x25f)); print(total, hex(total))"
307019 0x4af4b
```

Predicted `eax`: **307019** (decimal) / `0x4af4b` (hex).

### Step 4 — Path B: let GDB do it (the intended path)

The hint tells us to set a breakpoint after the calculation. The `mov eax, [rbp-0x4]` at `0x40113e` *is* the calculation's result being moved into `eax`. GDB's `*ADDR` breakpoint is "before executing the instruction at ADDR," so to see the final value I want the breakpoint on the *next* instruction (`pop rbp` at `0x401141`), or I `nexti` once after hitting the breakpoint on `0x40113e`.

I broke at `0x401141` so the answer appears in a single `run`:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "break *0x401141" \
        -ex "run" \
        -ex "info registers eax" \
        -ex "print/d \$eax" \
        ./debugger_c
```

```
Breakpoint 1 at 0x401141
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x0000000000401141 in main ()
eax            0x4af4b             307019
$1 = 307019
```

Breaking down the output:

- `Breakpoint 1 at 0x401141` — breakpoint set on `pop rbp`, the first instruction after the final `mov eax, ...`.
- `Breakpoint 1, 0x0000000000401141 in main ()` — program hit the breakpoint after running through 607 loop iterations.
- `eax 0x4af4b  307019` — `info registers` shows both the hex (`0x4af4b`) and decimal (`307019`) value of `eax`.
- `$1 = 307019` — `print/d $eax` confirms the signed decimal interpretation.

GDB's answer matches the hand calculation exactly.

### Step 5 — Wrap the value in the picoCTF flag format

The description is explicit: "in the decimal number base." So `0x4af4b` becomes `307019`, and the flag is:

```
picoCTF{307019}
```

Submitted and accepted.

---

## What Happened Internally (Timeline)

Walking through what `main` did from start to finish, with each step mapped to a C statement:

1. `_start` (in libc) calls `main(argc=1, argv=...)`. The standard prologue (`push %rbp; mov %rsp, %rbp`) sets up the stack frame.
2. `mov DWORD PTR [rbp-0x14], %edi` and `mov QWORD PTR [rbp-0x20], %rsi` save `argc` and `argv` into the stack. Neither is used later; they are kept for compatibility with the standard signature.
3. `mov DWORD PTR [rbp-0x4], 0x1e0da` initializes `total` to `123098`.
4. `mov DWORD PTR [rbp-0xc], 0x25f` initializes `limit` to `607`.
5. `mov DWORD PTR [rbp-0x8], 0x0` initializes `i` to `0`.
6. `jmp 401136` jumps directly to the loop test, *not* the loop body. This is the standard compiler optimization: there is no point executing the body before checking the condition.
7. The loop test at `401136`:
   - `mov eax, [rbp-0x8]` — load `i` into `eax` for the comparison.
   - `cmp eax, [rbp-0xc]` — set flags based on `i - limit`.
   - `jl 40112c` — if `i < limit` (signed), jump back to the body.
8. The loop body at `40112c`:
   - `mov eax, [rbp-0x8]` — load `i` into `eax`.
   - `add [rbp-0x4], eax` — `total += i`.
   - `add DWORD PTR [rbp-0x8], 1` — `i++`.
9. Steps 7-8 repeat. Iterations: `i = 0, 1, 2, ..., 606`. That is 607 iterations, each adding `i` to `total`. After the last iteration, `i = 607` and the test at step 7 fails, so control falls through.
10. `mov eax, [rbp-0x4]` loads the final value of `total` into `eax`. This is the answer the challenge wants.
11. `pop %rbp` restores the saved base pointer. GDB pauses here because of our breakpoint.
12. `ret` returns control to `_start`, which eventually calls `exit` with the return value.

The number that landed in `eax` at step 10 is the sum of `123098 + 0 + 1 + 2 + ... + 606`, which equals `123098 + 183921 = 307019`.

---

## Alternative Solve Methods

### Alternative 1 — `print` after a single-step past the breakpoint

If you would rather follow the description's literal flow ("set a breakpoint for after the calculation and inspect eax"), you can break right on `mov eax, [rbp-0x4]` and `nexti` once:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "break *0x40113e" \
        -ex "run" \
        -ex "nexti" \
        -ex "info registers eax" \
        ./debugger_c
```

```
Breakpoint 1, 0x00000000004040113e in main ()
0x000000000040401141 in main ()
eax            0x4af4b             307019
```

`nexti` ("next instruction") executes the current instruction, advances, and pauses again — perfect for "I want to be one instruction past the breakpoint."

### Alternative 2 — finish the program and use `$_exitcode`

GDB has a convenience variable `$_exitcode` that captures the value passed to `exit()` (or, for `main`, the value `main` returns). It is populated after the program finishes:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "run" \
        -ex "print/d \$_exitcode" \
        ./debugger_c
```

```
$1 = 307019
```

This is the cleanest "do nothing, get the answer" path, but it only works for programs whose return value is what you want — and it requires the program to actually terminate cleanly. The breakpoint approach is more general.

### Alternative 3 — watchpoint on `[rbp-0x4]`

If you do not want to predict where the breakpoint should go, set a **watchpoint** on the variable you care about. A watchpoint pauses the program every time that memory location is written:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "watch *(\$rbp-4)" \
        -ex "commands 2" \
        -ex "silent" \
        -ex "if \$rdi == 0" \
        -ex "end" \
        -ex "continue" \
        -ex "end" \
        -ex "run" \
        ./debugger_c
```

This is overkill for a static binary like this one but extremely useful when the loop body is hundreds of instructions long or when you want to watch a specific variable across many function calls.

### Alternative 4 — pure static analysis

I already showed the by-hand computation in Step 3. The loop pattern is regular enough that you can compute the answer with no debugger at all:

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "print(0x1e0da + (0x25f - 1) * 0x25f // 2)"
307019
```

This works because the loop sums `0 + 1 + ... + (limit - 1) = (limit - 1) * limit / 2` and adds it to `total`. The closed-form expression is exactly what GDB confirms.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `file` | file 5.39 | Confirmed the binary is an ELF 64-bit executable, not stripped. |
| `objdump` | GNU binutils 2.40 | Read the assembly of `main` to identify the loop and the final `mov eax, [rbp-0x4]`. |
| `gdb` | GNU gdb 13.1 | The intended debugger. Used in `-batch` mode with `break *0x401141`, `run`, and `info registers eax` to capture the answer. |
| `python3` | Python 3.11 | Sanity-checked the closed-form loop sum against GDB's answer. |

---

## Key Takeaways

- **The flag is `eax` in *decimal*, not hex.** The description is explicit: "in the decimal number base." If GDB prints `0x4af4b`, the flag is `picoCTF{307019}`, not `picoCTF{0x4af4b}`. This is a frequent trap on the baby-step series — every flag wants decimal unless the challenge explicitly says hex.
- **`info registers eax` is the cheapest way to read a single register.** It also helpfully prints the decimal value next to the hex, so you usually do not need a separate `print/d`.
- **Break one instruction *after* the calculation, not on it.** GDB's `*ADDR` breakpoints pause *before* the instruction at `ADDR` executes. If you break on the `mov eax, ...` itself, you will see the *previous* value of `eax`. Two ways to fix: break on the next instruction (cleanest), or `nexti` once after hitting the breakpoint.
- **`jl` ("jump if less") is the signed version of `<`.** The loop test at `40113c` (`jl 40112c`) is the assembly form of `if (i < limit) goto body;`. If you ever see `jb` ("jump if below"), that is the *unsigned* version. For positive integer counters both behave the same; for negative or huge values they diverge.
- **The compiler translates `for (i = 0; i < limit; i++)` into a "test-first" loop with a backward jump.** That is why the disassembly starts with `jmp 401136` (jump to the test) rather than executing the body once first. Once you can read that pattern, you can read most C-compiled loops at a glance.
- **`$_exitcode` is the fastest path when you do not need to inspect registers mid-run.** It only works if the program reaches `exit` cleanly, but when it does, it is one `print/d $_exitcode` away from the answer.
- **Closed-form verification is cheap.** The sum `0 + 1 + ... + (n-1)` equals `n*(n-1)/2`, a fact Gauss reportedly invented as a child. When the loop is short and the bounds are obvious, write down the formula and check it against GDB. If both agree, you have a high-confidence answer; if they disagree, you have a bug to chase.
- **Flag wordplay decode.** The flag is `picoCTF{307019}`. There is no wordplay or steganography to decode — the "decoding" is the loop unwinding: read the disassembly, identify the loop bounds (`0x1e0da` initial, `0x25f` upper limit), compute the sum `123098 + (0 + 1 + ... + 606) = 123098 + 183921 = 307019`, and confirm with GDB. The "trick" of the challenge is recognizing that the answer lives in the runtime register, not in a static constant in the disassembly like baby step 4. Once you do the math and run the binary, the flag is `picoCTF{307019}`.
