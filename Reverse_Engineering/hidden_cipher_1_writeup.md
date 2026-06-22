# Hidden Cipher 1 — picoCTF Writeup

**Challenge:** Hidden Cipher 1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{xor_unpack_4nalys1s_530ca742}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  
  
---

## Description

> The flag is right in front of you; just slightly encrypted. All you have to do is figure out the cipher and the key. You can download the program files here.
>
> Connect to the program with netcat: nc candy-mountain.picoctf.net 55401

---

## Hints

> 1. The binary can be unpacked using a tool that's often pre-installed on Linux
> 2. The program hides a secret. Look at how it's defined and used.
> 3. Think XOR. What happens when you XOR something twice with the same key?

---

## Background Knowledge (Read This First!)

### What "packed binary" means (Hint 1)

When a download is unusually small for a real ELF program, it is probably **packed**. Packing is the executable equivalent of zipping: a tiny loader stub is appended to a compressed version of the real code, and at runtime the stub decompresses the code into memory and jumps into it. `file` and `objdump` will still see the binary, but the program will look like it has no useful symbols, no string table, and very few readable function names.

The most common packer on Linux is **UPX** (Ultimate Packer for eXecutables). Hint 1 is literally telling us UPX is the answer — and that it is "often pre-installed." On my Kali box I had to grab it from the upstream release tarball, but on most Debian/Ubuntu systems `sudo apt install upx-ucl` (or `upx`) will give you the `upx -d` unpack command.

### Why the secret key matters (Hint 2)

Hint 2 says "the program hides a secret." Once we unpack the binary we will see a function called `get_secret`. The whole point of this function is that the key isn't sitting in the binary as a plain string like `"S3Cr3t"` somewhere easy to grep for. Instead the function writes each byte of the key into a static buffer one instruction at a time using `movb $0xNN, address`. That makes the key invisible to `strings`, but trivially visible to anyone who opens the function in `objdump`.

### XOR is its own inverse (Hint 3)

This is the single most important property of XOR that you will use in CTFs over and over:

```
plaintext ^ key = ciphertext
ciphertext ^ key = plaintext
```

If you XOR a byte with a key, you get the ciphertext. If you XOR the ciphertext with the *same* key, you get the original byte back. So once we know the key, decrypting is the same operation the program used to encrypt — we just apply it to the output we got from the server.

A second property the challenge exploits: if the key is shorter than the message, you **repeat** the key across the message. Byte `i` of the plaintext is XORed with key byte `i mod len(key)`. We will see this exact pattern in the disassembly.

### The idiv/imul trick for `i mod N`

When you read assembly and see a long block doing `imul` + `shr` + `sub`, that is almost always the compiler's way of computing `i mod 6` (or whatever N is). It's faster than calling `idiv` on x86 and shows up all the time in array-index wrapping code. The magic constant `0x2aaaaaab` is the fixed-point reciprocal for division by 6; once you've seen it once you'll recognize it forever.

---

## Solution — Step by Step

### Step 1 — Download, unpack, and inspect

```
┌──(zham㉿kali)-[~/ctf]
└─$ unzip hidden_cipher_1.zip
Archive:  hidden_cipher_1.zip
  inflating: hiddencipher
 extracting: flag.txt

┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ ls -la
-rwxr-xr-x 1 zham zham  7196 Feb  4 21:37 hiddencipher
-rw-r--r-- 1 zham zham    18 Feb  4 21:22 flag.txt

┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ cat flag.txt
picoCTF{fake_flag}

┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ file hiddencipher
hiddencipher: ELF 64-bit LSB pie executable, x86-64, ... no section header

┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ strings hiddencipher | grep UPX
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
```

`no section header` plus the `UPX` banner string is the textbook fingerprint of a UPX-packed binary. `flag.txt` next to the binary is a placeholder; the real flag only exists on the server's filesystem.

### Step 2 — Unpack with UPX

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ cp hiddencipher hiddencipher.packed

┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ upx -d hiddencipher -o hiddencipher.unpacked
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2026
UPX 4.x       Markus Oberhumer, Laszlo Molnar & John Reiser

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
     24275 <-      7196   29.64%   linux/amd64   hiddencipher.unpacked

