# file-run2 — picoCTF Writeup

**Challenge:** file-run2  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{F1r57_4rgum3n7_96f2195f}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Another program, but this time, it seems to want some input. What happens if you try to run it on the command line with input "Hello!"?

## Hints

> 1. Try running it and add the phrase "Hello!" with a space in front (i.e. "./run Hello!")

---

## Background Knowledge

This is a 100-point intro challenge that tests two beginner skills: passing command-line arguments to a program, and reading a tiny C program from its disassembly. Here is the vocabulary.

**1. `argc` and `argv`.**
When the OS loads a C program, `main` receives two arguments:
- `argc` (argument count) — an `int` equal to the number of space-separated tokens on the command line, **including the program name itself**.
- `argv` (argument vector) — a `char**`, an array of C strings. `argv[0]` is always the program path (`./run`), `argv[1]` is the first user argument, etc.

So `./run Hello!` runs the program with `argc == 2` and `argv[1] == "Hello!"`. `./run` alone gives `argc == 1` and there is no `argv[1]`. That distinction is the entire gate.

**2. `strcmp(a, b)` returns 0 on match.**
`strcmp` is *not* a boolean. It returns `0` when the two strings are identical, a negative number when `a` sorts before `b`, and a positive number when `a` sorts after `b`. That is why you see `if (strcmp(...) == 0)` everywhere — beginners often write `if (strcmp(...) == 1)` which only fires by accident.

**3. The x86 calling convention for these calls.**
`rdi` holds the first argument, `rsi` the second, `rdx` the third. So `strcmp(argv[1], "Hello!")` is:
- `mov rax, [rbp-0x10]` — load `argv`.
- `add rax, 8` — skip `argv[0]` (a pointer is 8 bytes on x86-64).
- `mov rax, [rax]` — dereference to get `argv[1]` (a `char*`).
- `lea rsi, "Hello!"` — second argument.
- `call strcmp`.

Reading that little dance in `objdump` is the fastest way to know what string the program compares against — without ever running it.

**4. Why this challenge does not require patching or heavy reversing.**
`file-run1` (a sibling challenge) probably just printed something on run; this one adds the simple "give me a CLI argument" gate. Once you understand `argc` / `argv` and `strcmp`, the disassembly is a one-screen read. The hint is also the entire solution spelled out — `"./run Hello!"`. The work here is *recognising* that the program wants `Hello!` as `argv[1]`, not as stdin, not as input redirection.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The downloaded file is `run`.

### 1. Move the file into a working directory and confirm what it is

```
┌──(zham㉿kali)-[~/file-run2]
└─$ mkdir -p ~/file-run2 && cd ~/file-run2

┌──(zham㉿kali)-[~/file-run2]
└─$ cp ~/Downloads/run .

┌──(zham㉿kali)-[~/file-run2]
└─$ chmod +x run

┌──(zham㉿kali)-[~/file-run2]
└─$ file run
run: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=303d7c50cf258e07aa508de37a50e4d85d5e475f, for GNU/Linux 3.2.0, not stripped
```

`not stripped` — symbols like `main` will resolve cleanly in `objdump` / `gdb`.

### 2. Try running it with no arguments

```
┌──(zham㉿kali)-[~/file-run2]
└─$ ./run
Run this file with only one argument.
```

The program prints a usage hint. That confirms it wants exactly one user argument (`argv[1]`), bringing the total argument count to `argc == 2`.

### 3. Try the hint's exact command

```
┌──(zham㉿kali)-[~/file-run2]
└─$ ./run Hello!
The flag is: picoCTF{F1r57_4rgum3n7_96f2195f}
```

The flag is `picoCTF{F1r57_4rgum3n7_96f2195f}`.

### 4. (For the writeup only) Confirm what a wrong argument does

```
┌──(zham㉿kali)-[~/file-run2]
└─$ ./run wrong
Won't you say 'Hello!' to me first?
```

Nice touch — the failure message spells out exactly what was expected. The two messages ("Run this file with only one argument." and "Won't you say 'Hello!' to me first?") are the *only* two `puts` strings in the binary, so there are exactly three code paths: too few args, right arg, wrong arg.

### 5. (Optional) Confirm by reading the disassembly

For the curious — and to prove we did not just get lucky — here is the relevant slice of `main`:

```
┌──(zham㉿kali)-[~/file-run2]
└─$ objdump -d -M intel run | sed -n '/<main>:/,/<__libc_csu_init>:/p'
```

```
0000000000001189 <main>:
    1189: endbr64
    118d: push   rbp
    118e: mov    rbp,rsp
    1191: sub    rsp,0x10
    1195: mov    DWORD PTR [rbp-0x4],edi        ; argc
    1198: mov    QWORD PTR [rbp-0x10],rsi       ; argv
    119c: cmp    DWORD PTR [rbp-0x4],0x1        ; argc > 1 ?
    11a0: jle    11a8                            ;   no  -> error
    11a2: cmp    DWORD PTR [rbp-0x4],0x2        ; argc <= 2 ?
    11a6: jle    11bb                            ;   yes -> compare argv[1]
    11a8: lea    rdi,[rip+0xe81]                 ; "Run this file with only one argument."
    11af: call   puts@plt
    11b4: mov    eax,0x0
    11b9: jmp    1207
    11bb: mov    rax,QWORD PTR [rbp-0x10]       ; argv
    11bf: add    rax,0x8                         ; skip argv[0]
    11c3: mov    rax,QWORD PTR [rax]             ; argv[1]
    11c6: lea    rsi,[rip+0xe89]                 ; "Hello!"
    11cd: mov    rdi,rax
    11d0: call   strcmp@plt
    11d5: test   eax,eax
    11d7: jne    11f6                            ; if not zero -> wrong
    11d9: mov    rax,QWORD PTR [rip+0x2e30]     ; flag pointer
    11e0: mov    rsi,rax
    11e3: lea    rdi,[rip+0xe73]                 ; "The flag is: %s"
    11ea: mov    eax,0x0
    11ef: call   printf@plt
    11f4: jmp    1202
    11f6: lea    rdi,[rip+0xe73]                 ; "Won't you say 'Hello!' to me first?"
    11fd: call   puts@plt
    1202: mov    eax,0x0
    1207: leave
    1208: ret
```

Three branches, all readable:

- `argc <= 1` *or* `argc > 2` → "Run this file with only one argument." (the `jle 11a8` covers `argc == 1`, and `argc == 2` falls through to the `jle 11bb`; any `argc > 2` falls through to `11a8` again).
- `argc == 2` and `strcmp(argv[1], "Hello!") == 0` → `printf("The flag is: %s", flag)`.
- `argc == 2` and the strings differ → "Won't you say 'Hello!' to me first?".

The flag itself lives in `.rodata` at offset `0x2008`, and `.data` at offset `0x4010` holds a pointer to it (loaded by `mov rax, [rip+0x2e30]`). `strings run | grep pico` would also pull the literal out directly:

```
┌──(zham㉿kali)-[~/file-run2]
└─$ strings run | grep pico
picoCTF{F1r57_4rgum3n7_96f2195f}
```

---

## Alternative Solves

**A. Skip running the binary — read the flag directly with `strings`.**
The flag is a plain ASCII literal sitting in `.rodata`. `strings` finds printable runs ≥ 4 chars by default, and `picoCTF{...}` is exactly that:

```
┌──(zham㉿kali)-[~/file-run2]
└─$ strings run | grep picoCTF
picoCTF{F1r57_4rgum3n7_96f2195f}
```

Useful when the program has side effects you do not want to trigger (network calls, file writes, hangs).

**B. Use `objdump -s -j .rodata run` to dump the read-only data section.**
`objdump -s` prints raw hex+ASCII of a section. Because the flag is a constant, it has to live in `.rodata`:

```
┌──(zham㉿kali)-[~/file-run2]
└─$ objdump -s -j .rodata run
```

```
Contents of section .rodata:
 2000 01000200 00000000 7069636f 4354467b  ........picoCTF{
 2010 46317235 375f3472 67756d33 6e375f39  F1r57_4rgum3n7_9
 2020 36663231 3935667d 00000000 00000000  6f2195f}........
```

The right-hand column is the ASCII rendering. The `picoCTF{...}` literal sits between offset `0x2008` and `0x2023`.

**C. Use `gdb` to break after the `strcmp` and read the flag from memory.**
If you wanted to be extra sure, fire up `gdb`, set a breakpoint after the `strcmp`, and either inspect the comparison result or skip straight to the `printf` branch:

```
┌──(zham㉿kali)-[~/file-run2]
└─$ gdb -batch \
    -ex "break *main+76" \
    -ex "run Hello!" \
    -ex "x/s *(void**)(0x4010 + 0x555555554000)" \
    ./run
```

(The exact GDB expression depends on the PIE base, but any "read the pointer at `0x4010` and dereference it as a `char*`" form works.) This is overkill for the challenge but a good habit when the flag is stored behind a pointer instead of inline.

---

## What Happened Internally

1. The shell parsed `./run Hello!` into two tokens (`./run` and `Hello!`) and called the kernel's `execve` with `argv = ["./run", "Hello!", NULL]`.
2. The dynamic loader mapped the ELF, set up the GOT/PLT for `puts`, `strcmp`, `printf`, and jumped to `main` at `0x1189`.
3. `main` saved `argc` (the `int` `2`) into `[rbp-0x4]` and `argv` (the `char**`) into `[rbp-0x10]` per the System V AMD64 ABI.
4. The first `cmp` checked `argc > 1`. With `argc == 2` it fell through to the second `cmp`, which checked `argc <= 2`. That also held, so control jumped to `0x11bb` — the comparison branch.
5. `0x11bb`–`0x11c3` computed `argv[1]` by reading `[rbp-0x10]` (the `argv` pointer), adding `8` to skip `argv[0]` (each pointer is 8 bytes), and dereferencing to load the `char*` to the user argument.
6. `lea rsi, [rip+0xe89]` loaded the address of the literal `"Hello!"` from `.rodata` (offset `0x2056`). `mov rdi, rax` put `argv[1]` in the first argument slot.
7. `call strcmp@plt` resolved `strcmp` through the PLT, which on first call jumped to the dynamic linker and from then on jumped directly into libc. `strcmp` walked both strings byte-by-byte, found them identical (`'H'`,`'e'`,`'l'`,`'l'`,`'o'`,`'!'` all match), and returned `0`.
8. `test eax, eax; jne 11f6` saw the zero result and fell through to the success path. `mov rax, [rip+0x2e30]` loaded the pointer stored in `.data` at `0x4010`, which pointed to the flag string in `.rodata` at `0x2008`.
9. `lea rdi, "The flag is: %s"; mov rsi, rax; call printf` formatted the flag into stdout. The user saw `The flag is: picoCTF{F1r57_4rgum3n7_96f2195f}`.
10. `main` returned `0` to `__libc_start_main`, which called `exit(0)` and the process terminated cleanly.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `file`           | Confirmed the artifact is a 64-bit ELF, PIE, not stripped.      |
| `chmod +x`       | Made the downloaded binary executable.                         |
| `./run` (no arg) | Verified the program prints a usage message — it wants exactly one argument. |
| `./run Hello!`   | The actual solve, straight from the challenge hint.             |
| `./run wrong`    | Captured the failure message for the writeup.                  |
| `objdump -d -M intel` | Disassembled `main` to confirm the `argc`/`strcmp` logic.   |
| `objdump -s -j .rodata` | Dumped the read-only string table including the flag.    |
| `strings`        | Alternative one-liner to extract the literal flag.             |

---

## Key Takeaways

- `argc` and `argv` are the *only* way most CLI programs receive user input. If `./prog` does nothing and `./prog foo` prints something, the program is reading `argv[1]`. That is the entire "input handling" of `file-run2`.
- "Won't you say 'Hello!' to me first?" is the program's polite hint. When a binary prints a specific string in its failure path, *that string is the literal it was expecting*. Read the error first, then supply what it asked for.
- The disassembly of a tiny C program fits in one screen. Once you recognise `argc`, `argv`, `strcmp`, and `printf`, you can read the whole flow without ever executing the binary. That is the actual skill `file-run2` is testing.
- `strings <binary> | grep picoCTF` is a one-liner that finds a hard-coded flag in `.rodata`. Use it any time the flag is a plain ASCII literal — it is faster than running the program.
- The flag's leet: `F1r57_4rgum3n7` reads as **first_argument** (`1→i`, `5→s`, `7→t`, `4→a`, `3→e`, `7→t`). The challenge is literally called *file-run2* and the gate is the *first argument* to the program — the flag's wordplay is the program mechanics in plain English.
