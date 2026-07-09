# file-run1 — picoCTF Writeup

**Challenge:** file-run1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{U51N6_Y0Ur_F1r57_F113_47cf2b7b}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> A program has been provided to you, what happens if you try to run it on the command line?
>
> Download the program `here`.

## Hints

> 1. To run the program at all, you must make it executable (i.e. $ `chmod +x run`)
> 2. Try running it by adding a `./` in front of the path to the file (i.e. $ `./run`)

---

## Background Knowledge

This is the simplest Reverse Engineering challenge in the picoCTF 2022 set — the entire solve is "treat the downloaded file like a Linux program." A few concepts make the move from "file in `~/Downloads`" to "executable you can run" feel obvious.

**1. File permissions and the executable bit.**
Linux keeps three permission bits per file for three audiences: *owner*, *group*, and *others*. Each audience can independently be granted **r**ead, **w**rite, and e**x**ecute. `chmod +x run` adds the execute bit for everyone, which is what turns a text file or a downloaded binary into something the kernel will load as a program. Without that bit, `./run` returns `Permission denied`.

**2. Why `./` and not just `run`.**
When you type `run`, the shell searches a list of directories called `$PATH` (typically `/usr/local/bin`, `/usr/bin`, `/bin`, ...). Your current directory — `.` — is **not** in `$PATH` by default for security reasons (otherwise an attacker could drop a `ls` script in any folder and trick you into running it). Typing `./run` says "the program is *here*, in this directory." Once you understand that, the `./` is no longer mysterious.

**3. `file` and the `ELF` header.**
A Linux executable starts with the four magic bytes `\x7fELF` (`0x7f`, `'E'`, `'L'`, `'F'`). `file run` reads those bytes and prints a one-line description: 32-bit vs 64-bit, little-endian vs big-endian, PIE vs non-PIE, dynamically linked vs statically linked, stripped vs not stripped. That single line tells you most of what you need to know before running it.

**4. PIE + not stripped.**
This binary is `pie executable` and `not stripped`:
- **PIE** = Position Independent Executable. The kernel loads it at a random base address (ASLR), but every offset inside the file (function addresses relative to the load base, byte offsets in sections) stays the same.
- **not stripped** = the symbol table is intact, so `main`, `printf`, and friends are real names you can use in `objdump`, `gdb`, and `nm`. With a stripped binary you would have to guess names from addresses.

**5. The "two hints" pattern.**
Hint 1 (`chmod +x`) and Hint 2 (`./run`) are the *complete* solve. Each one removes a single piece of friction:
- `chmod +x` removes the *permission* obstacle.
- `./` removes the *path* obstacle.

Together they are the smallest possible "from download to running program" recipe on Linux.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The downloaded challenge file is `run`.

### 1. Move the file into a working directory and confirm what it is

```
┌──(zham㉿kali)-[~/file-run1]
└─$ mkdir -p ~/file-run1 && cd ~/file-run1

┌──(zham㉿kali)-[~/file-run1]
└─$ cp ~/Downloads/run .

┌──(zham㉿kali)-[~/file-run1]
└─$ file run
run: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4d8e230e54db29e0879e7ed9f2b2231eb8c60032, for GNU/Linux 3.2.0, not stripped
```

`ELF 64-bit LSB pie executable`, `dynamically linked`, `not stripped` — the standard shape of a modern Linux C binary.

### 2. Try running it without `chmod +x` (to prove the hint matters)

```
┌──(zham㉿kali)-[~/file-run1]
└─$ ./run
bash: ./run: Permission denied
```

The shell refuses because the executable bit is not set. Notice that bash gave the failure *before* the kernel even saw the file — the kernel reports `EACCES`, but bash translates that into a friendlier message. If you see this error, your first move should always be `chmod +x` on the offending file.

### 3. Make it executable and try again

```
┌──(zham㉿kali)-[~/file-run1]
└─$ chmod +x run

┌──(zham㉿kali)-[~/file-run1]
└─$ ls -la run
-rwxr-xr-x 1 zham zham 16736 ... run
```

