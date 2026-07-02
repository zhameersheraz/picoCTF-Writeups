# GDB baby step 1 — picoCTF Writeup

**Challenge:** GDB baby step 1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{549698}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> Can you figure out what is in the eax register at the end of the main function? Put your answer in the picoCTF flag format: picoCTF{n} where n is the contents of the eax register in the decimal number base. If the answer was 0x11 your flag would be picoCTF{17}.

## Hints

> 1. gdb is a very good debugger to use for this problem and many others!
> 2. main is actually a recognized symbol that can be used with gdb commands.

---

## Background Knowledge (Read This First!)

If you have read the baby steps 2-4 writeups already, you have most of the context you need. This challenge is the same shape with a much smaller `main` — no loop, no function call, no memory load. Just one `mov` into `eax` and a `ret`. Here is the small extra vocabulary that makes the solution feel like a single confident step instead of a guess.

### The x86 `mov` Immediate Forms

`mov` has a thousand forms, but the one we care about here is "move a 32-bit immediate into `eax`":

```
1138: b8 42 63 08 00    mov    eax, 0x86342
```

That single instruction is the *entire* challenge. It puts the literal value `0x86342` into `eax` and that's it. When `main` returns a few instructions later, `eax` still holds `0x86342`.

### PIE vs. Non-PIE Binaries (Why This One Loads at `0x555555555000`)

`file` reported `pie executable` for this binary. **PIE** stands for **P**osition-**I**ndependent **E**xecutable. PIE is the modern default on most Linux distributions: the linker produces a binary whose addresses are *relative*, and the kernel picks a random base address at load time (this is part of ASLR, **A**ddress **S**pace **L**ayout **R**andomization).

What this means for our solve:

- `objdump` shows `main` at offset `0x1129` from the start of the binary.
- At runtime, `main` lives at some random address like `0x555555555129` (the offset `0x1129` plus the load base).
- GDB is smart enough to resolve `main` by symbol regardless of where it ends up in memory, so you can still `break main`, `disassemble main`, or `break *(main+15)` and have it work.

If you ever need to point GDB at an instruction by raw address, use the *symbol-offset* form (`main+15`) rather than the on-disk address (`0x1138`), because the on-disk address is not where the instruction lives at runtime.

### Why `break main` Sometimes Pauses Partway Through `main`

You might notice that `break main` followed by `info registers eax` does *not* show `0x86342` — it shows some garbage from libc's call to `main`. That is because on modern Linux with CET (Control-flow Enforcement Technology), the function starts with an `endbr64` instruction and a few bytes of prologue, and GDB's "function entry" breakpoint often resolves to *just after* the prologue. The cleanest fix is to break on a specific instruction by symbol-offset, exactly the way the previous baby steps did.

For this challenge, the cleanest pattern is:

```
break *(main+20)        # break on the pop rbp right after mov eax
```

That guarantees the `mov eax, 0x86342` has executed and `eax` is the answer.

### The Decimal-vs-Hex Trap

The description works through the example `0x11` → `picoCTF{17}`. The flag is the *decimal* value, not the hex spelling. If GDB prints `eax 0x86342`, the flag is `picoCTF{549698}`, not `picoCTF{0x86342}`. This trap is the entire reason the worked example is in the description — read it carefully before submitting.

---

## Solution — Step by Step

### Step 1 — Make the binary executable and identify it

```
┌──(zham㉿kali)-[~/ctf]
└─$ cp ~/Downloads/debugger_d .
┌──(zham㉿kali)-[~/ctf]
└─$ chmod +x debugger_d
┌──(zham㉿kali)-[~/ctf]
└─$ file debugger_d
debugger_d: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=15a10290db2cd2ec0c123cf80b88ed7d7f5cf9ff, for GNU/Linux 3.2.0, not stripped
```

Two things to note:

- **pie executable** — position-independent. The binary will be loaded at a random base address, so we will use symbol-relative breakpoints (`main+offset`) rather than raw addresses.
- **not stripped** — symbols are intact, so `main` is a valid symbol for GDB.

