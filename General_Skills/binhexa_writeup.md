# binhexa — picoCTF Writeup

**Challenge:** binhexa  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{b1tw^3se_0p3eR@tI0n_su33essFuL_1367e2c6}`  

---

## Description

> How well can you perform basic binary operations?
> Start searching for the flag here: `nc titan.picoctf.net 54369`

**Hints:** None

---

## Background Knowledge (Read This First!)

### What is Binary?

Computers store everything as **binary** — a number system that uses only `0` and `1`. Each digit is called a **bit**. An 8-bit binary number like `00100001` represents a normal decimal number — in this case, 33.

To convert binary to decimal: each position from right to left is a power of 2.
```
00100001
= 0×128 + 0×64 + 1×32 + 0×16 + 0×8 + 0×4 + 0×2 + 1×1
= 32 + 1 = 33
```

### What is Hexadecimal?

**Hexadecimal (hex)** is a base-16 number system using digits `0-9` and letters `a-f`. It's a compact way to represent binary — every 4 bits maps to exactly one hex digit. In Python, hex numbers are prefixed with `0x`.

```
33 decimal = 0x21 hex
```

### Binary Operations Used in This Challenge

| Operation | Symbol | What it does | Example |
|-----------|--------|--------------|---------|
| Multiply | `*` | Normal multiplication | 33 × 201 = 6633 |
| OR | `\|` | Bit is 1 if **either** bit is 1 | 00100001 \| 11001001 = 11101001 |
| Add | `+` | Normal addition | 33 + 201 = 234 |
| Left shift | `<<` | Shift all bits left (× 2) | 00100001 << 1 = 1000010 |
| Right shift | `>>` | Shift all bits right (÷ 2) | 11001001 >> 1 = 1100100 |
| AND | `&` | Bit is 1 only if **both** bits are 1 | 00100001 & 11001001 = 00000001 |

### What is `nc` (netcat)?

`nc` (netcat) is a tool that opens a raw TCP connection to a server — like a phone call to a remote computer. The server sends text and you type responses directly. It's commonly used in CTFs to interact with challenge servers.

### ⚠️ Important Note

**Never type shell commands into the netcat window.** Netcat sends everything you type directly to the server — if you type `python3 -c "..."` in the netcat terminal, the server receives it as your answer and marks it wrong. Always use a **separate terminal** for calculations.

---

## The Numbers

```
Binary Number 1: 00100001  →  decimal 33
Binary Number 2: 11001001  →  decimal 201
```

---

## Solution — Step by Step

Open **two terminals**: one for netcat, one for Python calculations.

**Terminal 1 — connect to the server:**
```
┌──(zham㉿kali)-[~]
└─$ nc titan.picoctf.net 54369
```

**Terminal 2 — run this Python cheat sheet to get all answers at once:**
```python
python3 -c "
b1='00100001'
b2='11001001'
n1,n2=int(b1,2),int(b2,2)
print('*:  ', bin(n1*n2)[2:])
print('|:  ', bin(n1|n2)[2:])
print('+:  ', bin(n1+n2)[2:])
print('<<: ', bin(n1<<1)[2:])
print('>>: ', bin(n2>>1)[2:])
print('&:  ', bin(n1&n2)[2:])
print('HEX:', hex(n1&n2))
"
```

Output:
```
*:   1100111101001
|:   11101001
+:   11101010
<<:  1000010
>>:  1100100
&:   1
HEX: 0x1
```

Now answer each question in netcat using those results:

### Question 1 — Multiply (`*`)
```
Operation 1: '*'
Enter the binary result: 1100111101001
Correct!
```
`33 × 201 = 6633` → binary: `1100111101001`

### Question 2 — OR (`|`)
```
Operation 2: '|'
Enter the binary result: 11101001
Correct!
```
`00100001 | 11001001 = 11101001`

### Question 3 — Add (`+`)
```
Operation 3: '+'
Enter the binary result: 11101010
Correct!
```
`33 + 201 = 234` → binary: `11101010`

### Question 4 — Left Shift (`<<`)
```
Operation 4: '<<'
Perform a left shift of Binary Number 1 by 1 bits.
Enter the binary result: 1000010
Correct!
```
`00100001 << 1 = 66` → binary: `1000010`

### Question 5 — Right Shift (`>>`)
```
Operation 5: '>>'
Perform a right shift of Binary Number 2 by 1 bits.
Enter the binary result: 1100100
Correct!
```
`11001001 >> 1 = 100` → binary: `1100100`

### Question 6 — AND (`&`) + Final Hex
```
Operation 6: '&'
Enter the binary result: 00000001
Correct!
Enter the results of the last operation in hexadecimal: 0x1
Correct answer!
The flag is: picoCTF{b1tw^3se_0p3eR@tI0n_su33essFuL_1367e2c6}
```
`00100001 & 11001001 = 00000001` = `0x1` in hex

✅ Got the flag! 🎯

---

## All Answers at a Glance

| Q | Operation | On | Result (binary) | Decimal |
|---|-----------|-----|-----------------|---------|
| 1 | `*` | B1 × B2 | `1100111101001` | 6633 |
| 2 | `\|` | B1 \| B2 | `11101001` | 233 |
| 3 | `+` | B1 + B2 | `11101010` | 234 |
| 4 | `<<` | B1 << 1 | `1000010` | 66 |
| 5 | `>>` | B2 >> 1 | `1100100` | 100 |
| 6 | `&` | B1 & B2 | `00000001` | 1 → **0x1** |

---

## Alternative — Automate with Python + pwntools

If you want to solve it in one shot without typing anything manually, use **pwntools** to script the entire interaction:

```python
from pwn import *

