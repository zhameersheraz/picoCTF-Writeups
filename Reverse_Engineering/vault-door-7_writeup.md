# vault-door-7 — picoCTF Writeup

**Challenge:** vault-door-7  
**Category:** Reverse Engineering  
**Difficulty:** Hard  
**Flag:** `picoCTF{A_b1t_0f_b1t_sh1fTiNg_7bf2b8dfa4}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This vault uses bit shifts to convert a password string into an array of integers. Hurry, agent, we are running out of time to stop Dr. Evil's nefarious plans!
> The source code for this vault is here: VaultDoor7.java

**Hint 1:** Use a decimal/hexademical converter such as this one: https://www.mathsisfun.com/binary-decimal-hexadecimal-converter.html  
**Hint 2:** You will also need to consult an ASCII table such as this one: https://www.asciitable.com/

---

## Background Knowledge (Read This First!)

### Reading the source code

`VaultDoor7.java` looks long, but the entire check boils down to two functions: `passwordToIntArray` (which encodes the password) and `checkPassword` (which compares the encoded result to 8 hardcoded ints). Here is the encoder:

```java
public int[] passwordToIntArray(String hex) {
    int[] x = new int[8];
    byte[] hexBytes = hex.getBytes();
    for (int i=0; i<8; i++) {
        x[i] = hexBytes[i*4]   << 24
             | hexBytes[i*4+1] << 16
             | hexBytes[i*4+2] << 8
             | hexBytes[i*4+3];
    }
    return x;
}
```

And the check that uses it:

```java
public boolean checkPassword(String password) {
    if (password.length() != 32) {
        return false;
    }
    int[] x = passwordToIntArray(password);
    return x[0] == 1096770097
        && x[1] == 1952395366
        && x[2] == 1600270708
        && x[3] == 1601398833
        && x[4] == 1716808014
        && x[5] == 1734293346
        && x[6] == 1714577976
        && x[7] == 1684431156;
}
```

The 32-character password must produce these exact 8 ints. Eight ints for 32 characters means **4 characters per int** — one int holds four bytes.

### What is a byte?

Every ASCII character has a numeric code that fits in a single byte (8 bits, values 0-255). For example:

- `'A'` is `0x41`
- `'_'` is `0x5F`
- `'0'` is `0x30`

So a 32-character password is really just 32 bytes.

### What are bit shifts?

A Java `int` is 32 bits wide, big enough to hold four 8-bit bytes side by side. The encoder uses two bitwise tricks:

- `b << n` shifts the bits of `b` to the **left** by `n` positions. Each shift left by 1 multiplies by 2.
- `a | b` merges two numbers by stacking their 1-bits together (bitwise **OR**).

For 4 characters at positions `i*4` to `i*4+3`, the loop:

- shifts byte 0 left by 24 bits (places it in the highest 8 bits)
- shifts byte 1 left by 16 bits
- shifts byte 2 left by 8 bits
- leaves byte 3 in place
- ORs all four together into one int

The author's own comment walks through the example `"01ab"` packing to `808542562`:

```
0x30: 00110000
0x31: 00110001
0x61: 01100001
0x62: 01100010
00110000001100010110000101100010 -> 808542562
```

### How do we reverse it?

Shifting right undoes a left shift, and masking with `0xff` keeps only the lowest 8 bits. So for each int:

```python
(n >> 24) & 0xff     # recover byte 0 (top 8 bits)
(n >> 16) & 0xff     # recover byte 1
(n >>  8) & 0xff     # recover byte 2
(n      ) & 0xff     # recover byte 3 (bottom 8 bits)
```

Each recovered byte is an ASCII character; concatenating them gives the password back.

---

## Solution — Step by Step

### Step 1 — Save the source and inspect the hardcoded ints

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-7]
└─$ mkdir -p ~/picoCTF/vault-door-7 && cd ~/picoCTF/vault-door-7
┌──(zham㉿kali)-[~/picoCTF/vault-door-7]
└─$ cp /path/to/VaultDoor7.java .
┌──(zham㉿kali)-[~/picoCTF/vault-door-7]
└─$ grep "x\[" VaultDoor7.java
        return x[0] == 1096770097
            && x[1] == 1952395366
            && x[2] == 1600270708
            && x[3] == 1601398833
            && x[4] == 1716808014
            && x[5] == 1734293346
            && x[6] == 1714577976
            && x[7] == 1684431156;
```

Those 8 numbers are the only inputs we need.

### Step 2 — Write the inverse in Python

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-7]
└─$ nano solve.py
```

Paste this into nano, then save with `Ctrl+O`, `Enter`, and exit with `Ctrl+X`:

```python
# solve.py -- invert VaultDoor7's passwordToIntArray()
ints = [
    1096770097, 1952395366, 1600270708, 1601398833,
    1716808014, 1734293346, 1714577976, 1684431156,
]