### Step 2 — Read the disassembly to see what `main` does

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d -M intel ./debugger_d | sed -n '/<main>:/,/^$/p'
```

```
0000000000001129 <main>:
    1129:	f3 0f 1e fa          	endbr64
    112d:	55                   	push   rbp
    112e:	48 89 e5             	mov    rbp,rsp
    1131:	89 7d fc             	mov    DWORD PTR [rbp-0x4],edi
    1134:	48 89 75 f0          	mov    QWORD PTR [rbp-0x10],rsi
    1138:	b8 42 63 08 00       	mov    eax,0x86342
    113d:	5d                   	pop    rbp
    113e:	c3                   	ret
    113f:	90                   	nop
```

Reading it as a C program:

| Address | C equivalent | Comment |
|---|---|---|
| `1129-112e` | (function prologue) | `endbr64; push rbp; mov rbp, rsp` |
| `1131` | `argc = edi;` | save `argc` to the stack (unused) |
| `1134` | `argv = rsi;` | save `argv` to the stack (unused) |
| `1138` | `return 0x86342;` | **the entire challenge** |
| `113d-113e` | (epilogue) | restore `rbp`, return to caller |

The disassembly alone tells us the answer: `eax` will be `0x86342` at the end of `main`. We will confirm this with GDB.

### Step 3 — Predict the answer on paper

```
0x86342 = 8*16^4 + 6*16^3 + 3*16^2 + 4*16 + 2
       = 8*65536 + 6*4096 + 3*256 + 64 + 2
       = 524288 + 24576 + 768 + 64 + 2
       = 549698
```

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "print(0x86342)"
549698
```

Predicted answer: **549698** (decimal) / `0x86342` (hex).

### Step 4 — Confirm with GDB

The hint tells us `main` is a recognized symbol, so we can use `break main` or, more precisely, `break *(main+20)` to land right after the `mov eax`:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "break *(main+20)" \
        -ex "run" \
        -ex "info registers eax" \
        -ex "print/d \$eax" \
        ./debugger_d
```

```
Breakpoint 1 at 0x113d
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x000055555555513d in main ()
eax            0x86342             549698
$1 = 549698
```

Breaking down the output:

- `Breakpoint 1 at 0x113d` — GDB resolved `main+20` to the runtime address `0x55555555513d` (PIE base plus offset `0x113d`).
- `Breakpoint 1, 0x000055555555513d in main ()` — the program ran and paused exactly where we wanted.
- `eax  0x86342  549698` — `info registers` shows the value in both hex and decimal side-by-side.
- `$1 = 549698` — `print/d $eax` confirms the signed decimal value is `549698`.

GDB's answer matches the static prediction: `0x86342` / `549698`.

### Step 5 — Wrap the value in the picoCTF flag format

The description's worked example is `0x11` → `picoCTF{17}`. Same conversion:

```
picoCTF{549698}
```

Submitted and accepted.

---

## What Happened Internally (Timeline)

Walking through `main` from start to finish, step by step:

1. `_start` (in libc) calls `main(argc=1, argv=...)`. `argc` lands in `edi`, `argv` lands in `rsi`. The runtime address of `main` is something like `0x555555555129` (PIE base + offset `0x1129`).
2. `endbr64` at `0x1129` is a no-op for our purposes — it is an indirect-branch landing pad for Intel CET, used by the hardware to validate that indirect calls and jumps land on a legitimate instruction.
3. `push %rbp` saves the caller's base pointer on the stack. `mov %rsp, %rbp` anchors a fresh stack frame so locals can be addressed relative to `rbp`.
4. `mov DWORD PTR [rbp-0x4], %edi` saves `argc` at `[rbp-0x4]`. `mov QWORD PTR [rbp-0x10], %rsi` saves `argv` at `[rbp-0x10]`. Neither is used again; they exist only to match the standard `int main(int argc, char **argv)` signature.
5. The whole challenge in one instruction: `mov $0x86342, %eax`. The CPU loads the 32-bit immediate `0x86342` into `eax`, zeroing the upper 32 bits of `rax` in the process (because the 32-bit operand encoding forces a `rax` zero-extension).
6. `pop %rbp` restores the caller's base pointer. GDB pauses here because of our breakpoint at `main+20`.
7. `ret` pops the saved return address and jumps back into libc startup code, which eventually calls `exit` with `eax`'s value (`0x86342`).

`eax` is the answer from step 5 onward. There is no further modification — no loop, no arithmetic, no function call. The value `549698` is what `main` returns.

---

## Alternative Solve Methods

### Alternative 1 — predict entirely from `objdump` (no GDB run)

Because the value is loaded with a single literal `mov`, the answer is right there in the disassembly. You can solve this challenge with `objdump` alone, no debugger needed:

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d -M intel ./debugger_d | sed -n '/<main>:/,/^$/p' | grep mov
    1131:	89 7d fc             	mov    DWORD PTR [rbp-0x4],edi
    1134:	48 89 75 f0          	mov    QWORD PTR [rbp-0x10],rsi
    1138:	b8 42 63 08 00       	mov    eax,0x86342
```

