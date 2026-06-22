# Hidden Cipher 2 — picoCTF Writeup

**Challenge:** Hidden Cipher 2  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{m4th_b3h1nd_c1ph3r_f92f803e}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  
  
---

## Description

> The flag is right in front of you… kind of. You just need to solve a basic math problem to see it. But to get the real flag, you'll have to understand how that math answer is used. You can download the program files here.
>
> Connect to the program with netcat: nc crystal-peak.picoctf.net 58827

---

## Hints

> 1. Focus on what the program does with your correct answer. Is it reused later?
> 2. Disassembling or decompiling tools (Ghidra, IDA) can help reveal the exact transformation on the flag.

---

## Background Knowledge (Read This First!)

### What the program does at a high level

The binary asks us a single random math question (`What is A + B?`, `A - B?`, or `A * B?`), waits for the answer, and then — if we got it right — prints the flag's contents as a sequence of integers. The trick is that those integers are **not** the ASCII codes of the flag. They have been multiplied by the math answer first. We have to undo that multiplication to recover the flag.

### Why the math answer is "reused later"

Hint 1 is dead-on: the answer isn't just a gatekeeper. After the program verifies our input equals the right answer, it calls `encode_flag(flag_buf, answer)` and `answer` is passed in as the multiplier. Look at the disassembly below — the function signature is `void encode_flag(char *flag, int multiplier)`, and inside the loop it does `printf("%d", flag[i] * multiplier)`. So the same number we typed as our math answer is the key to decoding the flag.

### The x86-64 instruction we care about

The encode loop boils down to one assembly idiom:

```
movzx  eax, BYTE PTR [flag + i]
movsx  eax, al                  ; sign-extend byte to int
imul   eax, DWORD PTR [rbp-0x1c]  ; eax *= multiplier
```

`imul reg, mem` multiplies. The argument is the user-supplied math answer. Each byte of the flag is multiplied by that value, then printed as `%d`. To recover the byte we just do `byte_value = encoded_value / multiplier` (the division is exact because multiplication was lossless for ASCII range).

### Why the flag survives the multiplication

ASCII printable characters are all in the 32..126 range. The multiplier is at most `9 * 9 = 81` (max from `rand()%10 + 1` and `rand()%10`). So `126 * 81 = 10,206` — well within int range and small enough that no information is lost. The reverse is therefore exact integer division.

---

## Solution — Step by Step

### Step 1 — Unpack and inspect the files

```
┌──(zham㉿kali)-[~/ctf]
└─$ unzip hidden_cipher_2.zip
Archive:  hidden_cipher_2.zip
  inflating: hiddencipher2
 extracting: flag.txt

┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ ls -la
-rwxr-xr-x 1 zham zham 16920 Feb  4 22:28 hiddencipher2
-rw-r--r-- 1 zham zham    18 Feb  4 21:22 flag.txt

┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ cat flag.txt
picoCTF{fake_flag}

┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ file hiddencipher2
hiddencipher2: ELF 64-bit LSB pie executable, x86-64, ... not stripped
```

`flag.txt` next to the binary is just a stub; the real flag lives only on the server's filesystem. Good thing the binary reads it from disk at runtime — which means the server's `flag.txt` will contain the real flag.

