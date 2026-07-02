# vault-door-8 — picoCTF Writeup

**Challenge:** vault-door-8  
**Category:** Reverse Engineering  
**Difficulty:** Hard  
**Flag:** `picoCTF{s0m3_m0r3_b1t_sh1fTiNg_785c5c77d}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> Apparently Dr. Evil's minions knew that our agency was making copies of their source code, because they intentionally sabotaged this source code in order to make it harder for our agents to analyze and crack into! The result is a quite mess, but I trust that my best special agent will find a way to solve it.
>
> The source code for this vault is here: VaultDoor8.java

**Hint 1:** Clean up the source code so that you can read it and understand what is going on.  
**Hint 2:** Draw a diagram to illustrate which bits are being switched in the `scramble()` method, then figure out a sequence of bit switches to undo it. You should be able to reuse the `switchBits()` method as is.

---

## Background Knowledge (Read This First!)

### Step 1: clean up the source

The provided `VaultDoor8.java` is one giant pile of obfuscated one-liners and commented-out lines. Hint 1 asks us to clean it up first. After reformatting and removing the noise, the program has just three real methods. Here is the readable version:

```java
import java.util.*;

class VaultDoor8 {
    public static void main(String args[]) {
        Scanner scanner = new Scanner(System.in);
        System.out.print("Enter vault password: ");
        String userInput = scanner.next();
        String input = userInput.substring("picoCTF{".length(), userInput.length() - 1);
        VaultDoor8 vaultDoor = new VaultDoor8();
        if (vaultDoor.checkPassword(input)) {
            System.out.println("Access granted.");
        } else {
            System.out.println("Access denied!");
        }
    }

    public char[] scramble(String password) {
        // Scramble a password by transposing pairs of bits.
        char[] a = password.toCharArray();
        for (int b = 0; b < a.length; b++) {
            char c = a[b];
            c = switchBits(c, 1, 2);
            c = switchBits(c, 0, 3);
            c = switchBits(c, 5, 6);
            c = switchBits(c, 4, 7);
            c = switchBits(c, 0, 1);
            c = switchBits(c, 3, 4);
            c = switchBits(c, 2, 5);
            c = switchBits(c, 6, 7);
            a[b] = c;
        }
        return a;
    }

    public char switchBits(char c, int p1, int p2) {
        // Move the bit in position p1 to position p2, and move the bit
        // that was in position p2 to position p1. Precondition: p1 < p2
        char mask1 = (char)(1 << p1);
        char mask2 = (char)(1 << p2);
        char bit1  = (char)(c & mask1);
        char bit2  = (char)(c & mask2);
        char rest  = (char)(c & ~(mask1 | mask2));
        char shift = (char)(p2 - p1);
        return (char)((bit1 << shift) | (bit2 >> shift) | rest);
    }

    public boolean checkPassword(String password) {
        char[] scrambled = scramble(password);
        char[] expected = {
            0xF4, 0xC0, 0x97, 0xF0, 0x77, 0x97, 0xC0, 0xE4,
            0xF0, 0x77, 0xA4, 0xD0, 0xC5, 0x77, 0xF4, 0x86,
            0xD0, 0xA5, 0x45, 0x96, 0x27, 0xB5, 0x77, 0xF1,
            0xC2, 0xD1, 0xB4, 0xD1, 0xB4, 0xF1, 0xF1, 0x85
        };
        return Arrays.equals(scrambled, expected);
    }
}
```

### What `switchBits` actually does

It takes the bit at position `p1` and the bit at position `p2`, swaps them, and leaves every other bit alone. `p1 < p2` so shifting the smaller position up by `p2 - p1` lines up with the bigger position and vice versa.

A quick worked example — swap bits 1 and 2 of `0b00100111` (`0x27`):

```
input:        0 0 1 0 0 1 1 1   (0x27 = "'")
positions:    7 6 5 4 3 2 1 0
swap bits 1,2:
              0 0 1 0 0 1 1 1   <-- bit 1 moves up to pos 2, bit 2 moves down to pos 1
                 ^   ^
