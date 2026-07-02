# packer — picoCTF Writeup

**Challenge:** packer  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{U9X_UnP4ck1N6_B1n4Ri3S_bdd84893}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> Reverse this linux executable?

## Hints

> 1. What can we do to reduce the size of a binary after compiling it.

---

## Background Knowledge (Read This First!)

### What is a Binary Packer?

When you compile a C/C++ program, the resulting ELF file contains the actual machine code plus a lot of metadata, debug symbols, and unused bytes. A **binary packer** is a tool that compresses (and sometimes encrypts) the executable sections of a binary so the file becomes much smaller. When you run the packed binary, a tiny stub at the start decompresses everything in memory and then jumps into the real code.

The most common packer on Linux is **UPX** — the **U**ltimate **P**acker for e**X**ecutables. It is free, open source, and reversible. If a binary was packed with UPX, you can usually just run `upx -d file` to get the original binary back.

### How do I Recognise a UPX-packed Binary?

A few easy tells:
- `file binary` reports `statically linked, no section header` — that is the classic UPX signature. Normal ELF binaries always have section headers.
- `strings binary | grep -i upx` shows a banner like `$Info: This file is packed with the UPX executable packer http://upx.sf.net $`.
- The file size is suspiciously small compared to the amount of functionality it has.

### Why Doesn't `file` Show the Sections?

When UPX compresses the binary, it strips most of the section headers and overlays a stub. The compressed data is stored in the `.text` segment and is decompressed into memory at runtime. Tools that read sections (like debuggers or normal `objdump`) cannot find much because the original sections are gone. Unpacking first restores them.

### Hex-Encoded Strings

The flag in this challenge is stored as a long hex string like `7069636f4354467b...`. Each pair of hex digits is one byte. Decode with `xxd -r -p` (reverse a plain hex dump) or in Python with `bytes.fromhex(...)` / `binascii.unhexlify(...)`. The decoded bytes are the ASCII flag.

---

## Solution — Step by Step

### Step 1 — Identify the Binary

I downloaded the attachment, gave it the executable bit, and asked `file` what it is:

```
┌──(zham㉿kali)-[~/packer]
└─$ chmod +x packer
└─$ file packer
packer: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, no section header
```

Two red flags at once:
1. `statically linked` — fine, lots of binaries are static.
2. **`no section header`** — almost always means a packer stripped them.

### Step 2 — Confirm It Is UPX

I scanned the strings for any packer banner:

```
┌──(zham㉿kali)-[~/packer]
└─$ strings packer | grep -i upx
UPX!@	
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
$Id: UPX 3.95 Copyright (C) 1996-2018 the UPX Team. All Rights Reserved. $
UPX!u
UPX!
UPX!
```

Confirmed: this is **UPX 3.95**.

### Step 3 — Install UPX (if Needed)

Kali usually ships `upx` already, but just in case:

```
┌──(zham㉿kali)-[~/packer]
└─$ which upx || sudo apt-get install -y upx-ucl
/usr/bin/upx
```

If `apt` does not have it, the official standalone binary works fine. I grabbed UPX 4.2.4 from GitHub as a backup:

```
┌──(zham㉿kali)-[~/packer]
└─$ curl -sL https://github.com/upx/upx/releases/download/v4.2.4/upx-4.2.4-amd64_linux.tar.xz | tar -xJ
└─$ ls upx-4.2.4-amd64_linux/upx
upx-4.2.4-amd64_linux/upx
```

### Step 4 — Unpack the Binary

I kept a copy of the packed original just in case, then unpacked:

```
┌──(zham㉿kali)-[~/packer]
└─$ cp packer packer.packed
└─$ upx -d packer -o packer.unpacked
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.4       Markus Oberhumer, Laszlo Molnar & John Reiser    May 9th 2024

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
[WARNING] bad b_info at 0x4b718

[WARNING] ... recovery at 0x4b714

    877724 <-    336520   38.34%   linux/amd64   packer.unpacked

Unpacked 1 file.
```

The size jumped from 336 KB to 877 KB — that 38% ratio line is UPX telling us it had compressed the file down to a bit more than a third of its real size. The two `[WARNING] bad b_info` lines are harmless; they happen because someone tampered with a UPX chunk and UPX auto-recovers.

Confirming it is now a normal ELF:

```
┌──(zham㉿kali)-[~/packer]
└─$ file packer.packed packer.unpacked
packer.packed:   ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, no section header
packer.unpacked: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=705654b705e0b7c1367d4f3761f5379b543607b2, for GNU/Linux 3.2.0, not stripped
```

Notice the difference: `no section header` became `not stripped`, and we now have a proper `BuildID`. UPX restored everything.

### Step 5 — Pull the Flag Out of Strings

