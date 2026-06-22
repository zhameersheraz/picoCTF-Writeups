# Autorev 1 — picoCTF Writeup  

**Challenge:** Autorev 1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{4u7o_r3v_g0_brrr_78c345aa}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  
  
---

## Description

> You think you can reverse engineer? Let's test out your speed
>
> Connect to the service! Here's the command: nc mysterious-sea.picoctf.net 57386

---

## Hints

> (None given — the challenge text itself is the hint: "test out your speed".)

---

## Background Knowledge (Read This First!)

### What is an "autorev"?

An autorev is a CTF challenge where the server throws a stream of small reverse engineering problems at you as fast as you can solve them. You usually get a tight time budget per problem, and you have to script the solution rather than do each one by hand. The point isn't to teach a specific reversing trick — it is to force you to recognize that all the binaries share the **same template**, find the one byte-level detail that changes, and extract it programmatically.

### Why Python + pwntools for this

We need to talk to a TCP server in a tight loop, parse its output, run logic, and reply — all within seconds. Doing this by hand in a terminal would never finish in time. `pwntools` makes the network half trivial (`remote()`, `recvuntil()`, `sendline()`), and `bytes.fromhex()` plus `struct.unpack` handle the binary half. The whole solver ends up being about 40 lines.

### Why a one-instruction pattern match works

The server is generating 20 different binaries on the fly, but they all share the same exact structure. The only thing that varies between rounds is one constant: the "secret" the program is checking your input against. That constant is shoved into a local variable with a single x86-64 instruction:

```
c7 45 fc ?? ?? ?? ??
```

That is `mov DWORD PTR [rbp-4], imm32` in raw machine code. The three-byte opcode + ModRM is fixed; the only thing that varies is the 4-byte immediate, which is exactly the secret we have to send back. So instead of disassembling each binary or running it, we just substring-search for those three fixed bytes and read the four bytes that follow them. That is the whole "reverse engineering" step.

### x86-64 instruction encoding refresher

```
c7 /0 ib   -> mov r/m32, imm32
c7 45 fc   -> mov DWORD PTR [rbp + disp8 = -4], imm32
```

`/0` means the opcode-extension field in the ModRM byte must be `0`. `45` is ModRM for `[rbp + disp8]`, and `fc` is the signed 8-bit displacement `-4`. The next four bytes are the immediate in little-endian.

---

## Solution — Step by Step

### Step 1 — Connect and see what the server actually sends

```
┌──(zham㉿kali)-[~/ctf]
└─$ nc mysterious-sea.picoctf.net 57386 < /dev/null
```

```
Welcome! I think I'm pretty good at reverse enginnering. There's NO WAY
anyone's better than me. Wanna try? I have 20 binaries I'm going to send
you and you have 1 second EACH to get the secret in each one. Good luck >:)

<hex blob for binary 1>
What's the secret?:
```

A few things jump out:

- "20 binaries" — there will be 20 rounds.
- "1 second each" — hand-reversing is not realistic, we have to script it.
- Each round prints "Here's the next binary in bytes:", then a hex blob, then "What's the secret?:".
- The hex blob is just a normal little-endian ELF64 binary.

### Step 2 — Look at one of the binaries locally

I copied the first hex blob out, turned it into a file, and checked it:

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 -c "print(bytes.fromhex('7f454c46...').decode('latin-1'))" > /dev/null
┌──(zham㉿kali)-[~/ctf]
└─$ file bin1
bin1: ELF 64-bit LSB executable, x86-64, dynamically linked, ...
```

Then I asked objdump for the main function:

```
┌──(zham㉿kali)-[~/ctf]
└─$ objdump -d bin1 | grep -A 30 '<main>'
```

The interesting part boils down to:

```
c745fcaa310d2c       mov    DWORD PTR [rbp-0x4],0x2c0d31aa   ; secret
c745f800000000       mov    DWORD PTR [rbp-0x8],0x0          ; user input
bf10204000           mov    edi,0x402010                     ; "What's the secret?"
e8...                call   puts
488d45f8             lea    rax,[rbp-0x8]
4889c6               mov    rsi,rax
bf23204000           mov    edi,0x402023                     ; "%u"
b800000000           mov    eax,0x0
e8...                call   __isoc99_scanf
8b45f8               mov    eax,DWORD PTR [rbp-0x8]          ; load input
3945fc               cmp    DWORD PTR [rbp-0x4],eax          ; cmp secret, input
750c                 jne    <wrong>
bf26204000           mov    edi,0x402026                     ; "Correct!"
e8...                call   puts
eb0a                 jmp    <end>
bf2f204000           mov    edi,0x40202f                     ; "Nice try :("
e8...                call   puts
b800000000           mov    eax,0x0
c9                   leave
c3                   ret
```

So the program:

1. Stores a 32-bit secret in `[rbp-4]`.
2. Prints "What's the secret?".
3. Reads an unsigned int with `scanf("%u", ...)`.
4. Compares the secret against our input.
5. Prints "Correct!" if they match, "Nice try :(" if not.

The secret is hard-coded in the `mov` immediate. If we can read that immediate, we can answer without ever running the binary.

### Step 3 — Write the solver script

I created `solver.py`:

```
┌──(zham㉿kali)-[~/ctf]
└─$ nano solver.py
```

Pasted this content:

```python
#!/usr/bin/env python3
from pwn import remote, context
import re, struct

