# GDB baby step 4 — picoCTF Writeup

**Challenge:** GDB baby step 4  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{12905}`  
**Platform:** picoCTF (General Skills origin, RE placement)  
**Writeup by:** zham  

---

## Description

> main calls a function that multiplies eax by a constant. The flag for this challenge is that constant in decimal base. If the constant you find is 0x1000, the flag will be picoCTF{4096}.

## Hints

> 1. A function can be referenced by either its name or its starting address in gdb.

---

## Background Knowledge (Read This First!)

If any of the terms below feel fuzzy, take a minute to read through them. The whole challenge collapses into "find one number in the disassembly," so it helps to know what the disassembly is actually saying.

### What is GDB?

**GDB** — the **G**NU **D**e**b**ugger — is the standard command-line debugger for Linux. It lets you start a program, pause it wherever you want, step through it one instruction at a time, inspect registers and memory, and change values on the fly. For reversing challenges it is overkill in the best possible way: you can ask it to *just disassemble a function* without ever running the program, which is exactly what we are going to do here.

A minimal GDB session looks like this:

```
gdb ./binary                      # open the binary in gdb
(gdb) info functions              # list every function in the binary
(gdb) disassemble main            # show the assembly of main
(gdb) break main                  # set a breakpoint at the start of main
(gdb) run                         # start the program
(gdb) info registers              # dump every CPU register
(gdb) quit                        # exit
```

You can also drive GDB non-interactively with `-batch -ex "command"`, which is great for scripting and for writing up clean terminal output for a writeup.

### What is Disassembly?

When you compile C into a binary, the compiler turns every line of C into one or more **machine code** instructions. Those instructions are stored in the binary as raw bytes. **Disassembly** is the process of turning those bytes back into a human-readable listing of the instructions, using the mnemonics the CPU understands.

For an x86-64 binary, an instruction might look like:

```
401114:	69 c0 69 32 00 00    	imul   eax,eax,0x3269
```

Reading that left to right:

- `401114` — the address of this instruction in memory.
- `69 c0 69 32 00 00` — the raw bytes of the instruction (the encoding).
- `imul` — the mnemonic. "`i`nteger `mul`tiply."
- `eax,eax,0x3269` — the operands: take `eax`, multiply it by `0x3269`, and put the result back into `eax`.

That single line is the entire challenge.

### The `imul` Family of Instructions

`imul` is the signed integer multiply instruction. x86-64 has three common forms you will see in disassembly:

| Form | What it does | Example |
|---|---|---|
| `imul src` | Signed multiply `eax` by `src`, store the low 64 bits in `edx:eax` | `imul ecx` |
| `imul dst, src` | Signed multiply `dst` by `src`, store the low 32 bits in `dst` | `imul eax, ecx` |
| `imul dst, src, imm` | Signed multiply `src` by the immediate `imm`, store the low 32 bits in `dst` | `imul eax, eax, 0x3269` |

Our challenge uses the third form. The third operand is the **immediate constant** — that is the number the description is asking us to find.

### What are the x86-64 General-Purpose Registers?

The registers are tiny on-CPU storage slots the program uses constantly. The ones that show up in this challenge:

- `eax` — 32-bit "accumulator," the lower 32 bits of `rax`. Often used as the return value of a function and for arithmetic.
- `edi` — 32-bit "destination index," the lower 32 bits of `rdi`. By the System V AMD64 ABI (the calling convention on Linux), `edi`/`rdi` holds the **first argument** to a function call.
- `rbp` — "base pointer." Used to anchor the current stack frame so local variables can be addressed relative to it.
- `rsp` — "stack pointer." Points at the top of the current stack frame.
- `rip` — "instruction pointer." The address of the next instruction to execute.

You do not need to memorize them all to solve this challenge — you only need to recognize that when the disassembly says `imul eax, eax, 0x3269`, the **0x3269 is the constant**.

### Why `0x1000` in the Description Example?

The description says "if the constant you find is 0x1000, the flag will be picoCTF{4096}." That is just teaching you the conversion rule:

```
0x1000 = 1*16^3 + 0*16^2 + 0*16^1 + 0*16^0
       = 4096 + 0 + 0 + 0
       = 4096
