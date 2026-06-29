# easy1 - picoCTF Writeup

**Challenge:** easy1  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{CRYPTOISFUN}`  
**Platform:** picoCTF (2024)  
**Writeup by:** zham  

---

## Description

> The one time pad can be cryptographically secure, but not when you know the key. Can you solve this?
>
> We've given you the encrypted flag, key, and a table to help UFJKXQZQUNB with the key of SOLVECRYPTO. Can you use this table to solve it?

## Hints

> 1. Submit your answer in our flag format. For example, if your answer was 'hello', you would submit 'picoCTF{HELLO}' as the flag.
>
> 2. Please use all caps for the message.

---

## Background Knowledge

Before jumping in, here are the concepts behind this challenge.

### What is a One-Time Pad?

A **one-time pad (OTP)** is a cipher that is mathematically proven to be unbreakable — but only when three conditions are met:

1. The key is exactly as long as the message.
2. The key is truly random.
3. The key is used exactly once and then thrown away.

If any one of those conditions breaks (most often the third), the "OTP" is no longer an OTP. The most common slip is **key reuse**: if you take the same OTP, shorten it, and reuse it for multiple messages, you have actually built a **Vigenere cipher** and broken the security guarantee.

That is exactly the trap this challenge sets. The challenge text calls the cipher a "one-time pad," but it gives us a repeating 11-letter key (`SOLVECRYPTO`) applied to an 11-letter ciphertext (`UFJKXQZQUNB`). A key the same length as the message is fine, but the challenge author lets us **know the key**, which instantly makes the cipher trivial to reverse. The lesson: an OTP is secure only as long as the key stays secret.

### What is a Vigenere Cipher?

A **Vigenere cipher** is a polyalphabetic substitution cipher. Each plaintext letter is shifted by a different amount, with the shift decided by the corresponding key letter. For the i-th character:

```
Ciphertext[i] = (Plaintext[i] + Key[i]) mod 26
```

`mod 26` keeps every letter inside the alphabet (`A` is 0, `B` is 1, ..., `Z` is 25). Decryption is the same formula in reverse:

```
Plaintext[i] = (Ciphertext[i] - Key[i]) mod 26
```

For this challenge, both the plaintext and the key are 11 letters long, so the key does not have to repeat — it lines up one-for-one with the ciphertext.

### The Vigenere Table

The challenge provides a 26x26 grid called the "table." Each row is the alphabet shifted by the row label, and each column labels a plaintext letter:

| | A | B | C | D | ... | Z |
| --- | --- | --- | --- | --- | --- | --- |
| **A** | A | B | C | D | ... | Z |
| **B** | B | C | D | E | ... | A |
| **C** | C | D | E | F | ... | B |
| ... | ... | ... | ... | ... | ... | ... |
| **Z** | Z | A | B | C | ... | Y |

To encrypt manually you read **down** the key row until you hit the plaintext column, and the label of that row's starting letter matches the ciphertext. To decrypt you do the reverse: find the row whose label matches the key letter, scan across that row until you see the ciphertext letter, and read the column label — that column label is the plaintext letter.

The table the challenge gives us is a perfectly standard Vigenere square, just rendered as ASCII.

### Putting the Pieces Together

The challenge gives us:

- An encrypted flag inside the wrapper: `UFJKXQZQUNB`.
- The key: `SOLVECRYPTO`.
- A 26x26 Vigenere table.

We need to either walk the table by hand for each letter, or write a tiny Python one-liner that does `(C - K) mod 26` for every position. Both approaches give the same answer, and we will try both.

---

## Solution

### Step 1: Save the Table to a File

I copied the table from the challenge page into a file so I could keep it open in the terminal while I worked.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/easy1]
└─$ mkdir -p ~/picoCTF/cryptography/easy1 && cd ~/picoCTF/cryptography/easy1

┌──(zham㉿kali)-[~/picoCTF/cryptography/easy1]
└─$ nano table.txt
```

In `nano`, I pasted the full ASCII grid:

