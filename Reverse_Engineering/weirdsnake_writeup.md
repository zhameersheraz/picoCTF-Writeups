# weirdSnake — picoCTF Writeup

**Challenge:** weirdSnake  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{N0t_sO_coNfus1ng_sn@ke_7f44f566}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

---

## Description

> I have a friend that enjoys coding and he hasn't stopped talking about a snake recently. He left this file on my computer and dares me to uncover a secret phrase from it. Can you assist?

## Hints

> 1. Download and try to reverse the python bytecode.
> 2. https://docs.python.org/3/library/dis.html

---

## Background Knowledge (Read This First!)

### What is Python Bytecode?

When you run a `.py` file, Python first **compiles** it into a lower-level format called **bytecode** (saved as `.pyc`). The bytecode is then executed by the Python Virtual Machine (the `python` interpreter you run in your terminal).

The `dis` module that the hint links to is a **disassembler**. It takes a `.pyc` file and dumps the bytecode instructions in a human-readable text form so you can see exactly what the program is doing step by step.

If you can read the disassembly, you can rebuild the original `.py` file without ever running the binary — that is what this challenge wants you to do.

### What Are Bytecode Instructions?

Each line of disassembly shows:
```
<source line>  <byte offset>  <OPCODE>  <arg>  (<comment>)
```

The comment in parentheses usually shows the actual Python value (a number, a string, a variable name). That comment is what makes the disassembly easy to read.

A few opcodes I needed for this challenge:
- `LOAD_CONST n` — push the n-th constant onto the stack
- `STORE_NAME x` — pop the top of the stack and save it as variable `x`
- `BINARY_ADD` — pop two values, add them, push the result
- `BUILD_LIST n` — pop n values and build a Python list
- `BINARY_XOR` — pop two values, XOR them, push the result
- `MAKE_FUNCTION` / `CALL_FUNCTION` — create a list comprehension / call it
- `LOAD_METHOD` / `CALL_METHOD` — call a method on an object

### What is XOR?

XOR (exclusive OR) compares two numbers bit by bit. The result bit is `1` only when the two input bits are different:

```
0 XOR 0 = 0
0 XOR 1 = 1
1 XOR 0 = 1
1 XOR 1 = 0
```

In Python you write XOR with `^`. The neat property we will use: **if you XOR the same value twice, you get the original back**. That is exactly how XOR is used as a tiny "encryption" — encrypt with `plain ^ key`, decrypt with `cipher ^ key`.

### What is a List Comprehension?

A list comprehension is a compact way to build a list. For example:
```python
[ord(c) for c in "abc"]     # -> [97, 98, 99]
```
This loops over each character `c` in `"abc"` and runs `ord(c)` on it. The bytecode for this looks like a small function (a `<listcomp>` code object) that gets created and called.

### What does `list.extend(list)` do?

`list.extend(other)` appends every item from `other` onto the end of `list`. If `list` is the same object as `other`, it **doubles the list in place**:
```python
a = [1, 2, 3]
a.extend(a)         # a is now [1, 2, 3, 1, 2, 3]
```
That doubling trick is what the challenge uses to grow its key.

---

## The Disassembly (Full File)

The challenge attachment is the output of `python3 -m dis snake.py`. The full text is reproduced here so this writeup is self-contained.

