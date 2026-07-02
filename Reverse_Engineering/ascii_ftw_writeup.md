# ASCII FTW — picoCTF Writeup

**Challenge:** ASCII FTW  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{ASCII_IS_EASY_8960F0AF}`  
**Platform:** picoCTF (picoGym)  
**Writeup by:** zham  

---

## Description

> This program has constructed the flag using hex ascii values. Identify the flag text by disassembling the program.
>
> You can download the file from [here].

## Hints

> 1. The combined range of hex-ascii for English alphabets and numerical digits is from 30 to 7A.
> 2. Online hex-ascii converters can be helpful.

---

## Background Knowledge (Read This First!)

If you have read the Bit-O-Asm-1 through Bit-O-Asm-4 writeups, you already know most of the cast: a few lines of standard prologue, some local-variable stores to memory, a return at the end. ASCII FTW keeps the same shape but is the first challenge in this repo where you are given a **real ELF binary** instead of a static assembly dump. That means there is one extra step at the start (disassemble the binary yourself) and one extra step at the end (run the program and notice that it does not actually print the whole flag). The new concept that ties it all together is **hex-to-ASCII conversion** — every byte the program stores on the stack is a printable ASCII character code, and once you decode them in order, the flag falls out. Here is the vocabulary that makes that decode a one-liner instead of a guessing game.

### What is ASCII?

ASCII is a 7-bit encoding that maps small integers to characters. For the purposes of this challenge, you only need a handful of ranges:

| Hex range | Decimal range | What it covers |
|---|---|---|
| `0x30`–`0x39` | `48`–`57`   | digits `0`–`9` |
| `0x41`–`0x5A` | `65`–`90`   | uppercase `A`–`Z` |
| `0x61`–`0x7A` | `97`–`122`  | lowercase `a`–`z` |
| `0x7B`        | `123`       | `{` |
| `0x7D`        | `125`       | `}` |
| `0x5F`        | `95`        | `_` |

The challenge hint is literally telling you this: "the combined range of hex-ascii for English alphabets and numerical digits is from `30` to `7A`." That is the printable range that covers digits, uppercase, lowercase, and (by extension) the braces and underscore the picoCTF flag format uses. If a byte in the dump is between `0x30` and `0x7A`, you can be confident it is a printable ASCII character and the conversion is just `chr(byte)`.

### `BYTE PTR` vs. `DWORD PTR`

The Bit-O-Asm series used `mov DWORD PTR [rbp-0x4], 0x9fe1a` — a 4-byte write. ASCII FTW uses `mov BYTE PTR [rbp-0x30], 0x70` — a **1-byte write**. Same idea, smaller size. The general syntax is the same:

```
mov    <size> PTR [<address-expression>], <value>
```

Where `<size>` is one of:

| Mnemonic | Bytes | Bits | C-equivalent |
|---|---|---|---|
| `BYTE`  | 1 | 8  | `char`        |
| `WORD`  | 2 | 16 | `short`       |
| `DWORD` | 4 | 32 | `int`         |
| `QWORD` | 8 | 64 | `long` / `void *` |

For this challenge, every flag byte is written as `BYTE PTR`. The compiler chose `BYTE` because the flag is a string (a sequence of `char` values in C), and writing one byte at a time is the natural way to build a string in assembly.

### `mov BYTE PTR [rbp-0x30], 0x70` in C

That single instruction is the C equivalent of:

```c
char buf[32];                // somewhere on the stack, anchored at rbp-0x30
buf[0] = 'p';                // 0x70 is the ASCII code for 'p'
```

If you read 31 consecutive `mov BYTE PTR` instructions all targeting `rbp-0x30` through `rbp-0x12`, you are reading 31 `char` writes to a 31-byte buffer. That is a 31-character string. Converting each byte to its ASCII character gives you the flag.

### The Trick: the Program Only Prints the First Character

If you run the binary naively, you see something like:

```
$ ./asciiftw
The flag starts with 70
```

That is **not** the flag. It is a hint, and a misleading one if you do not read the rest of the disassembly. The program constructs the *entire* flag on the stack (31 bytes, from `rbp-0x30` to `rbp-0x12`), but the call to `printf` only reads the **first** byte and prints it as hex:

```
movzx  eax, BYTE PTR [rbp-0x30]   ; load first byte into eax
movsx  eax, al                     ; sign-extend al into eax
mov    esi, eax                    ; pass it as the second printf arg
lea    rdi, [rip+0xdf4]            ; "The flag starts with %x\n"
mov    eax, 0x0
call   printf@plt
```

It is the C equivalent of `printf("The flag starts with %x\n", buf[0]);` — print the first byte of the flag in hex. The `%x` format specifier prints an integer in hexadecimal, so the program literally tells you the first character is `0x70` and stops. You have to read the rest of the flag from the disassembly yourself.

This is the entire reason the challenge is interesting. The program gives you one byte as a freebie, then dares you to find the other 30 bytes by reading the assembly. The other 30 bytes are right there in the disassembly; you just have to know that `mov BYTE PTR [rbp-0x2f], 0x69` is the second character (`'i'`, ASCII `0x69`) and so on.

### A Quick ASCII Hex-to-Character Cheat Sheet

For the bytes in this challenge, you only need:

| Hex | Char | Hex | Char | Hex | Char |
|---|---|---|---|---|---|
| `0x30` | `0` | `0x41` | `A` | `0x61` | `a` |
| `0x31` | `1` | `0x42` | `B` | `0x62` | `b` |
| `0x32` | `2` | `0x43` | `C` | `0x63` | `c` |
| `0x33` | `3` | `0x44` | `D` | `0x64` | `d` |
| `0x34` | `4` | `0x45` | `E` | `0x65` | `e` |
| `0x35` | `5` | `0x46` | `F` | `0x66` | `f` |
| `0x36` | `6` | `0x47` | `G` | `0x67` | `g` |
| `0x37` | `7` | `0x48` | `H` | `0x68` | `h` |
| `0x38` | `8` | `0x49` | `I` | `0x69` | `i` |
| `0x39` | `9` | ... | ... | `0x6A` | `j` |
| ... | ... | `0x54` | `T` | `0x6F` | `o` |
| `0x5F` | `_` | `0x59` | `Y` | `0x7B` | `{` |
| | | | | `0x7D` | `}` |

If a byte is in the printable range, you can just look it up. For everything else, `python3 -c "print(chr(0x70))"` returns `'p'` and you move on.

---

## Solution — Step by Step

### Step 1 — Get the binary and identify it

I downloaded the file the challenge linked to and saved it into a working folder. Let me call it `asciiftw`.

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ mkdir -p ~/ctf/ascii-ftw && cd ~/ctf/ascii-ftw
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ mv ~/Downloads/asciiftw .
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ file asciiftw
asciiftw: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=e1c32dace8ac1516160b771e493f5ebffcac9855, for GNU/Linux 3.2.0, not stripped
```

