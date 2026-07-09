# patchme.py — picoCTF Writeup

**Challenge:** patchme.py  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{p47ch1ng_l1f3_h4ck_21d62e33}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Can you get the flag?
>
> Run this `Python program` in the same directory as this `encrypted flag`.

## Hints

(Hint panel did not reveal any explicit hint for this instance — the challenge title itself is the hint: *patch* the script.)

---

## Background Knowledge

A few concepts worth understanding before we look at the script.

**1. What "patching" a script actually means.**
"Patching" is a soft term for *editing a program's source or binary to change its behavior*. In Reverse Engineering challenges, you usually do not need to crack a hash or reverse an algorithm — you simply nudge the program so it takes the path you want. Adding `or True` to an `if` statement, hard-coding a return value, or deleting a check entirely are all valid patches. The challenge is called `patchme.py`, so the intended approach is to edit the file.

**2. Python `input()` and string concatenation.**
`input("prompt")` reads one line from stdin and returns it as a Python string. The script then compares what you typed to a literal string. The literal here is built by concatenating four pieces with `+`:
`"ak98" + "-=90" + "adfjhgj321" + "sleuth9000"` = `ak98-=90adfjhgj321sleuth9000`. Even if you never type the password, simply reading the source tells you what the password is. That is one valid "solve" without patching anything.

**3. XOR with a repeating key.**
The decryption helper `str_xor(secret, key)` extends the short key by repeating it until it is the same length as the ciphertext, then XORs each byte pair. The key here is `"utilitarian"`. XOR is its own inverse, so the *exact same function* is used both to encrypt the flag and to decrypt it. Knowing the key (which is hard-coded in the source) is enough to recover the plaintext.

**4. `chr(ord(a) ^ ord(b))` — manual byte XOR.**
Python's `^` is bitwise XOR on integers, so the function converts each character to its code point with `ord`, XORs the two integers, and turns the result back into a character with `chr`. This is how XOR "encryption" is normally written in pure Python without any external library.

**5. Why patching is faster than guessing the password.**
Both routes work, but patching demonstrates the deeper lesson: in real-world software, an attacker who can modify the binary (or the script running on a server) does not need to know the secret at all. They simply remove the gate. That mindset — *control the code, not the data* — is the heart of this challenge.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. After downloading the challenge I had two files: `patchme.py` and `flag.txt.enc`.

### 1. Make a working directory and inspect the script

```
┌──(zham㉿kali)-[~/patchme]
└─$ mkdir -p ~/patchme && cd ~/patchme

┌──(zham㉿kali)-[~/patchme]
└─$ cp ~/Downloads/patchme.py .
┌──(zham㉿kali)-[~/patchme]
└─$ cp ~/Downloads/flag.txt.enc .

┌──(zham㉿kali)-[~/patchme]
└─$ cat patchme.py
```

The full source is short. The important parts:

```python
flag_enc = open('flag.txt.enc', 'rb').read()

def level_1_pw_check():
    user_pw = input("Please enter correct password for flag: ")
    if( user_pw == "ak98" + \
                   "-=90" + \
                   "adfjhgj321" + \
                   "sleuth9000"):
        print("Welcome back... your flag, user:")
        decryption = str_xor(flag_enc.decode(), "utilitarian")
        print(decryption)
        return
    print("That password is incorrect")

level_1_pw_check()
```

So the program reads `flag.txt.enc`, asks for a password, and only if the password matches does it XOR-decrypt the file and print the flag.

### 2. The intended solve: patch the script

The challenge is literally called `patchme.py`, so the cleanest solve is to edit the `if` line so the check always succeeds. I keep a backup first.

```
┌──(zham㉿kali)-[~/patchme]
└─$ cp patchme.py patchme.py.bak
```

