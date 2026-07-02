# Reverse — picoCTF Writeup

**Challenge:** Reverse  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d43}`  
**Platform:** picoCTF (picoGym Exclusive)  
**Writeup by:** zham  

---

## Description

> Try reversing this file? Can ya?
>
> I forgot the password to this file. Please find it for me?

**Hint given in the challenge:**

- No formal hint — the category name `Reverse Engineering` and the file extension `.bin` tell you to disassemble the binary.

---

## Background Knowledge (Read This First!)

### What is reverse engineering?

Reverse engineering (RE) is the process of taking a compiled program (machine code) and figuring out what it does — and usually extracting a hidden secret (flag, key, password) from it. You work **without source code**. The basic toolset is:

| Tool | Purpose |
|------|---------|
| `file` | Identify the binary format (ELF, PE, Mach-O) |
| `strings` | Print all readable ASCII/Unicode runs in the binary |
| `objdump -d` | Disassemble the `.text` section (machine code → assembly) |
| `nm` | List symbols (function names, globals) |
| `gdb` / `pwndbg` | Interactive debugger |
| `objdump -s -j .rodata` | Dump read-only data section (string literals) |
| `ltrace` / `strace` | Trace library calls / syscalls at runtime |

For a beginner-level challenge, **`strings` and `objdump -d`** are usually enough. Reach for `gdb` only when the binary hides the flag behind control flow or anti-RE tricks.

### ELF file structure (the parts that matter here)

An ELF binary on Linux is split into named sections:

| Section | Contents |
|---------|----------|
| `.text` | Executable machine code (what `objdump -d` shows) |
| `.rodata` | Read-only constants — string literals, jump tables |
| `.data` / `.bss` | Initialized / zero-initialized global variables |
| `.symtab` | Symbol table — function and global names (present only if not stripped) |

This binary is **not stripped** (`file` reports it), so symbol names like `main`, `strcmp`, `printf`, `puts`, `__isoc99_scanf` are visible in the disassembly.

### x86-64 calling convention (so you can read disassembly)

When the binary calls a function, arguments go in specific registers:

| Register | Role |
|----------|------|
| `rdi` | 1st argument |
| `rsi` | 2nd argument |
| `rdx` | 3rd argument |
| `rcx` | 4th argument |
| `rax` | return value |

So `call strcmp` with `rdi = input_buffer` and `rsi = flag_buffer` is `strcmp(input_buffer, flag_buffer)`.

### `movabs` — the giveaway for inlined strings

When you see `movabs $0x..., %rax` followed by `mov %rax, -OFFSET(%rbp)`, the binary is loading a constant 8-byte value into a stack buffer. Two `movabs` instructions next to each other = 16 ASCII characters being placed on the stack. Decode them as little-endian ASCII and you have a string. This is the most common way beginners hide a flag in C code without putting it in `.rodata`.

---

## Solution — Step by Step

### Step 1 — Copy and inspect the binary

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /tmp/reverse && cd /tmp/reverse
┌──(zham㉿kali)-[/tmp/reverse]
└─$ cp ~/Downloads/reverse.bin reverse && chmod +x reverse
┌──(zham㉿kali)-[/tmp/reverse]
└─$ file reverse
reverse: ELF 64-bit LSB pie executable, x86-64, ..., dynamically linked, ..., not stripped
┌──(zham㉿kali)-[/tmp/reverse]
└─$ ls -la reverse
-rwxr-xr-x 1 zham zham 16888 Jul  3 01:28 reverse
```

Key facts from `file`:

- 64-bit ELF, dynamically linked (uses libc for `printf`, `strcmp`, etc.)
- **Not stripped** — symbol names will be visible
- PIE executable (position-independent — common for modern distros)

### Step 2 — Try `strings` first (the 5-second solve)

Beginner's first reflex on any binary:

```
┌──(zham㉿kali)-[/tmp/reverse]
└─$ strings reverse | grep -i pico
picoCTF{H
Password correct, please see flag: picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d43}
```

Two hits — both part of the same flag. The flag is right there: `picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d43}`.