Two important things from the `file` output:

- **ELF 64-bit LSB pie executable** — the same x86-64 PIE binary format as the GDB baby steps. Position-independent, so the runtime addresses will be shifted by a random base, but `objdump` shows the on-disk offsets and that is all we need for static reading.
- **not stripped** — symbols are intact, so `main` is a real symbol we can disassemble by name.

### Step 2 — Run the binary to see what it does

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ chmod +x asciiftw
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ ./asciiftw
The flag starts with 70
```

So the program tells us the first byte of the flag is `0x70`. That is `'p'` in ASCII — which is the right first character for a `picoCTF{...}` flag. Good sanity check. Now we need the other 30 bytes.

### Step 3 — Disassemble `main` with `objdump`

The challenge says "identify the flag text by disassembling the program," so we use `objdump` to dump the assembly of `main` and read the bytes off it.

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ objdump -d -M intel ./asciiftw | sed -n '/<main>:/,/^$/p'
```

```
0000000000001169 <main>:
    1169:	f3 0f 1e fa          	endbr64
    116d:	55                   	push   rbp
    116e:	48 89 e5             	mov    rbp,rsp
    1171:	48 83 ec 30          	sub    rsp,0x30
    1175:	64 48 8b 04 25 28 00 	mov    rax,QWORD PTR fs:0x28
    117c:	00 00 
    117e:	48 89 45 f8          	mov    QWORD PTR [rbp-0x8],rax
    1182:	31 c0                	xor    eax,eax
    1184:	c6 45 d0 70          	mov    BYTE PTR [rbp-0x30],0x70
    1188:	c6 45 d1 69          	mov    BYTE PTR [rbp-0x2f],0x69
    118c:	c6 45 d2 63          	mov    BYTE PTR [rbp-0x2e],0x63
    1190:	c6 45 d3 6f          	mov    BYTE PTR [rbp-0x2d],0x6f
    1194:	c6 45 d4 43          	mov    BYTE PTR [rbp-0x2c],0x43
    1198:	c6 45 d5 54          	mov    BYTE PTR [rbp-0x2b],0x54
    119c:	c6 45 d6 46          	mov    BYTE PTR [rbp-0x2a],0x46
    11a0:	c6 45 d7 7b          	mov    BYTE PTR [rbp-0x29],0x7b
    11a4:	c6 45 d8 41          	mov    BYTE PTR [rbp-0x28],0x41
    11a8:	c6 45 d9 53          	mov    BYTE PTR [rbp-0x27],0x53
    11ac:	c6 45 da 43          	mov    BYTE PTR [rbp-0x26],0x43
    11b0:	c6 45 db 49          	mov    BYTE PTR [rbp-0x25],0x49
    11b4:	c6 45 dc 49          	mov    BYTE PTR [rbp-0x24],0x49
    11b8:	c6 45 dd 5f          	mov    BYTE PTR [rbp-0x23],0x5f
    11bc:	c6 45 de 49          	mov    BYTE PTR [rbp-0x22],0x49
    11c0:	c6 45 df 53          	mov    BYTE PTR [rbp-0x21],0x53
    11c4:	c6 45 e0 5f          	mov    BYTE PTR [rbp-0x20],0x5f
    11c8:	c6 45 e1 45          	mov    BYTE PTR [rbp-0x1f],0x45
    11cc:	c6 45 e2 41          	mov    BYTE PTR [rbp-0x1e],0x41
    11d0:	c6 45 e3 53          	mov    BYTE PTR [rbp-0x1d],0x53
    11d4:	c6 45 e4 59          	mov    BYTE PTR [rbp-0x1c],0x59
    11d8:	c6 45 e5 5f          	mov    BYTE PTR [rbp-0x1b],0x5f
    11dc:	c6 45 e6 38          	mov    BYTE PTR [rbp-0x1a],0x38
    11e0:	c6 45 e7 39          	mov    BYTE PTR [rbp-0x19],0x39
    11e4:	c6 45 e8 36          	mov    BYTE PTR [rbp-0x18],0x36
    11e8:	c6 45 e9 30          	mov    BYTE PTR [rbp-0x17],0x30
    11ec:	c6 45 ea 46          	mov    BYTE PTR [rbp-0x16],0x46
    11f0:	c6 45 eb 30          	mov    BYTE PTR [rbp-0x15],0x30
    11f4:	c6 45 ec 41          	mov    BYTE PTR [rbp-0x14],0x41
    11f8:	c6 45 ed 46          	mov    BYTE PTR [rbp-0x13],0x46
    11fc:	c6 45 ee 7d          	mov    BYTE PTR [rbp-0x12],0x7d
    1200:	0f b6 45 d0          	movzx  eax,BYTE PTR [rbp-0x30]
    1204:	0f be c0             	movsx  eax,al
    1207:	89 c6                	mov    esi,eax
    1209:	48 8d 3d f4 0d 00 00 	lea    rdi,[rip+0xdf4]        # 2004 <_IO_stdin_used+0x4>
    1210:	b8 00 00 00 00       	mov    eax,0x0
    1215:	e8 56 fe ff ff       	call   1070 <printf@plt>
    121a:	90                   	nop
    121b:	48 8b 45 f8          	mov    rax,QWORD PTR [rbp-0x8]
    121f:	64 48 33 04 25 28 00 	xor    rax,QWORD PTR fs:0x28
    1226:	00 00 
    1228:	74 05                	je     122f <main+0xc6>
    122a:	e8 31 fe ff ff       	call   1060 <__stack_chk_fail@plt>
    122f:	c9                   	leave
    1230:	c3                   	ret
```

