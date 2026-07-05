# unpackme — picoCTF Writeup

**Challenge:** unpackme  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{up><_m3_f7w_77ad107e}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Can you get the flag?
>
> Reverse engineer this `binary`.

## Hints

**Hint 1:** What is UPX?

---

## Background Knowledge

A few ideas before we touch the binary.

**1. UPX (Ultimate Packer for eXecutables).**
UPX is a free, open-source executable packer. It compresses an ELF, PE or Mach-O file and wraps it in a tiny decompressor stub. When you launch the packed binary, the stub inflates the original program in memory and jumps into it. From the outside, the file is smaller, has fewer sections, and shows `no section header` in `file` output — all useful telltales.

**2. `file` and `upx -l`.**
Two quick sanity checks. `file` will say `statically linked, no section header` for an UPX-packed ELF. `upx -l binary` lists the original size, packed size, and the format (`linux/amd64`). Both confirm we are dealing with a UPX-packed binary before we touch a disassembler.

**3. Unpacking with `upx -d`.**
`upx -d binary` writes the unpacked ELF alongside the original (it keeps a `.packed` copy if you ask, otherwise it overwrites). After unpacking, the binary is a normal ELF with full symbols and section headers — perfect for `objdump` / `gdb` / `radare2` / `Ghidra`.

**4. Caesar-style rotation on printable ASCII.**
The challenge's inner function rotates every printable character by a fixed amount, wrapping inside the printable range. The printable ASCII band runs from `0x21` (`!`) to `0x7e` (`~`) — exactly 94 characters. Adding `0x2f` (47) and wrapping modulo 94 is a textbook Caesar on that band. We don't actually need to crack it: running the unpacked binary with the right input prints the flag for us.

---

## Step-by-step Solution

I worked this out on my Kali VM (VirtualBox). The downloaded challenge file is just `unpackme` — no extension.

### 1. Inspect the file

```
┌──(zham㉿kali)-[~/unpackme]
└─$ mkdir -p ~/unpackme && cd ~/unpackme

┌──(zham㉿kali)-[~/unpackme]
└─$ cp ~/Downloads/unpackme .

┌──(zham㉿kali)-[~/unpackme]
└─$ chmod +x unpackme

┌──(zham㉿kali)-[~/unpackme]
└─$ file unpackme
unpackme: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, no section header
```

The "no section header" line is the first hint something is off. UPX strips section headers from the packed file.

### 2. Confirm it is UPX-packed

```
┌──(zham㉿kali)-[~/unpackme]
└─$ upx -l unpackme
```

```
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.2       Markus Oberhumer, Laszlo Molnar & John Reiser    Jan 3rd 2024

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
   1002528 ->    379188   37.82%   linux/amd64   unpackme
```

So the original ELF was `1002528` bytes and UPX shrunk it to `379188` (about 38%). Definitely UPX.

### 3. Unpack it

`upx -d` writes the unpacked ELF back over the file (it leaves a backup of the packed version next to it if you pass `-k`, but I just overwrote).

```
┌──(zham㉿kali)-[~/unpackme]
└─$ cp unpackme unpackme.packed

┌──(zham㉿kali)-[~/unpackme]
└─$ upx -d unpackme
```

```
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2024
UPX 4.2.2       Markus Oberhumer, Laszlo Molnar & John Reiser    Jan 3rd 2024

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
   1006445 <-    379188   37.68%   linux/amd64   unpackme

Unpacked 1 file.
```

```
┌──(zham㉿kali)-[~/unpackme]
└─$ file unpackme
unpackme: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 3.2.0
```

Section headers are back. The binary is now friendly to `objdump` and `gdb`.

### 4. Try running the unpacked binary

```
┌──(zham㉿kali)-[~/unpackme]
└─$ echo "" | ./unpackme
What's my favorite number? Sorry, that's not it!
```

It reads a number from stdin and compares it to a hard-coded value. We need to find that value.

### 5. Find the magic number with `objdump`

I disassembled `main` and looked for the comparison instruction:

```
┌──(zham㉿kali)-[~/unpackme]
└─$ objdump -d unpackme | sed -n '/<main>:/,/<get_common/p'
```

The interesting block:

```
  401ea8:	call   410ba0 <_IO_printf>     # printf("What's my favorite number? ")
  401ec0:	call   410d30 <__isoc99_scanf>  # scanf("%d", &input)
  401ec5:	mov    -0x3c(%rbp),%eax
  401ec8:	cmp    $0xb83cb,%eax            # if (input == 0xb83cb)
  401ecd:	jne    401f12 <main+0xcf>       #   else "Sorry, that's not it!"
  401ecf:	lea    -0x30(%rbp),%rax
  401edb:	call   401d85 <rotate_encrypt>  #   rotate_encrypt(0, &encoded_flag)
  401ef5:	call   420980 <_IO_fputs>       #   puts(rotated)
```

`0xb83cb` decimal is **754635**. That is our magic number.

### 6. Run the binary with the magic number

```
┌──(zham㉿kali)-[~/unpackme]
└─$ echo "754635" | ./unpackme
What's my favorite number? picoCTF{up><_m3_f7w_77ad107e}
```