`strings` finds it because the success message "Password correct, please see flag: picoCTF{...}" is a literal string stored in `.rodata`. For a beginner challenge this often *is* the solution. You can stop here and submit.

### Step 3 — Confirm with `objdump -d` (read the assembly)

To understand *why* the binary works the way it does, disassemble `main`:

```
┌──(zham㉿kali)-[/tmp/reverse]
└─$ objdump -d reverse | sed -n '/<main>:/,/^$/p'
```

The interesting part — the flag construction on the stack:

```
11e4:   movabs $0x7b4654436f636970,%rax   ; "picoCTF{" (LE)
11ee:   movabs $0x337633725f666c33,%rdx   ; "3lf_r3v3"
11f8:   mov    %rax,-0x30(%rbp)
11fc:   mov    %rdx,-0x28(%rbp)

1200:   movabs $0x75735f676e693572,%rax   ; "r5ing_su"
120a:   movabs $0x6c75663535656363,%rdx   ; "cce55ful"
1214:   mov    %rax,-0x20(%rbp)
1218:   mov    %rdx,-0x18(%rbp)

121c:   movabs $0x346434316237645f,%rax   ; "_d7b14d4"
1226:   mov    %rax,-0x10(%rbp)

122a:   lea    0xdd7(%rip),%rdi            ; "Enter the password to unlock this file: "
1231:   call   printf

123b:   lea    -0x60(%rbp),%rax            ; input buffer
1242:   lea    0xde8(%rip),%rdi            ; "%s"
1249:   call   scanf                       ; read user input

126b:   lea    -0x30(%rbp),%rdx            ; flag (built on stack)
126f:   lea    -0x60(%rbp),%rax            ; input buffer
1273:   mov    %rdx,%rsi
1276:   mov    %rax,%rdi
1279:   call   strcmp                       ; strcmp(input, flag)
127e:   test   %eax,%eax
1280:   jne    129c                         ; if not equal -> "Access denied"

1282:   lea    0xdbf(%rip),%rdi             ; "Password correct, please see flag: picoCTF{...}"
1289:   call   puts
128e:   lea    -0x30(%rbp),%rax
1292:   mov    %rax,%rdi
1295:   call   puts                         ; puts(flag)
```

**Logic in plain English:**

1. Build the flag on the stack as 5 `movabs` immediates (no null terminator at the end).
2. `printf("Enter the password to unlock this file: ")`
3. `scanf("%s", input_buffer)`
4. `strcmp(input, flag)` — compare.
5. If equal → `puts("Password correct, please see flag: picoCTF{...}"); puts(flag);`
6. Else → `puts("Access denied")`.

### Step 4 — Decode the immediates with Python

Each `movabs` loads 8 bytes. Read them as little-endian ASCII:

```
┌──(zham㉿kali)-[/tmp/reverse]
└─$ python3 -c "
import struct
parts = [
    0x7b4654436f636970,
    0x337633725f666c33,
    0x75735f676e693572,
    0x6c75663535656363,
    0x346434316237645f,
]
print(b''.join(struct.pack('<Q', p) for p in parts).decode())
"
picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d4
```

Five `movabs` give 40 ASCII characters. Note the flag is **truncated** — it stops at `_d7b14d4` and is missing the closing `3}`.

### Step 5 — Dump `.rodata` to get the closing `3}`

```
┌──(zham㉿kali)-[/tmp/reverse]
└─$ objdump -s -j .rodata reverse | grep -A2 "Password"
 2048 50617373 776f7264 20636f72 72656374  Password
 2050 20636f72 72656374 2c20706c 65617365   correct, please
 2060 20736565 20666c61 673a2070 69636f43   see flag: picoC
 2070 54467b33 6c665f72 33763372 35696e67  TF{3lf_r3v3r5ing
 2080 5f737563 63653535 66756c5f 64376231  _succe55ful_d7b1
 2090 34643433 7d004163 63657373 2064656e  4d43}.Access den
```

The full success message: `"Password correct, please see flag: picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d43}"`.

So the **complete flag** is:

```
picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d43}
```

### Step 6 — Verify by running the binary with the right password

