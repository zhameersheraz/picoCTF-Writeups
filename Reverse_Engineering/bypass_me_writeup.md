# Bypass Me — picoCTF Writeup

**Challenge:** Bypass Me
**Category:** Reverse Engineering
**Difficulty:** Medium
**Points:** 100
**Flag:** `picoCTF{d3bugg3r_p0w3r_is_4w3s0m3_b8f8478e}`
**Platform:** picoCTF 2026
**Writeup by:** zham

---

## Description

> Your task is to analyze and exploit a password-protected binary called bypassme.bin and binary performs input sanitization.
>
> However, instead of guessing the password, you are expected to reverse engineer or debug the program to bypass the authentication logic and retrieve the hidden flag.
>
> You'll need to think like an attacker using tool like LLDB to uncover how the binary works under the hood and leak the correct password.
>
> ssh to foggy-cliff.picoctf.net:63672, and run the binary named "bypassme.bin" once connected. Login as ctf-player with the Password, 6dd28e9b

---

## Hints

> 1. Try disassembling the binary to understand its inner workings.
> 2. Pay special attention to functions available
> 3. The password might be hidden or decoded at runtime

---

## Background Knowledge (Read This First!)

### What is "input sanitization" here?

The binary reads our input, then strips out everything that isn't an alphabetic character (it uses `isalpha` from libc). That is why we see two echoed lines: `Raw Input` shows exactly what we typed, and `Sanitized Input` shows the version the program actually compares against. So whatever the real password is, it must be made entirely of letters A-Z / a-z. Anything we type that isn't a letter gets thrown away.

### What is LLDB and why use it?

LLDB is the debugger that ships with LLVM. While a program is running, we can pause it at any function and inspect memory. The challenge hint points us at LLDB specifically because the password is built up in memory at runtime, never sitting as a plain ASCII string in the binary. If we only ran `strings` on the file, we would never see the password. We have to actually let the program run, then peek inside its memory.

### What does "decoded at runtime" mean?

The program stores the password as a scrambled byte array on the stack, then loops through it and XORs each byte with the constant `0xAA` to produce the real password. This is a classic obfuscation trick: a casual `strings` dump shows garbage like `\xf9\xdf\xda\xcf...`, but the actual password "SuperSecure" only ever exists in RAM while the program is running. That is exactly the kind of thing a debugger (or careful disassembly reading) is designed to expose.

### The functions inside the binary

```
$ nm bypassme.bin | grep " T "

0000000000001457 T _Z13auth_sequencev       # auth_sequence()
00000000000014c6 T _Z14intro_sequencev      # intro_sequence()
0000000000001333 T _Z15decode_passwordPc    # decode_password(char*)
00000000000013c4 T _Z8sanitizePKcPc         # sanitize(const char*, char*)
00000000000012c9 T _Z8type_outPKcj          # type_out(const char*, unsigned int)
000000000000162e T main
```

`_Z15decode_passwordPc` is the one we want. It is the function that builds the password in memory right before `strcmp` is called against our sanitized input.

---

## Solution — Step by Step

### Step 1 — Connect and look around

```
┌──(zham㉿kali)-[~]
└─$ sshpass -p '6dd28e9b' ssh -o StrictHostKeyChecking=no -p 63672 ctf-player@foggy-cliff.picoctf.net
```

Once in:

```
ctf-player@foggy-cliff:~$ ls -la
-rwsr-xr-x 1 root root 21672 Mar  6 20:10 bypassme.bin
```

That `s` in the permissions means `bypassme.bin` is a **setuid root** binary. We cannot read `/root/flag.txt` directly, but the binary can, so we have to play by its rules.

### Step 2 — Run it once to see what we are dealing with

```
┌──(zham㉿kali)-[~]
└─$ echo "test" | ./bypassme.bin
```

We see the SECURE PORTAL banner, then:

```
[3 tries left] Enter password:
Raw Input:      [test]
Sanitized Input:[test]
Hint: Input must match something special...
Access Denied
```

It gives us 3 tries and the same prompt repeats. Let me confirm the sanitization behavior:

```
┌──(zham㉿kali)-[~]
└─$ printf "a1b2!@#c\n" | ./bypassme.bin
...
Raw Input:      [a1b2!@#c]
Sanitized Input:[abc]
```

Confirmed: digits and symbols are dropped, only letters survive. The password itself must be all letters.

### Step 3 — Pull the binary locally for closer inspection

We could analyze it on the remote box, but it's nicer to copy it back to Kali so we can poke at it freely.

```
┌──(zham㉿kali)-[~/ctf]
└─$ sshpass -p '6dd28e9b' scp -P 63672 ctf-player@foggy-cliff.picoctf.net:bypassme.bin .
```

Quick reconnaissance:

```
┌──(zham㉿kali)-[~/ctf]
└─$ file bypassme.bin
bypassme.bin: setuid ELF 64-bit LSB shared object, x86-64, ... with debug_info, not stripped

┌──(zham㉿kali)-[~/ctf]
└─$ nm bypassme.bin | grep " T "
0000000000001457 T _Z13auth_sequencev
00000000000014c6 T _Z14intro_sequencev
0000000000001333 T _Z15decode_passwordPc
00000000000013c4 T _Z8sanitizePKcPc
00000000000012c9 T _Z8type_outPKcj
000000000000162e T main
```

Two interesting things:
- The binary is **not stripped**, so symbols and even debug info are present.
- `decode_password(char*)` literally builds the password for us. Let's read it.

### Step 4 — Disassemble decode_password with objdump

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d --disassemble=_Z15decode_passwordPc bypassme.bin
```

```
0000000000001333 <_Z15decode_passwordPc>:
    1333: endbr64
    1337: push   %rbp
    1338: mov    %rsp,%rbp
    133b: sub    $0x30,%rsp
    133f: mov    %rdi,-0x28(%rbp)        ; out buffer pointer saved
    1343: mov    %fs:0x28,%rax            ; stack canary
    134c: mov    %rax,-0x8(%rbp)
    1350: xor    %eax,%eax
    1352: movabs $0xc9cff9d8cfdadff9,%rax ; <-- 8 encoded bytes on stack
    135c: mov    %rax,-0x13(%rbp)
    1360: movw   $0xd8df,-0xb(%rbp)        ; <-- 2 more encoded bytes
    1366: movb   $0xcf,-0x9(%rbp)          ; <-- last encoded byte
    136a: movl   $0x0,-0x18(%rbp)          ; i = 0
    1371: mov    -0x18(%rbp),%eax
    1374: cltq
    1376: cmp    $0xa,%rax                ; while i <= 10
    137a: ja     13a2
    137c: mov    -0x18(%rbp),%eax
    1381: movzbl -0x13(%rbp,%rax,1),%eax   ; load enc[i]
    1386: xor    $0xffffffaa,%eax          ; enc[i] ^ 0xAA
    1389: mov    %eax,%ecx
    138b: mov    -0x18(%rbp),%eax
    1391: mov    -0x28(%rbp),%rax          ; out pointer
    1395: add    %rdx,%rax
    1398: mov    %ecx,%edx
    139a: mov    %dl,(%rax)                ; out[i] = decoded byte
    139c: addl   $0x1,-0x18(%rbp)          ; i++
    13a0: jmp    1371
    13a2: mov    -0x28(%rbp),%rax
    13a6: add    $0xb,%rax                 ; out[11] = 0  (NUL terminator)
    13aa: movb   $0x0,(%rax)
```

Reading this, the function:

1. Lays 11 encoded bytes on the stack at `-0x13(%rbp)`.
2. Loops `i` from 0 to 10 inclusive.
3. For each index, takes `enc[i]` and XORs it with `0xAA`.
4. Writes the result into `out[i]`.
5. Writes a NUL byte at `out[11]`.

The 11 encoded bytes, in order, are:

```
f9 df da cf d8 f9 cf c9 df d8 cf
```

### Step 5 — Decode the bytes (XOR with 0xAA)

I did it on the command line with Python so I did not have to do hex math by hand:

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "print(''.join(chr(b ^ 0xAA) for b in [0xf9,0xdf,0xda,0xcf,0xd8,0xf9,0xcf,0xc9,0xdf,0xd8,0xcf]))"
SuperSecure
```

The decoded password is `SuperSecure`. Quick sanity check by hand:

```
0xf9 ^ 0xAA = 0x53 -> 'S'
0xdf ^ 0xAA = 0x75 -> 'u'
0xda ^ 0xAA = 0x70 -> 'p'
0xcf ^ 0xAA = 0x65 -> 'e'
0xd8 ^ 0xAA = 0x72 -> 'r'
0xf9 ^ 0xAA = 0x53 -> 'S'
0xcf ^ 0xAA = 0x65 -> 'e'
0xc9 ^ 0xAA = 0x63 -> 'c'
0xdf ^ 0xAA = 0x75 -> 'u'
0xd8 ^ 0xAA = 0x72 -> 'r'
0xcf ^ 0xAA = 0x65 -> 'e'
```

`SuperSecure`. All letters, so the sanitizer will pass it through unchanged.

### Step 6 — Log in on the remote box and grab the flag

```
┌──(zham㉿kali)-[~]
└─$ sshpass -p '6dd28e9b' ssh -p 63672 ctf-player@foggy-cliff.picoctf.net
```

```
ctf-player@foggy-cliff:~$ printf "SuperSecure\n" | ./bypassme.bin
```

The intro plays, then:

```
[3 tries left] Enter password:
Raw Input:      [SuperSecure]
Sanitized Input:[SuperSecure]
Hint: Input must match something special...

Authenticating...
Flag: picoCTF{d3bugg3r_p0w3r_is_4w3s0m3_b8f8478e}
```

Got the flag on the first attempt.

---

## Alternative — Solve with LLDB (the hinted approach)

