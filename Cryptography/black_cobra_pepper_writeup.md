# Black Cobra Pepper - picoCTF Writeup

**Challenge:** Black Cobra Pepper  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{spi1cy!}`  
**Platform:** picoCTF (2024)  
**Writeup by:** zham  

---

## Description

> i like peppers. (change!) chall.py, output.txt

## Hints

> 1. Does this remind you of any other popular encryption?
> 2. What purpose does the s-box serve?

---

## Background Knowledge

Before jumping in, here are the concepts behind this challenge.

### What is AES?

**AES (Advanced Encryption Standard)** is the symmetric block cipher everyone uses. AES-128 takes a 16-byte plaintext block and a 16-byte key, runs them through 10 rounds of four operations each — `SubBytes`, `ShiftRows`, `MixColumns`, `AddRoundKey` — and spits out a 16-byte ciphertext block. Each round also derives its own 16-byte subkey from the master key using a small key schedule.

```
┌──(zham㉿kali)-[~/black-cobra-pepper]
└─$ python3 -c "
def aes_round(state, rk):
    state = sub_bytes(state)
    state = shift_rows(state)
    state = mix_columns(state)
    state = xor(state, rk)
    return state
"
```

### The S-box Is the Secret Sauce

There is exactly one non-linear operation in AES: the **S-box** in `SubBytes`. It maps each byte through a fixed lookup table (`s[i] -> sbox[s[i]]`). Without that single step, every other operation is either a permutation of bytes (`ShiftRows`), a linear matrix multiply over GF(2^8) (`MixColumns`), or a bit XOR (`AddRoundKey`). All of those are linear.

Strip the S-box out and the cipher collapses from "everyone's favorite block cipher" into "one matrix multiply over GF(2)." A linear cipher that takes a known plaintext and outputs a ciphertext is a one-shot linear-algebra problem. That is the entire challenge.

### The Clever Wrinkle: Nibble-Mixing in the Key Schedule

The author's `split()` does something I did not notice on first read:

```python
def split(full_key):
    k = full_key
    sub_keys = ["", "", "", ""]
    for i in range(len(k)):
        sub_keys[i % 4] += k[0]    # walks one HEX CHARACTER at a time
        k = k[1:]
    return sub_keys
```

It walks the 32-character round key one hex character at a time, building four "sub-keys" out of *nibbles*. That means bits *within* a single byte get shuffled around by the key schedule. The cipher is therefore linear over **GF(2)** (bit XOR), but **not** linear over **GF(2^8)** (byte-by-byte multiplication like `0x02 * c`, `0x03 * c` in the AES field). I learned this the hard way: my first solver assumed byte-level GF(2^8) linearity and got a key that did not verify. The fix is to drop down to bit-level Gaussian elimination.

### Putting the Pieces Together

The challenge gives us two files and three things to figure out:

- `chall.py` — a hand-rolled AES-128 with `sub_bytes`, `sub_word`, and `rcon` all replaced by `return word` (identity).
- `output.txt` — two 32-hex-char ciphertexts: the first is the encryption of a known plaintext, the second is the encryption of the flag.
- The hint pair — "popular encryption" plus "s-box purpose" — points straight at AES-without-the-S-box.

Attack plan: build a 128x128 binary matrix `B` from 128 single-bit-set keys, solve `B*k = rhs` over GF(2) to recover the master key, then decrypt the flag with the same trick on the plaintext side.

---

## Solution

### Step 1: Pull the Files and Look Around

I made a working directory, dropped the two challenge files into it, and inspected them.

```
┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ mkdir -p ~/picoCTF/black-cobra-pepper && cd ~/picoCTF/black-cobra-pepper

┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ ls
chall.py  output.txt
```

```
┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ cat output.txt
d7481d89f1aaf5a857f56edd2ae8994c
8c7d66558130eb5796d131beb43c9934
```

The first line is `AES(pt1, key)` and the second line is `AES(flag, key)`. The `chall.py` confirms it:

```python
pt1 = "72616e646f6d64617461313131313131"
print(AES(pt1, key))   # -> first line of output.txt
print(AES(flag, key))  # -> second line of output.txt
```

`pt1` hex-decodes to the ASCII string `randomdata11111111` — a known plaintext handed to us on a plate. That is the entire key-recovery input we need.

### Step 2: Spot the Vulnerability

Three functions in `chall.py` jumped out:

```python
def sub_bytes(state):
    return state

def sub_word(word):
    return word

def rcon(word):
    return word
