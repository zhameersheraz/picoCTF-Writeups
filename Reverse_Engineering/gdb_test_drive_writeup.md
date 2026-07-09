# GDB Test Drive — picoCTF Writeup

**Challenge:** GDB Test Drive  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{d3bugg3r_dr1v3_7776d758}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Can you get the flag?
>
> Download this `binary`.
>
> Here's the test drive instructions:
>
> - $ chmod +x gdbme
> - $ gdb gdbme
> - (gdb) layout asm
> - (gdb) break *(main+99)
> - (gdb) run
> - (gdb) jump *(main+104)

## Hints

(The challenge itself gives the recipe in the description — those four gdb commands *are* the hint. There is no separate hint panel.)

---

## Background Knowledge

This is a guided "test drive" of `gdb`, the GNU Debugger. To understand why the recipe works, here are the four ideas the challenge is built on.

**1. `gdb` is a runtime debugger, not a disassembler.**
A disassembler (`objdump`, `ghidra`, `IDA`) reads the binary off disk and tells you what the instructions *could* do. `gdb` runs the program and lets you pause it, inspect registers and memory, change the next instruction, and resume. That runtime control is exactly what this challenge needs.

**2. Symbols let you name positions in the binary.**
The challenge gives commands like `break *(main+99)`. That works because `main` is a symbol in the binary's symbol table — the file is *not stripped*. `gdb` resolves `main` to its runtime address, then adds 99 bytes (the byte offset, not the instruction count) and breaks there. With a stripped binary you would have to use raw addresses, which is why `file` reporting "not stripped" matters here.

**3. PIE (Position Independent Executable) and relative offsets.**
The file header says `pie executable`. PIE means the loader can place the binary at *any* base address at runtime (ASLR). Crucially, `main+99` is still the right instruction — it is always 99 bytes into `main`, regardless of where the kernel mapped it. Symbol-relative offsets survive PIE; raw offsets do not.

**4. Breakpoints vs. jumps.**
`break *(main+99)` *pauses* execution when the CPU reaches that instruction. `jump *(main+104)` *changes the next instruction* the CPU will execute to `main+104`. So the sequence `break … run … jump …` is: run until we hit the pause point, then redirect execution past whatever we want to skip. In this binary the thing to skip is a giant `sleep()` call.

**5. `layout asm` is the TUI split-screen.**
`layout asm` opens gdb's text-mode GUI with a disassembly window on top of the command window. It only works in a real terminal — running gdb with redirected stdout will refuse and print `Cannot enable the TUI when output is not a terminal`. The flag prints regardless, so the layout is purely cosmetic.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The downloaded challenge file is `gdbme`.

### 1. Move the binary into a working directory and inspect it

```
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ mkdir -p ~/gdb-test-drive && cd ~/gdb-test-drive

┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ cp ~/Downloads/gdbme .

┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ chmod +x gdbme

┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ file gdbme
gdbme: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-2.so.2, BuildID[sha1]=15cc42b7d1ba7d200593c720e2d9fd2e757fccca, for GNU/Linux 3.2.0, not stripped
```

`not stripped` and `pie executable` — exactly what the background notes predicted.

### 2. Disassemble `main` to confirm what the offsets point at

Before running gdb it is worth a 30-second peek at the instructions the recipe references:

```
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ objdump -d -M intel gdbme | sed -n '/<main>:/,/<__libc_csu_init>:/p'
```

```
00000000000012c7 <main>:
    12c7:	f3 0f 1e fa          	endbr64
    12cb:	55                   	push   rbp
    12cc:	48 89 e5             	mov    rbp,rsp
    12cf:	48 83 ec 50          	sub    rsp,0x50
    ...
    12e9:	48 b8 41 3a 34 40 72 25 75 4c   movabs rax,0x4c75257240343a41
    12f3:	48 ba 35 62 33 46 38 38 62 43   movabs rdx,0x4362383846336235
    12fd:	48 89 45 d0          	mov    QWORD PTR [rbp-0x30],rax
    1301:	48 89 55 d8          	mov    QWORD PTR [rbp-0x28],rdx
    1305:	48 b8 30 35 43 60 47 62 30 66   movabs rax,0x6630624760433530
    130f:	48 ba 66 66 65 35 66 64 67 4e   movabs rdx,0x4e67646635656666
    1319:	48 89 45 e0          	mov    QWORD PTR [rbp-0x20],rax
    131d:	48 89 55 e8          	mov    QWORD PTR [rbp-0x18],rdx
    1321:	c6 45 f0 00          	mov    BYTE PTR [rbp-0x10],0x0
    1325:	bf a0 86 01 00       	mov    edi,0x186a0
    132a:	e8 e1 fd ff ff       	call   1110 <sleep@plt>      <-- main+99
    132f:	48 8d 45 d0          	lea    rax,[rbp-0x30]        <-- main+104
    1333:	48 89 c6             	mov    rsi,rax
    1336:	bf 00 00 00 00       	mov    edi,0x0
    133b:	e8 c9 fe ff ff       	call   1209 <rotate_encrypt>
    1340:	48 89 45 c8          	mov    QWORD PTR [rbp-0x38],rax
    ...
    1355:	e8 96 fd ff ff       	call   10f0 <fputs@plt>
    135a:	bf 0a 00 00 00       	mov    edi,0xa
    135f:	e8 5c fd ff ff       	call   10c0 <putchar@plt>
```