```

Hex digits go 0–9 then a–f, and each position is a power of 16. As soon as you spot the constant in the `imul`, convert it from base-16 to base-10 and wrap it in `picoCTF{...}`.

### Why the Hint Says "Name or Address"

In GDB, `disassemble func1` and `disassemble 0x401106` do exactly the same thing — GDB resolves the name `func1` to its starting address (`0x401106` in our binary) and then disassembles from there. The hint is reminding you that if you ever strip the symbol table or have a stripped binary, you can still disassemble by address.

---

## Solution — Step by Step

### Step 1 — Make the binary executable and identify it

I copied the challenge file into a working directory, made it executable, and ran `file` on it to confirm it is an ELF binary.

```
┌──(zham㉿kali)-[~/ctf]
└─$ cp ~/Downloads/debugger_a .
┌──(zham㉿kali)-[~/ctf]
└─$ chmod +x debugger_a
┌──(zham㉿kali)-[~/ctf]
└─$ file debugger_a
debugger_a: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=96ad8d8a802a567a7a1a27cf9b7231e2f7fa15f7, for GNU/Linux 3.2.0, not stripped
```

Two things to note:

- **ELF 64-bit LSB** — it is a 64-bit Linux binary, little-endian. That tells us the disassembly will be x86-64 (AT&T or Intel syntax, our choice).
- **not stripped** — the symbol table is intact. `main` and any function it calls will have their names preserved, which is exactly what we need.

### Step 2 — List every function in the binary

The challenge says `main` calls a function that multiplies `eax` by a constant. I want to know the *name* of that function before I disassemble anything. GDB's `info functions` lists every symbol it can see:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch -ex "info functions" ./debugger_a
```

```
All defined functions:

Non-debugging symbols:
0x0000000000401000  _init
0x0000000000401020  _start
0x0000000000401050  _dl_relocate_static_pie
0x0000000000401060  deregister_tm_clones
0x0000000000401090  register_tm_clones
0x00000000004010d0  __do_global_dtors_aux
0x0000000000401100  frame_dummy
0x0000000000401106  func1
0x000000000040111c  main
0x0000000000401150  __libc_csu_init
0x00000000004011c0  __libc_csu_fini
0x00000000004011c8  _fini
```

The output is sorted by address. The custom functions in this binary are:

- `func1` at `0x401106`
- `main` at `0x40111c`

Everything else is libc startup boilerplate. Since `main` is at `0x40111c` and the next user-defined symbol is `func1` at `0x401106` — and `func1` is *before* `main` — it is a great bet that `main` calls `func1`. Let me confirm by disassembling both.

### Step 3 — Disassemble `main` to confirm it calls `func1`

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch -ex "disassemble main" ./debugger_a
```

```
Dump of assembler code for function main:
   0x000000000040111c <+0>:	endbr64
   0x0000000000401120 <+4>:	push   %rbp
   0x0000000000401121 <+5>:	mov    %rsp,%rbp
   0x0000000000401124 <+8>:	sub    $0x20,%rsp
   0x0000000000401128 <+12>:	mov    %edi,-0x14(%rbp)
   0x000000000040112b <+15>:	mov    %rsi,-0x20(%rbp)
   0x000000000040112f <+19>:	movl   $0x28e,-0x4(%rbp)
   0x0000000000401136 <+26>:	movl   $0x0,-0x8(%rbp)
   0x000000000040113d <+33>:	mov    -0x4(%rbp),%eax
   0x0000000000401140 <+36>:	mov    %eax,%edi
   0x0000000000401142 <+38>:	call   0x401106 <func1>
   0x0000000000401147 <+43>:	mov    %eax,-0x8(%rbp)
   0x000000000040114a <+46>:	mov    -0x4(%rbp),%eax
   0x000000000040114d <+49>:	leave
   0x000000000040114e <+50>:	ret
End of assembler dump.
```

Look at the `call` instruction at `main+38`:

```
401142:	e8 bf ff ff ff       	call   0x401106 <func1>
```

`main` is calling `func1`. That matches the description exactly.

The setup for the call is just two instructions:

```
40113d:	mov    -0x4(%rbp),%eax     ; load the local variable into eax
401140:	mov    %eax,%edi           ; move it into the first argument register
401142:	call   0x401106 <func1>    ; call func1 with that argument
```

The variable loaded at `main+33` is whatever was stored at `[rbp-0x4]` two instructions earlier (`movl $0x28e,-0x4(%rbp)`), so `main` is calling `func1(0x28e)`. The return value comes back in `%eax` and is stashed at `[rbp-0x8]`. None of this changes the flag — the constant we want lives inside `func1`.

### Step 4 — Disassemble `func1` to find the constant

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch -ex "disassemble func1" ./debugger_a
```

