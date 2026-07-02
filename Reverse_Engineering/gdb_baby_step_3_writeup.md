# GDB baby step 3 — picoCTF Writeup

**Challenge:** GDB baby step 3  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{0x6bc96222}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> Now for something a little different. 0x2262c96b is loaded into memory in the main function. Examine the byte-wise memory that the constant is loaded in by using the GDB command x/4xb addr. The flag is the four bytes as they are stored in memory. If you find the bytes 0x11 0x22 0x33 0x44 in the memory location, your flag would be: picoCTF{0x11223344}.

## Hints

> 1. You'll need to breakpoint the instruction after the memory load.
> 2. Use the gdb command x/4xb addr with the memory location as the address addr to examine. GDB manual page.
> 3. Any registers in addr should be prepended with a $ like $rbp.
> 4. Don't use square brackets for addr
> 5. What is endianness?

---

## Background Knowledge (Read This First!)

The whole challenge is one sentence from the hint list: "What is endianness?" If you already know what endianness is, you can skip straight to the solution. If not, here is the whole story this challenge depends on.

### Memory is Just a Long Line of Bytes

To a C program, memory is one giant line of bytes. Each byte has its own address, and adjacent addresses are adjacent bytes. When the program stores a multi-byte value like an `int` (4 bytes) or a `long long` (8 bytes), the bytes have to go *somewhere* — and the question of "where exactly" is what endianness answers.

### Little-Endian: the Low Byte Goes First

x86, x86-64, and most ARM operating modes (Android, iOS, Linux on Raspberry Pi, etc.) store multi-byte values in **little-endian** order. The rule is dead simple:

> **The least-significant byte is stored at the lowest address. The most-significant byte is stored at the highest address.**

For the value `0x2262c96b` (a 32-bit integer), the bytes from most-significant to least-significant are:

```
most-significant                                 least-significant
       0x22        0x62        0xc9        0x6b
       (byte 3)    (byte 2)    (byte 1)    (byte 0)
```

Little-endian reverses that when storing into memory:

```
lowest address                                       highest address
       0x6b        0xc9        0x62        0x22
       byte 0      byte 1      byte 2      byte 3
```

That is the layout we have to read off with GDB.

### Big-Endian: the High Byte Goes First (for contrast)

The opposite convention is **big-endian**, used by old Motorola 68k CPUs, some old PowerPC systems, and most network protocols. Big-endian stores the most-significant byte at the lowest address. So big-endian `0x2262c96b` would be laid out as `0x22 0x62 0xc9 0x6b` — the same way humans write the number.

This challenge is testing whether you understand that x86 is little-endian. If you naively read `0x2262c96b` left-to-right and submitted `picoCTF{0x2262c96b}`, you would fail. The bytes in memory are the *little-endian* layout: `0x6b 0xc9 0x62 0x22`.

### The `mov` Encoding Itself Spills the Beans

Here is the giveaway baked right into the disassembly:

```
401115: c7 45 fc 6b c9 62 22    mov    DWORD PTR [rbp-0x4], 0x2262c96b
```

Read the raw bytes after the opcode (`c7 45 fc`): they are `6b c9 62 22`. That is the immediate value as it is encoded in the instruction stream — and the instruction stream is a sequence of bytes in memory, written low-to-high. So even without *running* the program, you can see the answer sitting right there. The GDB run is there to confirm it, not to reveal it.

### What `x/4xb` Means

GDB's `x` ("examine memory") command has the form:

```
x/<count><format><unit-size> <address>
```

For `x/4xb $rbp-4`:

- `4` — print 4 units.
- `x` — print each unit as unsigned hex.
- `b` — unit size is one byte.
- `$rbp-4` — the address to read from. `$rbp` is the current value of the base pointer register; subtracting 4 lands us at `[rbp-0x4]`, which is exactly where `main` just stored the constant.

Each unit is one byte, displayed as `0xNN`, separated by tabs. The output is laid out left-to-right as **lowest-address-first** — which is the same left-to-right order as the bytes in the disassembly above, which is the same as little-endian. So the leftmost byte you see in the `x` output is the low byte of the value.

### Why a Breakpoint and Not Just Disassembly