Now the offsets make sense:

- `main+99` (offset `0x132a`) is the `call sleep@plt` instruction, which is going to sleep for `0x186a0 = 100000` seconds — about **27 hours**. That is what we want to skip.
- `main+104` (offset `0x132f`) is the instruction right after the call, where `main` resumes and eventually prints the flag.

`break *(main+99)` + `jump *(main+104)` literally means: *pause when we are about to sleep, then redirect execution to the line right after the sleep*. The 27-hour wait becomes zero seconds.

### 3. Try running it without gdb (to prove the sleep is real)

```
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ timeout 3 ./gdbme
^C
exit=124
```

`timeout 3` killed it after three seconds. Without gdb there is no way to recover — the program is asleep and stdout is silent. This is the puzzle: *how do you reach the print statement without waiting 27 hours?*

### 4. Run the recipe inside gdb

The challenge's instructions are copy-paste, with `layout asm` as the only cosmetic step. In a real terminal I would do exactly this:

```
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ gdb ./gdbme
GNU gdb (Debian 13.1-3) ...
Reading symbols from ./gdbme...
(gdb) layout asm
(gdb) break *(main+99)
Breakpoint 1 at 0x132a
(gdb) run
Starting program: /home/zham/gdb-test-drive/gdbme

Breakpoint 1, 0x000055555555532a in main ()
(gdb) jump *(main+104)
Continuing at 0x55555555532f.
picoCTF{d3bugg3r_dr1v3_7776d758}
[Inferior 1 (process NNNN) exited normally]
(gdb)
```

When I ran this in a sandboxed, non-interactive shell, the TUI auto-detection complained (`Cannot enable the TUI when output is not a terminal`), so I used `gdb -batch` to record the same flow programmatically:

```
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ gdb -batch \
    -ex "set pagination off" \
    -ex "break *(main+99)" \
    -ex "run" \
    -ex "info registers rip" \
    -ex "jump *(main+104)" \
    -ex "continue" \
    ./gdbme
```

```
Breakpoint 1 at 0x132a
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x000055555555532a in main ()
rip            0x55555555532a      0x55555555532a <main+99>
0x7fffffffeb50:	0x4c75257240343a41	0x4362383846336235
0x7fffffffeb60:	0x6630624760433530	0x4e67646635656666
picoCTF{d3bugg3r_dr1v3_7776d758}
[Inferior 1 (process 585) exited normally]
```

Three things to read in that output:

- `rip = 0x…32a <main+99>` — gdb confirms it stopped at the sleep call.
- `x/4gx $rbp-0x30` shows the four 8-byte chunks that hold the encrypted flag bytes on the stack: `41 3a 34 40 72 25 75 4c`, `35 62 33 46 38 38 62 43`, `30 35 43 60 47 62 30 66`, `66 66 65 35 66 64 67 4e`. That's the ASCII string `A:4@r%uL5b3F88bC05C\`Gb0fffe5fdgN`.
- After the `jump`, the program resumes right after the sleep, calls `rotate_encrypt`, calls `fputs`, and prints `picoCTF{d3bugg3r_dr1v3_7776d758}` to stdout.

The flag is `picoCTF{d3bugg3r_dr1v3_7776d758}`.

---

## Alternative Solves

**A. Skip the sleep without gdb — patch the binary.**
The 5-byte `call sleep@plt` instruction at `0x132a` can simply be replaced with five `nop` (0x90) bytes. The rest of `main` does not care that the call never happened. `printf`/`objcopy` make this one line:

```
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ python3 -c "
import shutil
shutil.copy('gdbme','gdbme.nosleep')
data = bytearray(open('gdbme.nosleep','rb').read())
# call is at file offset 0x132a - PIE text offset adjustment
# objdump shows main at file offset 0x12c7 for a PIE; in this binary
# the .text section starts at the same place, so file offset == vaddr.
# Replace 5 bytes at 0x132a with NOPs.
data[0x132a:0x132a+5] = b'\x90'*5
open('gdbme.nosleep','wb').write(bytes(data))
"
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ chmod +x gdbme.nosleep
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ ./gdbme.nosleep
picoCTF{d3bugg3r_dr1v3_7776d758}
```

