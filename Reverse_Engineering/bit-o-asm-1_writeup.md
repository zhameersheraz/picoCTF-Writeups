# Bit-O-Asm-1 — picoCTF Writeup

**Challenge:** Bit-O-Asm-1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{48}`  
**Platform:** picoCTF 2022 (picoGym)  
**Writeup by:** zham  

---

## Description

> Can you figure out what is in the `eax` register? Put your answer in the picoCTF flag format: `picoCTF{n}` where n is the contents of the `eax` register in the decimal number base. If the answer was `0x11` your flag would be `picoCTF{17}`.
>
> Download the assembly dump [here].

## Hints

> 1. As with most assembly, there is a lot of noise in the instruction dump. Find the one line that pertains to this question and don't second guess yourself!

---

## Background Knowledge (Read This First!)

This is a "bit of assembly" challenge — short disassembly, no binary to run, just read the lines and find the answer. You do not need GDB, you do not need to compile anything, you do not even need to be on a specific architecture. You only need to be able to read one line of x86 assembly and convert one hex number to decimal. Here is the vocabulary that makes the whole thing a one-minute solve.

### What is an "Assembly Dump"

An assembly dump is just a text listing of machine instructions written in their human-readable mnemonics (`mov`, `push`, `ret`, …). When you run `objdump -d` on a binary, or paste output from a debugger like GDB, or even just open a `.txt` file the challenge gives you, you are looking at an assembly dump. Each line tells you "the CPU did this at this point in the program."

For this challenge, picoCTF hands you the dump directly. No binary, no GDB, no objdump. You just open the file and read it.

### What is `eax`?

`eax` is a CPU register on x86 systems. Think of it as one of the small, super-fast named boxes inside the processor that programs use to do arithmetic, pass function return values, and store temporary results. `eax` is the lower 32 bits of the 64-bit `rax` register.

In a C-like mental model:

```c
int eax;            // 32-bit register, used for return values
```

When a function in C does `return 42;`, the compiled assembly moves `42` into `eax` and then runs `ret`. The calling code reads `eax` to get the return value. So if we know what got moved into `eax` right before `ret`, we know what the function "returned" — and that is the entire question.

### The `mov` Instruction

`mov` is "copy this value into that place." The syntax is:

```
mov   destination, source
```

The hint at the top of the challenge literally tells us this: **"the `mov` instruction moves the second operand into the first operand."** So if you see `mov eax, 0x30`, the CPU takes the value `0x30` (the second operand) and copies it into `eax` (the first operand). After that instruction, `eax` holds `0x30`.

### The Other Lines Are Noise

The hint also tells us there is a "lot of noise." That is the standard prologue and epilogue every C function gets compiled with:

- `endbr64` — Intel CET landing pad. Modern CPUs use this to validate indirect branches. We can ignore it.
- `push rbp` / `mov rbp, rsp` — set up the stack frame so locals can be addressed as `[rbp-0x4]`, `[rbp-0x10]`, etc.
- `mov DWORD PTR [rbp-0x4], edi` and `mov QWORD PTR [rbp-0x10], rsi` — save the `argc` and `argv` arguments that came in from the caller. They are never read again.
- `pop rbp` / `ret` — tear down the stack frame and return to whoever called us.

None of that changes `eax`. The only instruction in the entire dump that touches `eax` is the one we care about.

### Hex to Decimal

The flag format is decimal. The dump gives us a hex literal. So we need to convert. For a single-digit hex like `0x30` you can do this by hand:

```
0x30 = 3 * 16^1 + 0 * 16^0
     = 3 * 16  + 0 * 1
     = 48 + 0
     = 48
```

For anything bigger, use `python3 -c "print(0x30)"` or the calculator. The challenge worked example `0x11` → `17` is the same trick: `1 * 16 + 1 = 17`.

---

## Solution — Step by Step

### Step 1 — Get the assembly dump

I downloaded the file the challenge linked to. Let us call it `bit-o-asm-1.S` (or `.txt`, the extension does not matter — it is just text). Save it into a working folder.

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ mkdir -p ~/ctf/bit-o-asm && cd ~/ctf/bit-o-asm
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ls
```

