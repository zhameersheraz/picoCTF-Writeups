# bloat.py — picoCTF Writeup

**Challenge:** bloat.py  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{d30bfu5c4710n_f7w_161a4f09}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Can you get the flag?
>
> Run this `Python program` in the same directory as this `encrypted flag`.

## Hints

(Hint panel did not reveal any explicit hint for this instance — the tag **obfuscation** under the challenge is the hint: the source has been deliberately made unreadable, but the logic is still plain Python.)

---

## Background Knowledge

`bloat.py` is the sequel to `patchme.py`. The structure is identical (XOR-encrypted `flag.txt.enc` gated by a password prompt), but every visible string in the program has been replaced by a *table-lookup expression*. A few concepts clear this up.

**1. Python strings are just sequences of characters.**
There is no difference between `'hello'` and `chr(104)+chr(101)+chr(108)+chr(108)+chr(111)` — both produce the same `str` object. The second form just makes the source impossible to skim. That is the entire obfuscation.

**2. Indexing into an alphabet table.**
The script defines a global lookup string once at the top:

```python
a = "!\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\\]^_`abcdefghijklmnopqrstuvwxyz{|}~ "
```

That is the 95 printable ASCII characters in order, from `!` (index `0`) to ` ` (index `94`). Every "string" in the file is built as `a[71]+a[64]+a[79]+...` — concatenating single characters pulled by index. The "password" is just `a[71]+a[64]+a[79]+a[79]+a[88]+a[66]+a[71]+a[64]+a[77]+a[66]+a[68]`. Add a tiny decoder and the entire program collapses back to plain English.

**3. The two functions that matter.**
- `arg133(arg432)` is the password gate. It compares your input to a literal built from `a[i]+a[i]+...`. If it does not match, it prints a rejection message (also built from `a[i]+a[i]+...`) and `sys.exit(0)`.
- `arg111(arg444)` is the decryption routine. It calls `arg122` — the same XOR-with-repeating-key helper from `patchme.py` — using *another* literal built from `a[i]+a[i]+...` as the key. The key is the XOR key for the flag.

So the only two secrets in the file are: (1) the password for the gate, and (2) the XOR key for the cipher. Both are sitting right there in the source — just expressed as table lookups.

**4. Why "obfuscation" is not "encryption."**
All the obfuscation here is *cosmetic* — it makes a human skim the file and see only `a[71]+a[64]+...`. It does not actually hide the values from someone who pauses to evaluate them. A real attack on this kind of program is to add `print(...)` to dump the constants, or to replace the obfuscated expressions with the strings they evaluate to, or just to evaluate them once with a small Python helper.

**5. The `rapscallion`/`happychance` literary flavor.**
Both the password (`happychance`) and the XOR key (`rapscallion`) are unusual English words. picoCTF authors like doing this — see `correctstaple` from the earlier `unpackme.py` challenge — so the words are not security; they are just memorable placeholders.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The downloaded challenge gives two files: `bloat.py` and `flag.txt.enc`.

### 1. Move the files into a working directory and look at the source

```
┌──(zham㉿kali)-[~/bloat]
└─$ mkdir -p ~/bloat && cd ~/bloat

┌──(zham㉿kali)-[~/bloat]
└─$ cp ~/Downloads/bloat.py .
┌──(zham㉿kali)-[~/bloat]
└─$ cp ~/Downloads/flag.txt.enc .

┌──(zham㉿kali)-[~/bloat]
└─$ cat bloat.py
```

The file is short (41 lines) but every visible string has been replaced by `a[i]+a[i]+...` expressions. The structure is identical to `patchme.py`:

```python
a = "!\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\\]^_`abcdefghijklmnopqrstuvwxyz{|}~ "

def arg133(arg432):
  if arg432 == a[71]+a[64]+a[79]+a[79]+a[88]+a[66]+a[71]+a[64]+a[77]+a[66]+a[68]:
    return True
  else:
    print(a[51]+a[71]+a[64]+a[83]+a[94]+a[79]+a[64]+a[82]+a[82]+a[86]+a[78]+\
a[81]+a[67]+a[94]+a[72]+a[82]+a[94]+a[72]+a[77]+a[66]+a[78]+a[81]+\
a[81]+a[68]+a[66]+a[83])
    sys.exit(0)
    return False