The work section is the 31 `mov BYTE PTR` instructions at `<+27>` through `<+153>` (the lines starting at `0x1184` and ending at `0x11fc`). Each one writes a single byte to a consecutive address on the stack, starting at `[rbp-0x30]` and ending at `[rbp-0x12]`. The whole 31-byte block at `[rbp-0x30..rbp-0x12]` is the flag.

### Step 4 — Extract the bytes in order

I read the destination address and the literal from each line, in source order. The destination addresses go from `rbp-0x30` up to `rbp-0x12`, so the source order matches the natural string order:

| Line | Address | Byte | Char |
|---|---|---|---|
| `<+27>`  | `[rbp-0x30]` | `0x70` | `p` |
| `<+31>`  | `[rbp-0x2f]` | `0x69` | `i` |
| `<+35>`  | `[rbp-0x2e]` | `0x63` | `c` |
| `<+39>`  | `[rbp-0x2d]` | `0x6f` | `o` |
| `<+43>`  | `[rbp-0x2c]` | `0x43` | `C` |
| `<+47>`  | `[rbp-0x2b]` | `0x54` | `T` |
| `<+51>`  | `[rbp-0x2a]` | `0x46` | `F` |
| `<+55>`  | `[rbp-0x29]` | `0x7b` | `{` |
| `<+59>`  | `[rbp-0x28]` | `0x41` | `A` |
| `<+63>`  | `[    -0x27]` | `0x53` | `S` |
| `<+67>`  | `[rbp-0x26]` | `0x43` | `C` |
| `<+71>`  | `[rbp-0x25]` | `0x49` | `I` |
| `<+75>`  | `[rbp-0x24]` | `0x49` | `I` |
| `<+79>`  | `[rbp-0x23]` | `0x5f` | `_` |
| `<+83>`  | `[rbp-0x22]` | `0x49` | `I` |
| `<+87>`  | `[rbp-0x21]` | `0x53` | `S` |
| `<+91>`  | `[rbp-0x20]` | `0x5f` | `_` |
| `<+95>`  | `[rbp-0x1f]` | `0x45` | `E` |
| `<+99>`  | `[rbp-0x1e]` | `0x41` | `A` |
| `<+103>` | `[rbp-0x1d]` | `0x53` | `S` |
| `<+107>` | `[rbp-0x1c]` | `0x59` | `Y` |
| `<+111>` | `[rbp-0x1b]` | `0x5f` | `_` |
| `<+115>` | `[rbp-0x1a]` | `0x38` | `8` |
| `<+119>` | `[rbp-0x19]` | `0x39` | `9` |
| `<+123>` | `[rbp-0x18]` | `0x36` | `6` |
| `<+127>` | `[rbp-0x17]` | `0x30` | `0` |
| `<+131>` | `[rbp-0x16]` | `0x46` | `F` |
| `<+135>` | `[rbp-0x15]` | `0x30` | `0` |
| `<+139>` | `[rbp-0x14]` | `0x41` | `A` |
| `<+143>` | `[rbp-0x13]` | `0x46` | `F` |
| `<+147>` | `[rbp-0x12]` | `0x7d` | `}` |