```

The S-box — the only non-linear step in AES — does nothing. Same for `sub_word` and `rcon` in the key schedule. MixColumns, ShiftRows, and AddRoundKey are intact, but those are all linear operations. The whole cipher is a linear map on 128-bit vectors over GF(2).

Quick sanity check that AES is still linear in the key:

```
┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ python3 -c "
from chall import AES
k1 = '00' * 15 + '01'
k2 = '00' * 14 + '01' + '00'
k12 = '00' * 14 + '01' + '01'
print(AES('00'*16, k1))
print(AES('00'*16, k2))
print(AES('00'*16, k12))
"
27b8c672ad5d09e02fe18b5d75f0a40a
1aa0ad0a362921368b230e9bf7fae2c7
3d186b789b7428d6a4c285c6820a46cd
```

The XOR of the first two outputs equals the third exactly. Linearity in the key holds.

### Step 3: Confirm GF(2^8) Linearity Fails

If you naively try to treat each byte as a GF(2^8) element and multiply by the AES field constants `0x02`, `0x03`, things seem to work for the small scalars but blow up at `0x10`:

```
┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ python3 -c "
from chall import AES, gmul
c_e0 = bytes.fromhex(AES('00'*16, '00'*15+'01'))
for s in [2, 3, 4, 0x10]:
    mul = bytes(gmul(s, format(c_e0[i], '02x')) for i in range(16))
    aes = AES('00'*16, '00'*15+format(s, '02x'))
    print(f's=0x{s:02x}', 'match' if mul.hex() == aes else 'MISMATCH')
"
s=0x02 match
s=0x03 match
s=0x04 match
s=0x10 MISMATCH
```

The reason is the `split()` function I called out earlier — it mixes nibbles, so the linear map does not respect byte boundaries. The cipher is GF(2)-linear (bit XOR), not GF(2^8)-linear. We have to do Gaussian elimination on a 128-bit system, not a 16-byte system.

### Step 4: Write the Solver in `nano`

I dropped into `nano` and pasted a Python script that builds the 128x128 binary matrix `B`, solves for the key, then decrypts the flag.

```
┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ nano solve.py
```

```python
#!/usr/bin/env python3
"""
Black Cobra Pepper — picoCTF Crypto (Medium, 200 pts)

The cipher is AES-128 with sub_bytes / sub_word / rcon replaced by the
identity function. Without the S-box the whole cipher becomes linear
over GF(2) (but NOT over GF(2^8) because split() mixes nibbles).

  c = AES(p, k) = A * p  XOR  B * k     (128x128 binary matrices)

  1. Build B's 128 columns by encrypting p=0 with single-bit-set keys.
  2. From (pt1, ct1) compute rhs = ct1 XOR AES(pt1, 0) = B*k.
  3. Solve B*k = rhs by Gaussian elimination over GF(2).
  4. Decrypt ct2 with flag = A^-1 * (ct2 XOR B*k).
"""
from chall import AES

pt1_hex = "72616e646f6d64617461313131313131"
ct1_hex = "d7481d89f1aaf5a857f56edd2ae8994c"
ct2_hex = "8c7d66558130eb5796d131beb43c9934"


def hex_to_bits(h):
    out = []
    for byte in bytes.fromhex(h):
        for i in range(8):
            out.append((byte >> (7 - i)) & 1)
    return out


def bits_to_hex(bits):
    b = bytearray()
    for i in range(0, 128, 8):
        byte = 0
        for j in range(8):
            byte = (byte << 1) | bits[i + j]
        b.append(byte)
    return bytes(b).hex()


def set_bit(h, bit_index):
    h_list = list(h)
    byte_idx, bit_within = bit_index // 8, bit_index % 8
    val = int(h_list[2*byte_idx], 16) * 16 + int(h_list[2*byte_idx+1], 16)
    val |= (1 << (7 - bit_within))
    h_list[2*byte_idx] = f"{val >> 4:x}"
    h_list[2*byte_idx+1] = f"{val & 0xf:x}"
    return "".join(h_list)


# Build matrix B (one column per bit of the key, by encrypting p=0)
print("[*] Building 128x128 binary matrix B ...")
B_cols = [hex_to_bits(AES("00"*16, set_bit("00"*16, bit))) for bit in range(128)]


def solve_gf2(M_cols, rhs_bits):
    """Solve (M x = rhs) over GF(2)."""
    n = 128
    aug = [[M_cols[c][r] for c in range(n)] + [rhs_bits[r]] for r in range(n)]
    row = 0
    for col in range(n):
        pivot = next((r for r in range(row, n) if aug[r][col] == 1), None)
        if pivot is None:
            continue
        aug[row], aug[pivot] = aug[pivot], aug[row]
        for r in range(n):
            if r != row and aug[r][col] == 1:
                for c in range(col, n + 1):
                    aug[r][c] ^= aug[row][c]
        row += 1
    return [aug[r][n] for r in range(n)]