result:       0 0 1 0 0 1 1 1   -> 0x27  (coincidence for this input)
```

### What `scramble` does on a single character

It applies **eight** bit swaps in a fixed order:

```
(1,2)  (0,3)  (5,6)  (4,7)  (0,1)  (3,4)  (2,5)  (6,7)
```

Every swap is its own inverse (swapping two bits twice gives back the original), and the same swap order applied to the whole byte does not interact between bytes — `scramble` operates on each character independently. That means we can answer two related questions about any byte:

1. **Bijection:** is `scramble` a 1-to-1 mapping on values 0-255? (Yes — composed swaps are still a permutation.)
2. **Inverse:** what input byte maps to a given output byte?

### Step 2: draw the permutation (the answer to hint 2)

Hint 2 suggests drawing a diagram. To do that, run `scramble` eight times, each time with only one bit set, and watch where that bit lands:

| Input bit set | `scramble()` output | Output position |
|---------------|---------------------|-----------------|
| bit 0 (0x01)  | 0b00010000          | position 4      |
| bit 1 (0x02)  | 0b00100000          | position 5      |
| bit 2 (0x04)  | 0b00000001          | position 0      |
| bit 3 (0x08)  | 0b00000010          | position 1      |
| bit 4 (0x10)  | 0b01000000          | position 6      |
| bit 5 (0x20)  | 0b10000000          | position 7      |
| bit 6 (0x40)  | 0b00000100          | position 2      |
| bit 7 (0x80)  | 0b00001000          | position 3      |

Read down the right column to see the full mapping:

```
  position:    0  1  2  3  4  5  6  7
  input bit:   2  3  6  7  0  1  4  5
```

In cycle notation:

```
(0 4 6 2)(1 5 7 3)   -- two 4-cycles
```

The inverse permutation sends input bit `i` BACK to position `perm_inv[perm[i]] = i`:

```
  position:    0  1  2  3  4  5  6  7
  came from:   4  5  0  1  6  7  2  3
```

So to **invert** a scrambled byte `c`, for each input bit position `i` we read bit `perm[i]` from `c`.

### Two ways to use this

Either of these is a complete solve:

1. **Direct inverse** — read bits out of `c` using the `perm_inv` table and re-pack them in original order. Pythonic version: build the 256-entry lookup with `for c in range(256): scramble(c)` and use it as a reverse dictionary. This is what we do below — it's three lines and obviously correct.
2. **Reverse the swap order** — applying the same swaps in reverse undoes them, because `switchBits` is self-inverse. That gives the inverse chain `(6,7) (2,5) (3,4) (0,1) (4,7) (5,6) (0,3) (1,2)`. Either path works.

---

## Solution — Step by Step

### Step 1 — Save the source and port the helpers to Python

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-8]
└─$ mkdir -p ~/picoCTF/vault-door-8 && cd ~/picoCTF/vault-door-8
┌──(zham㉿kali)-[~/picoCTF/vault-door-8]
└─$ cp /path/to/VaultDoor8.java .
┌──(zham㉿kali)-[~/picoCTF/vault-door-8]
└─$ nano solve.py
```

Paste this in nano (it's a direct line-by-line port of `switchBits` and `scramble`). Save with `Ctrl+O`, `Enter`, and exit with `Ctrl+X`:

```python
# solve.py -- invert VaultDoor8's scramble()

def switch_bits(c, p1, p2):
    mask1 = 1 << p1
    mask2 = 1 << p2
    bit1  = c & mask1
    bit2  = c & mask2
    rest  = c & ~(mask1 | mask2)
    shift = p2 - p1
    return (bit1 << shift) | (bit2 >> shift) | rest

def scramble(c):
    c = switch_bits(c, 1, 2)
    c = switch_bits(c, 0, 3)
    c = switch_bits(c, 5, 6)
    c = switch_bits(c, 4, 7)
    c = switch_bits(c, 0, 1)
    c = switch_bits(c, 3, 4)
    c = switch_bits(c, 2, 5)
    c = switch_bits(c, 6, 7)
    return c & 0xFF

expected = [
    0xF4, 0xC0, 0x97, 0xF0, 0x77, 0x97, 0xC0, 0xE4,
    0xF0, 0x77, 0xA4, 0xD0, 0xC5, 0x77, 0xF4, 0x86,
    0xD0, 0xA5, 0x45, 0x96, 0x27, 0xB5, 0x77, 0xF1,
    0xC2, 0xD1, 0xB4, 0xD1, 0xB4, 0xF1, 0xF1, 0x85,
]

# Build a 256-entry reverse lookup once
reverse = {scramble(c): c for c in range(256)}
assert len(reverse) == 256, "scramble is NOT a bijection -- check your port"

password = ''.join(chr(reverse[e]) for e in expected)
print(password)
print('picoCTF{' + password + '}')
```

### Step 2 — Run it

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-8]
└─$ python3 solve.py
s0m3_m0r3_b1t_sh1fTiNg_785c5c77d
picoCTF{s0m3_m0r3_b1t_sh1fTiNg_785c5c77d}
```

### Step 3 — Verify by re-scrambling

The flag is only real if running the **forward** scramble over it reproduces `expected`:

```
┌──(zham㉿kali)-[~/picoCTF/vault-door-8]
└─$ python3 -c "
def switch_bits(c, p1, p2):
    mask1 = 1 << p1; mask2 = 1 << p2
    bit1 = c & mask1; bit2 = c & mask2
    rest = c & ~(mask1 | mask2); shift = p2 - p1
    return (bit1 << shift) | (bit2 >> shift) | rest