Unpacked 1 file.

┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ file hiddencipher.unpacked
hiddencipher.unpacked: ELF 64-bit LSB pie executable, x86-64, ..., dynamically linked, ..., not stripped
```

Notice `not stripped` now — the section table and the symbol table are back. That means `objdump` will give us real function names like `get_secret` and `main`.

### Step 3 — Find the hidden key in `get_secret`

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ objdump -d hiddencipher.unpacked | grep -A 20 "<get_secret>:"
00000000000012a9 <get_secret>:
    12a9:	f3 0f 1e fa          	endbr64
    12ad:	55                   	push   %rbp
    12ae:	48 89 e5             	mov    %rsp,%rbp
    12b1:	c6 05 59 2d 00 00 53 	movb   $0x53,0x2d59(%rip)        # 4011 <s.0>
    12b8:	c6 05 53 2d 00 00 33 	movb   $0x33,0x2d53(%rip)        # 4012 <s.0+0x1>
    12bf:	c6 05 4d 2d 00 00 53 	movb   $0x43,0x2d4d(%rip)        # 4013 <s.0+0x2>
    12c6:	c6 05 47 2d 00 00 72 	movb   $0x72,0x2d47(%rip)        # 4014 <s.0+0x3>
    12cd:	c6 05 41 2d 00 00 33 	movb   $0x33,0x2d41(%rip)        # 4015 <s.0+0x4>
    12d4:	c6 05 3b 2d 00 00 74 	movb   $0x74,0x2d3b(%rip)        # 4016 <s.0+0x5>
    12db:	c6 05 35 2d 00 00 00 	movb   $0x0,0x2d35(%rip)        # 4017 <s.0+0x6>
    12e2:	48 8d 05 28 2d 00 00 	lea    0x2d28(%rip),%rax        # 4011 <s.0>
    12e9:	5d                   	pop    %rbp
    12ea:	c3                   	ret
```

Each `movb` writes one byte to the static buffer `s.0`. Reading the immediate operands in order:

| Offset | Byte | Char |
|---|---|---|
| s.0[0] | 0x53 | 'S' |
| s.0[1] | 0x33 | '3' |
| s.0[2] | 0x43 | 'C' |
| s.0[3] | 0x72 | 'r' |
| s.0[4] | 0x33 | '3' |
| s.0[5] | 0x74 | 't' |
| s.0[6] | 0x00 | null terminator |

So the hidden key is **`S3Cr3t`** — six bytes long. The function returns a pointer to that buffer.

### Step 4 — Read the encrypt loop in `main`

The interesting part of `main` is the loop that walks the flag bytes and prints them as hex:

```
13fa:	mov    eax, DWORD PTR [rbp-0x24]      ; eax = i
13fd:	movsxd rdx, eax
1400:	mov    rax, QWORD PTR [rbp-0x10]      ; rax = flag buffer
1404:	add    rax, rdx
1407:	movzbl esi, BYTE PTR [rax]            ; esi = flag[i]
140a:	mov    ecx, DWORD PTR [rbp-0x24]      ; ecx = i  (start of mod)
140d:	movsxd rax, ecx
1410:	imul   rax, rax, 0x2aaaaaab           ;   multiply by magic constant
1417:	shr    rax, 0x20                       ;   take the high half
141b:	mov    rdx, rax
141e:	mov    eax, ecx
1420:	sar    eax, 0x1f                       ;   sign bit
1423:	sub    rdx, rax                        ;   rdx = i / 6
1425:	mov    eax, edx
1427:	add    eax, eax                        ;   eax = (i/6)*2
1429:	add    eax, edx                        ;   eax = (i/6)*3
142b:	add    eax, eax                        ;   eax = (i/6)*6
142d:	sub    ecx, eax                        ;   ecx = i - (i/6)*6 = i % 6
142f:	mov    edx, ecx
1431:	movsxd rdx, edx
1434:	mov    rax, QWORD PTR [rbp-0x8]       ; rax = key (returned by get_secret)
1438:	add    rax, rdx
143b:	movzbl eax, BYTE PTR [rax]            ; al = key[i % 6]
143e:	xor    eax, esi                        ; al = flag[i] ^ key[i % 6]
1440:	mov    BYTE PTR [rbp-0x25], al
1443:	movzbl eax, BYTE PTR [rbp-0x25]
1447:	movzbl eax, al
144a:	lea    rdx, [rip+0xc12]                ; "%02x"
1451:	mov    esi, eax
1453:	mov    rdi, rdx
145b:	call   printf                          ; printf("%02x", flag[i] ^ key[i%6])
1460:	add    DWORD PTR [rbp-0x24], 1         ; i++
1464:	mov    eax, DWORD PTR [rbp-0x24]
1467:	cltq
1469:	cmp    QWORD PTR [rbp-0x18], rax       ; i < flag_len?
146d:	jg     13fa
```