def arg111(arg444):
  return arg122(arg444.decode(), a[81]+a[64]+a[79]+a[82]+a[66]+a[64]+a[75]+\
a[75]+a[72]+a[78]+a[77])
```

Two secrets I need to recover:

- The password literal in `arg133`: `a[71]+a[64]+a[79]+a[79]+a[88]+a[66]+a[71]+a[64]+a[77]+a[66]+a[68]`.
- The XOR key in `arg111`: `a[81]+a[64]+a[79]+a[82]+a[66]+a[64]+a[75]+a[75]+a[72]+a[78]+a[77]`.

### 2. Decode the table once and use it everywhere

The cleanest move is to write a tiny helper that evaluates every `a[NNN]+a[NNN]+...` expression in the file. I drop this at the top of a scratch file:

```
┌──(zham㉿kali)-[~/bloat]
└─$ nano decode.py
```

In `nano` I pasted:

```python
import re

a = "!\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\\]^_`abcdefghijklmnopqrstuvwxyz{|}~ "

src = open('bloat.py').read()

def decode(expr):
    return ''.join(a[int(n)] for n in re.findall(r'a\[(\d+)\]', expr))

# Find every concatenated "a[i]+a[i]+..." expression (handles line continuations too)
exprs = re.findall(r'(?:a\[\d+\]\+)+a\[\d+\]', src)
for e in exprs:
    print(repr(e[:60]).ljust(65), '->', decode(e))
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X` to exit `nano`.

### 3. Run the decoder

```
┌──(zham㉿kali)-[~/bloat]
└─$ python3 decode.py
```

```
'a[71]+a[64]+a[79]+a[79]+a[88]+a[66]+a[71]+a[64]+a[77]+a[66]+a[68]' -> happychance
'a[51]+a[71]+a[64]+a[83]+a[94]+a[79]+a[64]+a[82]+a[82]+a[86]+a[78]+ ...' -> That password is incorrect
'a[81]+a[64]+a[79]+a[82]+a[66]+a[64]+a[75]+a[75]+a[72]+a[78]+a[77]' -> rapscallion
'a[54]+a[68]+a[75]+a[66]+a[78]+a[76]+a[68]+a[94]+a[65]+a[64]+a[66]+ ...' -> Welcome back... your flag, user:
'a[47]+a[75]+a[68]+a[64]+a[82]+a[68]+a[94]+a[68]+a[77]+a[83]+a[68]+ ...' -> Please enter correct password for flag: 
```

Five decoded strings:

- The **password** is `happychance`.
- The **rejection message** is `That password is incorrect`.
- The **XOR key** is `rapscallion`.
- The **success banner** is `Welcome back... your flag, user:`.
- The **input prompt** is `Please enter correct password for flag: `.

So the program is exactly the same shape as `patchme.py` — read `flag.txt.enc`, ask for the password, on a match XOR-decrypt with the key, print the flag.

### 4. Run `bloat.py` with the recovered password

```
┌──(zham㉿kali)-[~/bloat]
└─$ echo "happychance" | python3 bloat.py
Please enter correct password for flag: Welcome back... your flag, user:
picoCTF{d30bfu5c4710n_f7w_161a4f09}
```

The flag is `picoCTF{d30bfu5c4710n_f7w_161a4f09}`.

---

## Alternative Solves

**A. Skip the source — call the decoder routine directly.**
`arg122` (the XOR helper) and `arg111` (the key loader) are both in scope at the module level after the script defines them. We can replicate the decryption in one line without ever touching the password gate:

```
┌──(zham㉿kali)-[~/bloat]
└─$ python3 -c "
a = '!\"#\$%&\'()*+,-./0123456789:;<=>?@ABCDEFGHIJKLMNOPQRSTUVWXYZ[\\\\]^_\`abcdefghijklmnopqrstuvwxyz{|}~ '
key = a[81]+a[64]+a[79]+a[82]+a[66]+a[64]+a[75]+a[75]+a[72]+a[78]+a[77]
print('key =', key)
data = open('flag.txt.enc','rb').read().decode()
k = key
i = 0
while len(k) < len(data):
    k += key[i]
    i = (i+1) % len(key)
print(''.join(chr(ord(s)^ord(kk)) for s,kk in zip(data,k)))
"
key = rapscallion
picoCTF{d30bfu5c4710n_f7w_161a4f09}
```

This is the *no-source-required* path: read the key once from the obfuscated source (or hard-code it after you know it), then run only the XOR. Same flag, zero interaction.

**B. Patch the script to skip the password gate.**
Add `or True` to the `if` line so any input passes:

```
┌──(zham㉿kali)-[~/bloat]
└─$ cp bloat.py bloat.patched.py