The third line is the answer: `eax = 0x86342`.

### Alternative 2 — `$_exitcode` after the run

Just like baby step 2, you can let the program finish and read `$_exitcode`:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "run" \
        -ex "print/d \$_exitcode" \
        ./debugger_d
```

```
$1 = 549698
```

Cleanest single-command path, but only useful because we know `eax` is never modified between the `mov` and `exit`. If the function did anything else after the `mov eax`, this would give the *final* `eax`, which may or may not be the same as the one we wanted.

### Alternative 3 — `print` directly without running the program

GDB can disassemble and even evaluate expressions against a binary without actually running it. That said, for this challenge you would still need to read the immediate from the disassembly — so this path gives no advantage over `objdump`. Listed for completeness only.

### Alternative 4 — breakpoint by raw address

If you want to see the PIE base address in action, you can breakpoint by raw address instead of symbol:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "starti" \
        -ex "info proc mappings" \
        ./debugger_d
```

This prints the runtime base address. You can then set the breakpoint as `break *0x55555555513d` (or whatever the resolved runtime address is). For real-world reversing this is rarely useful — symbol-relative breakpoints (`break *(main+20)`) are easier and survive ASLR for free.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `file` | file 5.39 | Confirmed the binary is an ELF 64-bit PIE executable (so we know to use symbol-relative breakpoints). |
| `objdump` | GNU binutils 2.40 | Read the assembly of `main` and identified the `mov eax, 0x86342` instruction — the entire challenge. |
| `gdb` | GNU gdb 13.1 | The intended debugger. Used in `-batch` mode with `break *(main+20)`, `run`, and `info registers eax` to confirm the answer. |
| `python3` | Python 3.11 | Sanity-checked the hex-to-decimal conversion `0x86342` → `549698`. |

---

## Key Takeaways

- **This is the "GDB hello world" of the series.** There is no loop, no function call, no memory trick. The challenge is just "can you find the constant, convert hex to decimal, and wrap it in `picoCTF{...}`?" Treat it as a confidence-builder: if you can solve this, you have the full toolchain working — `file`, `objdump`, `gdb -batch`, `info registers`, and the decimal flag format.
- **PIE changes nothing about the *logic* of the solve, but it changes the *breakpoint syntax*.** On a PIE binary, the runtime addresses shift every run. Use symbol-relative breakpoints (`break *(main+20)`) instead of raw addresses (`break *0x40113d`) so the breakpoint survives ASLR.
- **`break main` does not always mean "break at the very first instruction."** On modern Linux with CET, GDB often resolves `main` to *after* the `endbr64` and prologue. That is fine for most reverse engineering, but it means `eax` (and other caller-saved registers) still hold values from before `main` ran. If you want a specific instruction's effect visible, break *just after* it.
- **`info registers <reg>` is the cheat code for register questions.** It prints the value in hex *and* decimal simultaneously. You almost never need a separate `print/d $reg` unless you want the value in a script.
- **Read the worked example in the description.** "If the answer was 0x11 your flag would be picoCTF{17}" is not flavor text — it is the format spec. Decimal in, hex out, no `0x` prefix in the flag. Get this wrong and you submit `picoCTF{0x86342}` instead of `picoCTF{549698}`.
- **`b8` is the opcode for `mov eax, imm32`.** When you scan disassembly and see `b8 ?? ?? ?? ??`, that is "load this 4-byte little-endian immediate into `eax`." You can mentally parse the answer without even running the program.
- **Flag wordplay decode.** The flag is `picoCTF{549698}`. There is no wordplay to decode — the trick is the same hex-to-decimal conversion that runs through the whole baby-step series. Read the disassembly, find `0x86342`, convert to decimal (`8 * 65536 + 6 * 4096 + 3 * 256 + 4 * 16 + 2 = 549698`), wrap in `picoCTF{...}`. Once you do the conversion, the flag is `picoCTF{549698}`.