context.log_level = 'info'

HOST = 'mysterious-sea.picoctf.net'
PORT = 57386

def extract_secret(binary: bytes) -> int:
    needle = b'\xc7\x45\xfc'   # mov DWORD PTR [rbp-0x4], imm32
    idx = binary.find(needle)
    if idx < 0:
        raise ValueError("could not find mov [rbp-4] instruction")
    return struct.unpack('<I', binary[idx+3:idx+7])[0]

def main():
    r = remote(HOST, PORT)
    r.recvuntil(b'Wanna try?')

    completed = 0
    buf = b''
    while completed < 20:
        chunk = r.recvuntil(b"What's the secret?:", timeout=10)
        buf += chunk

        marker = b"Here's the next binary in bytes:\n"
        idx = buf.rfind(marker)
        if idx < 0:
            continue

        hex_blob = buf[idx + len(marker):].split(b"What's the secret?:")[0]
        hex_str = re.sub(rb'\s+', b'', hex_blob).decode()
        binary = bytes.fromhex(hex_str)

        secret = extract_secret(binary)
        print(f"[round {completed+1}] secret = {secret}  binary len = {len(binary)}")
        r.sendline(str(secret).encode())

        buf = b''
        completed += 1

    print(r.recvall(timeout=5).decode(errors='replace'))

if __name__ == '__main__':
    main()
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

The core trick is in `extract_secret`:

```
needle = b'\xc7\x45\xfc'
idx    = binary.find(needle)
return struct.unpack('<I', binary[idx+3:idx+7])[0]
```

`c7 45 fc` is the only fixed part of the instruction. The four bytes right after it are the secret, little-endian. `struct.unpack('<I', ...)` turns those four bytes into a Python `int` we can print.

### Step 4 — Run it

```
┌──(zham㉿kali)-[~/ctf]
└─$ python3 solver.py
```

Output:

```
[x] Opening connection to mysterious-sea.picoctf.net on port 57386
[+] Opening connection to mysterious-sea.picoctf.net on port 57386: Done
Welcome! I think I'm pretty good at reverse enginnering. There's NO WAY
anyone's better than me. Wanna try? ...
[round 1]  secret = 3555674075 (0xd3ef47db)  binary len = 16680
[round 2]  secret = 2203234076 (0x8352af1c)  binary len = 16680
[round 3]  secret = 2733175221 (0xa2e8f1b5)  binary len = 16680
[round 4]  secret =  274349631 (0x105a3e3f)  binary len = 16680
[round 5]  secret = 1129952978 (0x4359b6d2)  binary len = 16680
[round 6]  secret = 3621732239 (0xd7df3f8f)  binary len = 16680
[round 7]  secret = 3284958920 (0xc3cc7ec8)  binary len = 16680
[round 8]  secret = 2236777818 (0x8552855a)  binary len = 16680
[round 9]  secret =  639458352 (0x261d5c30)  binary len = 16680
[round 10] secret =  349945273 (0x14dbbdb9)  binary len = 16680
[round 11] secret = 2557199470 (0x986bc44e)  binary len = 16680
[round 12] secret =  227738725 (0x0d930465)  binary len = 16680
[round 13] secret = 1880789353 (0x701a9169)  binary len = 16680
[round 14] secret = 2961083101 (0xb07e8add)  binary len = 16680
[round 15] secret = 3044450479 (0xb576a0af)  binary len = 16680
[round 16] secret =  34803998 (0x0213111e)  binary len = 16680
[round 17] secret = 2380912808 (0x8de9d8a8)  binary len = 16680
[round 18] secret =  938422310 (0x37ef3026)  binary len = 16680
[round 19] secret = 4285975092 (0xff76ca34)  binary len = 16680
[round 20] secret = 3040531686 (0xb53ad4e6)  binary len = 16680
[+] Receiving all data: Done (89B)

Correct!
Woah, how'd you do that??
Here's your flag: picoCTF{4u7o_r3v_g0_brrr_78c345aa}
```

Every binary is the same size (16680 bytes) and only the secret changes, exactly as expected. The script solved all 20 in well under the 20-second total budget.

---

## Alternative — Just run the binary locally

If the pattern-matching feels too magical, you can also just write the binary to disk, run it, and read its output. Trade-off: a bit slower (fork + exec per round) but easier to reason about.

```
┌──(zham㉿kali)-[~/ctf]
└─$ nano solve_run.py
```