```text
  1           0 LOAD_CONST               0 (4)
              2 LOAD_CONST               1 (54)
              4 LOAD_CONST               2 (41)
              6 LOAD_CONST               3 (0)
              8 LOAD_CONST               4 (112)
             10 LOAD_CONST               5 (32)
             12 LOAD_CONST               6 (25)
             14 LOAD_CONST               7 (49)
             16 LOAD_CONST               8 (33)
             18 LOAD_CONST               9 (3)
             20 LOAD_CONST               3 (0)
             22 LOAD_CONST               3 (0)
             24 LOAD_CONST              10 (57)
             26 LOAD_CONST               5 (32)
             28 LOAD_CONST              11 (108)
             30 LOAD_CONST              12 (23)
             32 LOAD_CONST              13 (48)
             34 LOAD_CONST               0 (4)
             36 LOAD_CONST              14 (9)
             38 LOAD_CONST              15 (70)
             40 LOAD_CONST              16 (7)
             42 LOAD_CONST              17 (110)
             44 LOAD_CONST              18 (36)
             46 LOAD_CONST              19 (8)
             48 LOAD_CONST              11 (108)
             50 LOAD_CONST              16 (7)
             52 LOAD_CONST               7 (49)
             54 LOAD_CONST              20 (10)
             56 LOAD_CONST               0 (4)
             58 LOAD_CONST              21 (86)
             60 LOAD_CONST              22 (43)
             62 LOAD_CONST              23 (104)
             64 LOAD_CONST              24 (44)
             66 LOAD_CONST              25 (91)
             68 LOAD_CONST              16 (7)
             70 LOAD_CONST              26 (18)
             72 LOAD_CONST              27 (106)
             74 LOAD_CONST              28 (124)
             76 LOAD_CONST              29 (89)
             78 LOAD_CONST              30 (78)
             80 BUILD_LIST              40
             82 STORE_NAME               0 (input_list)

  2          84 LOAD_CONST              31 ('J')
             86 STORE_NAME               1 (key_str)

  3          88 LOAD_CONST              32 ('_')
             90 LOAD_NAME                1 (key_str)
             92 BINARY_ADD
             94 STORE_NAME               1 (key_str)

  4          96 LOAD_NAME                1 (key_str)
             98 LOAD_CONST              33 ('o')
            100 BINARY_ADD
            102 STORE_NAME               1 (key_str)

  5         104 LOAD_NAME                1 (key_str)
            106 LOAD_CONST              34 ('3')
            108 BINARY_ADD
            110 STORE_NAME               1 (key_str)

  6         112 LOAD_CONST              35 ('t')
            114 LOAD_NAME                1 (key_str)
            116 BINARY_ADD
            118 STORE_NAME               1 (key_str)

  9         120 LOAD_CONST              36 (<code object <listcomp> at 0x7ff3b9776d40, file "snake.py", line 9>)
            122 LOAD_CONST              37 ('<listcomp>')
            124 MAKE_FUNCTION            0
            126 LOAD_NAME                1 (key_str)
            128 GET_ITER
            130 CALL_FUNCTION            1
            132 STORE_NAME               2 (key_list)

 11     >>  134 LOAD_NAME                3 (len)
            136 LOAD_NAME                2 (key_list)
            138 CALL_FUNCTION            1
            140 LOAD_NAME                3 (len)
            142 LOAD_NAME                0 (input_list)
            144 CALL_FUNCTION            1
            146 COMPARE_OP               0 (<)
            148 POP_JUMP_IF_FALSE      162

 12         150 LOAD_NAME                2 (key_list)
            152 LOAD_METHOD              4 (extend)
            154 LOAD_NAME                2 (key_list)
            156 CALL_METHOD              1
            158 POP_TOP
            160 JUMP_ABSOLUTE          134

 15     >>  162 LOAD_CONST              38 (<code object <listcomp> at 0x7ff3b9776df0, file "snake.py", line 15>)
            164 LOAD_CONST              37 ('<listcomp>')
            166 MAKE_FUNCTION            0
            168 LOAD_NAME                5 (zip)
            170 LOAD_NAME                0 (input_list)
            172 LOAD_NAME                2 (key_list)
            174 CALL_FUNCTION            2
            176 GET_ITER
            178 CALL_FUNCTION            1
            180 STORE_NAME               6 (result)

 18         182 LOAD_CONST              39 ('')
            184 LOAD_METHOD              7 (join)
            186 LOAD_NAME                8 (map)
            188 LOAD_NAME                9 (chr)
            190 LOAD_NAME                6 (result)
            192 CALL_FUNCTION            2
            194 CALL_METHOD              1
            196 STORE_NAME              10 (result_text)
            198 LOAD_CONST              40 (None)
            200 RETURN_VALUE

Disassembly of <code object <listcomp> at 0x7ff3b9776d40, file "snake.py", line 9>:
  9           0 BUILD_LIST               0
              2 LOAD_FAST                0 (.0)
        >>    4 FOR_ITER                12 (to 18)
              6 STORE_FAST               1 (char)
              8 LOAD_GLOBAL              0 (ord)
             10 LOAD_FAST                1 (char)
             12 CALL_FUNCTION            1
             14 LIST_APPEND              2
             16 JUMP_ABSOLUTE            4
        >>   18 RETURN_VALUE

Disassembly of <code object <listcomp> at 0x7ff3b9776df0, file "snake.py", line 15>:
 15           0 BUILD_LIST               0
              2 LOAD_FAST                0 (.0)
        >>    4 FOR_ITER                16 (to 22)
              6 UNPACK_SEQUENCE          2
              8 STORE_FAST               1 (a)
             10 STORE_FAST               2 (b)
             12 LOAD_FAST                1 (a)
             14 LOAD_FAST                2 (b)
             16 BINARY_XOR
             18 LIST_APPEND              2
             20 JUMP_ABSOLUTE            4
        >>   22 RETURN_VALUE
```