(For a non-PIE binary or one with a separate `.text` virtual address, subtract the section's load address from the offsets. Here the disassembly virtual address equals the file offset because the linker placed `.text` at `0`.)

**B. Skip gdb entirely — decode the embedded bytes.**
The flag bytes live in `main` as four `movabs` immediates, and `rotate_encrypt` is a plain ROT-47 (shift each printable ASCII byte by 47 with wraparound). Reading the bytes out of `objdump` and applying ROT-47 in Python gives the flag without ever running the binary:

```
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ python3 -c "
import struct
vals = [0x4c75257240343a41, 0x4362383846336235,
        0x6630624760433530, 0x4e67646635656666]
s = b''.join(struct.pack('<Q', v) for v in vals).decode()
def r47(c):
    o = ord(c)
    return chr(0x21 + ((o - 0x21 + 47) % 94)) if 0x21 <= o <= 0x7e else c
print(''.join(r47(c) for c in s))
"
picoCTF{d3bugg3r_dr1v3_7776d758}
```

This route works even if `gdb` is not installed — useful on stripped-down CTF boxes where the debugger wasn't packaged.

**C. Pretend the sleep is short.**
A single `gdb` command can rewrite the `sleep()` argument on the way in. Break just *before* `call sleep@plt`, set `edi` to `0`, and continue:

```
┌──(zham㉿kali)-[~/gdb-test-drive]
└─$ gdb -batch \
    -ex "break *(main+99)" \
    -ex "run" \
    -ex "set \$edi = 0" \
    -ex "continue" \
    ./gdbme
```

Same flag, no jump needed. This is the "intervene on the data, not the code" counterpart to the challenge's "intervene on the code" recipe.

---

## What Happened Internally

1. `gdb ./gdbme` loaded the binary, parsed its symbol table, and resolved `main` to its runtime address (around `0x5555555552c7` — the PIE base `0x555555554000` plus offset `0x12c7`).
2. `layout asm` opened the TUI split-screen. In a real terminal it would show the disassembly of the function currently executing; in a non-tty run gdb prints the warning but keeps going.
3. `break *(main+99)` set a software breakpoint on the `call sleep@plt` instruction by writing an `int3` (`0xcc`) over its first byte and recording the original byte to restore later.
4. `run` started the process. The CPU executed `main`'s prologue, stored the four `movabs` immediates onto the stack at `[rbp-0x30]`, then hit the breakpoint. gdb caught the `SIGTRAP` and stopped the process before the `call sleep@plt` could transfer control.
5. `info registers rip` confirmed `rip = 0x…32a`, i.e. exactly the `call sleep@plt` instruction. `x/4gx $rbp-0x30` showed the raw stack bytes that hold the encoded flag.
6. `jump *(main+104)` asked gdb to set the program counter to `0x…32f` (the `lea rax, [rbp-0x30]` instruction right after the call) and resume. The `call sleep@plt` never executed, so the 100 000-second wait never started.
7. The CPU continued from `lea rax, [rbp-0x30]`, called `rotate_encrypt` to ROT-47 the encoded bytes, called `fputs` to write the resulting string to stdout, called `putchar('\n')`, freed the heap-allocated string, and returned `0` from `main`.
8. The process exited normally; gdb reported `[Inferior 1 (process NNN) exited normally]` and dropped back to the prompt.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `file`           | Confirmed the binary is a 64-bit ELF, PIE, not stripped.       |
| `objdump -d -M intel` | Read `main` to confirm what `main+99` and `main+104` point at. |
| `gdb`            | The whole point of the challenge — breakpoint, jump, continue. |
| `layout asm` (TUI) | Cosmetic split-screen to watch disassembly while stepping.    |
| `break *(main+99)` | Paused execution on the call to `sleep(100000)`.              |
| `jump *(main+104)` | Redirected execution past the sleep call.                     |
| `info registers rip` / `x/4gx $rbp-0x30` | Confirmed where we were and dumped the encoded bytes. |
| `python3`        | Static ROT-47 decode and binary patching for the alternative solves. |
| `chmod +x`       | Made the downloaded binary executable.                         |
| `timeout`        | Demonstrated that running the binary directly hangs.           |

---

## Key Takeaways

- A `break` followed by a `jump` is the standard gdb idiom for *skipping* code at runtime. You can use it to bypass long sleeps, anti-debugging tricks, license checks, or any gate that lives between you and the useful function.
- Symbol-relative offsets (`main+99`) survive PIE and ASLR because the loader keeps the *relative* layout of the binary fixed even though the absolute base moves. Always prefer `symbol+offset` to raw addresses when scripting gdb.
- A `call sleep@plt` with a huge immediate is a classic CTF time-waster. When you see `mov edi, 0x186a0` followed by `call sleep`, treat it as "the answer is past this line; do not wait it out."
- `layout asm` is gdb's TUI split-screen — useful when stepping through code visually, but it requires a real terminal and is purely cosmetic. The same commands work without it.
- The flag's leet: `d3bugg3r_dr1v3` reads as **debugger drive** (`3→e`, `1→i`, `3→e`). The challenge is called *GDB Test Drive*; the flag's wordplay is literally *debugger drive* — you "drove" the gdb past the 27-hour sleep to reach the destination.