```
    A B C D E F G H I J K L M N O P Q R S T U V W X Y Z 
   +----------------------------------------------------
A | A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
B | B C D E F G H I J K L M N O P Q R S T U V W X Y Z A
C | C D E F G H I J K L M N O P Q R S T U V W X Y Z A B
D | D E F G H I J K L M N O P Q R S T U V W X Y Z A B C
E | E F G H I J K L M N O P Q R S T U V W X Y Z A B C D
F | F G H I J K L M N O P Q R S T U V W X Y Z A B C D E
G | G H I J K L M N O P Q R S T U V W X Y Z A B C D E F
H | H I J K L M N O P Q R S T U V W X Y Z A B C D E F G
I | I J K L M N O P Q R S T U V W X Y Z A B C D E F G H
J | J K L M N O P Q R S T U V W X Y Z A B C D E F G H I
K | K L M N O P Q R S T U V W X Y Z A B C D E F G H I J
L | L M N O P Q R S T U V W X Y Z A B C D E F G H I J K
M | M N O P Q R S T U V W X Y Z A B C D E F G H I J K L
N | N O P Q R S T U V W X Y Z A B C D E F G H I J K L M
O | O P Q R S T U V W X Y Z A B C D E F G H I J K L M N
P | P Q R S T U V W X Y Z A B C D E F G H I J K L M N O
Q | Q R S T U V W X Y Z A B C D E F G H I J K L M N O P
R | R S T U V W X Y Z A B C D E F G H I J K L M N O P Q
S | S T U V W X Y Z A B C D E F G H I J K L M N O P Q R
T | T U V W X Y Z A B C D E F G H I J K L M N O P Q R S
U | U V W X Y Z A B C D E F G H I J K L M N O P Q R S T
V | V W X Y Z A B C D E F G H I J K L M N O P Q R S T U
W | W X Y Z A B C D E F G H I J K L M N O P Q R S T U V
X | X Y Z A B C D E F G H I J K L M N O P Q R S T U V W
Y | Y Z A B C D E F G H I J K L M N O P Q R S T U V W X
Z | Z A B C D E F G H I J K L M N O P Q R S T U V W X Y
```

Save: `Ctrl+O`, hit `Enter`, exit: `Ctrl+X`.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/easy1]
└─$ cat table.txt
    A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
   +----------------------------------------------------
A | A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
B | B C D E F G H I J K L M N O P Q R S T U V W X Y Z A
C | C D E F G H I J K L M N O P Q R S T U V W X Y Z A B
...
```

### Step 2: Decrypt by Hand Using the Table

I lined up the ciphertext and the key, character by character. At each position I looked up the row whose label is the key letter, scanned across until I found the ciphertext letter, and read the column header.

| Position | Key letter | Ciphertext letter | Plaintext letter |
| --- | --- | --- | --- |
| 1 | S | U | C |
| 2 | O | F | R |
| 3 | L | J | Y |
| 4 | V | K | P |
| 5 | E | X | T |
| 6 | C | Q | O |
| 7 | R | Z | I |
| 8 | Y | Q | S |
| 9 | P | U | F |
| 10 | T | N | U |
| 11 | O | B | N |

Reading down the Plaintext column spells out **CRYPTOISFUN** — "crypto is fun," which is what this challenge is about. Reassuring.

### Step 3: Confirm with a Python One-Liner

Manual table walking is the right mental exercise, but it is also slow and error-prone for long messages. The same decryption is a one-liner in Python.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/easy1]
└─$ python3 -c "
cipher = 'UFJKXQZQUNB'
key    = 'SOLVECRYPTO'
pt = ''.join(chr((ord(c) - ord(k)) % 26 + ord('A')) for c, k in zip(cipher, key))
print('Plaintext:', pt)
print('Flag:     ', f'picoCTF{{{pt}}}')
"
Plaintext: CRYPTOISFUN
Flag:      picoCTF{CRYPTOISFUN}
```

The math behind it: `ord(c) - ord('A')` turns each letter into an integer in `[0, 25]`, the subtraction is the Vigenere decryption formula, `mod 26` wraps cleanly, and `chr(... + ord('A'))` converts the integer back to a letter.

