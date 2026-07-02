# Bit-O-Asm-2 — picoCTF Writeup

**Challenge:** Bit-O-Asm-2  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{654874}`  
**Platform:** picoCTF 2022 (picoGym)  
**Writeup by:** zham  

---

## Description

> Can you figure out what is in the `eax` register? Put your answer in the picoCTF flag format: `picoCTF{n}` where n is the contents of the `eax` register in the decimal number base. If the answer was `0x11` your flag would be `picoCTF{17}`.
>
> Download the assembly dump [here].

## Hints

> 1. PTR's or 'pointers', reference a location in memory where values can be stored.

---

## Background Knowledge (Read This First!)

If you solved Bit-O-Asm-1 first, you already know the cast of characters: a few lines of standard prologue/epilogue noise, one or two instructions that touch `eax`, and a hex-to-decimal conversion at the end. Bit-O-Asm-2 keeps that skeleton but introduces the **one** new concept that makes this challenge different from Bit-O-Asm-1: the value does not go straight into `eax`. It detours through memory first. Here is the vocabulary that lets you follow that detour without getting lost.

### Registers vs. Memory

The CPU has two places to keep a value while a function runs:

1. **Registers** — small, named, inside the CPU. `eax`, `rbp`, `rsp`, `edi`, `rsi`, etc. There are only a handful. They are the fastest storage the CPU has.
2. **Memory (RAM)** — vast, but slow relative to registers. The function's "local variables" live here. The CPU accesses memory by **address**.

Registers are like the variables in your pocket. Memory is like a filing cabinet. The CPU can only do arithmetic on values in registers, but it can shuffle values between memory and registers with `mov`.

### What `[rbp-0x4]` Means

In Intel syntax, square brackets mean "the memory at this address." So `[rbp-0x4]` means "the memory location whose address is `rbp` minus 4 bytes."

In a C-like mental model:

```c
int local;                 // stored at some address on the stack
*(int *)(rbp - 0x4) = ...; // write to that local
... = *(int *)(rbp - 0x4); // read from that local
```

`rbp` is the **base pointer** — it anchors the current stack frame, so every local variable can be addressed as `[rbp + something]` or `[rbp - something]`. The exact numeric addresses do not matter for this challenge; what matters is that `[rbp-0x4]` is *one specific slot* in memory.

### `DWORD PTR` and `QWORD PTR`

These tell you how many bytes the instruction reads or writes:

- `DWORD` = "double word" = **4 bytes** (32 bits). Same width as `eax`.
- `QWORD` = "quad word" = **8 bytes** (64 bits). Same width as `rax`.

`mov DWORD PTR [rbp-0x4], 0x9fe1a` writes 4 bytes (the value `0x9fe1a`) into the 4-byte slot at `[rbp-0x4]`. `mov eax, DWORD PTR [rbp-0x4]` reads 4 bytes back out of that same slot and drops them into `eax`.

### The Two-Step Pattern: Store, Then Load

The challenge hint is literally explaining the pattern:

> PTR's or 'pointers', reference a location in memory where values can be stored.

Read that as: "When you see `PTR [...]`, you are looking at a memory location. The value is being **stored there first**, and a later instruction will **read it back** into a register." In this challenge, the store happens at `<+15>` and the load (into `eax`) happens at `<+22>`. Both refer to the same slot, `[rbp-0x4]`, so the value that ends up in `eax` is the value that was written at `<+15>`.

This is the C-equivalent of:

```c
int x;
x = 0x9fe1a;     // store to memory
eax = x;          // load back into a register
return eax;
```

If the two instructions referred to different slots, the puzzle would be different. They do not — they are the same slot.

### Why Bother With the Detour?

If you can `mov eax, 0x9fe1a` directly (which is what Bit-O-Asm-1 does), why does this challenge write to memory first? In real C code, you would do this when:

- The value came from another function call that returned into the stack.
- The compiler decided to keep a frequently-read value in memory for register-allocation reasons.
- The value is the result of arithmetic on several locals, and the compiler is breaking it into smaller steps.

For this challenge, the reason is pedagogical: picoCTF wants you to learn the `[rbp-0x4]` indirection pattern before Bit-O-Asm-3 piles arithmetic on top of it. Treat it as a stepping stone, not a trick.

### Hex to Decimal (Again)

Same conversion as Bit-O-Asm-1, but the literal now has five hex digits instead of two:

```
0x9fe1a = 9 * 16^4 + f * 16^3 + e * 16^2 + 1 * 16 + a
        = 9 * 65536 + 15 * 4096 + 14 * 256 + 1 * 16 + 10
        = 589824 + 61440 + 3584 + 16 + 10
        = 654874