# Recover the master key
rhs = [a ^ b for a, b in zip(hex_to_bits(ct1_hex),
                              hex_to_bits(AES(pt1_hex, "00"*16)))]
print("[*] Solving B*k = ct1 XOR AES(pt1, 0) ...")
key_hex = bits_to_hex(solve_gf2(B_cols, rhs))
print("[+] Recovered key:", key_hex)
assert AES(pt1_hex, key_hex) == ct1_hex, "verification failed"
print("[+] Verification passed.")

# Decrypt the flag by building matrix A and solving A*flag = ct2 XOR B*k
print("[*] Building 128x128 binary matrix A ...")
A_cols = [hex_to_bits(AES(set_bit("00"*16, bit), "00"*16)) for bit in range(128)]
rhs2 = [a ^ b for a, b in zip(hex_to_bits(ct2_hex),
                               hex_to_bits(AES("00"*16, key_hex)))]
print("[*] Solving A*flag = ct2 XOR B*k ...")
flag_hex = bits_to_hex(solve_gf2(A_cols, rhs2))
print("[+] Flag plaintext (hex):", flag_hex)
print("[+] Flag plaintext:", bytes.fromhex(flag_hex).decode())
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X` (the usual nano dance).

### Step 5: Install pwntools and Run

The cipher uses `from pwn import xor`, so I need pwntools:

```
┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ pip install pwntools
```

```
┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ python3 solve.py
[*] Building 128x128 binary matrix B ...
[*] Solving B*k = ct1 XOR AES(pt1, 0) ...
[+] Recovered key: a1a1a1a1b2b2b2b2c3c3c3c3d4d4d4d4
[+] Verification passed.
[*] Building 128x128 binary matrix A ...
[*] Solving A*flag = ct2 XOR B*k ...
[+] Flag plaintext (hex): 7069636f4354467b737069316379217d
[+] Flag plaintext: picoCTF{spi1cy!}
```

The recovered master key is `a1a1a1a1b2b2b2b2c3c3c3c3d4d4d4d4` (a recognisable repeating pattern, which is a nice sanity check that we got the bits right). Independent verification — re-encrypt the recovered flag with the recovered key and check it matches `ct2`:

```
┌──(zham㉿kali)-[~/picoCTF/black-cobra-pepper]
└─$ python3 -c "
from chall import AES
key = 'a1a1a1a1b2b2b2b2c3c3c3c3d4d4d4d4'
flag = b'picoCTF{spi1cy!}'.hex()
print('AES(flag, key) =', AES(flag, key))
print('expected ct2   = 8c7d66558130eb5796d131beb43c9934')
"
AES(flag, key) = 8c7d66558130eb5796d131beb43c9934
expected ct2   = 8c7d66558130eb5796d131beb43c9934
```

**Flag:** `picoCTF{spi1cy!}`

---

## Alternative Solve Methods

### Method 1: Use Sympy for the Linear Algebra

If you prefer to let a library do the matrix work, `sympy.Matrix` does Gaussian elimination cleanly. Replace the custom `solve_gf2` with:

```python
from sympy import Matrix