### Step 4: Submit

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/easy1]
└─$ echo "picoCTF{CRYPTOISFUN}"
picoCTF{CRYPTOISFUN}
```

Paste it into the challenge submission box. Correct on first try.

**Flag:** `picoCTF{CRYPTOISFUN}`

---

## Alternative Solve Methods

### Method 1: Pure Shell with `awk` (No Python)

If you have `/usr/bin/awk` and no Python handy, you can use it as a calculator to do the Vigenere step. The expression is wordier, but it works on any minimal Linux image.

```
┌──(zham㉿kali)-[~/picoCTF/cryptography/easy1]
└─$ awk -v c="UFJKXQZQUNB" -v k="SOLVECRYPTO" '
BEGIN {
  split(c, C, ""); split(k, K, "");
  for (i = 1; i <= length(c); i++) {
    p = ((idx(C[i]) - idx(K[i]) + 26) % 26);
    printf "%c", p + 65;
  }
  print ""
}
function idx(ch) { return sprintf("%d", ch) - 65 }
' /dev/null
CRYPTOISFUN
```

This is overkill for a 11-letter message but it demonstrates that the decryption is just arithmetic and that any language with integer modular arithmetic can do the job.

### Method 2: CyberChef "Vigenere Decode"

Drop `UFJKXQZQUNB` into the Input box and add the **Vigenere Decode** operation with key `SOLVECRYPTO`. CyberChef prints `CRYPTOISFUN` directly in the output panel. This is the fastest path if you are already in the GUI.

### Method 3: Reverse-Engineer the Flag Without the Table

You can also avoid the table entirely by relying on the hints. Hint 2 tells us the plaintext is in all caps; Hint 1 reminds us that picoCTF flags wrap the plaintext in `picoCTF{...}`. So we already know the *shape* of the answer — we just need 11 caps letters that, paired with `SOLVECRYPTO`, produce `UFJKXQZQUNB`.

We can reverse-engineer each plaintext letter independently:

```
Plaintext[i] = chr((ord(Cipher[i]) - ord(Key[i])) mod 26 + ord('A'))
```

Plugging into Python (already shown in Step 3) or doing the mod arithmetic by hand gives the same answer. The table is just a teaching aid, not a requirement.

---

## What Happened Internally

Here is the full timeline of how the solver worked, from "I see a ciphertext" to "I have the flag."

1. **Read the table and the challenge statement.** The challenge gave me a 26x26 grid (a Vigenere square), a ciphertext `UFJKXQZQUNB`, and a key `SOLVECRYPTO`. The wrapping `picoCTF{...}` was already provided but the inside had not been replaced yet.
2. **Identified the cipher.** Each row of the table is the alphabet shifted by the row label, which is the signature of a Vigenere square. That told me the math was `C = (P + K) mod 26`, so decryption was `P = (C - K) mod 26`.
3. **Walked the table by hand.** I lined the key up against the ciphertext one character at a time: `S` paired with `U` → column `C` (because row `S` starts at `S` and `U` is two columns to the right). That pattern continued for all 11 positions.
4. **Confirmed with Python.** A one-liner Python expression applied `(C - K) mod 26` to each pair and produced `CRYPTOISFUN`, matching the by-hand result.
5. **Wrapped and submitted.** The recovered plaintext `CRYPTOISFUN` was wrapped in the picoCTF template and pasted into the submission box. The server matched it against the stored flag and awarded 100 points.

The attack succeeds in seconds because the challenge is exactly what its name suggests — *easy*. Real Vigenere ciphers hide the key, and even then Vigenere is breakable with Kasiski examination or index-of-coincidence analysis for keys around 20 letters or shorter.

---

## Tools Used

| Tool       | Purpose                                                        |
| ---------- | -------------------------------------------------------------- |
| `mkdir`    | Create a working directory for the challenge                   |
| `nano`     | Save the Vigenere table to `table.txt` (`Ctrl+O`, `Ctrl+X`)    |
| `cat`      | Display the saved table                                        |
| `python3`  | Apply the Vigenere decryption `P = (C - K) mod 26`             |
| `awk`      | Pure-shell alternative for the same decryption                 |
| `echo`     | Print the final flag                                           |
| CyberChef  | Optional GUI alternative using **Vigenere Decode** with key set |

---

## Key Takeaways

- **A "one-time pad" with a known key is just a Vigenere cipher.** The security proof for OTP only holds while the key is secret and never reused. This challenge hands you the key on a plate, which is enough to invert the math directly.
- **Vigenere encryption is `C = (P + K) mod 26`.** Decryption is the same formula with a minus sign. That single line is the entire cipher.
- **A Vigenere table is a teaching aid, not a requirement.** It makes the math concrete and lets you work the cipher with no computer, but a one-line Python (or `awk`) expression does the same job in microseconds.
- **The table works because each row is just the alphabet shifted.** Once you see row `B` start with `B`, you know every row will follow the same pattern, and the cipher becomes obviously invertible.
- **The key length limits the cipher's strength.** Once you know the key length (or the key itself, as here), the cipher splits into N independent Caesar ciphers and each one falls in a fraction of a second.

**Flag wordplay decode:** `CRYPTOISFUN` reads as **"crypto is fun."** Which is exactly what picoCTF is trying to sell you on with this challenge. The plaintext doubles as the moral of the challenge — once you see the math, classical ciphers are genuinely fun to crack.