The disassembly shows you the *encoded* instruction bytes — those bytes are part of the instruction stream, not part of the runtime value. At runtime, the CPU has *executed* the `mov` and the immediate bytes are now sitting in the stack frame at `[rbp-0x4]`. To see them there, the program has to actually run to that point. A breakpoint is the cleanest way to pause it at exactly that instant, before `main` returns and the stack frame goes away.

---

## Solution — Step by Step

### Step 1 — Make the binary executable and identify it

I copied the challenge file into a working directory, made it executable, and confirmed what kind of binary it is.

```
┌──(zham㉿kali)-[~/ctf]
└─$ cp ~/Downloads/debugger_b .
┌──(zham㉿kali)-[~/ctf]
└─$ chmod +x debugger_b
┌──(zham㉿kali)-[~/ctf]
└─$ file debugger_b
debugger_b: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=a10a8fa896351748020d158a4e18bb4be15cd3aa, for GNU/Linux 3.2.0, not stripped
```

Two things to note:

- **ELF 64-bit LSB** — "LSB" stands for "Least Significant Byte first," i.e. little-endian. The file format itself is little-endian on disk, and the CPU will be little-endian at runtime. Same answer both ways.
- **not stripped** — symbols like `main` are intact.

### Step 2 — Find the load instruction and the breakpoint target

The challenge says "you'll need to breakpoint the instruction after the memory load." So I need two addresses:

1. The instruction that *loads* the constant — so I know where the value is going.
2. The instruction that *uses* the constant after the load — that's the breakpoint target.

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d -M intel ./debugger_b | sed -n '/<main>:/,/^$/p'
```

```
0000000000401106 <main>:
  401106:	f3 0f 1e fa          	endbr64
  40110a:	55                   	push   rbp
  40110b:	48 89 e5             	mov    rbp,rsp
  40110e:	89 7d ec             	mov    DWORD PTR [rbp-0x14],edi
  401111:	48 89 75 e0          	mov    QWORD PTR [rbp-0x20],rsi
  401115:	c7 45 fc 6b c9 62 22 	mov    DWORD PTR [rbp-0x4],0x2262c96b
  40111c:	8b 45 fc             	mov    eax,DWORD PTR [rbp-0x4]
  40111f:	5d                   	pop    rbp
  401120:	c3                   	ret
```

Two relevant lines:

- **Load instruction at `0x401115`** — `mov DWORD PTR [rbp-0x4], 0x2262c96b`. This writes the constant `0x2262c96b` into the four bytes at addresses `[rbp-4]`, `[rbp-3]`, `[rbp-2]`, `[rbp-1]`.
- **Breakpoint target at `0x40111c`** — `mov eax, DWORD PTR [rbp-0x4]`. This is the very next instruction; it reads the constant back into `eax`. Setting a breakpoint here guarantees the load has already happened but the value is still on the stack.

If you look closely at the raw bytes of the load instruction itself — `c7 45 fc 6b c9 62 22` — you can already see the answer `6b c9 62 22` embedded in the instruction stream. The little-endian layout is the same on disk and at runtime.

### Step 3 — Set the breakpoint and run

I drove GDB non-interactively. The pattern is the same as baby step 4: open the binary, set a breakpoint, run, examine, done.

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "break *0x40111c" \
        -ex "run" \
        -ex "x/4xb \$rbp-4" \
        ./debugger_b
```

```
Breakpoint 1 at 0x40111c
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x00000000004011c in main ()
0x7fffffffebbc:	0x6b	0xc9	0x62	0x22
```

Breaking this output down:

- `Breakpoint 1 at 0x40111c` — GDB confirmed the breakpoint was set at the right address.
- `Breakpoint 1, 0x000000000040111c in main ()` — the program ran, hit the breakpoint, and is now paused at the start of `main+22` (the `mov eax, ...` instruction).
- `0x7fffffffebbc:` — the address being examined. This is `$rbp - 4`, the exact slot `main` just wrote to.
- `0x6b   0xc9   0x62   0x22` — the four bytes stored there, in memory order (lowest address first).

Reading the bytes left-to-right gives us the flag value: `6b c9 62 22`.

### Step 4 — Sanity check the answer with the disassembly bytes