The mode string changed from `-rw-r--r--` (no execute bits) to `-rwxr-xr-x` (executable for everyone). Now:

```
┌──(zham㉿kali)-[~/file-run1]
└─$ ./run
The flag is: picoCTF{U51N6_Y0Ur_F1r57_F113_47cf2b7b}
```

The flag is `picoCTF{U51N6_Y0Ur_F1r57_F113_47cf2b7b}`. The whole program is one `printf` — no gates, no comparisons, no input.

### 4. (For the writeup) Confirm by reading the binary

This challenge is over once you run it, but for completeness, the disassembly of `main` is six real instructions:

```
┌──(zham㉿kali)-[~/file-run1]
└─$ objdump -d -M intel run | sed -n '/<main>:/,/<__libc_csu_init>:/p'
```

```
0000000000001149 <main>:
    1149: endbr64
    114d: push   rbp
    114e: mov    rbp,rsp
    1151: sub    rsp,0x10
    1155: mov    DWORD PTR [rbp-0x4],edi        ; save argc
    1158: mov    QWORD PTR [rbp-0x10],rsi       ; save argv
    115c: mov    rax,QWORD PTR [rip+0x2ead]     ; load flag pointer from .data
    1163: mov    rsi,rax
    1166: lea    rdi,[rip+0xec3]                 ; "The flag is: %s"
    116d: mov    eax,0x0
    1172: call   printf@plt
    1177: mov    eax,0x0
    117c: leave
    117d: ret
```

There is no input handling, no comparison, no branch. `main` unconditionally loads the flag string from a pointer in `.data` and prints it.

You can also confirm by inspecting `.rodata` directly:

```
┌──(zham㉿kali)-[~/file-run1]
└─$ objdump -s -j .rodata run
```

```
Contents of section .rodata:
 2000 01000200 00000000 7069636f 4354467b  ........picoCTF{
 2010 5535314e 365f5930 55725f46 31723537  U51N6_Y0Ur_F1r57
 2020 5f463131 335f3437 63663262 37627d00  _F113_47cf2b7b}.
 2030 54686520 666c6167 2069733a 20257300  The flag is: %s.
```

The flag literal starts at offset `0x2008` (right after the eight-byte `01000200 00000000` padding), and the `printf` format string `"The flag is: %s"` sits at offset `0x2030`.

---

## Alternative Solves

**A. Skip running the binary — read the flag directly with `strings`.**
The flag is a 38-character ASCII literal in `.rodata`. `strings` finds it in one line:

```
┌──(zham㉿kali)-[~/file-run1]
└─$ strings run | grep picoCTF
picoCTF{U51N6_Y0Ur_F1r57_F113_47cf2b7b}
```

Use this when you do not want to risk executing an unknown binary, or when the binary is interactive and you just want the literal.

**B. Use `nm` or `objdump -t` to read the symbol table, then dereference the `flag` global.**
The binary declares a global variable named `flag` (visible because it is `not stripped`). You can see its address and then read the pointed-to string:

```
┌──(zham㉿kali)-[~/file-run1]
└─$ nm run | grep -w flag
0000000000004010 D flag

┌──(zham㉿kali)-[~/file-run1]
└─$ objdump -s --start-address=0x2008 --stop-address=0x202f run
```

```
Contents of section .rodata:
 2008 7069636f 4354467b 5535314e 365f5930  picoCTF{U51N6_Y0Ur
 2018 55725f46 31723537 5f463131 335f3437  Ur_F1r57_F113_47
 2028 63663262 37627d00                 cf2b7b}.
```

The `flag` global at `0x4010` is a pointer whose value is `0x2008` — that points straight at the start of the `picoCTF{` literal in `.rodata`.

**C. Use `gdb` and just `print flag`.**
With the symbol table intact, `gdb` lets you print the value of the global directly:

```
┌──(zham㉿kali)-[~/file-run1]
└─$ gdb -batch -ex "start" -ex "print (char*)flag" ./run
```