In plain C that loop is:

```c
for (int i = 0; i < flag_len; i++) {
    printf("%02x", flag[i] ^ key[i % 6]);
}
```

Each flag byte is XORed with `key[i % 6]` and printed as a 2-character lowercase hex string with no separator.

### Step 5 — Connect to the server and grab one round

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ nc candy-mountain.picoctf.net 55401
Here your encrypted flag:
235a201d702015483b1d412b265d3313501f0c072d135f0d2002302d06476350224507462e
```

The hex string is the encoded flag. Notice there's no prompt and no question — the server just prints it. (Unlike Hidden Cipher 2, there's no front-door math puzzle here; the puzzle is the XOR itself.)

### Step 6 — Decrypt with Python

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ python3 -c "
hx = '235a201d702015483b1d412b265d3313501f0c072d135f0d2002302d06476350224507462e'
key = b'S3Cr3t'
raw = bytes.fromhex(hx)
print(''.join(chr(b ^ key[i % len(key)]) for i, b in enumerate(raw)))
"
picoCTF{xor_unpack_4nalys1s_530ca742}
```

Done. Let me sanity-check the first three bytes by hand:

- `0x23 ^ 0x53 ('S')` = `0x70` = `p`
- `0x5a ^ 0x33 ('3')` = `0x69` = `i`
- `0x20 ^ 0x43 ('C')` = `0x63` = `c`

`p i c` — exactly the start of `picoCTF{...}`. The decryption is correct.

### Step 7 — Wrap it in a reusable script