```

Or just let Python do it: `python3 -c "print(0x9fe1a)"` returns `654874`.

---

## Solution — Step by Step

### Step 1 — Get the assembly dump

Same workflow as Bit-O-Asm-1. I downloaded the file the challenge linked to and saved it into the working folder.

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ls
bit-o-asm-1.txt
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ mv ~/Downloads/*.txt ./bit-o-asm-2.txt
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ls
bit-o-asm-1.txt  bit-o-asm-2.txt
```

I keep both files side-by-side because the series is short and I want to refer back to Bit-O-Asm-1 when I compare.

### Step 2 — Read the dump

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ cat bit-o-asm-2.txt
```

```
<+0>:     endbr64
<+4>:     push   rbp
<+5>:     mov    rbp,rsp
<+8>:     mov    DWORD PTR [rbp-0x14],edi
<+11>:    mov    QWORD PTR [rbp-0x20],rsi
<+15>:    mov    DWORD PTR [rbp-0x4],0x9fe1a
<+22>:    mov    eax,DWORD PTR [rbp-0x4]
<+25>:    pop    rbp
<+26>:    ret
```

Eight lines. Two more than Bit-O-Asm-1, but the structure is identical: prologue, two `argc`/`argv` saves, **the work**, epilogue.

### Step 3 — Trace the value of `eax` through the dump

The hint tells us to watch the `PTR [...]` lines. There are two of them in "the work" section, and both reference the same slot, `[rbp-0x4]`:

| Line | What it does | `eax` after |
|---|---|---|
| `<+8>`  | save `argc` (in `edi`) to `[rbp-0x14]` | unchanged |
| `<+11>` | save `argv` (in `rsi`) to `[rbp-0x20]` | unchanged |
| `<+15>` | write literal `0x9fe1a` to `[rbp-0x4]` | unchanged |
| `<+22>` | read 4 bytes from `[rbp-0x4]` into `eax` | `eax = 0x9fe1a` |
| `<+25>` | pop saved `rbp` | `eax = 0x9fe1a` |
| `<+26>` | `ret`, exit with `eax` as the return value | `eax = 0x9fe1a` |

The two `mov` instructions at `<+15>` and `<+22>` form a store-then-load pair around the same memory slot. Reading line `<+22>` as English: "move the 32-bit value at `[rbp-0x4]` into `eax`." Since `[rbp-0x4]` was just set to `0x9fe1a` three instructions earlier and nothing else in the dump writes to it, the load at `<+22>` pulls `0x9fe1a` straight back out.

### Step 4 — Convert `0x9fe1a` to decimal

The flag format wants decimal, so I run the conversion.

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 -c "print(0x9fe1a)"
654874
```

Or, if I want to follow the worked example in the description, by hand:

```
0x9fe1a
= 9 * 16^4 + f * 16^3 + e * 16^2 + 1 * 16 + a
= 9 * 65536 + 15 * 4096 + 14 * 256 + 1 * 16 + 10
= 589824 + 61440 + 3584 + 16 + 10
= 654874
```

Either way: `eax` holds **654874** when `main` returns.

### Step 5 — Wrap in the flag format

The challenge description worked the example `0x11` → `picoCTF{17}`. Same conversion, same format:

```
picoCTF{654874}
```