The browser saves the dump into `~/Downloads` by default. Let us move it:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ mv ~/Downloads/*.txt ./bit-o-asm-1.txt
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ls -la
total 8
drwxr-xr-x 2 zham zham 4096 Jul  2 22:15 .
drwxr-xr-x 3 zham zham 4096 Jul  2 22:15 ..
-rw-r--r-- 1 zham zham  140 Jul  2 22:15 bit-o-asm-1.txt
```

Tiny file, 140 bytes. Pure text.

### Step 2 — Read the dump

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ cat bit-o-asm-1.txt
```

```
<+0>:     endbr64
<+4>:     push   rbp
<+5>:     mov    rbp,rsp
<+8>:     mov    DWORD PTR [rbp-0x4],edi
<+11>:    mov    QWORD PTR [rbp-0x10],rsi
<+15>:    mov    eax,0x30
<+20>:    pop    rbp
<+21>:    ret
```

That is the whole challenge. Eight lines.

### Step 3 — Find the line that touches `eax`

Per the hint, scan for the only line that mentions `eax`. It is line `<+15>`:

```
<+15>:    mov    eax,0x30
```

Reading it as English: "move the value `0x30` into `eax`." After this instruction executes, `eax == 0x30`. Nothing else in the dump writes to `eax` (the prologue only touches `rbp`, `rsp`, and stack memory), so `eax` stays at `0x30` until `ret` returns it to the caller.

### Step 4 — Convert `0x30` to decimal

The flag format wants the decimal value, so I run the conversion. Two ways.

The mental way:

```
0x30 = 3 * 16 + 0 = 48
```

The terminal way:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(0x30)"
48
```

Either way: `eax` holds **48** when `main` returns.

### Step 5 — Wrap in the flag format

The challenge description worked the example `0x11` → `picoCTF{17}`. Same conversion, same format:

```
picoCTF{48}
```

Submit it on the picoCTF challenge page, accept.

---

## What Happened Internally (Timeline)

Walking through the dump from start to finish, step by step, so we can see why every line other than `<+15>` is irrelevant:

1. The function is entered. The CPU jumps to `<+0>` because `_start` (in libc) called us. At this point `rax` holds whatever the C runtime left in it (some return value from the previous function). `eax` is the low 32 bits of that — some garbage we do not care about.
2. `endbr64` at `<+0>` is a CET landing pad. It does not modify any general-purpose registers. We continue.
3. `push rbp` saves the caller's base pointer on the stack. `mov rbp, rsp` anchors a fresh stack frame. `eax` is still untouched.
4. `mov DWORD PTR [rbp-0x4], edi` saves `argc` to the stack. `mov QWORD PTR [rbp-0x10], rsi` saves `argv` to the stack. Neither instruction writes to `eax`. They are just complying with the C `int main(int argc, char **argv)` signature.
5. `<+15>: mov eax, 0x30` — the entire challenge. The CPU loads the 32-bit immediate `0x30` into `eax`. Because the 32-bit destination zero-extends into the full 64-bit `rax`, `rax` is now also `0x30`. From here on, `eax` is `0x30`.
6. `pop rbp` restores the caller's base pointer. `eax` is still `0x30`.
7. `ret` pops the saved return address and jumps back into libc startup code, which eventually calls `exit` with `eax`'s value (`0x30`). The program terminates.

`eax` is the answer from step 5 onward. There is no further modification — no arithmetic, no function call, no comparison, no jump that could skip the assignment. The decimal value `48` is what `main` "returned."

---

## Alternative Solve Methods

### Alternative 1 — `printf` instead of `python3`

If you do not want to spawn the Python interpreter just to convert one byte, `printf` does it inline:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ printf '%d\n' 0x30
48
```

`printf` accepts the `0x` prefix and prints the value in decimal. Same answer, no script.

### Alternative 2 — mental math only (no terminal at all)

For small hex literals like `0x30`, you can just multiply in your head. The hex digits are `3` and `0`, so the value is `3 * 16 + 0 = 48`. This is faster than opening a terminal, and it is what you will end up doing by reflex once you have solved a few Bit-O-Asm challenges.

A handy mental table for the most common cases:

| Hex | Decimal |
|---|---|
| `0x10` | 16 |
| `0x20` | 32 |
| `0x30` | 48 |
| `0x40` | 64 |
| `0x50` | 80 |
| `0x100` | 256 |
| `0x1000` | 4096 |

If the literal in the dump happens to be in this table, you do not even need a calculator.

### Alternative 3 — `bc` (the Unix bench calculator)

`bc` is the old-school way to do arbitrary-precision math on the command line. It accepts hex with the `-l` and `-i` flags (well, technically `bc` does not directly parse hex, but you can pipe in `echo "obase=10; ibase=16; 30" | bc`):

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ echo "obase=10; ibase=16; 30" | bc
48
```

This is overkill for a one-digit hex, but it is the right tool if you ever face a 10-digit hex literal in a harder Bit-O-Asm problem and want to script the conversion.

### Alternative 4 — write a tiny reusable script

If you are about to solve the whole Bit-O-Asm-1/2/3/4 series, you will do this conversion four times. A small script pays for itself. Create `hex2dec.py`:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ nano hex2dec.py
```

Inside nano, type the following content:

```
#!/usr/bin/env python3
"""Convert a hex literal (with or without the 0x prefix) to decimal."""

import sys

if len(sys.argv) != 2:
    print(f"usage: {sys.argv[0]} <hex>", file=sys.stderr)
    sys.exit(1)

literal = sys.argv[1]
if literal.startswith(("0x", "0X")):
    literal = literal[2:]

print(int(literal, 16))
```

Save with `Ctrl+O`, press `Enter` to confirm the filename, then exit with `Ctrl+X`.

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ chmod +x hex2dec.py
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ./hex2dec.py 0x30
48
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ./hex2dec.py 30
48
```

Now you have a one-shot tool for the rest of the series. The Bit-O-Asm-2 dump gives you `0x9fe1a`, Bit-O-Asm-3 dumps a computed value, Bit-O-Asm-4 dumps a branchy conditional — same script handles all of them.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `cat` | coreutils 9.1 | Read the assembly dump file directly. No special tooling needed — it is just text. |
| `python3` | Python 3.11 | Converted the hex literal `0x30` to its decimal value `48`. |
| `printf` | coreutils 9.1 | Alternative one-liner for the same hex-to-decimal conversion. |
| `bc` | GNU bc 1.07 | Listed as an alternative for users who prefer the classic Unix calculator to Python. |
| `nano` | GNU nano 7.2 | Wrote the small reusable `hex2dec.py` helper script in the Alternative Methods section. |

---

## Key Takeaways

- **This is the "assembly hello world" of the Bit-O-Asm series.** There is no binary, no debugger, no control flow. The challenge is just "find the one `mov eax` line, convert the hex literal to decimal, wrap in `picoCTF{...}`." Treat it as a confidence-builder: if you can solve this, you have the hex-to-decimal reflex you will reuse on Bit-O-Asm-2, -3, and -4.
- **The hint is doing real work.** "Find the one line that pertains to this question" is telling you that the prologue (`push rbp`, `mov rbp, rsp`, the `argc`/`argv` saves) and the epilogue (`pop rbp`, `ret`) never touch `eax`. Scan for the literal string `eax` in the dump; whatever instruction writes to it is the answer. Do not try to mentally execute every line.
- **"`mov` moves the second operand into the first" is the most important sentence in the prompt.** AT&T-syntax assembly (`mov src, dst`) and Intel-syntax assembly (`mov dst, src`) disagree on operand order. picoCTF uses the **Intel** convention here (`mov eax, 0x30` means "put `0x30` into `eax`"). If you have only seen AT&T syntax before, it is easy to read this backwards and submit the wrong flag. Read the worked example in the description to confirm the order.
- **The flag format wants decimal, not hex.** The challenge description works through `0x11` → `picoCTF{17}` for exactly this reason. If the dump says `0x30`, the flag is `picoCTF{48}`, not `picoCTF{0x30}`. Submitting the hex spelling is the most common failure on this challenge.
- **No `0x` prefix in the flag.** The braces wrap the bare decimal digits, not the hex spelling. `picoCTF{0x30}` would be marked wrong even though `0x30 == 48`.
- **You can solve this entire challenge from your phone.** No VM, no Kali, no GDB. The file is 140 bytes of plain text, the conversion is single-digit hex, and the only "tool" you need is the ability to read one line of assembly. Keep that in mind when the next challenge in the series ramps up the difficulty.
- **Flag wordplay decode.** The flag is `picoCTF{48}`. There is no wordplay to decode — the entire challenge is a one-line `mov` (`mov eax, 0x30`), a hex-to-decimal conversion (`0x30 = 3 * 16 + 0 = 48`), and the picoCTF brace format. The "bit" in Bit-O-Asm is a play on "a bit of assembly" — one tiny assembly snippet per challenge. Once you do the conversion, the flag is `picoCTF{48}`.