The stack-built flag (40 chars, no null terminator) is the password. It works because the stack canary right after it starts with a `\x00` byte, so `strcmp` reads up to 40 chars and stops.

```
┌──(zham㉿kali)-[/tmp/reverse]
└─$ printf "picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d4" | ./reverse
Enter the password to unlock this file: You entered: picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d4
Password correct, please see flag: picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d43}
picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d4
```

Confirmed: enter the 40-char stack-built password (no `3}`), and the binary prints the full 41-char flag.

If you enter the full 41-char flag as the password, it fails — because `strcmp` is comparing against the 40-char buffer plus the canary's null byte, not against the full string.

```
┌──(zham㉿kali)-[/tmp/reverse]
└─$ printf "picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d43}" | ./reverse
Enter the password to unlock this file: You entered: picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d43}
Access denied
```

---

## What Happened Internally (Timeline)

1. The source program put the flag in two places: (a) five `movabs` immediates that build a 40-byte buffer on the stack at run-time, and (b) a full 41-char success message in `.rodata`.
2. The 40-byte buffer was deliberately **not null-terminated** by the program. It relies on the Linux stack canary (which always starts with `\x00`) to terminate the string for `strcmp`.
3. `main` reads the user's input with `scanf("%s", ...)`, calls `strcmp(input, flag_buffer)`, and if they match, prints the full flag string from `.rodata` followed by `puts(flag_buffer)` (which prints the 40-byte stack-built string again).
4. `strings` immediately revealed the flag because the success message is a literal in `.rodata`. `objdump -d` confirmed the structure and let us reconstruct the missing `3}` from the immediates vs. the literal.

---

## Alternative Methods

**Method 1 — `strings` only (5-second solve)**

```
strings reverse | grep pico
```

Done. The flag is in `.rodata` as part of the success message.

**Method 2 — `gdb` interactive**

```bash
gdb ./reverse
(gdb) break main
(gdb) run
(gdb) x/s $rbp-0x30
```

`gdb` shows the flag buffer contents directly. Same idea as the disassembly but interactive.

**Method 3 — Just run the binary and feed it the password you extracted**

```bash
printf 'picoCTF{3lf_r3v3r5ing_succe55ful_d7b14d4' | ./reverse
```

The binary prints the full flag for you. Useful as a confirmation step.

**Method 4 — `ltrace` to watch the `strcmp` call**

```bash
ltrace ./reverse <<< "test"
```

You'll see `strcmp(input, flag)` and could potentially intercept it, but `strings` is faster.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Confirm ELF 64-bit, not stripped |
| `strings` | Extract readable strings — instantly finds the flag |
| `objdump -d` | Disassemble `main` to see how the flag is built |
| `objdump -s -j .rodata` | Dump the read-only data section to confirm the closing `3}` |
| `python3` (struct.pack) | Decode the 5 `movabs` immediates as little-endian ASCII |
| `printf \| ./reverse` | Run the binary with the right password to confirm |

---

## Key Takeaways

- **`strings | grep pico` is your first reflex on any binary RE challenge.** It catches the easy cases — secrets stored in `.rodata` (literal strings). 30 % of beginner CTF challenges can be solved with this one command.
- A `movabs $0x..., %rax` followed by `mov %rax, -OFFSET(%rbp)` pattern means the binary is **constructing a string on the stack at runtime**. Decode the immediates as little-endian ASCII (Python's `struct.pack('<Q', value).decode()`).
- The Linux stack canary **always starts with a `\x00` byte**. A stack buffer that "has no null terminator" effectively inherits the canary's null as its terminator for `strcmp`. That is exactly how this challenge compares a 40-byte input against a 40-byte buffer.
- The full flag may be split between **two locations** in the binary: the stack-built buffer (used for the comparison) and the `.rodata` literal (printed on success). Always check both — `.rodata` is what `strings` finds, the stack buffer is what the binary actually compares against.
- Flag wordplay: `3lf_r3v3r5ing_succe55ful` → **"elf reversing successful"** (3→e, 5→s). The challenge is literally about reversing an ELF binary — and the flag spells out the lesson. The trailing `_d7b14d4` is just an instance hash.