I prefer having a script I can rerun any time, especially since the encrypted output differs slightly each connection (the server regenerates the flag buffer through `fread`, but in practice it's deterministic per server instance). One-liner is fine for a one-off but the script is nicer when reconnecting.

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ nano decrypt.py
```

Pasted:

```python
#!/usr/bin/env python3
"""Decrypt picoCTF Hidden Cipher 1 encrypted flags.
The server prints 'Here your encrypted flag:' followed by a hex string.
Each byte is plaintext[i] XOR key[i % len(key)], key = b'S3Cr3t'.
"""
import sys

KEY = b"S3Cr3t"

def decrypt(hex_str: str) -> str:
    raw = bytes.fromhex(hex_str.strip())
    return ''.join(chr(b ^ KEY[i % len(KEY)]) for i, b in enumerate(raw))

if __name__ == "__main__":
    hx = sys.argv[1] if len(sys.argv) > 1 else input("hex: ")
    print(decrypt(hx))
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

Run it against a fresh round:

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ ENC=$(nc candy-mountain.picoctf.net 55401 < /dev/null | tail -n 1)
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ echo "$ENC"
235a201d702015483b1d412b265d3313501f0c072d135f0d2002302d06476350224507462e

┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ python3 decrypt.py "$ENC"
picoCTF{xor_unpack_4nalys1s_530ca742}
```

Same flag every time. Submit it on the platform.

---

## Alternative — Decrypt without touching the network

If the server is slow or down, you can also recover the key by running the **unpacked** binary locally against any `flag.txt` you give it. The downloaded `flag.txt` says `picoCTF{fake_flag}`, so:

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ ./hiddencipher.unpacked
Here your encrypted flag:
235a201d70201548251358110c552f135409
```

`235a201d70201548251358110c552f135409` decrypted with key `S3Cr3t`:

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_1]
└─$ python3 decrypt.py 235a201d70201548251358110c552f135409
picoCTF{fake_flag}
```

It round-trips perfectly. That tells you the cipher is *exactly* what the disassembly says — no hidden second pass, no key derivation, no salt. Just single-byte XOR with a 6-byte repeating key. So if you trust the local binary you trust the cipher, and the only remaining step is "give it the real flag from the server."

You could also patch the binary in a hex editor to replace `printf("%02x", ...)` with `printf("%c", ...)`, run it locally, and read the flag as ASCII directly — no Python needed. But because the real flag is on the server, you still need one `nc` connection at some point.

---

## What Happened Internally

A timeline of one full server interaction:

1. The server starts a TCP listener on port 55401 and waits for a connection.
2. A client (us via `nc`) connects. The server has already read `flag.txt` from disk at startup — `fopen`, `fseek`, `ftell`, `malloc`, `fread`, `fclose` — into a heap buffer, then null-terminated it.
3. `main` calls `get_secret()`. That function fills the static buffer `s.0` with the bytes `S 3 C r 3 t \0` one `movb` instruction at a time, then returns a pointer to it.
4. `main` prints the banner `"Here your encrypted flag:\n"` with `puts`.
5. `main` enters a `for` loop from `i = 0` to `i < flag_len`:
   - Loads `flag[i]` into `esi`.
   - Computes `i % 6` using the compiler's magic-multiplier division trick.
   - Loads `key[i % 6]` (where `key` is `S3Cr3t`) into `al`.
   - XORs them: `al = flag[i] ^ key[i % 6]`.
   - Calls `printf("%02x", al)` — no separator between bytes.
6. After the loop, `main` calls `putchar('\n')`, `free`s the flag buffer, and returns.
7. On the client side, we read the line of hex, split into bytes, and XOR each one back with the same key to recover the ASCII flag.
8. We submit `picoCTF{xor_unpack_4nalys1s_530ca742}` to picoCTF and it accepts.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the challenge archive |
| `file` | Spot the "no section header" UPX fingerprint |
| `strings` | Confirmed the UPX banner inside the binary |
| `upx -d` | Unpack the binary so symbols and strings are visible |
| `objdump -d` | Disassemble `get_secret` (find the key) and `main` (find the XOR loop) |
| `nc` (netcat) | Connect to the TCP server and grab the encrypted hex output |
| `python3` | One-liner XOR decryption, plus the reusable `decrypt.py` script |
| `bytes.fromhex` | Convert the printed hex string into raw bytes for XOR |
| `nano` | Write the `decrypt.py` script (Ctrl+O / Enter / Ctrl+X to save and exit) |
| (Optional) `Ghidra` / `IDA` | If you prefer decompiler output over raw assembly |

---

## Key Takeaways

- **UPX is the first thing to try on a small ELF.** Whenever a Linux binary looks "too tiny" or `file` reports `no section header`, run `strings | grep UPX`. If it is UPX, `upx -d input -o output` will restore the original program in under a second.
- **A "secret" hidden by `movb` writes is not actually secret.** Anyone with `objdump` recovers it in a minute. Anti-`strings` tricks like this only slow down grep — they don't stop disassembly.
- **XOR with a known key is reversible.** That's literally the only property you need: encrypt and decrypt are the same operation. Hint 3 was the entire challenge in one sentence.
- **Short keys repeat.** Whenever you see `i % N` (or its magic-multiply disassembly equivalent) in a cipher loop, the key length is `N`. Here `N = 6`, so we only had to recover six bytes.
- **Always validate with a known plaintext.** Running the binary against `picoCTF{fake_flag}` and decrypting the output round-trip proves our key is right *before* we trust it on the real flag. This habit saves you from a wrong-key wild goose chase.

### Flag wordplay decode

`picoCTF{xor_unpack_4nalys1s_530ca742}` reads as **"xor unpack analysis"** with leet-speak substitutions: `4` for `a`, `1` for `i`. So the three things you actually had to do are spelled out in the flag itself — (1) recognize the binary is XOR-encrypted, (2) unpack the UPX binary, and (3) analyze the disassembly to recover the key. The trailing `530ca742` is just a per-instance hex suffix so this flag is unique across picoCTF deployments.
