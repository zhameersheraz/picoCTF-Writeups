# Substitution 0 — picoCTF Writeup

**Challenge:** Substitution 0  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{5UB5717U710N_3V0LU710N_03055505}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> A message has come in but it seems to be all scrambled. Luckily it seems to have the key at the beginning. Can you crack this substitution cipher?
>
> Download the message [here](https://artifacts.picoctf.net/c/154/message.txt).

## Hints

> 1. Try a frequency attack. An online tool might help.

---

## Background Knowledge

A **substitution cipher** is one of the simplest classical ciphers. Each letter in the original message (the *plaintext*) is swapped for another letter, and the same swap is used everywhere in the message. The scrambled output is called the *ciphertext*.

The whole cipher is described by a single 26-letter **key** (also called the *cipher alphabet*). If I write the key underneath the normal alphabet, every substitution becomes a one-to-one lookup:

```
Plain :  A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Cipher:  O H N F U M W S V Z L X E G C P T A J D Y I R K Q B
```

So `A` becomes `O`, `B` becomes `H`, `C` becomes `N`, and so on. The person who receives the message reverses the same table to read it.

To **decrypt** a substitution cipher you have to recover that table. There are three common ways, going from easiest to hardest:

1. **Given the key.** The key (cipher alphabet) is handed to you. You just build the lookup table and decode. This is what Substitution 0 does.
2. **Frequency analysis.** You count how often each letter shows up in the ciphertext, compare those counts to known English letter frequencies (`e` is most common, then `t`, `a`, `o`, `i`...), and guess the mapping.
3. **Known-plaintext attack.** You already know some of the plaintext. picoCTF flags always start with `picoCTF{`, so that prefix instantly gives you seven mappings for free.

In this challenge the key is right at the top of the file, so I only need method 1. The hint about a "frequency attack" is a bit of a red herring — it is more useful for Substitution 1, where the key is missing.

---

## Step-by-Step Solution

### Step 1 — Save the message

I created a working directory and dropped the downloaded ciphertext into `message.txt`.

```bash
┌──(zham㉿kali)-[~/picoctf/substitution0]
└─$ mkdir -p ~/picoctf/substitution0 && cd ~/picoctf/substitution0
```

I pasted the ciphertext (everything from `OHNFUMWSVZLXEGCPTAJDYIRKQB` to the closing `}`) into `message.txt`.

### Step 2 — Look at the file

Before I do anything clever I just peek at the file.

```bash
┌──(zham㉿kali)-[~/picoctf/substitution0]
└─$ cat message.txt
OHNFUMWSVZLXEGCPTAJDYIRKQB 

Suauypcg Xuwaogf oacju, rvds o waoiu ogf jdoduxq ova, ogf hacywsd eu dsu huudxu
mace o wxojj noju vg rsvns vd roj ugnxcjuf. Vd roj o huoydvmyx jnoaohouyj, ogf, od
dsod dveu, yglgcrg dc godyaoxvjdj—cm ncyaju o wauod pavbu vg o jnvugdvmvn pcvgd
cm ivur. Dsuau ruau drc acygf hxonl jpcdj guoa cgu ukdauevdq cm dsu honl, ogf o
xcgw cgu guoa dsu cdsua. Dsu jnoxuj ruau uknuufvgwxq soaf ogf wxcjjq, rvds oxx dsu
oppuoaognu cm hyagvjsuf wcxf. Dsu ruvwsd cm dsu vgjund roj iuaq aueoalohxu, ogf,
dolvgw oxx dsvgwj vgdc ncgjvfuaodvcg, V ncyxf soafxq hxoeu Zypvdua mca svj cpvgvcg
aujpundvgw vd.

Dsu mxow vj: pvncNDM{5YH5717Y710G_3I0XY710G_03055505}
```

The very first line is exactly 26 letters long and it is followed by a blank line, then a scrambled paragraph, then a line that ends in a brace. That first line is screaming "I am the key."

### Step 3 — Confirm the key length

```bash
┌──(zham㉿kali)-[~/picoctf/substitution0]
└─$ head -1 message.txt | tr -d '\n' | wc -c
26
```

Twenty-six characters. That matches the size of the English alphabet, so the line really is a one-to-one mapping from `A`–`Z` to the letters above.

### Step 4 — Build the lookup table

The first line tells me the encryption alphabet. Aligning it under `A`–`Z`:

| Plain  | A | B | C | D | E | F | G | H | I | J | K | L | M | N | O | P | Q | R | S | T | U | V | W | X | Y | Z |
| ------ | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - |
| Cipher | O | H | N | F | U | M | W | S | V | Z | L | X | E | G | C | P | T | A | J | D | Y | I | R | K | Q | B |

To **decrypt**, I read the table the other way: `O -> A`, `H -> B`, `N -> C`, `F -> D`, ... `B -> Z`. Doing it by hand for every letter is tedious, so I drop the table into a tiny Python script.

### Step 5 — Write the decoder in nano

```bash
┌──(zham㉿kali)-[~/picoctf/substitution0]
└─$ nano solve.py
```

In nano I pasted:

```python
#!/usr/bin/env python3
"""Decode a substitution cipher when the key alphabet is given."""
import sys, re

# Cipher alphabet from line 1 of the message.
cipher_alpha = "OHNFUMWSVZLXEGCPTAJDYIRKQB"
plain_alpha  = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"

with open(sys.argv[1]) as f:
    raw = f.read()

# Skip the first (key) line, keep the rest.
body = raw.split("\n", 1)[1]

# Build a translation table that handles both upper and lower case,
# leaving digits, punctuation and whitespace untouched.
trans = {}
for c, p in zip(cipher_alpha, plain_alpha):
    trans[c] = p
    trans[c.lower()] = p.lower()

plain = "".join(trans.get(ch, ch) for ch in body)
print(plain)

flag = re.search(r"picoCTF\{[^}]+\}", plain)
if flag:
    print("\nFLAG:", flag.group(0))
```

Save and exit nano: `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 6 — Run the decoder

```bash
┌──(zham㉿kali)-[~/picoctf/substitution0]
└─$ python3 solve.py message.txt
Hereupon Legrand arose, with a grave and stately air, and brought me the beetle
from a glass case in which it was enclosed. It was a beautiful scarabaeus, and, at
that time, unknown to naturalists—of course a great prize in a scientific point
of view. There were two round black spots near one extremity of the back, and a
long one near the other. The scales were exceedingly hard and glossy, with all the
appearance of burnished gold. The weight of the insect was very remarkable, and,
taking all things into consideration, I could hardly blame Jupiter for his opinion
respecting it.

The flag is: picoCTF{5UB5717U710N_3V0LU710N_03055505}

FLAG: picoCTF{5UB5717U710N_3V0LU710N_03055505}
```

The decoded body is the opening paragraph of Edgar Allan Poe's short story **"The Gold-Bug"** (1843). The flag is sitting at the bottom of the file.

### Step 7 — Submit

```bash
┌──(zham㉿kali)-[~/picoctf/substitution0]
└─$ echo "picoCTF{5UB5717U710N_3V0LU710N_03055505}"
picoCTF{5UB5717U710N_3V0LU710N_03055505}
```

I pasted `picoCTF{5UB5717U710N_3V0LU710N_03055505}` into the picoCTF submission box and the challenge turned green.

---

## Alternative Solve — CyberChef "Substitute" recipe

If you do not want to write any code, [CyberChef](https://gchq.github.io/CyberChef/) has a built-in **Substitute** operation that does the same job.

1. Open CyberChef and paste the ciphertext (everything from the second line down) into the **Input** box.
2. Drag the **Substitute** operation into the recipe.
3. In the **Substitution** field type the cipher alphabet `OHNFUMWSVZLXEGCPTAJDYIRKQB`.
4. In the **Replace** field type `ABCDEFGHIJKLMNOPQRSTUVWXYZ`.
5. The **Output** box shows the same Poe paragraph and the flag `picoCTF{5UB5717U710N_3V0LU710N_03055505}` at the bottom.

CyberChef is honestly faster for a single-shot decode like this. Writing the script is still worth doing once because it forces you to understand exactly what a substitution cipher is doing under the hood.

---

## What Happened Internally

| # | Step                                | What the cipher / solver did                                                                                  |
| - | ----------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| 1 | Read `message.txt`                  | Pulled in 7 lines: a 26-character key, a blank line, 7 lines of ciphertext, and a flag line.                  |
| 2 | Confirmed the first line is a key   | `wc -c` on line 1 returned 26 — exactly the size of the English alphabet.                                      |
| 3 | Built the inverse mapping           | Paired `O H N F U M W S V Z L X E G C P T A J D Y I R K Q B` with `A B C D E F G H I J K L M N O P Q R S T U V W X Y Z`. |
| 4 | Skipped the key line                | `body = raw.split("\n", 1)[1]` dropped line 1 and kept the cipher body.                                       |
| 5 | Per-character lookup                | For each char in the body, looked up the inverse table; uppercase hit the uppercase map, lowercase hit the lowercase map, everything else was passed through. |
| 6 | Recovered plaintext                 | Walked through every character and produced the "Gold-Bug" paragraph verbatim.                                |
| 7 | Regex for the flag                  | `picoCTF\{[^}]+\}` matched `picoCTF{5UB5717U710N_3V0LU710N_03055505}`.                                        |
| 8 | Submission                          | Pasted the flag into the picoCTF submission box.                                                              |

---

## Tools Used

| Tool          | Purpose                                                                          |
| ------------- | -------------------------------------------------------------------------------- |
| `mkdir` / `cd` | Create a clean working directory for the challenge.                             |
| `cat`         | Inspect the ciphertext file.                                                     |
| `wc -c`       | Confirm the first line is 26 characters long (i.e. a real cipher alphabet).      |
| `nano`        | Write the decoder script `solve.py`.                                             |
| `python3`     | Build the inverse mapping, translate the ciphertext, and regex out the flag.     |
| CyberChef     | Alternative no-code solve using the Substitute operation.                       |

---

## Key Takeaways

- **If the key is given, the cipher is already broken.** Substitution 0 is a good reminder that the *security* of a substitution cipher lives entirely in the secrecy of its key. Hand the attacker the key and there is nothing left to do.
- **Decryption is just the inverse of encryption.** Same table, read the other way. Once you see the alignment `A B C ... Z` over `O H N ... B`, you already have everything you need.
- **Self-consistency checks catch mistakes.** Counting 26 characters on line 1 is a one-line way to make sure the line you think is the key really is the key. Skip that and you can spend twenty minutes wondering why the decryption is garbage.
- **Skip the key line before translating.** If you feed the cipher alphabet into your own translator, every letter in that line becomes garbage and it buries any errors you might want to see. `split("\n", 1)[1]` is the simplest way to drop it.
- **CyberChef is great for quick wins.** When the cipher is this straightforward, a no-code recipe is faster than a script. Keep both options in your back pocket.

### Flag wordplay decode

The flag `picoCTF{5UB5717U710N_3V0LU710N_03055505}` is leetspeak for **SUBSTITUTION EVOLUTION**:

```
5UB5717U710N   ->   SUBSTITUTION       (5=S, 7=T, 1=I, 0=O)
3V0LU710N      ->   EVOLUTION          (3=E, 0=O, 7=T, 1=I, 0=O)
03055505       ->   OEOSSSOS           (0=O, 3=E, 5=S) — playful extra suffix
```

So the flag reads "**SUBSTITUTION EVOLUTION**" — a tongue-in-cheek nod to the fact that even the very simplest classical cipher only "evolves" if you actually keep the key secret. Hand it out and the cipher collapses on its own.