def scramble(c):
    for p in [(1,2),(0,3),(5,6),(4,7),(0,1),(3,4),(2,5),(6,7)]:
        c = switch_bits(c, *p)
    return c & 0xFF
expected = [0xF4,0xC0,0x97,0xF0,0x77,0x97,0xC0,0xE4,0xF0,0x77,0xA4,0xD0,0xC5,0x77,0xF4,0x86,0xD0,0xA5,0x45,0x96,0x27,0xB5,0x77,0xF1,0xC2,0xD1,0xB4,0xD1,0xB4,0xF1,0xF1,0x85]
password = 's0m3_m0r3_b1t_sh1fTiNg_785c5c77d'
got = bytes([scramble(ord(ch)) for ch in password])
print('match:', got == bytes(expected))
"
match: True
```

The recovered password re-scrambles exactly to the expected array. Locked in: **`picoCTF{s0m3_m0r3_b1t_sh1fTiNg_785c5c77d}`**

### Spot-check one byte end to end

For total confidence, trace one byte from end to end:

- Plaintext byte: `'b'` = `0x62` = `0b01100010`
- Apply the eight swaps; trace bit-by-bit:
  - bit 7: 0 -> swap(6,7) -> goes to position 6 -> later swap(4,7) (untouched here, but...)
  
Instead of hand-tracing, just trust the assertion — `len(reverse) == 256` proves every byte has a unique preimage, so the lookup is invertible by construction.

---

## Alternative Method — Run scramble in reverse

Because each `switchBits` is its own inverse, applying the same swaps in **reverse order** recovers the original:

```python
def unscramble(c):
    for p in [(6,7),(2,5),(3,4),(0,1),(4,7),(5,6),(0,3),(1,2)]:
        c = switch_bits(c, *p)
    return c & 0xFF

# Walk through expected[] once and apply unscramble() per byte
inner = ''.join(chr(unscramble(e)) for e in expected)
print('picoCTF{' + inner + '}')
```

This produces the same flag. Either path works — the reverse-swap route is more "manual" but reinforces Hint 2 directly.

---

## Alternative Method — Use the permutation table directly

If you actually drew the diagram from Hint 2, you got:

```
  position:    0  1  2  3  4  5  6  7
  came from:   4  5  0  1  6  7  2  3
```

That inverse permutation can be applied to each byte in plain Python with no lookup table:

```python
inv_perm = [4, 5, 0, 1, 6, 7, 2, 3]  # output position -> input bit it came from

def unpermute(c):
    out = 0
    for dst in range(8):
        src = inv_perm[dst]
        bit = (c >> src) & 1
        out |= bit << dst
    return out

inner = ''.join(chr(unpermute(e)) for e in expected)
print('picoCTF{' + inner + '}')
```

This is the shortest correct answer if you trust your diagram.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Reformatting `VaultDoor8.java` | Hint 1 — strip the obfuscation to see the real algorithm | Easy |
| Mental / paper bit-tracking | Hint 2 — draw the (0 4 6 2)(1 5 7 3) permutation diagram | Medium |
| `nano` | Edit `solve.py` | Easy |
| `python3` | Port `switchBits` + `scramble`, build the 256-entry reverse lookup, run the assertion, verify forward | Medium |

---

## Key Takeaways

- **Reformat before you read.** `VaultDoor8.java` looks scary in its obfuscated one-liner form, but the actual algorithm fits in under 30 lines once commented cruft is deleted
- **Bit swaps compose into a permutation.** Tracking each of the 8 input bits through the 8 swaps tells you exactly where every bit lands — `(0 4 6 2)(1 5 7 3)` in cycle notation. With a permutation in hand you can invert with one shift/mask pass per byte
- **A brute-force 256-entry lookup is often the fastest solve.** `for c in range(256): scramble(c)` is small enough to write, runs in microseconds, and the `len(reverse) == 256` assertion tells you on the spot whether your port is a bijection
- **`switchBits` is its own inverse**, so the same swaps applied in reverse order also recover the original — pick whichever path is shorter to write down
- The flag `s0m3_m0r3_b1t_sh1fTiNg` is leetspeak for **"some more bit shifting"**, picking up directly from vault-door-7's "a bit of bit shifting" — the vault-door series is one long bit-shifting pun. The `785c5c77d` tail is a per-instance random hash picoCTF appends to keep flags unique across users