### Step 2 — Disassemble main and the two helper functions

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ objdump -d -M intel hiddencipher2 > disasm.txt
```

The four interesting functions the binary exports:

```
0000000000001369 <generate_math_question>
000000000000149c <encode_flag>
0000000000001544 <read_flag_file>
0000000000001636 <main>
```

`main` does the high-level orchestration. Let me look at it:

```
1674: call   1369 <generate_math_question>   ; returns the correct answer
...
1697: call   printf                          ; "What is A op B?"
16c1: call   scanf                           ; our answer
16e4: cmp    [rbp-0x14], eax                 ; compare to correct answer
16e7: je     16ff                            ; jump if correct
...
1709: call   1544 <read_flag_file>           ; if correct, read flag.txt
172c: call   149c <encode_flag>              ; encode it
```

`generate_math_question` returns the answer through `[rbp-0x14]` (the `eax` return value), and `main` later passes that same value as the second argument to `encode_flag`:

```
1720: mov    edx, DWORD PTR [rbp-0x14]   ; edx = correct answer
1727: mov    esi, edx                     ; esi = 2nd arg
1729: mov    rdi, rax                     ; rdi = flag buf
172c: call   encode_flag
```

Hint 1 confirmed: the answer we typed is reused as the multiplier.

### Step 3 — Read encode_flag to confirm the math

The encode loop is small enough to grasp as pseudo-C:

```
14c5: mov    DWORD PTR [rbp-0x4], 0x0     ; i = 0
14c7: mov    eax, DWORD PTR [rbp-0x4]
14d1: add    rax, rdx
14d4: movzx  eax, BYTE PTR [rax]          ; al = flag[i]
14d7: movsx  eax, al                      ; sign-extend byte
14da: imul   eax, DWORD PTR [rbp-0x1c]    ; eax *= multiplier
14e5: mov    esi, eax
14ea: call   printf                       ; printf("%d", eax)
...
1530: movzx  eax, BYTE PTR [rax]          ; check flag[i]
1533: test   al, al
1535: jne    14c7                         ; loop while flag[i] != 0
```

So the loop is literally:

```c
for (int i = 0; flag[i] != '\0'; i++) {
    printf("%d", flag[i] * multiplier);
    if (flag[i+1] != '\0') printf(", ");
}
printf("\n");
```

Decoding is one division per value. We just divide each integer by the multiplier we sent.

### Step 4 — Connect to the server and grab one round

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ nc crystal-peak.picoctf.net 58827
What is 5 + 7? 12
Encoded flag values:
1344, 1260, 1188, 1332, 804, 1008, 840, 1476, 1308, 624, 1392, 1248, 1140, 1176, 612, 1248, 588, 1320, 1200, 1140, 1188, 588, 1344, 1248, 612, 1368, 1140, 1224, 684, 600, 1224, 672, 576, 612, 1212, 1500
```

Math answer was 12. Let me confirm the first byte:

```
1344 / 12 = 112 = ord('p')   # picoCTF starts with 'p' ✓
```

### Step 5 — Decode the values