The challenge explicitly calls out LLDB, so I also did it that way for fun. Instead of reading objdump and XORing by hand, I let the program build the password in RAM and then dumped it.

```
┌──(zham㉿kali)-[~/ctf]
└─$ lldb ./bypassme.bin
(lldb) b _Z15decode_passwordPc          ; break on the decode function
(lldb) b _Z13auth_sequencev              ; and also on auth_sequence
(lldb) r                                   ; run the program
```

LLDB stops right inside `decode_password` before the loop has finished. At this point the encoded bytes are sitting on the stack at `-0x13(%rbp)`. I just let it continue past the function and stop at the auth prompt, then dumped the first argument (which is the buffer the function just filled in):

```
(lldb) c                                   ; continue past the break
Process stopped at auth_sequence
(lldb) register read rdi                   ; rdi still holds the buffer pointer
rdi = 0x00007ffe...                        ; truncated for clarity
(lldb) memory read -c 16 `$rdi`            ; read 16 bytes from that pointer
0x7ffe...: 53 75 70 65 72 53 65 63 75 72 65 00 ...
          S  u  p  e  r  S  e  c  u  r  e  \0
```

LLDB hands us the string right out of memory: `SuperSecure`. From there it's the same login step as above.

This is the more "attacker-minded" approach the hint nudges us toward: never guess, just observe.

---

## What Happened Internally

A timeline of what the binary is doing behind the curtain:

1. `main` is called. The very first thing it does is call `intro_sequence()`, which prints the SECURE PORTAL banner character by character (with `usleep` in between to look fancy).
2. `auth_sequence()` is then called. Inside it, the program allocates two buffers: one for raw input, one for sanitized input.
3. `decode_password(password_buffer)` is invoked. The password is **not stored as a string anywhere**. Instead, 11 scrambled bytes (`f9 df da cf d8 f9 cf c9 df d8 cf`) are pushed onto the stack. A loop XORs each one with `0xAA` and writes the result into the buffer. After the loop, a NUL terminator is appended, so the buffer finally holds the ASCII string `SuperSecure`.
4. We are prompted for a password. `fgets` reads up to a newline into the raw buffer.
5. `sanitize(raw, clean)` walks the raw buffer. Every character that is **not** alphabetic is dropped. The result is copied into the clean buffer.
6. The program calls `strcmp(clean, password_buffer)`. If they match, the flag file `../../root/flag.txt` is opened (the binary is setuid, so it can read root's files) and the flag is printed.
7. If `strcmp` fails, we get one of the three "Access Denied" prompts. After all three are used, the program prints "All attempts used. Try harder!" and exits.

The sanitization step is what makes brute force annoying: typing `123SuperSecure456` still results in `SuperSecure` being compared, but typing the wrong thing three times loses our chances.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `sshpass` | Auto-fill the SSH password so I could script the login |
| `ssh` | Connect to the picoCTF remote instance |
| `scp` | Copy `bypassme.bin` from the remote box back to Kali for analysis |
| `file` | Confirm the binary format, that it is ELF 64-bit, and that it has debug info |
| `nm` | List the exported functions inside the binary |
| `objdump` | Disassemble `decode_password` and read the XOR loop by eye |
| `python3` | Quick one-liner to XOR the encoded bytes with `0xAA` |
| `lldb` | Breakpoint-based approach: stop in `decode_password`, dump the resulting buffer from memory |
| `printf` | Pipe a clean input string into the binary (no newline issues like `echo -e`) |

---

## Key Takeaways

- `strings` is not enough. If the password is built in memory at runtime from XOR-scrambled bytes, you will never see it as a literal in the binary. You have to look at the code that builds it.
- Disassembly is your friend when the binary is not stripped. `nm` to find the function list, then `objdump -d --disassemble=<symbol>` to read just that one function cleanly.
- LLDB's "break on the decode function, then dump the destination buffer" trick is the textbook way to extract a runtime-built password. You don't have to step through every instruction, just inspect the right memory region at the right moment.
- XOR with a single repeating byte is the most basic form of obfuscation. If you see a `xor reg, 0xAA` inside a loop in a "decrypt" function, you are looking at the exact pattern used here.
- Input sanitization tells you a lot about the password format. Here, the sanitized view kept only letters, which immediately told us the password has no digits or symbols. That narrows the search space even before we read the disassembly.
- The setuid bit on `bypassme.bin` is why we cannot just `cat /root/flag.txt` directly. The binary is the bridge that has the privileges; we have to make it succeed.

### Flag wordplay decode

`picoCTF{d3bugg3r_p0w3r_is_4w3s0m3_b8f8478e}` -> "debugger power is awesome", written in l33t-speak (`3` for `e`, `0` for `o`, `4` for `a`, `3` for `e` again). The whole challenge is about using a debugger (LLDB) to do the dirty work for us, which is exactly what the flag is bragging about. The trailing `b8f8478e` is just a unique hex suffix to make this flag different from anyone else's.