Once the binary is unpacked, the flag is just sitting in the data section. A simple `strings` and `grep` finds it:

```
┌──(zham㉿kali)-[~/packer]
└─$ strings packer.unpacked | grep -iE "pico|flag|ctf"
Password correct, please see flag: 7069636f4354467b5539585f556e5034636b314e365f42316e34526933535f62646438343839337d
```

The line `Password correct, please see flag:` is a print format string that the program uses after you enter the correct password. The hex blob right after the colon is the flag, hex-encoded.

### Step 6 — Decode the Hex

```
┌──(zham㉿kali)-[~/packer]
└─$ echo "7069636f4354467b5539585f556e5034636b314e365f42316e34526933535f62646438343839337d" | xxd -r -p
picoCTF{U9X_UnP4ck1N6_B1n4Ri3S_bdd84893}
```

Or, the Python way:

```
┌──(zham㉿kali)-[~/packer]
└─$ python3 -c "import binascii; print(binascii.unhexlify('7069636f4354467b5539585f556e5034636b314e365f42316e34526933535f62646438343839337d').decode())"
picoCTF{U9X_UnP4ck1N6_B1n4Ri3S_bdd84893}
```

Got the flag: `picoCTF{U9X_UnP4ck1N6_B1n4Ri3S_bdd84893}`

---

## Alternative Method — Run the Unpacked Binary

Just to confirm the binary actually does what the strings say it does, I ran it and fed it some input:

```
┌──(zham㉿kali)-[~/packer]
└─$ chmod +x packer.unpacked
└─$ echo "test" | ./packer.unpacked
Enter the password to unlock this file: You entered: test

Access denied
```

The program prompts for a password, prints whatever you typed, and either shows the flag (on success) or says "Access denied" (on failure). Because the flag hex is a hardcoded literal in the binary, we do not actually need the correct password — `strings` already gives it to us.

### Bonus — Static Analysis with `objdump`

Once unpacked, normal reverse-engineering tools work again. For example:

```
┌──(zham㉿kali)-[~/packer]
└─$ objdump -d packer.unpacked | grep -A2 "<main>:" | head -10
0000000000401416 <main>:
  401416:	55                   	push   %rbp
  401417:	48 89 e5             	mov    %rsp,%rbp
```

You can now open the unpacked binary in Ghidra, Binary Ninja, radare2, or whatever you prefer and trace the password check properly.

---

## What Happened Internally

Here is the timeline of what was going on behind the scenes:

1. **The author compiled** a normal C program that asks for a password, checks it, and on success prints a hex-encoded flag.
2. **The author ran UPX** on the compiled ELF. UPX compressed the `.text` and `.data` sections, stripped section headers, and prepended a tiny decompression stub.
3. **The original sections disappeared** from the file. `file` reported `no section header`, `strings` returned very little, and any tool relying on sections could not read it.
4. **I ran `upx -d`** to undo step 2. UPX detected its own marker at the end of the file, decompressed the payload, and rebuilt the section table.
5. **`strings` on the unpacked ELF** immediately revealed the format string `Password correct, please see flag:` followed by the hex blob.
6. **Hex decode** of that blob gave the plaintext flag. No reverse engineering of the actual password check was needed.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Identify the ELF type and notice the missing section headers |
| `strings` + `grep` | Find the UPX banner and later the hidden flag |
| `curl` + `tar` | Fetch the standalone UPX binary as a backup |
| `upx -d` | Decompress the binary and restore its sections |
| `xxd -r -p` (or Python `binascii.unhexlify`) | Decode the hex-encoded flag |
| `chmod +x` | Make the unpacked binary executable so I could test it |

---

## Key Takeaways

- `file` reporting `no section header` on an ELF is the classic UPX signature. Always check it before doing any heavy reverse engineering.
- `strings binary | grep -i upx` is the fastest way to confirm UPX is in use.
- `upx -d binary -o unpacked` is the entire solve for any UPX-packed challenge. No need for a debugger, no need to crack the password.
- The hint "What can we do to reduce the size of a binary after compiling it?" is pointing directly at packers / UPX. The wording is the hint.
- The flag wordplay decode: **U9X_UnP4ck1N6_B1n4Ri3S** reads as "UPX Unpacking Binaries" in leet-speak — `9` for `P`, `4` for `a`, `1` for `i`, `6` for `g`, `3` for `e`. The challenge is literally telling you what the answer technique is.
- The trailing `bdd84893` looks like a fragment of a hash and is just an ID suffix. It does not decode to anything meaningful on its own.
- A packed binary can hide its behaviour, but it cannot hide data that is referenced from uncompressed code. If you unpack first, every reverse-engineering tool you know suddenly works again.