B = Matrix(B_cols).T                              # 128x128 over GF(2)
rhs_sym = Matrix([a ^ b for a, b in zip(
    hex_to_bits(ct1_hex),
    hex_to_bits(AES(pt1_hex, "00" * 16))
)])
key_bits = [int(x) % 2 for x in B.solve(rhs_sym)]
key_hex = bits_to_hex(key_bits)
```

Same answer, more dependency overhead but the linear-algebra code is one line instead of thirty.

### Method 2: Treat the Cipher as a Pure Linear System (Skip the Two-Stage Trick)

My main solver runs the cipher once with the unknown key, once with `k=0`, then XORs the two outputs to isolate the key's contribution. That works because the cipher is bilinear in `(p, k)` and there is no constant term (`AES(0, 0) = 0`). A more paranoid alternative is to verify the "no constant" assumption explicitly with a second sample and use a small affine-correction term if needed. For this challenge the simple XOR is enough — the flag verifies on the first try.

### Method 3: Attack via the Key Schedule Directly

Because the key schedule is linear, you can attack the **first round key** instead of the master key. The first round key equals the master key (`keys[0] = master_key` in `gen_keys`), so the two approaches are equivalent here. In a more complex variant where round keys differed from the master key, you could pick whichever round key was easiest to recover and walk back through the linear key schedule to get the master. The cipher stays linear either way.

---

## What Happened Internally

Here is the full timeline of how the solver worked, from "I see two ciphertexts" to "I have the flag."

1. **Pulled the files.** `cat output.txt` showed two 32-hex-char lines; reading `chall.py` confirmed the second line is `AES(flag, key)` and the first line is `AES("7261...3131", key)` where the plaintext hex decodes to ASCII `randomdata11111111`. That gave me one known plaintext/ciphertext pair for free.
2. **Spotted the bug.** Three functions — `sub_bytes`, `sub_word`, `rcon` — were stubbed out as `return state` / `return word`. The S-box, the only non-linear step in AES, was disabled.
3. **Verified linearity.** A two-key XOR test (`AES(0, k1) XOR AES(0, k2)` vs. `AES(0, k1 XOR k2)`) showed bit-level XOR-linearity in the key. The same is true in the plaintext.
4. **Caught the GF(2^8) trap.** Trying to multiply by AES field constants (`0x02 * c`, `0x10 * c`) at the byte level failed for `0x10` and above. The `split()` function in the key schedule walks nibbles, not bytes, so the cipher is GF(2)-linear only. Switched the solver to bit-level Gaussian elimination.
5. **Built matrix `B`.** Ran `AES("00"*16, set_bit("00"*16, bit))` for each of the 128 bits of the key. Each 128-bit output is one column of `B`. 128 AES calls total — about a second on a laptop.
6. **Computed `rhs = ct1 XOR AES(pt1, 0)`.** Because the cipher is bilinear in `(p, k)` with no constant term, `rhs` is exactly `B*k` for the unknown key.
7. **Ran Gaussian elimination over GF(2).** Standard row reduction on the 128x129 augmented matrix. Solved in milliseconds. Output: key bits = `a1a1a1a1b2b2b2b2c3c3c3c3d4d4d4d4`.
8. **Verified.** Re-encrypted `pt1` with the recovered key and confirmed it produced `ct1`. Then independently re-encrypted the recovered flag with the recovered key and confirmed it produced `ct2`.
9. **Decrypted the flag.** Built the analogous 128x128 matrix `A` for the plaintext side (128 more AES calls, same trick), computed `rhs2 = ct2 XOR AES(0, key)`, and solved `A*flag = rhs2`. Output: `picoCTF{spi1cy!}`.

---

## Tools Used

| Tool         | Purpose                                                          |
| ------------ | ---------------------------------------------------------------- |
| `cat`        | Read the challenge output file                                   |
| `nano`       | Write the Python solver (`Ctrl+O`, `Enter`, `Ctrl+X`)            |
| `python3`    | Run the bit-level Gaussian elimination over GF(2)                 |
| `pwntools`   | The `xor()` helper used inside `chall.py`                        |
| `pip`        | Install `pwntools` if it is not already on the system            |

---

## Key Takeaways

* **The S-box is what makes AES non-linear, and therefore secure.** Strip it out and the cipher becomes a one-line linear-algebra problem. Every other operation in AES (ShiftRows, MixColumns, AddRoundKey) is already linear — that is intentional, but the S-box is what makes the *combination* resistant to known-plaintext attacks. This challenge is a clean demonstration that one missing non-linear step is enough to break the whole thing.
* **A "linear cipher" means `E(p, k)` is linear in `(p, k)`.** With one known plaintext/ciphertext pair you get a 128-bit linear system in the 128-bit key. Solve by Gaussian elimination. No brute force, no chosen-plaintext oracle, no exotic math.
* **Watch which field your linearity lives in.** This cipher is linear over **GF(2)** (bit XOR) but **not** over **GF(2^8)** (byte multiplication by AES field constants). The `split()` function in the key schedule is the reason — it walks the round key one hex character at a time, which shuffles bits within bytes. If you blindly treat each byte as a GF(2^8) element, scalars like `0x10 * c` will silently give the wrong answer. Drop down to bit-level operations and it all works.
* **One known plaintext/ciphertext pair is all you need.** The challenge hands you `pt1 = "7261...3131"` for free in the source code. That single pair is the entire input to the key-recovery system. You do not need a chosen-plaintext oracle or any other interaction with the cipher.
* **128 AES calls to build a 128x128 matrix is fine.** Even on a slow laptop, the matrix build and Gaussian elimination together take under a second. The expensive step in real AES attacks (chosen plaintexts, brute force, side channels) is completely absent here.

**Flag wordplay decode:** `spi1cy!` is leetspeak for **spicy**. The challenge is called "Black Cobra Pepper" — a cobra pepper is a hot variety, the kind you call *spicy*. The cipher itself is the *de-spiced* version of AES (the S-box, which is the cryptographic "spice," was ripped out and replaced with identity), and the flag calls that out by celebrating the spice: `spi1cy!`, with a `1` for the `i` and an exclamation point because peppers are loud.
