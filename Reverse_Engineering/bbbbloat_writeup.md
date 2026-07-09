# Bbbbloat — picoCTF Writeup

**Challenge:** Bbbbloat  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{cu7_7h3_bl047_44f74a60}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Can you get the flag?
>
> Reverse engineer this `binary`.

## Hints

(Hint panel did not reveal any explicit hint for this instance — the tags **binary** + **obfuscation** are the hint, and so is the name *Bbbbloat*: the program is full of deliberately useless code that makes it look bigger and harder than it is.)

---

## Background Knowledge

`Bbbbloat` is the stripped-compiled-binary version of `bloat.py`. Same problem shape (encoded flag bytes, a gate, a decode routine), but every visible string in the C source has been replaced by either a `movabs` literal or — and this is the new twist — a long chain of useless arithmetic that the compiler kept around. A few concepts make sense of the mess.

**1. `stripped` vs `not stripped`.**
`file` reported this binary as `stripped`. That means the symbol table was removed after linking: there are no function names in the binary. `main` is at offset `0x1307` in the binary's virtual address space, but you will never see the word `main` in `objdump` or `nm`. To navigate the file you have to find `main` by its position in `_start`'s `lea rdi, [rip+0x17f]` (which loads `0x1307`) and follow the code from there. Everything else — `printf`, `scanf`, `fputs`, `free`, `strdup`, `strlen`, `putchar` — is visible only as a PLT entry, never as a named symbol.

**2. What `Bbbbloat`'s "bloat" actually is.**
If you read `main` top-to-bottom, between the `printf("What's my favorite number? ")` and the `scanf("%d", &input)`, you find *six* identical arithmetic blocks. Each block is roughly:

```asm
mov   DWORD [rbp-0x3c], 0x3078
add   DWORD [rbp-0x3c], 0x13c29e
sub   DWORD [rbp-0x3c], 0x30a8
shl   DWORD [rbp-0x3c], 1
mov   eax, DWORD [rbp-0x3c]
movsxd rdx, eax
imul  rdx, rdx, 0x55555556
shr   rdx, 0x20
sar   eax, 0x1f
mov   ecx, edx
sub   ecx, eax
mov   eax, ecx
mov   DWORD [rbp-0x3c], eax
```

The constant `0x55555556` is the compiler's magic-number division by 3 (`2^32 / 3 ≈ 0x55555556`). The full sequence is the assembly form of `x = ((0x3078 + 0x13c29e - 0x30a8) << 1) / 3`, which always evaluates to the same number. The result is then stored back into `[rbp-0x3c]` … and *immediately overwritten by the next block*. **None of these six blocks is read by the program.** They are pure dead code. The optimizer left them behind — probably because the original source had a few `volatile` writes or assignments that prevented the compiler from deleting them. That is the "bloat": six copies of a useless computation, totalling about 90 instructions, all of which produce a number that is then thrown away.

**3. The real shape of `main`.**
Strip out the six dead blocks and the program is tiny:

```
print "What's my favorite number? "
read input into [rbp-0x40] with scanf("%d")
compare [rbp-0x40] to 0x86187
  if equal → decode the flag with rotate_encrypt(flag_bytes), fputs, putchar('\n'), free
  if not   → puts("Sorry, that's not it!")
return 0
```

That is the entire program. The obfuscation never adds behavior — only visual noise. The `rotate_encrypt` helper at `0x1249` is the same ROT-47 from `gdbme` and `bloat.py`.

**4. Why the program "hangs" when you run it without input.**
`scanf("%d", &input)` reads one integer from stdin. If stdin is a TTY with nothing typed, `scanf` blocks waiting for a newline. That is what makes `./bbbbloat` look frozen — it is not in a busy loop, it is politely waiting for a number. Pipe a number in (`echo 549255 | ./bbbbloat`) and it completes in microseconds. This is a useful general lesson: when a `scanf`-using program "hangs," it is almost always blocked on I/O, not stuck in an infinite loop.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The downloaded challenge file is `bbbbloat`.

### 1. Move the file into a working directory and confirm what it is

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ mkdir -p ~/bbbbloat && cd ~/bbbbloat

┌──(zham㉿kali)-[~/bbbbloat]
└─$ cp ~/Downloads/bbbbloat .

┌──(zham㉿kali)-[~/bbbbloat]
└─$ chmod +x bbbbloat

┌──(zham㉿kali)-[~/bbbbloat]
└─$ file bbbbloat
bbbbloat: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=1989eb2c7cb4aad23df277d8aed3c97482740d7a, for GNU/Linux 3.2.0, stripped
```

`stripped` is the new wrinkle compared to `gdbme`. No symbols — we navigate by addresses.

### 2. Try running it (and watch it hang)

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ ./bbbbloat
^C
```