r = remote('titan.picoctf.net', 54369)

r.recvuntil(b'Binary Number 1: ')
b1 = r.recvline().decode().strip()
r.recvuntil(b'Binary Number 2: ')
b2 = r.recvline().decode().strip()

n1, n2 = int(b1, 2), int(b2, 2)

ops = {
    '*':  lambda: bin(n1 * n2)[2:],
    '|':  lambda: bin(n1 | n2)[2:],
    '+':  lambda: bin(n1 + n2)[2:],
    '<<': lambda: bin(n1 << 1)[2:],
    '>>': lambda: bin(n2 >> 1)[2:],
    '&':  lambda: bin(n1 & n2)[2:],
}

for i in range(6):
    data = r.recvuntil(b'Enter the').decode()
    op = data.split("'")[1] if "'" in data else ''
    is_hex = 'hex' in r.recvline().decode().lower()
    
    result_bin = ops[op]()
    result = hex(int(result_bin, 2)) if is_hex else result_bin
    r.sendline(result.encode())

print(r.recvall(timeout=3).decode())
```

Run with: `python3 solve.py`

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` (netcat) | Connect to the challenge server | ⭐ Easy |
| Python `int(b, 2)` | Convert binary string to decimal | ⭐ Easy |
| Python `bin(n)[2:]` | Convert decimal back to binary string | ⭐ Easy |
| Python `hex(n)` | Convert decimal to hex | ⭐ Easy |
| pwntools (optional) | Automate the full interaction | ⭐⭐ Medium |

---

## Key Takeaways

- **Never type commands into the netcat window** — it sends them to the server as your answer
- **Always open a second terminal** for calculations when working with interactive netcat challenges
- Python is the fastest calculator for binary/hex conversions: `int('00100001', 2)` converts binary to decimal, `bin(33)[2:]` converts back, `hex(33)` gives hex
- **Left shift `<< 1` = multiply by 2**, **right shift `>> 1` = integer divide by 2** — useful shortcuts to remember
- The flag name `b1tw^3se` = "bitwise" — the entire challenge is a bitwise operations quiz