Concatenating the right column: `picoCTF{ASCII_IS_EASY_8960F0AF}`.

I confirmed each byte is in the printable range `0x30`–`0x7A` (with the single exception of the closing brace `0x7d`, which is just past `0x7a` — still well within the printable ASCII range). That matches the hint.

### Step 5 — Convert with Python (sanity check)

I do not trust my table-reading in step 4 for a flag submission, so I do the same conversion in Python:

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ python3 -c "
bytes_hex = [0x70,0x69,0x63,0x6f,0x43,0x54,0x46,0x7b,0x41,0x53,0x43,0x49,0x49,0x5f,0x49,0x53,0x5f,0x45,0x41,0x53,0x59,0x5f,0x38,0x39,0x36,0x30,0x46,0x30,0x41,0x46,0x7d]
print(''.join(chr(b) for b in bytes_hex))
"
picoCTF{ASCII_IS_EASY_8960F0AF}
```

Predicted answer: **`picoCTF{ASCII_IS_EASY_8960F0AF}`**.

### Step 6 — Submit the flag

```
picoCTF{ASCII_IS_EASY_8960F0AF}
```

Submit it on the picoCTF challenge page, accept.

---

## What Happened Internally (Timeline)

Walking through `main` from start to finish, step by step, so the byte-by-byte construction and the partial printout both stop looking like magic:

1. `_start` (in libc) calls `main(argc=1, argv=...)`. The runtime address of `main` is something like `0x555555555169` (PIE base + offset `0x1169`).
2. `endbr64` is the CET landing pad. No effect on registers. Continue.
3. `push rbp` saves the caller's base pointer. `mov rbp, rsp` anchors a fresh stack frame. `sub rsp, 0x30` reserves 48 bytes of stack space for locals and alignment.
4. `mov rax, QWORD PTR fs:0x28` is the stack canary. The prologue reads the canary from the thread-local storage and saves it at `[rbp-0x8]`. The epilogue checks it and calls `__stack_chk_fail` if it changed. This is the "stack protector" feature of modern GCC; you can ignore it for the challenge.
5. `xor eax, eax` zeroes `eax` to mark the variadic-argument register as empty for the upcoming `printf` call. Standard ABI hygiene.
6. The 31 `mov BYTE PTR [...]` instructions at `0x1184`–`0x11fc` execute in source order, each writing one byte to the next slot in the stack buffer. After this sequence, the 31-byte block at `[rbp-0x30..rbp-0x12]` holds the flag string. Nothing else writes to those addresses, so the buffer is now stable.
7. `movzx eax, BYTE PTR [rbp-0x30]` reads the **first** byte of the buffer (`0x70`) into `eax`, zero-extending to 32 bits. `movsx eax, al` sign-extends `al` into `eax` (a no-op here because `0x70` is positive). `mov esi, eax` puts the byte in `esi`, which is the second argument register for `printf`.
8. `lea rdi, [rip+0xdf4]` loads the address of the format string `"The flag starts with %x\n"`. `mov eax, 0x0` zeros the variadic-argument count. `call printf@plt` calls `printf` with the format string in `rdi` and the first flag byte in `esi`.
9. `printf` formats the byte as hex and prints it. The user sees `The flag starts with 70` on stdout.
10. The epilogue checks the stack canary, restores `rbp`, and returns. The 30 remaining flag bytes are still on the stack but are not printed.

The whole challenge is steps 6 and 9. Step 6 builds the flag. Step 9 prints one byte of it. The other 30 bytes are sitting in the disassembly, and step 4 of the Solution section is what reads them.

---

## Alternative Solve Methods

### Alternative 1 — use GDB to dump the buffer as ASCII at runtime

If you would rather see the flag appear in your terminal than read it off the disassembly, GDB can do it. The buffer is at `[rbp-0x30]` once `main` has executed the 31 stores but before the `printf`. The cleanest GDB invocation:

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ gdb -batch \
        -ex "break *main+153" \
        -ex "run" \
        -ex "x/31bx \$rbp-0x30" \
        -ex "x/1s \$rbp-0x30" \
        ./asciiftw
```