No output, then a Ctrl-C. As predicted in Background #4, the program is blocked on `scanf` waiting for an integer. We need to *type* one in — that is the gate.

### 3. Find `main` and the magic number

`_start` loads `0x1307` into `rdi` and calls `__libc_start_main`. That is `main`. Let me read just that function:

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ objdump -d -M intel bbbbloat | sed -n '/1307:/,/15a9:/p'
```

The interesting bit — *after* stripping out the six dead arithmetic blocks — is:

```
    13cb: lea    rdi,[rip+0xc32]              ; "What's my favorite number? "
    13d7: mov    eax,0x0
    13dc: call   printf@plt
    ...
    1446: lea    rax,[rbp-0x40]               ; address of input buffer
    144d: lea    rdi,[rip+0xbcc]              ; "%d"
    1459: call   __isoc99_scanf@plt
    ...
    14c8: mov    eax,DWORD PTR [rbp-0x40]     ; eax = our input
    14cb: cmp    eax,0x86187                   ; compare to 0x86187
    14d0: jne    1583                          ; if not equal → "Sorry, that's not it!"
    ...
    1540: lea    rax,[rbp-0x30]               ; the encoded flag bytes
    154c: call   1249                         ; rotate_encrypt (the ROT-47 helper)
    1555: ... fputs, putchar('\n'), free
    1583: lea    rdi,[rip+0xa99]              ; "Sorry, that's not it!"
    158a: call   puts@plt
    158f: mov    eax,0x0
    15a8: leave
    15a9: ret
```

So the magic number is `0x86187`. In decimal:

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ python3 -c "print(0x86187)"
549255
```

### 4. Run it with the magic number on stdin

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ echo "549255" | ./bbbbloat
What's my favorite number? picoCTF{cu7_7h3_bl047_44f74a60}
```

The flag is `picoCTF{cu7_7h3_bl047_44f74a60}`. The program prints the prompt, reads our `549255`, the `cmp eax, 0x86187` is true, and `fputs` writes the decoded flag.

### 5. Sanity-check the wrong number

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ echo "1" | ./bbbbloat
What's my favorite number? Sorry, that's not it!
```

Confirms the gate is just a `cmp eax, 0x86187` and the only "useful" branches are the success path and the rejection path.

---

## Alternative Solves

**A. Skip the program entirely — decode the embedded flag with ROT-47.**
The four `movabs` immediates at the start of `main` are the encoded flag bytes. `rotate_encrypt` is the same ROT-47 used in `gdbme` and `bloat.py` (add `0x2F` = 47 to each printable byte, wrap by `0x5E` = 94). Apply it once and you have the flag:

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ python3 -c "
import struct
vals = [0x4c75257240343a41, 0x3062396630664634,
        0x63633066635f3d33, 0x4e5f6532636637]
s = b''.join(struct.pack('<Q', v) for v in vals).decode()
def r47(c):
    o = ord(c)
    return chr(0x21 + ((o - 0x21 + 47) % 94)) if 0x21 <= o <= 0x7e else c
print(''.join(r47(c) for c in s))
"
picoCTF{cu7_7h3_bl047_44f74a60}
```

Useful when you cannot run the binary (e.g. wrong architecture) or simply do not want to type `549255` correctly.

**B. Skip the dead-code blocks with `gdb` breakpoints.**
If you want to use gdb the way the previous challenges taught, set a breakpoint right before `scanf` (so you can paste in the magic number interactively), then *another* breakpoint right before the `cmp`:

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ gdb -batch \
    -ex "break *0x1459" \
    -ex "run" \
    -ex "print \$eax = 0x86187" \
    -ex "continue" \
    ./bbbbloat
```

Or simply: set `eax` to `0x86187` right before the `cmp` and let the program run.

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ gdb -batch \
    -ex "break *0x14c8" \
    -ex "run" \
    -ex "set \$eax = 0x86187" \
    -ex "continue" \
    ./bbbbloat
What's my favorite number? picoCTF{cu7_7h3_bl047_44f74a60}
```

The first form overrides our `eax` *before* `scanf` even runs; the second form lets `scanf` take whatever we typed and then overwrites the result just before the comparison. Either works.

**C. Patch the binary to skip the comparison.**
The `cmp eax, 0x86187; jne 1583` is two instructions (6 bytes) at `0x14cb`–`0x14d0`. Replace them with `xor eax, eax; nop; nop` (effectively forcing `eax == 0x86187` is harder, but forcing `ZF=1` by replacing the `jne` with two NOPs and skipping the branch is enough):

```
┌──(zham㉿kali)-[~/bbbbloat]
└─$ python3 -c "
import shutil
shutil.copy('bbbbloat','bbbbloat.unlocked')
data = bytearray(open('bbbbloat.unlocked','rb').read())
# Replace the 6 bytes at file offset 0x14cb with NOPs (0x90)
data[0x14cb:0x14cb+6] = b'\x90'*6
open('bbbbloat.unlocked','wb').write(bytes(data))
"