Now patch. The line `if( user_pw == "ak98" + \` becomes `if( True or user_pw == "ak98" + \`. The `or True` short-circuits the comparison and the block is entered no matter what we type.

I used `sed` for a one-liner patch:

```
┌──(zham㉿kali)-[~/patchme]
└─$ sed -i 's|^    if( user_pw ==|    if( True or user_pw ==|' patchme.py
```

Verify the patch landed on the right line:

```
┌──(zham㉿kali)-[~/patchme]
└─$ sed -n '20,26p' patchme.py
def level_1_pw_check():
    user_pw = input("Please enter correct password for flag: ")
    if( True or user_pw == "ak98" + \
                   "-=90" + \
                   "adfjhgj321" + \
                   "sleuth9000"):
        print("Welcome back... your flag, user:")
```

If you prefer `nano`, open the file, change line 22 from
`    if( user_pw == "ak98" + \`
to
`    if( True or user_pw == "ak98" + \`
then `Ctrl+O`, `Enter`, `Ctrl+X` to save and exit.

### 3. Run the patched script

Any input will now be accepted because the check short-circuits. I pipe `anything` in just so the prompt is satisfied:

```
┌──(zham㉿kali)-[~/patchme]
└─$ echo "anything" | python3 patchme.py
Please enter correct password for flag: Welcome back... your flag, user:
picoCTF{p47ch1ng_l1f3_h4ck_21d62e33}
```

The flag is `picoCTF{p47ch1ng_l1f3_h4ck_21d62e33}`.

---

## Alternative Solves

**A. Skip the patch — read the password from the source and supply it.**
The literal is just four concatenated strings, so the password sits in the source in plain text. Concat them by hand (or with Python) and feed it in:

```
┌──(zham㉿kali)-[~/patchme]
└─$ python3 -c 'print("ak98" + "-=90" + "adfjhgj321" + "sleuth9000")'
ak98-=90adfjhgj321sleuth9000

┌──(zham㉿kali)-[~/patchme]
└─$ echo "ak98-=90adfjhgj321sleuth9000" | python3 patchme.py.bak
Please enter correct password for flag: Welcome back... your flag, user:
picoCTF{p47ch1ng_l1f3_h4ck_21d62e33}
```

Same flag, no file edits. I used the original `patchme.py.bak` here so the unmodified script is the one producing the output.

**B. Skip the password check entirely by calling the decrypt code directly.**
You do not actually need the password routine at all. Open `flag.txt.enc`, call `str_xor` with the same key, and print the result. This is the "I read the source so I will re-implement just the useful bits" approach:

```
┌──(zham㉿kali)-[~/patchme]
└─$ python3 -c "
flag_enc = open('flag.txt.enc','rb').read()
secret = flag_enc.decode()
key = 'utilitarian'
new_key = key
i = 0
while len(new_key) < len(secret):
    new_key += key[i]
    i = (i + 1) % len(key)
print(''.join(chr(ord(s) ^ ord(k)) for s, k in zip(secret, new_key)))
"
picoCTF{p47ch1ng_l1f3_h4ck_21d62e33}
```

This is overkill for a 100-point challenge, but it is a good habit: when a script guards a routine you actually want, just call the routine yourself.

---

## What Happened Internally

1. Python loaded `patchme.py` and ran the module-level line `flag_enc = open('flag.txt.enc', 'rb').read()`. The 36 bytes of the encrypted flag were stored in the variable `flag_enc`.
2. The script defined two functions: `str_xor(secret, key)` (a repeating-key XOR helper) and `level_1_pw_check()` (the gate).
3. The last line of the file, `level_1_pw_check()`, called the gate function.
4. Inside the gate, `input("Please enter correct password for flag: ")` printed the prompt and read one line from stdin. With the patch, whatever we typed was bound to `user_pw`.
5. The condition `if( True or user_pw == "ak98" + "-=90" + "adfjhgj321" + "sleuth9000"):` evaluated `True or <anything>`, which short-circuits to `True` before the comparison ever runs. The body was entered.
6. `"Welcome back... your flag, user:"` was printed.
7. `str_xor(flag_enc.decode(), "utilitarian")` extended the key `"utilitarian"` by repeating it until it matched the length of the decoded ciphertext, then XORed each byte pair to produce the plaintext flag bytes.
8. `print(decryption)` wrote those bytes to stdout, which is where the flag `picoCTF{p47ch1ng_l1f3_h4ck_21d62e33}` appeared.
9. The `return` inside the `if` block exited `level_1_pw_check()` and the script ended cleanly.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `cat`            | Read the entire `patchme.py` source in one go.                 |
| `cp`             | Made a `patchme.py.bak` backup before patching.                |
| `sed`            | In-place one-line patch on the `if` condition.                 |
| `nano` (alt)     | Alternative way to make the same edit interactively.           |
| `python3`        | Ran the patched script and confirmed the flag.                 |
| `echo + pipe`    | Sent arbitrary input into the patched script non-interactively. |
| `grep / sed -n`  | Sanity-checked which lines were actually changed.              |

---

## Key Takeaways

- The challenge name is a real hint: when a problem says "patch me," editing the program is the intended solve. Look for the smallest change that flips the program's behavior.
- A short-circuit `or True` / `and False` is the cleanest Python patch for any `if` guard. It avoids renaming variables, deleting lines, or breaking indentation.
- Always make a backup (`cp file file.bak`) before patching. If the first edit breaks the script, you can re-run it against the untouched source to confirm your patch is what produced the answer.
- "Guarded" code is rarely the *only* way to reach the result. If a script does `if check: do_secret()`, you can usually call `do_secret()` directly or copy its body into your own one-liner.
- The flag's leet: `p47ch1ng_l1f3_h4ck` reads as **patching life hack** (`4→a`, `7→t`, `1→i`, `3→e`). The challenge is called *patchme*, the flag's wordplay is *patching life hack* — the solve *is* the punchline: a tiny edit to the source is the "life hack" that unlocks the flag.