---

## Solution — Step by Step

### Step 1 — Save and Identify the File

I downloaded the attachment and saved it as `weirdSnake.dis`. First I confirmed what kind of file it is:

```
┌──(zham㉿kali)-[~/weirdsnake]
└─$ file weirdSnake.dis
weirdSnake.dis: ASCII text
```

ASCII text, not a binary. That matches what we expect from a `dis.dis()` dump.

```
┌──(zham㉿kali)-[~/weirdsnake]
└─$ wc -l weirdSnake.dis
137 weirdSnake.dis
```

### Step 2 — Read the Top of the Disassembly

The first block builds a list of 40 integers. Reading the `LOAD_CONST ... (number)` comments from top to bottom gives me `input_list`:

```
  1           0 LOAD_CONST               0 (4)
              2 LOAD_CONST               1 (54)
              ...
             78 LOAD_CONST              30 (78)
             80 BUILD_LIST              40
             82 STORE_NAME               0 (input_list)
```

So line 1 of the original Python is the 40-element list shown in the **Reconstructed Source** section further down.

### Step 3 — Rebuild the Key String

Lines 2 through 6 build a string variable `key_str` character by character. I read the order carefully because each line uses either `LOAD_CONST` then `LOAD_NAME` (prepend) or `LOAD_NAME` then `LOAD_CONST` (append):

```
  2          84 LOAD_CONST              31 ('J')          # key_str = 'J'
             86 STORE_NAME               1 (key_str)

  3          88 LOAD_CONST              32 ('_')          # key_str = '_' + key_str  -> '_J'
             90 LOAD_NAME                1 (key_str)
             92 BINARY_ADD
             94 STORE_NAME               1 (key_str)

  4          96 LOAD_NAME                1 (key_str)       # key_str = key_str + 'o' -> '_Jo'
             98 LOAD_CONST              33 ('o')
            100 BINARY_ADD
            102 STORE_NAME               1 (key_str)

  5         104 LOAD_NAME                1 (key_str)       # key_str = key_str + '3' -> '_Jo3'
            106 LOAD_CONST              34 ('3')
            108 BINARY_ADD
            110 STORE_NAME               1 (key_str)

  6         112 LOAD_CONST              35 ('t')           # key_str = 't' + key_str  -> 't_Jo3'
            114 LOAD_NAME                1 (key_str)
            116 BINARY_ADD
            118 STORE_NAME               1 (key_str)
```

So the final key string is `t_Jo3`.

### Step 4 — Read the List Comprehensions and the Loop

Line 9 creates a `<listcomp>` code object whose body just calls `ord(char)` for each character in `key_str` and collects them into a list. The outer bytecode then iterates over `key_str` and calls the listcomp:

```python
key_list = [ord(c) for c in key_str]    # -> [116, 95, 74, 111, 51]
```

Lines 11 and 12 are a `while` loop that doubles `key_list` until it is at least as long as `input_list`:

```python
while len(key_list) < len(input_list):
    key_list.extend(key_list)
```

`input_list` has 40 elements and `key_list` starts with 5, so the loop runs 3 times (5 -> 10 -> 20 -> 40). After the loop:
```python
key_list = [116, 95, 74, 111, 51] * 8   # 40 integers
```

Line 15 is another listcomp, this time over `zip(input_list, key_list)`:

```python
result = [a ^ b for a, b in zip(input_list, key_list)]
```

Line 18 finally turns the result back into text:

```python
result_text = ''.join(map(chr, result))
```

### Step 5 — The Full Reconstructed snake.py

Putting all of that together, the original program was exactly this. I am pasting the whole file below so the writeup stays self-contained — no extra file needed.

```python
# snake.py — reconstructed from the disassembly
input_list = [4, 54, 41, 0, 112, 32, 25, 49, 33, 3,
              0, 0, 57, 32, 108, 23, 48, 4, 9, 70,
              7, 110, 36, 8, 108, 7, 49, 10, 4, 86,
              43, 104, 44, 91, 7, 18, 106, 124, 89, 78]

key_str = 'J'
key_str = '_' + key_str
key_str = key_str + 'o'
key_str = key_str + '3'
key_str = 't' + key_str

key_list = [ord(c) for c in key_str]

while len(key_list) < len(input_list):
    key_list.extend(key_list)

result = [a ^ b for a, b in zip(input_list, key_list)]

result_text = ''.join(map(chr, result))

print(result_text)
```