out = ""
for n in ints:
    out += chr((n >> 24) & 0xff)   # byte 0
    out += chr((n >> 16) & 0xff)   # byte 1
    out += chr((n >>  8) & 0xff)   # byte 2
    out += chr( n        & 0xff)   # byte 3

print(out)
print("picoCTF{" + out + "}")
```

### Step 3 — Run it

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-7]
└─$ python3 solve.py
A_b1t_0f_b1t_sh1fTiNg_7bf2b8dfa4
picoCTF{A_b1t_0f_b1t_sh1fTiNg_7bf2b8dfa4}
```

### Step 4 — Verify the flag actually unlocks the vault

Before celebrating, run the **forward** transform on the recovered password and confirm it produces the same 8 ints.

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-7]
└─$ nano verify.py
```

```python
password = "A_b1t_0f_b1t_sh1fTiNg_7bf2b8dfa4"
got = []
for i in range(8):
    n = (ord(password[i*4])     << 24
       | ord(password[i*4 + 1]) << 16
       | ord(password[i*4 + 2]) << 8
       | ord(password[i*4 + 3]))
    got.append(n)
expected = [1096770097, 1952395366, 1600270708, 1601398833,
            1716808014, 1734293346, 1714577976, 1684431156]
print("Match:", got == expected)
```

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-7]
└─$ python3 verify.py
Match: True
```

Flag is confirmed: **`picoCTF{A_b1t_0f_b1t_sh1fTiNg_7bf2b8dfa4}`**

---

## Alternative Method — pwntools in one line

If pwntools is installed, the unpacking collapses to a single line using `p32` in big-endian mode:

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-7]
└─$ python3 -c "
from pwn import p32
ints=[1096770097,1952395366,1600270708,1601398833,1716808014,1734293346,1714577976,1684431156]
print(''.join(p32(n, endian='big').decode() for n in ints))
"
A_b1t_0f_b1t_sh1fTiNg_7bf2b8dfa4
```

Same answer, no manual shifting.

---

## Alternative Method — Follow the hints with mathsisfun + asciitable

The hints steer you toward a fully manual route, which also works:

1. Convert each int to hex.
2. Split each hex value into 4 two-digit bytes.
3. Look each byte up on the ASCII table to get a character.

Quick worked example for `x[0] = 1096770097`:

| Step | Value |
|------|-------|
| Decimal | 1096770097 |
| Hex | `0x415F6231` |
| Bytes (big-endian) | `41 5F 62 31` |
| ASCII | `A  _  b  1` |

Repeat for all 8 ints. It works — just slow.

---

## Per-Int Decode Table

| # | Decimal | Hex | ASCII |
|---|---------|-----|-------|
| 0 | 1096770097 | `415F6231` | `A_b1` |
| 1 | 1952395366 | `745F3066` | `t_0f` |
| 2 | 1600270708 | `5F623174` | `_b1t` |
| 3 | 1601398833 | `5F736831` | `_sh1` |
| 4 | 1716808014 | `6654694E` | `fTiN` |
| 5 | 1734293346 | `675F3762` | `g_7b` |
| 6 | 1714577976 | `66326238` | `f2b8` |
| 7 | 1684431156 | `64666134` | `dfa4` |

Concatenating the ASCII columns gives the inner flag content: `A_b1t_0f_b1t_sh1fTiNg_7bf2b8dfa4`.

Note: the `7bf2b8dfa4` tail is per-instance. Older online writeups show a different ending because picoCTF appends a random hex hash to keep shared flags unique. If you copy a flag from another source, the vault will reject it — you must recover yours.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reading `VaultDoor7.java` | Understand exactly how the password is encoded | Easy |
| `grep` | Pull the 8 hardcoded ints out of the source in one line | Easy |
| `nano` | Edit `solve.py` and `verify.py` in place | Easy |
| `python3` bitwise ops | `>>` and `& 0xff` to unpack bytes from each int | Medium |
| (Optional) `pwntools` `p32` | Unpack an int into bytes in one call | Medium |

---

## Key Takeaways

- **Reversing bit-packing is just shifting in the opposite direction.** Every `<<` in the encoder becomes a `>>` plus `& 0xff` to isolate a single byte in the decoder
- **A 32-character password can be fully described by 8 integers.** Anywhere you see the pattern `(b0 << 24) | (b1 << 16) | (b2 << 8) | b3`, you can reverse it the same way
- **Verify by running the forward transform on your recovered string.** One more for-loop confirms the flag would actually open the vault instead of just looking plausible
- The flag `A_b1t_0f_b1t_sh1fTiNg` is leetspeak for **"a bit of bit shifting"** — the challenge name describes its own solution