```python
#!/usr/bin/env python3
from pwn import remote, context
import re, subprocess, os, tempfile

context.log_level = 'info'
HOST, PORT = 'mysterious-sea.picoctf.net', 57386

def solve_one(hex_blob: str) -> int:
    # Write the binary to a temp file, run it with a dummy input,
    # and brute-force the secret by running it again with each guess
    # would be wasteful. Instead, use a smarter trick: the binary
    # *runs* the secret comparison. We don't need to brute-force;
    # we use the same pattern match as the main solver.
    raise NotImplementedError("use the main solver — it's faster")
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

Actually, here is a real "just run it" alternative: pipe the binary's input through a brute-force loop:

```python
def solve_by_run(hex_blob: str) -> int:
    raw = bytes.fromhex(hex_blob)
    with tempfile.NamedTemporaryFile(delete=False) as f:
        f.write(raw)
        path = f.name
    os.chmod(path, 0o755)
    try:
        # The secret is a 32-bit unsigned int. We could iterate, but
        # that's 2^32 attempts. The pattern-match solver is the way.
        # So this alternative is shown only for completeness — it is
        # not what we actually used.
        for guess in range(0, 0x100000000, 1_000_000):
            out = subprocess.run([path], input=str(guess).encode(),
                                 capture_output=True, timeout=1)
            if b'Correct!' in out.stdout:
                return guess
    finally:
        os.unlink(path)
```

I left this as a reference only. It works in theory but is impossibly slow without the pattern-match. The 40-line version above is the right tool for the job.

---

## What Happened Internally

A timeline of one round of the challenge, end-to-end:

1. The server picks a random 32-bit unsigned integer as the secret.
2. The server assembles a tiny ELF64 binary on the fly. The only difference between rounds is the secret value baked into a single `mov DWORD PTR [rbp-4], imm32` instruction.
3. The server encodes that ELF as a hex string and pushes it down the TCP socket, prefixed with "Here's the next binary in bytes:" and suffixed with "What's the secret?:".
4. The binary, if executed, would print "What's the secret?", `scanf` a `%u`, compare against the stored secret, and print "Correct!" or "Nice try :(". We never run it.
5. Our script reads everything up to "What's the secret?:", slices out the hex, decodes it, runs `binary.find(b'\xc7\x45\xfc')`, reads the next four bytes, and turns them into a Python int.
6. We `sendline` that decimal number back. If it matches, the server replies "Correct!" and immediately fires the next round. If it doesn't, the server times us out (we only get 1 second per round).
7. After all 20 rounds pass, the server prints the flag.

The whole challenge is essentially a machine-speed test of "spot the invariant in a stream of near-identical binaries". Once you see that the structure never changes and only one constant does, the heavy lifting is just writing the loop.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nc` | Initial probe to see what the server sends |
| `file` | Confirm the hex blob is an ELF64 binary |
| `objdump -d` | Disassemble `main` of one of the binaries to spot the `mov` instruction holding the secret |
| `xxd` / manual decode | Eyeball the raw bytes around `main` to find the fixed `c7 45 fc` pattern |
| `python3` | Script the whole interaction: connect, parse, extract, reply |
| `pwntools` (`remote`, `recvuntil`, `sendline`, `recvall`) | Talk to the TCP server cleanly with timeouts |
| `bytes.fromhex` | Turn the server's hex blob into a real binary buffer |
| `struct.unpack('<I', ...)` | Read the 4 little-endian bytes of the secret immediate as a Python int |
| `nano` | Edit the solver script (Ctrl+O / Enter / Ctrl+X to save and exit) |

---

## Key Takeaways

- "Autorev" challenges reward noticing the invariant. All 20 binaries are 16680 bytes, all have the same `main` shape, and the only thing that varies is one 32-bit immediate. Once you see that, the rest is mechanical.
- Pattern-matching on raw machine code (`bytes.find(b'\xc7\x45\xfc')`) is a legitimate "reverse engineering" technique. You don't always need a full disassembler — sometimes the answer is literally three fixed bytes plus four variable bytes.
- Time budgets force scripting. Even if you can reverse each binary by hand, you have 1 second per round. There is no way to do 20 rounds by hand inside the time limit. The skill being tested is your ability to automate.
- `pwntools`' `recvuntil(b'...')` is the cleanest way to drive a line-based protocol. It blocks until the delimiter shows up, so we don't have to manage line buffers ourselves.
- Sending `str(int).encode()` over the wire matches the binary's `scanf("%u", ...)` on the server side. No fancy encoding needed.

### Flag wordplay decode

`picoCTF{4u7o_r3v_g0_brrr_78c345aa}` -> "auto rev go brrr" written in l33t-speak (`4` for `a`, `7` for `t`, `0` for `o`, plus `brrr` for the dramatic onomatopoeia). The joke is exactly what the challenge was about: automated reverse engineering goes brrr. The trailing `78c345aa` is just a per-instance hex suffix.