The immediate bytes in the `mov` instruction itself are the same four bytes, just in a different location:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch -ex "x/4xb 0x401117" ./debugger_b
0x401117:	0x6b	0xc9	0x62	0x22
```

`0x401117` is one byte past the opcode prefix `c7 45 fc` of the load instruction, which is exactly where the 32-bit immediate starts in the encoded instruction. Same bytes, same order — little-endian, all the way down.

### Step 5 — Sanity check the answer with `print/x`

GDB can also read the same location as a 32-bit integer and format it as hex. The value should round-trip back to `0x2262c96b`:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "break *0x40111c" \
        -ex "run" \
        -ex "x/1xw \$rbp-4" \
        -ex "print/x *(unsigned int*)(\$rbp-4)" \
        ./debugger_b
```

```
0x7fffffffebbc:	0x2262c96b
$1 = 0x2262c96b
```

Three different views, same answer:

| View | Command | Result |
|---|---|---|
| Byte-wise (little-endian) | `x/4xb $rbp-4` | `0x6b 0xc9 0x62 0x22` |
| 32-bit word (GDB re-groups) | `x/1xw $rbp-4` | `0x2262c96b` |
| Cast back to `unsigned int` | `print/x *(unsigned int*)($rbp-4)` | `0x2262c96b` |

The 4-byte view *is* `0x2262c96b`. The bytes of that view, in memory order, are `0x6b 0xc9 0x62 0x22`. Both representations are correct; the flag just wants the byte layout.

### Step 6 — Wrap the bytes in the picoCTF flag format

The flag template from the description is `picoCTF{0x<bytes in memory order>}`:

```
picoCTF{0x6bc96222}
```

Submitted and accepted.

---

## What Happened Internally (Timeline)

Walking through what the CPU actually did from the moment `main` was called to the moment GDB paused it:

1. `_start` (in libc) sets up the process and calls `main(argc=1, argv=...)`. `argc` (`1`) goes into `edi`, `argv` (a pointer) goes into `rsi`. The standard prologue is skipped here for brevity.
2. `main` saves `rbp` and anchors a new stack frame: `push %rbp; mov %rsp, %rbp`.
3. `main` saves its two incoming arguments: `mov %edi, -0x14(%rbp)` and `mov %rsi, -0x20(%rbp)`. Neither one matters for the flag.
4. The CPU executes the load instruction: `mov DWORD PTR [rbp-0x4], 0x2262c96b`. The instruction encoding `c7 45 fc 6b c9 62 22` is decoded by the CPU:
   - `c7 /0` is the opcode for "move imm32 to r/m32."
   - `45 fc` is the ModRM + SIB byte that specifies `[rbp + disp8 = -4]` as the destination.
   - `6b c9 62 22` is the 32-bit immediate `0x2262c96b`, read low-byte-first from the instruction stream.
5. The CPU writes the immediate to `[rbp-0x4]` in **little-endian** order:
   - `[rbp-4] = 0x6b`
   - `[rbp-3] = 0xc9`
   - `[rbp-2] = 0x62`
   - `[rbp-1] = 0x22`
6. The CPU advances the instruction pointer to `0x40111c` — `mov eax, DWORD PTR [rbp-0x4]`. At this exact moment, GDB pauses the program because of our breakpoint.
7. `x/4xb $rbp-4` reads the four bytes from `[rbp-4]` upward. GDB displays them left-to-right as `0x6b 0xc9 0x62 0x22`, which is the answer the flag wants.

If we let the program continue, `main` would copy the bytes into `eax`, restore `rbp`, and return. By that point the stack frame is gone and the bytes are no longer sitting in the place we read them from — which is why the breakpoint matters.

---

## Alternative Solve Methods

### Alternative 1 — read the answer straight out of the disassembly

Because the instruction encoding already contains the immediate in little-endian byte order, you can answer this challenge with nothing but `objdump`. No debugger, no breakpoints, no execution. The relevant line:

```
401115:	c7 45 fc 6b c9 62 22 	mov    DWORD PTR [rbp-0x4], 0x2262c96b
```

Skipping the three opcode/ModRM bytes (`c7 45 fc`), the next four bytes — `6b c9 62 22` — are the immediate as it lives in the instruction stream. Same bytes, same order, same flag.

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d -M intel ./debugger_b | grep -A1 movl | head -5
  401115:	c7 45 fc 6b c9 62 22 	mov    DWORD PTR [rbp-0x4],0x2262c96b