```
Breakpoint 1, 0x000055555555511b in main ()
0x7fffffffdcb0:	0x70	0x69	0x63	0x6f	0x43	0x54	0x46	0x7b
0x7fffffffdcb8:	0x41	0x53	0x43	0x49	0x49	0x5f	0x49	0x53
0x7fffffffdcc0:	0x5f	0x45	0x41	0x53	0x59	0x5f	0x38	0x39
0x7fffffffdcc8:	0x36	0x30	0x46	0x30	0x41	0x46	0x7d	0x00
0x7fffffffdcb0:	"picoCTF{ASCII_IS_EASY_8960F0AF}"
```

Two GDB commands do the work here:

- `x/31bx $rbp-0x30` — examine 31 bytes in hex (`x` = examine, `/31` = 31 units, `b` = bytes, `x` = hex). GDB prints them in a 16-byte-wide table, exactly like `hexdump -C`.
- `x/1s $rbp-0x30` — examine 1 *string* (`s`) at the same address. GDB dereferences the bytes as a C string and prints the result. This is the "ASCII view" the challenge hint is asking about.

The `x/1s` form is the canonical "show me this memory as ASCII" command in GDB. If you ever do a memory dump challenge in the future, this is the right tool. Note that the 32nd byte (`0x00`) is a null terminator — the C string ends at the closing brace, but the C compiler leaves a `\0` byte just past it. That is why the buffer is 32 bytes long but the flag is 31 characters.