┌──(zham㉿kali)-[~/bbbbloat]
└─$ chmod +x bbbbloat.unlocked

┌──(zham㉿kali)-[~/bbbbloat]
└─$ echo "0" | ./bbbbloat.unlocked
What's my favorite number? picoCTF{cu7_7h3_bl047_44f74a60}
```

Same flag — but now any input passes.

---

## What Happened Internally

1. `_start` called `__libc_start_main`, which initialised libc, set up `argc`/`argv`/`envp`, and jumped to `main` at `0x1307`.
2. `main` set up the stack frame and stored four `movabs` literals into `[rbp-0x30]` — the encoded flag bytes `A:4@r%uL` + `4Ff0f9b0` + `3=_cf0cc` + `7fc2e_N`.
3. The first of six dead-code blocks executed. Each block set `[rbp-0x3c] = 0x3078`, added `0x13c29e`, subtracted `0x30a8`, shifted left by 1, then used the compiler's multiply-by-magic-and-shift trick to divide by 3. The result was stored back into `[rbp-0x3c]` — and immediately overwritten by the next block's `mov [rbp-0x3c], 0x3078`. None of these results were ever read.
4. The actual useful code ran: `printf("What's my favorite number? ")` printed the prompt. `scanf("%d", &[rbp-0x40])` read our integer `549255` from stdin into `[rbp-0x40]`. More dead-code blocks ran (same useless pattern). Then `mov eax, [rbp-0x40]; cmp eax, 0x86187; jne 1583` made the comparison.
5. `0x86187 == 549255`, so the `jne` did not branch. The success path ran: `lea rax, [rbp-0x30]` loaded the address of the encoded flag bytes, `call 0x1249` invoked `rotate_encrypt`, which `strdup`'d the input and ROT-47'd each printable character. The decoded `char*` was passed to `fputs(stdout, ...)`, followed by `putchar('\n')` and `free(decoded)`.
6. The control-flow merged at `0x158f`, set `eax = 0`, ran the stack canary check, and `main` returned 0.
7. `__libc_start_main` called `exit(0)`.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `file`           | Confirmed the artifact is a 64-bit ELF, PIE, **stripped**.     |
| `chmod +x`       | Made the downloaded binary executable.                         |
| `./bbbbloat` (no input) | Demonstrated the `scanf` block for the writeup.        |
| `objdump -d -M intel` | Located `main` at `0x1307` (via `_start`'s `lea rdi`) and read the `cmp eax, 0x86187` instruction. |
| `python3 -c "print(0x86187)"` | Converted the hex magic number to decimal (`549255`). |
| `echo 549255 \| ./bbbbloat` | The actual solve — pipe the right number in.         |
| `echo 1 \| ./bbbbloat`     | Captured the rejection path for the writeup.          |
| `gdb -batch`     | Used in Alternative Solve B to set `$eax = 0x86187` and continue. |
| `python3` (alt)  | ROT-47 decode of the embedded `movabs` bytes for Alternative Solve A. |

---

## Key Takeaways

- "Stripped" is the only real new difficulty compared to `gdbme`. The challenge is the same shape; the only added work is finding `main` and reading its instructions without symbol names. `objdump -d` plus the `lea rdi, [rip+…]` in `_start` is enough — `gdb` will also accept the address.
- "Bloat" in this binary is *dead code*: six identical arithmetic blocks that the compiler kept but the program never reads. Always check whether the value being computed is ever consumed — if not, it is decoration.
- The pattern `mov a, K; add a, K2; sub a, K3; shl a, 1; imul rdx, rdx, 0x55555556; shr rdx, 32; sar eax, 31; sub ecx, eax` is a compiler-generated `x = ((K + K2 - K3) << 1) / 3`. Recognising it lets you predict the result without running the program.
- A binary that "hangs" with no output is *probably* blocked on `scanf`/`fgets`/`read`. Type a value in (or pipe one in with `echo`) and try again before assuming the program is in an infinite loop.
- The flag's leet: `cu7_7h3_bl047` reads as **cut_the_bloat** (`7→t`, `7→t`, `3→e`, `0→o`, `4→a`). The challenge is called *Bbbbloat* and the flag's wordplay is *cut the bloat* — exactly the lesson: ignore the dead-code blocks, focus on the real `cmp`/`jne`/`call` chain, and the program collapses to a five-line puzzle.