Submit it on the picoCTF challenge page, accept.

---

## What Happened Internally (Timeline)

Walking through the dump from start to finish, step by step, so the indirection through `[rbp-0x4]` stops looking like magic:

1. The function is entered. `rax` (and therefore `eax`) holds whatever the previous call left in it — some libc startup garbage we do not care about.
2. `endbr64` is the CET landing pad. No effect on registers. Continue.
3. `push rbp` saves the caller's base pointer. `mov rbp, rsp` anchors a fresh stack frame, so the four slots `[rbp-0x14]`, `[rbp-0x20]`, `[rbp-0x4]`, and (later) `[rbp-0xc]` / `[rbp-0x8]` (Bit-O-Asm-3) become valid local-variable addresses. `eax` is untouched.
4. `mov DWORD PTR [rbp-0x14], edi` saves `argc`. `mov QWORD PTR [rbp-0x20], rsi` saves `argv`. Neither writes to `eax`. They are just satisfying the C signature `int main(int argc, char **argv)`.
5. `<+15>: mov DWORD PTR [rbp-0x4], 0x9fe1a` writes the 32-bit literal `0x9fe1a` to the 4-byte slot at `[rbp-0x4]`. This is the **store** half of the pattern the hint was talking about. Memory now holds `0x9fe1a` at that address. `eax` is still untouched.
6. `<+22>: mov eax, DWORD PTR [rbp-0x4]` reads the 4-byte value at `[rbp-0x4]` and puts it in `eax`. This is the **load** half of the pattern. Because of step 5, `[rbp-0x4]` still holds `0x9fe1a`, so after this instruction `eax == 0x9fe1a`. The 32-bit destination also zero-extends into `rax`, so `rax == 0x9fe1a` too.
7. `pop rbp` restores the caller's base pointer. `eax` is still `0x9fe1a`.
8. `ret` pops the saved return address and jumps back into libc startup, which eventually calls `exit` with `eax`'s value (`0x9fe1a` → 654874 in decimal). Program terminates.

`eax` is the answer from step 6 onward. There is no further modification — no arithmetic, no comparison, no jump. The decimal value `654874` is what `main` "returned."

---

## Alternative Solve Methods

### Alternative 1 — use the `hex2dec.py` helper from Bit-O-Asm-1

The little reusable script from the Bit-O-Asm-1 writeup pays off immediately on Bit-O-Asm-2. No need to re-derive the manual hex-to-decimal arithmetic:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ ./hex2dec.py 0x9fe1a
654874
```

Same script, same answer, no new work. If you are about to do Bit-O-Asm-3 and Bit-O-Asm-4 as well, this script is the right tool for the whole series.

### Alternative 2 — `printf` instead of `python3`

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ printf '%d\n' 0x9fe1a
654874
```

`printf` parses the `0x` prefix and prints decimal. Slightly faster than spawning Python when you only need a single conversion.

### Alternative 3 — `bc` (the Unix bench calculator)

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ echo "obase=10; ibase=16; 9FE1A" | bc
654874
```

`bc` does not need the `0x` prefix when you set `ibase=16` — you write the literal in pure hex digits. Same answer.

### Alternative 4 — solve it without running anything at all

This is the "I read it in the dump" path. Because the load at `<+22>` reads directly from the slot the store at `<+15>` just wrote to, and nothing else writes to that slot in between, the value of `eax` at `ret` is *the same hex literal that appears in the dump*. You do not even need to convert it for the *reasoning* — the only reason you convert is to put it into the decimal flag format. So:

```
eax at <+22> == value at [rbp-0x4] == 0x9fe1a (the literal from <+15>)
```

If the challenge accepted hex flags, you would be done at `<+15>`. As it does not, one `python3 -c "print(0x9fe1a)"` later you have the answer.

### Alternative 5 — confirm by simulating the stack frame in Python

If you want to see exactly why the indirection does not change the answer, you can simulate the whole function in Python:

```
┌──(zham㉿kali)-[~/ctf/bit-o-asm]
└─$ python3 - <<'PY'
mem = {}                        # our fake memory
eax = "garbage"                 # whatever was in eax on entry
rbp = 0x7fffffff                # fake stack frame base