### Alternative 2 — use GDB to confirm the disassembly-derived answer

The same GDB session also doubles as a sanity check for the static read. If the bytes GDB prints match the bytes in the disassembly table, the disassembly is correct. If they do not match, the binary has been tampered with or you are reading the wrong function.

### Alternative 3 — `printf '%c'` to convert individual bytes

If you want a shell-only path (no Python), `printf '%c'` can print a single character from its ASCII code:

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ printf '%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c%c\n' \
        0x70 0x69 0x63 0x6f 0x43 0x54 0x46 0x7b 0x41 0x53 0x43 0x49 0x49 0x5f 0x49 0x53 0x5f 0x45 0x41 0x53 0x59 0x5f 0x38 0x39 0x36 0x30 0x46 0x30 0x41 0x46 0x7d
picoCTF{ASCII_IS_EASY_8960F0AF}
```

`printf '%c'` interprets the argument as a number and prints the corresponding character. This is the shell-only analog of `python3 -c "print(chr(0x70))"`. It is faster than Python for a one-shot decode, but harder to script because you have to type the format specifier once for each byte.

### Alternative 4 — extract the bytes with `awk` / `sed`

If you do not want to read the table by eye, you can pull the literal bytes straight out of the `objdump` output with a small shell pipeline:

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ objdump -d -M intel ./asciiftw \
    | sed -n '/<main>:/,/ret/p' \
    | grep -oE 'BYTE PTR \[rbp-0x[0-9a-f]+\],0x[0-9a-f]+' \
    | grep -oE '0x[0-9a-f]+$' \
    | tail -31 \
    | tr -d '\n' \
    | xxd -r -p
picoCTF{ASCII_IS_EASY_8960F0AF}
```

What is going on here:

- `objdump -d -M intel ./asciiftw | sed -n '/<main>:/,/ret/p'` — keep only the lines from `<main>:` to the first `ret`.
- `grep -oE 'BYTE PTR \[rbp-0x[0-9a-f]+\],0x[0-9a-f]+'` — keep only the lines that look like `mov BYTE PTR [rbp-0xNN],0xXX`. The `-oE` prints only the matching part of each line.
- `grep -oE '0x[0-9a-f]+$'` — keep only the literal byte (the `0xXX` at the end of the line).
- `tail -31` — keep the last 31 matches. The 31 `mov BYTE PTR` instructions are the last 31 lines of the work section, and the other instructions in `main` (the prologue, the `printf` call, the epilogue) do not match the `BYTE PTR [rbp-0xNN]` pattern, so this filter is a clean way to isolate them.
- `tr -d '\n'` — concatenate the lines into one big hex string.
- `xxd -r -p` — interpret the hex string as raw bytes and print it. `-r` is "reverse" (hex to binary), `-p` is "plain hex" (no offsets, no spaces).

This is the "no Python, no GDB" path. It only needs `objdump`, `sed`, `grep`, `tail`, `tr`, and `xxd`, all of which are coreutils / binutils staples that are installed on any Linux box.

### Alternative 5 — `grep` for the flag prefix in the binary directly