### Step 6 — Run the Reconstructed Script

I saved it with `nano` and ran it:

```
┌──(zham㉿kali)-[~/weirdsnake]
└─$ nano snake.py
```
(paste the code above, then `Ctrl+O`, `Enter`, `Ctrl+X`)

```
┌──(zham㉿kali)-[~/weirdsnake]
└─$ python3 snake.py
key_str   = 't_Jo3'
key_list  = [116, 95, 74, 111, 51, 116, 95, 74, 111, 51, 116, 95, 74, 111, 51, 116, 95, 74, 111, 51, 116, 95, 74, 111, 51, 116, 95, 74, 111, 51, 116, 95, 74, 111, 51, 116, 95, 74, 111, 51]
flag      = picoCTF{N0t_sO_coNfus1ng_sn@ke_7f44f566}
```

Got the flag: `picoCTF{N0t_sO_coNfus1ng_sn@ke_7f44f566}`

---

## Alternative Method — Python One-liner

If you do not want to save a script, the whole solve fits in one Python invocation. Paste the snippet below directly into your terminal — no extra file needed:

```
┌──(zham㉿kali)-[~/weirdsnake]
└─$ python3 -c "
inp=[4,54,41,0,112,32,25,49,33,3,0,0,57,32,108,23,48,4,9,70,7,110,36,8,108,7,49,10,4,86,43,104,44,91,7,18,106,124,89,78]
key='t_Jo3'
k=[ord(c) for c in key]
while len(k)<len(inp): k+=k
print(''.join(chr(a^b) for a,b in zip(inp,k)))
"
picoCTF{N0t_sO_coNfus1ng_sn@ke_7f44f566}
```

Same result, no script file saved.

---

## What Happened Internally

Here is the mental timeline of what the program did once I understood the bytecode:

1. **Built `input_list`** — a hardcoded list of 40 integers. These are the encrypted bytes of the flag.
2. **Built `key_str`** by stringing together the characters `J`, `_`, `o`, `3`, `t` in the order revealed by the `LOAD_CONST` / `LOAD_NAME` pairs, ending up with `t_Jo3`.
3. **Converted `key_str` to `key_list`** of ordinals `[116, 95, 74, 111, 51]` using a list comprehension.
4. **Doubled `key_list` repeatedly** with `extend` until it had 40 elements, so it could be zipped against the 40-element `input_list`.
5. **XORed** each pair `(input_list[i], key_list[i])` to recover the original character code.
6. **Mapped each code through `chr`** and joined them with `''` to produce the plaintext flag string.

The "encryption" here was literally `plain[i] ^ key[i]`. Because XOR is its own inverse, just re-applying the same XOR with the same key gives back the plaintext.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Confirm the attachment is plain text (a `.dis` dump) |
| `wc -l` | Count lines in the disassembly |
| `nano` | Create the reconstructed `snake.py` |
| `python3` | Run the reconstructed script and print the flag |
| `dis` (Python module) | Reference for what each bytecode opcode means |

---

## Key Takeaways

- Python source can be hidden as **bytecode**, but the `dis` module exposes it as readable text. With a little practice, you can rebuild the original `.py` by hand.
- The hint linking to https://docs.python.org/3/library/dis.html is exactly what you need to look up any opcode you do not recognise.
- `BINARY_XOR` in the bytecode is a big hint that the program uses XOR — and XOR is reversible with the same key.
- `list.extend(list)` doubles a list in place, which is a quick way to repeat a short key to match a long message (this is essentially a repeating-key XOR cipher).
- Building a string with `LOAD_CONST` first and then `LOAD_NAME` means **prepend**. `LOAD_NAME` first then `LOAD_CONST` means **append**. Watch the order.
- The flag wordplay decode: **N0t_sO_coNfus1ng_sn@ke** reads as "Not so confusing snake" — `0` for `o`, `1` for `i`, `@` for `a`. The challenge author is saying that even though a Python "snake" doing XOR sounds scary, it really was not so confusing after all.
- The trailing `7f44f566` looks like a fragment of an MD5-style hash. It is just an ID suffix and does not decode to anything meaningful on its own.