┌──(zham㉿kali)-[~/bloat]
└─$ sed -i 's|^  if arg432 == |  if True or arg432 == |' bloat.patched.py

┌──(zham㉿kali)-[~/bloat]
└─$ echo "anything" | python3 bloat.patched.py
Please enter correct password for flag: Welcome back... your flag, user:
picoCTF{d30bfu5c4710n_f7w_161a4f09}
```

Identical flag, but the program never compared anything — the gate was bypassed.

**C. Brute-force the password character-by-character.**
The `arg133` gate returns at the first mismatch, so a binary-search brute for each of the 11 characters recovers the password in at most `11 * 95 = 1045` attempts. That is overkill here (the password is sitting in the source) but it is the right reflex for any "keygen-style" gate where you do not have the source.

---

## What Happened Internally

1. Python loaded `bloat.py` and evaluated the module-level `a = "!\"#$%&'()*+..."` definition, building a 95-character lookup string. Every other "string" in the file is a slice of this one.
2. The five functions were defined. None of them executed yet — `def` is just binding the function object to a name.
3. The last three lines ran in order:
   - `arg444 = arg132()` — opened `flag.txt.enc` and read the 35 bytes of ciphertext.
   - `arg432 = arg232()` — `input(...)` was called. Python printed the prompt `Please enter correct password for flag: ` and blocked waiting for stdin. We sent `happychance` on a line, so `arg432` became the string `happychance`.
   - `arg133(arg432)` — the password gate. `arg133` concatenated `a[71]+a[64]+a[79]+a[79]+a[88]+a[66]+a[71]+a[64]+a[77]+a[66]+a[68]` into the literal `happychance` and compared it against the input. The comparison was equal, so it returned `True`.
   - `arg112()` — printed `Welcome back... your flag, user:`.
   - `arg423 = arg111(arg444)` — called `arg111`, which called `arg122(arg444.decode(), "rapscallion")`. `arg122` extended the key `rapscallion` by repeating it until it matched the 35-byte ciphertext length, then XORed each byte pair to produce the plaintext flag bytes.
   - `print(arg423)` — wrote `picoCTF{d30bfu5c4710n_f7w_161a4f09}` to stdout.
   - `sys.exit(0)` — terminated the process cleanly.
4. The whole file is the same five-step recipe as `patchme.py`. The only difference is that *every* literal in the source is hidden behind a table lookup, and the function/argument names are renamed to `argXXX` for maximum visual noise.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `cat`            | Read `bloat.py` end to end and confirmed the obfuscation pattern. |
| `nano`           | Wrote the small `decode.py` helper that evaluates the table.   |
| `python3`        | Ran the decoder, ran `bloat.py`, and ran the alternative XOR one-liner. |
| `re` (regex)     | Pulled every `a[NNN]` index out of the source for decoding.    |
| `sed`            | Patched the password gate in `bloat.patched.py`.               |
| `echo + pipe`    | Sent the recovered password into `bloat.py` non-interactively. |
| `grep` / `wc -l` | Quick stats — file is 41 lines, 35-byte ciphertext.            |

---

## Key Takeaways

- String-table obfuscation (`a = "..."; expression = a[1]+a[2]+...`) hides nothing from anyone willing to evaluate the expressions. A 5-line Python helper that decodes every `a[NNN]` reference instantly turns the file back into readable English.
- "obfuscation" is a tag, not a security property. Treat obfuscated code like a magic trick: the *explanation* is right there, you just have to look at it the right way. The first reflex should be "what does this evaluate to?" — not "how do I reverse-engineer the algorithm?"
- Compare this challenge to `patchme.py`: same structure, same XOR helper, same prompt-and-gate flow. The difference is purely cosmetic. Once you have seen one, you have seen both.
- When you find the password and key in the source, *verify* by running the program. Knowing the literals is not the same as having a working answer — running it proves the gate accepts the password and the XOR key is correct.
- The flag's leet: `d30bfu5c4710n_f7w` reads as **deobfuscation_few** (`3→e`, `0→o`, `5→s`, `4→a`, `7→t`, `1→i`, `0→o`, `7→e`). The flag's wordplay is the whole lesson: a *few* lines of deobfuscation — literally our 5-line `decode.py` — is all you need to crack the file.