I did it in Python straight from the terminal:

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ python3 -c "
encoded = [1344, 1260, 1188, 1332, 804, 1008, 840, 1476, 1308, 624, 1392, 1248, 1140, 1176, 612, 1248, 588, 1320, 1200, 1140, 1188, 588, 1344, 1248, 612, 1368, 1140, 1224, 684, 600, 1224, 672, 576, 612, 1212, 1500]
mult = 12
print(''.join(chr(v // mult) for v in encoded))
"
picoCTF{m4th_b3h1nd_c1ph3r_f92f803e}
```

There it is.

### Step 6 — Automate it with a small solver

To be safe I scripted the whole interaction so I could re-run on a different math question (which might be `+`, `-`, or `*`):

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ nano solver.py
```

Pasted:

```python
#!/usr/bin/env python3
import socket, re

HOST, PORT = 'crystal-peak.picoctf.net', 58827

def recv_until(s, marker, timeout=4):
    s.settimeout(timeout)
    buf = b''
    while marker not in buf:
        chunk = s.recv(4096)
        if not chunk: break
        buf += chunk
    return buf

def parse_question(text):
    m = re.search(r"What is\s+(\d+)\s*([\+\-\*])\s*(\d+)\s*\?", text)
    if not m: return None
    a, op, b = int(m.group(1)), m.group(2), int(m.group(3))
    if op == '+': return a + b
    if op == '-': return abs(a - b)   # binary swaps to keep result non-negative
    if op == '*': return a * b

s = socket.create_connection((HOST, PORT))
line = recv_until(s, b'? ').decode()
mult = parse_question(line)
print('Q:', line.strip(), '-> multiplier =', mult)
s.sendall(f"{mult}\n".encode())

block = recv_until(s, b'\n')
if b',' not in block:
    block += recv_until(s, b'\n')

for ln in block.decode().splitlines():
    if ',' in ln:
        encoded = [int(x) for x in ln.strip().split(',')]
        break

flag = ''.join(chr(v // mult) for v in encoded)
print('FLAG:', flag)
s.close()
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`.

```
┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ python3 solver.py
Q: What is 8 + 2? -> multiplier = 10
FLAG: picoCTF{m4th_b3h1nd_c1ph3r_f92f803e}

┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ python3 solver.py
Q: What is 1 * 8? -> multiplier = 8
FLAG: picoCTF{m4th_b3h1nd_c1ph3r_f92f803e}

┌──(zham㉿kali)-[~/ctf/hidden_cipher_2]
└─$ python3 solver.py
Q: What is 3 + 0? -> multiplier = 3
FLAG: picoCTF{m4th_b3h1nd_c1ph3r_f92f803e}
```

Three different random questions, three different multipliers, same flag. The flag is deterministic; only the encoded form changes.

---

## Alternative — Patch the binary and run it locally

If you don't want to script the network interaction, you can also:

1. Open the binary in Ghidra or a hex editor.
2. Find the `imul` instruction inside `encode_flag`.
3. NOP it out (replace with `mov eax, edx` so each byte passes through unchanged), or replace the printf format string with `"%c "` to print characters instead of numbers.
4. Put the real `flag.txt` next to the binary and run it.

That works on Linux only if you have GLIBC 2.38 available (the binary was built against a newer libc than most distros ship). On the VM I used, I got `version 'GLIBC_2.38' not found`. The static-disassembly + script approach above sidesteps that entirely.

---

## What Happened Internally

A timeline of one round end-to-end:

1. `main` calls `srand(time(NULL))` to seed the RNG with the current epoch second. This means the math question changes every second, but it is the same for the whole second for everyone who connects.
2. `generate_math_question` is called with three output pointers: the operator character, operand A, and operand B. Inside, it calls `rand()` three times.
   - First `rand()` becomes `operand_A = (rand() % 10) + 1` — guaranteed 1..10.
   - Second `rand()` becomes `operand_B = rand() % 10` — guaranteed 0..9.
   - Third `rand()` selects the operator: `0` is `+`, `1` is `-`, `2` is `*`.
   - For `-`, the operands are swapped if `B > A` so the subtraction is always non-negative.
   - The function returns the computed answer via `eax`.
3. `main` prints `"What is A op B? "`, then `scanf("%d", &input)`.
4. If `input == answer`, `main` calls `read_flag_file("flag.txt")`. That function `fopen`s the file, `fseek`s to the end, `malloc`s a buffer of that size, reads it in, and returns the buffer pointer.
5. `main` then calls `encode_flag(flag_buf, answer)`. `encode_flag` walks the buffer one byte at a time, multiplies each byte by `answer`, and `printf`s it as a decimal integer with `, ` separators.
6. `main` `free`s the buffer and returns.
7. The TCP socket then closes, and on our end we just divide each printed number by the answer we sent to recover the original byte. Concatenate, decode, done.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the challenge archive |
| `file` | Confirm the binary is ELF64, not stripped |
| `objdump -d -M intel` | Disassemble `main`, `encode_flag`, `generate_math_question` |
| `grep` / `sed` | Pull just the relevant function out of the disassembly |
| `nc` (netcat) | Manual probe to see the protocol and one round's worth of encoded values |
| `python3` | One-liner decode + the full automation script |
| `socket` (stdlib) | Talk to the TCP server in the script |
| `re` (stdlib) | Parse the `What is A op B?` question regardless of which operator appears |
| `nano` | Edit the solver script (Ctrl+O / Enter / Ctrl+X to save and exit) |
| (Optional) `Ghidra` / `IDA` | The challenge hints suggest a decompiler if you don't want to read raw assembly |

---

## Key Takeaways

- **Multiplication is reversible when the multiplier is known.** As long as you know the key, dividing out is a one-liner. This is the simplest possible "cipher": a Caesar shift in disguise.
- **The answer to the front-door challenge becomes the key to the back-door cipher.** Whenever you see a server gate that asks for a value and then uses that value somewhere else, treat that value as a secret parameter and figure out what it does.
- **Disassembly is faster than guessing.** Two minutes of `objdump` reading gave us the exact transform. We never had to brute-force anything.
- **Math answer = decoder key.** Once we knew each value was `byte * answer`, we had everything we needed: divide, chr, concatenate.
- **Random operands don't change the answer.** Different multipliers produce different encodings, but the underlying flag is the same across connections. So you only need one successful round.

### Flag wordplay decode

`picoCTF{m4th_b3h1nd_c1ph3r_f92f803e}` -> "math behind cipher" with l33t-speak substitutions: `4` for `a`, `3` for `e`, `1` for `i`, `0` for `o`, `3` for `e` again. The whole challenge is literally about the math hiding behind the cipher — the math answer is the cipher key. The trailing `f92f803e` is just a per-instance hex suffix to make this flag unique.