```

Read the four bytes after the `c7 45 fc` prefix: `6b c9 62 22`.

### Alternative 2 — use a Python one-liner without ever running the binary

The little-endian conversion is just `int.to_bytes` in reverse:

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "print('0x' + (0x2262c96b).to_bytes(4, 'little').hex())"
0x6bc96222
```

That is the entire flag string, with no debugger required. (This is only useful as a way to *check* the manual answer — the challenge wants you to use GDB, and the GDB round-trip is the part that teaches the concept.)

### Alternative 3 — breakpoint by symbol offset, not raw address

If you would rather set the breakpoint symbolically, you can use GDB's "line +N" syntax. `main+22` is the address of the `mov eax, ...` instruction:

```
┌──(zham㉿kali)-[~/ctf]
└─$ gdb -batch \
        -ex "break *(main+22)" \
        -ex "run" \
        -ex "x/4xb \$rbp-4" \
        ./debugger_b
```

Same result, more readable. The hint about "by name or by starting address" from baby step 4 applies here too: `break *0x40111c` and `break *(main+22)` are equivalent.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `file` | file 5.39 | Confirmed the binary is an ELF 64-bit LSB executable. "LSB" here tells us the binary itself is little-endian on disk, matching the runtime endianness. |
| `objdump` | GNU binutils 2.40 | Pre-flight disassembly to identify the load instruction at `0x401115` and the breakpoint target at `0x40111c`. Also gives us the answer bytes for free. |
| `gdb` | GNU gdb 13.1 | The intended debugger. Used in `-batch` mode with `break *0x40111c`, `run`, and `x/4xb $rbp-4` to inspect the four bytes at runtime. |
| `python3` | Python 3.11 | Sanity-checked the little-endian conversion with `int.to_bytes(4, 'little')`. |

---

## Key Takeaways

- **Little-endian is the entire point of this challenge.** x86-64 stores multi-byte values with the least-significant byte at the lowest address. The bytes `0x6b 0xc9 0x62 0x22` in memory are the same number as `0x2262c96b` — they are just laid out in the order the CPU actually stores them. If the binary were big-endian (Motorola 68k, some old PowerPC configs, network byte order), the answer would be `picoCTF{0x2262c96b}` instead. The hint "What is endianness?" is the whole challenge in three words.
- **The instruction encoding already contains the answer.** The load instruction `c7 45 fc 6b c9 62 22` has the four answer bytes baked into the opcode stream after the `c7 45 fc` prefix. You could solve this with nothing but `objdump` if you read raw bytes carefully — the GDB round-trip is there to teach the concept of *runtime* memory layout.
- **Breakpoint *after* the load, not on it.** Setting the breakpoint on `0x401115` itself is wrong: GDB pauses *before* the load executes, so `[rbp-0x4]` still holds whatever was there from a previous call (typically garbage from a prior stack frame). The breakpoint at `0x40111c` is the first instruction where the value is guaranteed to be in place. Hint 1 was steering you here.
- **`x/<count><format><unit> <addr>` is the universal GDB memory recipe.** The four flags worth remembering:
  - Count: how many units to print.
  - Format: `x` hex, `d` signed decimal, `u` unsigned decimal, `t` binary, `c` char, `i` instruction, `s` string.
  - Unit: `b` byte, `h` half-word (2 bytes), `w` word (4 bytes), `g` giant word (8 bytes).
  - Address: register expressions need `$` (`$rbp`), not `[rbp]`. No square brackets.
- **The `*` in `break *0x40111c` means "interpret this as a raw address, not a symbol."** `break 0x40111c` would also work here (GDB guesses), but `break *ADDR` is the explicit form and is safer for CTFs where you might be breakpointing inside library code with the same numeric address.
- **Flag wordplay decode.** The flag is `picoCTF{0x6bc96222}`. The "decoding" is the little-endian reversal of `0x2262c96b`: take the constant, split it into bytes from high to low (`22 62 c9 6b`), then read the bytes back from low to high (`6b c9 62 22`). That byte order is the answer. The flag template just wraps it in `picoCTF{0x<bytes>}`. Once you do the reversal, the flag is `picoCTF{0x6bc96222}`.