mem[rbp - 0x14] = "argc"        # <+8>
mem[rbp - 0x20] = "argv"        # <+11>
mem[rbp - 0x4]  = 0x9fe1a       # <+15>

eax = mem[rbp - 0x4]            # <+22>
print(f"eax = {hex(eax)} = {eax}")
PY
```

```
eax = 0x9fe1a = 654874
```

Same answer, but now you have a written-down proof that the store-then-load pattern is equivalent to a direct `mov eax, 0x9fe1a`. This is the right mental model for Bit-O-Asm-3, where the same trick is used with two slots and an arithmetic operation on top.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `cat` | coreutils 9.1 | Read the assembly dump file directly. It is plain text, no special tooling. |
| `python3` | Python 3.11 | Converted the hex literal `0x9fe1a` to its decimal value `654874`, and optionally simulated the stack frame in the "Alternative Methods" section. |
| `printf` | coreutils 9.1 | Alternative one-liner for the same hex-to-decimal conversion. |
| `bc` | GNU bc 1.07 | Listed as an alternative for users who prefer the classic Unix calculator to Python. |
| `hex2dec.py` | (script from Bit-O-Asm-1) | Reused the helper script from the Bit-O-Asm-1 writeup to convert the literal without re-typing the conversion logic. |

---

## Key Takeaways

- **The whole new concept is "store to memory, then load back into a register."** Bit-O-Asm-1 was a one-step `mov eax, 0x30`. Bit-O-Asm-2 is a two-step `mov [rbp-0x4], 0x9fe1a` followed by `mov eax, [rbp-0x4]`. If you understand that pair, you understand 80% of what this challenge is testing.
- **`PTR [rbp-N]` is the way x86 talks about local variables.** Once you internalize that `[rbp-0x4]` means "the local at offset 4 below the frame pointer," you can read any plain C function's disassembly at a glance. The number after `rbp` is the byte offset of the local; the `DWORD` vs `QWORD` is the size of the local; the instruction tells you whether it is a store or a load.
- **Match the store to its load.** The trick in this challenge is noticing that `<+15>` and `<+22>` both reference the *same* slot, `[rbp-0x4]`. If they had referenced different slots, the value in `eax` would be whatever was last written to `[rbp-0x4]` (or leftover stack garbage). Always trace which slot each `PTR` line is reading or writing before you commit to an answer.
- **The flag format still wants decimal.** Same trap as Bit-O-Asm-1: the worked example `0x11` → `picoCTF{17}` is the format spec. The dump gives you a hex literal, the flag wants the decimal spelling, no `0x` prefix inside the braces.
- **Nothing else touches `eax` between `<+22>` and `ret`.** No arithmetic, no function call, no comparison. The value `0x9fe1a` is what `eax` holds at `ret`. Bit-O-Asm-3 will change that — it will add an `imul` and an `add` between the load and the return. If you ever get a wrong answer on this series, the first thing to check is whether you missed an arithmetic instruction between the load and the `ret`.
- **Reuse your tooling across the series.** The `hex2dec.py` script from Bit-O-Asm-1 works unchanged here. Build helpers as you go and you stop repeating work by challenge 3.
- **Flag wordplay decode.** The flag is `picoCTF{654874}`. There is no wordplay to decode — the entire challenge is the two-step `mov` pattern (store `0x9fe1a` to `[rbp-0x4]`, then load `[rbp-0x4]` into `eax`), the hex-to-decimal conversion (`0x9fe1a = 9 * 65536 + 15 * 4096 + 14 * 256 + 16 + 10 = 654874`), and the picoCTF brace format. The series name "Bit-O-Asm" still means "a bit of assembly" — this time, that bit happens to detour through a stack slot before reaching `eax`. Once you do the conversion, the flag is `picoCTF{654874}`.