```
Temporary breakpoint 1 at 0x1149
Temporary breakpoint 1, 0x0000555555555149 in main ()
$1 = 0x555555558008 "picoCTF{U51N6_Y0Ur_F1r57_F113_47cf2b7b}"
```

This is overkill for `file-run1` but a great pattern when the flag is hidden behind a global pointer in a larger program.

---

## What Happened Internally

1. The shell parsed `./run` into a single token. Because `.` is not in `$PATH`, it called `execve("./run", ["run", NULL], envp)` directly on the file.
2. The kernel saw the executable bit was *not* set and returned `EACCES`. `bash` translated that to the friendly `Permission denied` message and never reached `execve`.
3. After `chmod +x`, the executable bit was on. The kernel read the first four bytes of the file (`\x7f E L F`), recognised the ELF magic, mapped the segments into memory, set up the stack with `argc=1`, `argv=["./run", NULL]`, and `envp`, and jumped to the dynamic loader (`/lib64/ld-linux-x86-64.so.2`).
4. The loader resolved `printf` from `libc`, patched the GOT entry for `printf@plt`, and transferred control to `main` at `0x1149`.
5. `main` saved `argc` and `argv` (unused), then loaded the pointer stored in the global `flag` at `.data` offset `0x4010` into `rax`. That pointer's value was `0x2008`, the address of the `picoCTF{...}` literal in `.rodata`.
6. `main` set `rdi` to the format string `"The flag is: %s"`, set `rsi = rax` (the flag pointer), zeroed `eax` (no floating-point args for printf on x86-64 SysV), and called `printf@plt`.
7. `printf` walked the format string, hit the `%s`, read the `char*` from `rsi`, and wrote the flag bytes to stdout. The user saw `The flag is: picoCTF{U51N6_Y0Ur_F1r57_F113_47cf2b7b}`.
8. `main` set `eax = 0`, ran `leave; ret`, and the process exited with status `0`.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `mkdir` / `cd`   | Created an isolated working directory for the challenge.       |
| `cp`             | Copied the downloaded `run` file into the working directory.   |
| `file`           | Confirmed the artifact is a 64-bit ELF, PIE, not stripped.      |
| `./run` (no chmod) | Demonstrated the `Permission denied` failure for the writeup. |
| `chmod +x`       | Set the executable bit, exactly as the hint instructed.        |
| `./run`          | The actual solve — one command.                                |
| `objdump -d -M intel` | Disassembled `main` to confirm the one-`printf` shape.     |
| `objdump -s -j .rodata` | Dumped the read-only string table to show the flag literal. |
| `strings`        | Alternative one-liner to extract the literal flag.             |
| `nm`             | Showed the global `flag` symbol at `0x4010` for the alt solve. |
| `gdb -batch`     | The `print (char*)flag` demo for the alt solve.                |

---

## Key Takeaways

- `chmod +x` is the standard first move on any downloaded ELF. Until you do it, the kernel will refuse with `Permission denied` regardless of how the file looks.
- The leading `./` in `./run` is not decoration — it is a path. `.` is "this directory", and the shell refuses to look for executables in `.` until you ask it to. Knowing why avoids the "why does `./` work but `run` doesn't?" trap.
- A PIE, dynamically-linked, non-stripped ELF is the *default* shape of a Linux C binary. Once you recognise it, you can stop reading `file` output and get on with the actual challenge.
- `main` does not always need input, gates, or comparisons. The shortest possible `main` is "load a string and printf it." For those, `chmod +x && ./run` is genuinely the whole solve.
- The flag's leet: `U51N6_Y0Ur_F1r57_F113` reads as **using_your_first_file** (`5→s`, `1→i`, `6→g`, `0→o`, `1→i`, `5→s`, `7→t`, `1→i`, `1→i`, `3→e`). The challenge is the *first* binary you ever run as part of picoCTF, and the flag's wordplay spells out the exact lesson: *you are now using your first file*.