A sneakier path: grep the raw binary for the ASCII bytes of the flag prefix. The flag starts with `picoCTF{`, so the bytes `70 69 63 6f 43 54 46 7b` should appear in the binary as a sequence (in the instruction stream or in the embedded data). You can search for them with `xxd`:

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ xxd ./asciiftw | grep -i "7069 636f 4354 467b"
```

This may or may not match depending on how the compiler laid out the constants, but it is a fun trick for "the flag is in the binary" challenges in general. For this specific challenge, the flag is built byte-by-byte at runtime, so a literal `picoCTF{` substring does not appear in the binary — the bytes are spread across 31 separate `mov` instructions.

### Alternative 6 — use `ghidra` or `radare2` for a higher-level view

If you have Ghidra installed, you can open the binary, let it decompile `main`, and the decompiler will show you the flag as a C string literal. Same answer, less assembly reading. `radare2` (`r2 -A ./asciiftw; pdf @main`) does the same thing in a more command-line-friendly package.

For the actual flag-submission part, this is overkill — `objdump` is enough. But for a real reverse engineering workflow, Ghidra and `radare2` are the right tools when the disassembly gets longer than a screen.

### Alternative 7 — write a tiny `objdump | decode` script

If you are about to do more "build a string on the stack" challenges, the pipeline in Alternative 4 is worth packaging as a one-liner script. Save it as `~/bin/decode_bytes`:

```bash
#!/usr/bin/env bash
# Usage: decode_bytes <objdump-line-source>
objdump -d -M intel "$1" \
    | sed -n '/<main>:/,/ret/p' \
    | grep -oE 'BYTE PTR \[rbp-0x[0-9a-f]+\],0x[0-9a-f]+' \
    | grep -oE '0x[0-9a-f]+$' \
    | tr -d '\n' \
    | xxd -r -p
```

Make it executable:

```
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ chmod +x ~/bin/decode_bytes
┌──(zham㉿kali)-[~/ctf/ascii-ftw]
└─$ ~/bin/decode_bytes ./asciiftw
picoCTF{ASCII_IS_EASY_8960F0AF}
```

One command, no copy-paste of the byte table. Reusable for any future challenge of this shape.

---

## Tools Used

| Tool | Version | Why we used it |
|---|---|---|
| `file` | file 5.39 | Confirmed the binary is an ELF 64-bit PIE executable (so we know to use `objdump` and to expect the standard prologue/epilogue). |
| `chmod` | coreutils 9.1 | Made the binary executable before running it. |
| `./asciiftw` | (the binary) | Ran the program to see the misleading single-byte hint. The output `The flag starts with 70` confirmed the first flag character and that the program is meant to be disassembled rather than run blind. |
| `objdump` | GNU binutils 2.40 | The main tool. Dumped the assembly of `main` and showed the 31 `mov BYTE PTR` instructions that build the flag. |
| `python3` | Python 3.11 | Sanity-checked the byte table from the disassembly and confirmed the decoded flag. |
| `gdb` | GNU gdb 13.1 | (Optional) Showed the runtime memory of the flag buffer as both hex (`x/31bx`) and ASCII (`x/1s`). The `x/1s` form is the "show me this memory as ASCII" command the challenge hint is asking about. |
| `sed` / `grep` / `tail` / `tr` | GNU textutils | Used in the shell-only Alternative 4 to extract the 31 hex bytes from the `objdump` output and concatenate them into a flag. |
| `xxd` | xxd 1.10 | Reversed the hex string back into raw bytes in the shell-only Alternative 4. |
| `printf '%c'` | coreutils 9.1 | Converted individual ASCII codes to characters in the shell-only Alternative 3. |

---

## Key Takeaways

- **This is the first challenge in the series that gives you a real binary instead of a static dump.** Bit-O-Asm-1 through -4 handed you a text file with the assembly already extracted. ASCII FTW hands you an ELF, so the first move is to run `objdump` (or Ghidra, or `radare2`, or `gdb`) to get the disassembly. The actual reverse engineering is the same — read the `main` function, find the work section, interpret the instructions — but the entry point is one tool deeper.
- **`mov BYTE PTR [...]` is the C equivalent of `buf[i] = 'c'`.** The Bit-O-Asm series used `DWORD` (4 bytes, like an `int`). ASCII FTW uses `BYTE` (1 byte, like a `char`). The size suffix is the only difference. Once you have seen both forms, you can read any combination of `BYTE` / `WORD` / `DWORD` / `QWORD` writes in any future disassembly.
- **The program prints one byte, you read the rest.** The challenge is a small puzzle: the program's `printf` only reads the first flag byte, and the disassembly holds the other 30. If you trust the program's output and stop there, you submit nothing. If you read past the `printf` call back to the work section, you have the full flag. The lesson: when the runtime output looks suspiciously short, the disassembly is where the real answer lives.
- **Hex-to-ASCII is a one-line mental conversion if you memorize the printable range.** `0x30`–`0x39` is `0`–`9`, `0x41`–`0x5A` is `A`–`Z`, `0x61`–`0x7A` is `a`–`z`, plus a few special characters (`_` is `0x5f`, `{` is `0x7b`, `}` is `0x7d`). The challenge hint explicitly tells you the printable range is `0x30`–`0x7A`. If a byte in the dump is in that range, you can decode it on sight.
- **GDB's `x/1s <addr>` is the canonical "show me this memory as ASCII" command.** This is the "GDB" half of the "is there a way to examine memory in GDB that presents data as ASCII" question the challenge title is referencing. You will use this command for every memory-dump challenge from here on. Pair it with `x/Nbx` to see the same memory in hex when you want to confirm the bytes.
- **The picoCTF flag format `picoCTF{...}` is itself a sanity check.** The first 8 bytes of the flag in this challenge decode to `picoCTF{` — which is the exact picoCTF flag prefix. If your decoded string does not start with `picoCTF{`, you have either read the bytes in the wrong order or skipped one. Always cross-check the prefix before submitting.
- **The `0x00` byte at the end is the C string terminator.** The 32nd byte GDB shows in `x/31bx` is `0x00`. That is the null terminator the C compiler inserts after the 31-character flag string so functions like `printf("%s", buf)` know where to stop. The flag is 31 characters; the buffer is 32 bytes. Do not submit the `0x00`.
- **Flag wordplay decode.** The flag is `picoCTF{ASCII_IS_EASY_8960F0AF}`. The wordplay is the entire challenge name: "ASCII FTW" = "ASCII For The Win." The program is *literally* the ASCII table made concrete — every flag byte is a printable ASCII character, and once you decode them in order, the flag spells itself out. The flag body (`ASCII_IS_EASY_8960F0AF`) is the challenge author's wink: "ASCII is easy, you just have to look up the bytes." The `8960F0AF` suffix is the typical picoCTF random hex tail. Once you decode the 31 bytes from the disassembly, the flag is `picoCTF{ASCII_IS_EASY_8960F0AF}`.