```
Dump of assembler code for function func1:
   0x0000000000401106 <+0>:	endbr64
   0x000000000040110a <+4>:	push   %rbp
   0x000000000040110b <+5>:	mov    %rsp,%rbp
   0x000000000040110e <+8>:	mov    %edi,-0x4(%rbp)
   0x0000000000401111 <+11>:	mov    -0x4(%rbp),%eax
   0x0000000000401114 <+14>:	imul   $0x3269,%eax,%eax
   0x000000000040111a <+20>:	pop    %rbp
   0x000000000040111b <+21>:	ret
End of assembler dump.
```

There it is, the entire challenge on a single line:

```
401114:	imul   $0x3269,%eax,%eax
```

Reading it in Intel syntax: signed-multiply `eax` by the immediate `0x3269`, store the low 32 bits back into `eax`. The constant is `0x3269`.

If you happen to be reading AT&T syntax instead (GDB's default), the same instruction prints as:

```
401114:	69 c0 69 32 00 00    	imul   %eax,0x3269
```

The `69 c0 ...` is the opcode, and the two operands shown (`%eax, 0x3269`) are *swapped* relative to Intel syntax — AT&T prints `src, dst`, so it means "`eax` = `eax` * `0x3269`". Either way, the constant is `0x3269`.

### Step 5 — Convert the constant from hex to decimal

The flag wants the constant in **decimal base**. The description explicitly tells us how to think about the example: `0x1000` → `4096`. Same idea, different number.

```
0x3269 = 3*16^3 + 2*16^2 + 6*16^1 + 9*16^0
       = 3*4096 + 2*256 + 6*16 + 9
       = 12288 + 512 + 96 + 9
       = 12905
```

I sanity-checked the conversion three different ways: by hand, with Python, and by asking GDB to print the constant as a signed decimal integer directly.

By hand:

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "print(0x3269)"
12905
```

With GDB itself. The `/d` format specifier tells GDB to print the value as a signed decimal integer, regardless of how it was written in the disassembly:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch -ex "print/d 0x3269" ./debugger_a
$1 = 12905
```

All three agree: the constant is **12905**.

### Step 6 — Wrap the constant in the picoCTF flag format

```
┌──(zham㉿kali)-[~/ctf]
└─$ echo "picoCTF{12905}"
picoCTF{12905}
```

Submitted and accepted.

---

## What Happened Internally (Timeline)

Walking through what `main` and `func1` actually do once `func1(0x28e)` is called:

1. `main` reserves 32 bytes of stack space (`sub $0x20, %rsp`) and stores `0x28e` at `[rbp-0x4]` — this is the input value it will pass to `func1`.
2. `main` loads that value into `eax` (`mov -0x4(%rbp), %eax`) and then copies it into `edi` (`mov %eax, %edi`) so it is in the correct argument register for the System V AMD64 calling convention.
3. `main` executes `call 0x401106 <func1>`. The CPU pushes the return address onto the stack and jumps to `func1`.
4. Inside `func1`, the `endbr64` is a no-op for our purposes — it is an indirect-branch landing pad for CET (Control-flow Enforcement Technology). `push %rbp; mov %rsp, %rbp` sets up a fresh stack frame.
5. `func1` stores the incoming `edi` argument at `[rbp-0x4]` (`mov %edi, -0x4(%rbp)`), then loads it back into `eax` (`mov -0x4(%rbp), %eax`). The double trip through memory is just the compiler's standard pattern for using a local variable.
6. `func1` executes the entire challenge on one line: `imul $0x3269, %eax, %eax`. The result lands back in `eax` — for the input `0x28e`, the product is `0x28e * 0x3269 = 0x80c83e` (decimal `8439870`) — but for the flag we only care about the constant itself, which is `0x3269`.
7. `pop %rbp; ret` tears down the stack frame and returns to `main+43`.
8. Back in `main`, the return value in `eax` is stored at `[rbp-0x8]`. `main` then loads `[rbp-0x4]` into `eax` one more time (so `main`'s own return value is the original input, `0x28e`, not the multiplied one), and returns to whoever called it (libc startup, which then calls `exit`).

The flag was sitting in plain sight the whole time as the third operand of that single `imul`.

---

## Alternative Solve Methods

### Alternative 1 — `objdump` instead of `gdb`

GDB is the intended tool, but the binary is not stripped, so plain `objdump` gives you exactly the same information. Useful if you are scripting in a pipeline or if you do not have GDB installed.

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d -M intel ./debugger_a | sed -n '/<func1>:/,/^$/p'
```

```
0000000000401106 <func1>:
  401106:	f3 0f 1e fa          	endbr64
  40110a:	55                   	push   rbp
  40110b:	48 89 e5             	mov    rbp,rsp
  40110e:	89 7d fc             	mov    DWORD PTR [rbp-0x4],edi
  401111:	8b 45 fc             	mov    eax,DWORD PTR [rbp-0x4]
  401114:	69 c0 69 32 00 00    	imul   eax,eax,0x3269
  40111a:	5d                   	pop    rbp
  40111b:	c3                   	ret
```

The constant `0x3269` is visible on the `imul` line. Same answer, no debugger required.

You can also `grep` directly for it:

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d -M intel ./debugger_a | grep imul
  401114:	69 c0 69 32 00 00    	imul   eax,eax,0x3269
```

### Alternative 2 — `r2` (radare2)

If you prefer radare2, the workflow is:

```
┌──(zham㉿kali)-[~/ctf]
└─$ r2 -q -c "aaa; pdf @ sym.func1" ./debugger_a
```

The `pdf` ("print disassembly function") command at the `func1` symbol shows the same `imul eax, eax, 0x3269`.

### Alternative 3 — run it under GDB, break on the multiply, and read the register

If for some reason the constant were not a literal immediate but built up dynamically, you would still want GDB running the program. The workflow is:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb ./debugger_a
(gdb) break *0x401114
(gdb) run
(gdb) info registers eax
```

That is overkill for this challenge (the constant is right there in the disassembly), but it is the right move for the next challenge in the series.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `file` | file 5.39 | Confirmed the binary is an ELF 64-bit executable and that it is not stripped (so symbols like `main` and `func1` are usable). |
| `gdb` | GNU gdb 13.1 | The intended debugger. Used in `-batch` mode with `info functions`, `disassemble main`, `disassemble func1`, and `print/d` to get the constant in decimal. |
| `objdump` | GNU binutils 2.40 | Fallback disassembler. Used `-M intel` so the operands read like English instead of AT&T's `src, dst` order. |
| `python3` | Python 3.11 | Hex-to-decimal sanity check on `0x3269` and on the running product. |
| `chmod` | coreutils 9.1 | Marked the downloaded binary executable so `gdb` could load it. |

---

## Key Takeaways

- **`imul dst, src, imm` is the "multiply by a constant" pattern.** When you see this three-operand form, the third operand is almost always the interesting number — the kind of thing a challenge description will ask you to read off. The two-operand `imul dst, src` form is also common, but there the constant is hidden inside whatever `src` was loaded from earlier.
- **The description gives you the answer format for free.** "If the constant you find is 0x1000, the flag will be picoCTF{4096}" is a worked example. Read it, then apply the same conversion to whatever constant you actually find. The temptation to overthink is the main trap on this challenge.
- **The hint is hinting at a *future* trap, not the current one.** "A function can be referenced by either its name or its starting address in gdb" only matters once symbols get stripped. On this binary we have full symbols, so we use `disassemble func1` directly. Keep the address-based form (`disassemble 0x401106`) in your back pocket for later challenges.
- **`gdb -batch -ex "..."` is your best friend for clean writeup output.** Every command in this writeup was driven non-interactively. The same trick works in CI pipelines and shell scripts.
- **`/d` format specifier converts hex to decimal in GDB.** `print/d 0x3269` returns `12905`. The other useful format specifiers are `/x` (hex), `/u` (unsigned decimal), `/t` (binary), and `/c` (treat as character). `/d` is signed decimal, which is what the flag wants.
- **`objdump -M intel` is the cleanest one-liner for quick reversing.** No debugger session, no breakpoints, no stepping. When you only need to *read* the assembly, `objdump` is faster.
- **Flag wordplay decode.** The flag itself is `picoCTF{12905}` — there is no wordplay to decode here. The "trick" is purely a base conversion: hex `0x3269` reads as "3-2-6-9" in hex shorthand, but in decimal it is `12905`. The description's worked example (`0x1000` → `4096`) is meant to make you realize you have to do that conversion before wrapping the number in `picoCTF{...}`. Once you do, the flag is `picoCTF{12905}`.