There it is: `picoCTF{up><_m3_f7w_77ad107e}`.

---

## Alternative Solves

**A. GDB instead of `objdump`.**
If you prefer a live debugger, set a breakpoint after `scanf`, then run with any input and inspect the comparison register:

```
┌──(zham㉿kali)-[~/unpackme]
└─$ gdb -q ./unpackme
(gdb) break *0x401ec8
(gdb) run
What's my favorite number? 1
Breakpoint 1, 0x0000000000401ec8 in main ()
(gdb) info registers rax
rax            0x1                 1
(gdb) print/d 0xb83cb
$1 = 754635
(gdb) quit
```

Then feed `754635` to the actual binary.

**B. Decrypt the encoded bytes without running the binary.**
The encoded flag lives at `-0x30(%rbp)` in `main`, pushed via three `movabs` writes. The `rotate_encrypt` function does `new = c + 0x2f` and wraps with `new -= 0x5e` whenever `new > 0x7e`. I dropped the bytes into a tiny Python helper and applied the same rotation:

```
┌──(zham㉿kali)-[~/unpackme]
└─$ nano decode.py
```

```python
encoded = bytes([
    0x41, 0x3a, 0x34, 0x40, 0x72, 0x25, 0x75, 0x4c,
    0x46, 0x41, 0x6d, 0x6b, 0x30, 0x3e, 0x62, 0x30,
    0x37, 0x66, 0x48, 0x30, 0x66, 0x66, 0x32, 0x35,
    0x60, 0x5f, 0x66, 0x36, 0x4e,
])

out = []
for c in encoded:
    if 0x20 < c < 0x7f:
        c = c + 0x2f
        if c > 0x7e:
            c -= 0x5e
    out.append(c)
print(bytes(out).decode())
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`, then run:

```
┌──(zham㉿kali)-[~/unpackme]
└─$ python3 decode.py
picoCTF{up><_m3_f7w_77ad107e}
```

Same flag, no need to launch the binary at all.

**C. Brute-force the magic number.**
If you really wanted to brute-force, you could script the pipe and walk from `0` upward — but with no upper bound hint, that's `O(2^31)` work. Disassembling is faster and teaches you more about the binary.

---

## What Happened Internally

1. `main` allocated a 0x50-byte stack frame and reserved room for the encoded flag at `-0x30(%rbp)`.
2. Three `movabs` instructions (and two smaller stores) wrote the encoded flag bytes onto the stack. In little-endian order, those bytes spell out `A:4@r%uLFAmk0>b07fH0ff25`_6N`.
3. `printf("What's my favorite number? ")` printed the prompt.
4. `scanf("%d", &n)` read an integer into `-0x3c(%rbp)`.
5. The `cmp $0xb83cb` instruction checked `n` against `754635`. If not equal, control jumped to the `Sorry, that's not it!` branch.
6. If equal, `rotate_encrypt(0, &encoded)` ran. Inside, the encoded bytes were `strdup`'d, and for every byte `c` with `0x20 < c < 0x7f`, the function computed `c + 0x2f`, wrapping by `-0x5e` whenever the result exceeded `0x7e`. The result was a clean ASCII string.
7. `fputs(result, stdout)` printed the flag, then `free()` released the `strdup`'d buffer.

The UPX packing was just a packaging layer — once `upx -d` removed it, the binary behaved like a normal statically-linked ELF and the disassembly was straightforward.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `file`           | Quick check on the binary's format and section header status. |
| `chmod`          | Made the downloaded binary executable.                        |
| UPX 4.2.2        | Listed packed metadata and unpacked the binary.                |
| `objdump -d`     | Disassembled `main` and `rotate_encrypt`.                      |
| `gdb`            | Optional alternative — breakpoint right after `scanf` to read `rax`. |
| `python3`        | Re-implemented the rotation in a helper script (`decode.py`). |
| `echo + pipe`    | Fed the magic number `754635` into the binary non-interactively. |
| `nano`           | Wrote the small `decode.py` helper.                           |

---

## Key Takeaways

- A statically-linked ELF with `no section header` and a file-size drop of ~60% is almost always a UPX-packed binary. `upx -l` confirms it in one command.
- `cmp $imm, %reg` (or `cmp imm, reg` in AT&T syntax) is your best friend in `objdump`. Any time you see an integer comparison right before a jump, that integer is a candidate key.
- When a binary reads user input and compares it to a constant, you have three options: read the constant from the disassembly, brute-force (slow), or patch the jump with a debugger. Disassembly wins.
- Caesar-style rotations on printable ASCII are everywhere in beginner RE. The shift is usually `add 0x??` and the wrap is `sub 0x5e` (94, the printable range width). Recognize the pattern and you can decode any of them.
- The flag's leet: `up><_m3_f7w_77ad107e` reads as **up_and_me_few_trade_a_lot** in spirit — `up><` as arrows pointing at each other, `_m3_` = "me", `f7w` = "few", `77ad107e` = "77ad lo te" / "trad-lot-e" using `7=t`, `1=l`, `0=o`. So the wordplay is roughly "up and me, a few trades lots" — playful, not a clean sentence.
