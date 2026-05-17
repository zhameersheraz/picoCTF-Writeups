# Secure Password Database — picoCTF Writeup

**Challenge:** Secure Password Database  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Flag:** `picoCTF{d0nt_trust_us3rs}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  

---

## Description

> I made a new password authentication program that even shows you the password you entered saved in the database! Isn't that cool?
> Download: `system.out`
> `$ nc candy-mountain.picoctf.net 60138`

**Hint 1:** `How does the hashing algorithm work?`

---

## Background Knowledge (Read This First!)

### What is the Heartbleed bug?

**Heartbleed** (CVE-2014-0160) was a famous real-world vulnerability in OpenSSL. The server would allocate a buffer for a "heartbeat" message and return as many bytes as the **user claimed** the message was — without checking if that matched the actual message length. If you sent 1 byte but claimed it was 64KB, the server would return 64KB of raw memory — leaking secrets, private keys, and other sensitive data.

This challenge recreates that exact vulnerability in a custom password program.

### What is `system.out`?

`system.out` is a compiled **ELF shared library** (`.so`) — a Linux binary. Running `exiftool` confirms this. We analyse it with `strings`, `objdump`, and `readelf` to understand how it works.

### What is the djb2 hash?

Looking at the disassembly of the `hash` function:

```asm
movq $0x1505, -0x8(%rbp)   ; h = 0x1505 (initial value)
; loop for each character c:
shl  $0x5, %rax            ; rax = h * 32
add  %rax, %rdx            ; rdx = h * 32 + h = h * 33
add  %rdx, %rax            ; rax = h * 33 + c
```

This is the classic **djb2** hash:
```
h = 0x1505
for each character c:
    h = h * 33 + c
```

### What is the secret?

The binary contains an `obf_bytes` array at offset `0x2008` in the `.rodata` section:
```
c3 ff c8 c2 92 9b 8b c0 80 c2 c4 8b
```

The `make_secret` function XORs each byte with `0xAA` to produce the real secret:
```python
secret = bytes([b ^ 0xaa for b in obf_bytes])
# → b'iUbh81!j*hn!'
```

---

## The Vulnerability

When you enter a password, the program asks:
```
How many bytes in length is your password?
```

It then allocates a buffer, copies your password in, and **prints exactly as many bytes as you claimed** — not as many as you actually typed. If you claim more bytes than your password length, it reads beyond your buffer into adjacent memory.

The secret `iUbh81!j*hn!` is stored right next to the password buffer in memory. By claiming a large length, we force the program to print the secret bytes — leaking it.

---

## Solution — Step by Step

### Step 1 — Analyse the binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool system.out
File Type : ELF shared library
CPU Architecture : 64 bit
```

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings system.out
heartbleed.c          ← filename tells us everything
Please set a password for your account:
How many bytes in length is your password?
Your successfully stored password:
Enter your hash to access your account!
flag.txt
```

The source filename `heartbleed.c` confirms the vulnerability type immediately.

### Step 2 — Extract the secret and compute the hash

```python
# obf_bytes extracted from binary at offset 0x2008
obf_bytes = bytes([0xc3,0xff,0xc8,0xc2,0x92,0x9b,0x8b,0xc0,0x80,0xc2,0xc4,0x8b])

# XOR each byte with 0xAA (from make_secret function)
secret = bytes([b ^ 0xaa for b in obf_bytes])
print(secret)   # b'iUbh81!j*hn!'

# Compute djb2 hash (from hash function disassembly)
h = 0x1505
for c in secret:
    h = (h * 33 + c) & 0xFFFFFFFFFFFFFFFF
print(h)   # 15237662580160011234
```

### Step 3 — Trigger the Heartbleed overflow

Connect and enter a 1-character password, but claim it is 50 bytes long:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nc candy-mountain.picoctf.net 60138
Please set a password for your account:
a
How many bytes in length is your password?
50
You entered: 50
Your successfully stored password:
97 10 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
Enter your hash to access your account!
15237662580160011234
picoCTF{d0nt_trust_us3rs}
```

- `97` = ASCII `a` (our password)
- `10` = newline
- Everything after = leaked memory (zeroed in this run, but the secret would appear in different memory layouts)

The hash `15237662580160011234` matches the pre-computed djb2 hash of the secret → flag is printed. ✅ Got the flag! 🎯

---

## Why the Heartbleed Approach Worked

Even though the leaked bytes were all zeros in this run, **we didn't need the leaked bytes** — we extracted the secret directly from the binary by reverse engineering `obf_bytes` and `make_secret`. The Heartbleed vulnerability is the intended mechanic to find the secret at runtime without reverse engineering, but static analysis is faster here.

Both approaches lead to the same hash value: **15237662580160011234**

---

## Alternative Method — Runtime Heartbleed (without RE)

If you didn't want to reverse the binary, you could trigger the overflow with the right memory layout to actually see the secret bytes leak:

```
Password: a
Length: 100
```

The output would include the bytes of `iUbh81!j*hn!` after your password bytes, which you then decode and hash manually.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `exiftool` | Identify the binary type | ⭐ Easy |
| `strings` | Find `heartbleed.c` filename and program strings | ⭐ Easy |
| `objdump -d` | Disassemble the `hash` and `make_secret` functions | ⭐⭐ Medium |
| `readelf -S` | Find section offsets to locate `obf_bytes` | ⭐⭐ Medium |
| Python | Extract secret and compute djb2 hash | ⭐⭐ Medium |
| `nc` | Connect and exploit the heartbleed overflow | ⭐ Easy |

---

## Key Takeaways

- **Never trust user-supplied lengths** — always validate that claimed sizes match actual data sizes. This is the root cause of Heartbleed and this challenge
- **Source filenames in binaries** reveal the programmer's intent — `heartbleed.c` immediately told us the vulnerability type
- **djb2** (`h = h*33 + c` starting at `0x1505`) is a recognisable hash pattern in disassembly — the `shl $0x5` + add idiom means multiply by 33
- **XOR obfuscation with a constant** (here `0xAA`) is trivially reversible — not real encryption
- The flag `d0nt_trust_us3rs` → "don't trust users" — never let users control how many bytes you read from their own input
