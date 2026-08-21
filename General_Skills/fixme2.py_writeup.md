# fixme2.py — picoCTF Writeup

**Challenge:** fixme2.py  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** 100 pts  
**Flag:** `picoCTF{3qu4l1ty_n0t_4551gnm3nt_e8814d03}`  
**Platform:** picoMini 2022  
**Writeup by:** zham

---

## Description

> Fix the syntax error in the Python script to print the flag.

**Attachment:** fixme2.py (Python source code)

---

## Hints

> 1. Are equality and assignment the same symbol?
> 2. To view the file in the webshell, do: `$ nano fixme2.py`
> 3. To exit nano, press Ctrl and x and follow the on-screen prompts.
> 4. The str_xor function does not need to be reverse engineered for this challenge.

---

## Background Knowledge (Read This First!)

### Python: Assignment vs Comparison

Python uses two different operators that look similar:

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | **Assignment** — store a value in a variable | `flag = ""` (set flag to empty string) |
| `==` | **Equality** — compare two values, returns `True` or `False` | `flag == ""` (check if flag is empty) |

In `if` statements and `while` loops, you almost always want `==` (or `!=`), not `=`. Using `=` inside an `if` is a common beginner mistake.

In Python, this mistake is a **SyntaxError** because Python parses `if flag = "":` as trying to assign to `flag`, which is not a valid statement. C and JavaScript would silently accept it (assigning the empty string, which is falsy, so the if-body runs), but Python catches it at parse time.

### Reading Python Syntax Errors

A `SyntaxError` traceback shows:
1. The file name and line number
2. A `^` arrow pointing to the problematic token
3. The category of error (e.g., `invalid syntax`)

Always check the line number and the `^` arrow to find the exact problem.

---

## Solution

### Step 1 — Download the challenge file

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ wget <challenge_url> -O fixme2.py
└─$ cat fixme2.py
```

The file contents:

```python
import random



def str_xor(secret, key):
    #extend key to secret length
    new_key = key
    i = 0
    while len(new_key) < len(secret):
        new_key = new_key + key[i]
        i = (i + 1) % len(key)        
    return "".join([chr(ord(secret_c) ^ ord(new_key_c)) for (secret_c,new_key_c) in zip(secret,new_key)])


flag_enc = chr(0x15) + chr(0x07) + chr(0x08) + chr(0x06) + chr(0x27) + chr(0x21) + chr(0x23) + chr(0x15) + chr(0x58) + chr(0x18) + chr(0x11) + chr(0x41) + chr(0x09) + chr(0x5f) + chr(0x1f) + chr(0x10) + chr(0x3b) + chr(0x1b) + chr(0x55) + chr(0x1a) + chr(0x34) + chr(0x5d) + chr(0x51) + chr(0x40) + chr(0x54) + chr(0x09) + chr(0x05) + chr(0x04) + chr(0x57) + chr(0x1b) + chr(0x11) + chr(0x31) + chr(0x0e) + chr(0x51) + chr(0x5c) + chr(0x44) + chr(0x51) + chr(0x0a) + chr(0x5b) + chr(0x5a) + chr(0x19)

  
flag = str_xor(flag_enc, 'enkidu')

# Check that flag is not empty
if flag = "":
  print('String XOR encountered a problem, quitting.')
else:
  print('That is correct! Here\'s your flag: ' + flag)
```

The bug is on **line 22**: `if flag = "":` uses `=` (assignment) instead of `==` (comparison).

### Step 2 — Try to run the script and see the error

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fixme2.py
  File "fixme2.py", line 22
    if flag = "":
           ^
SyntaxError: invalid syntax. Maybe you meant '==' instead of '='?
```

Python is even helpful here — it suggests using `==` instead of `=`! The `^` points right at the `=` that should be `==`.

### Step 3 — Fix the comparison

Open the file in nano:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano fixme2.py
```

In nano:
- Use the **arrow keys** to move the cursor to the `=` on line 22
- Type `=` again to make it `==`
- Press **Ctrl+O**, then **Enter** to save
- Press **Ctrl+X** to exit

The fixed line should look like:

```python
if flag == "":
```

### Step 4 — Run the fixed script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fixme2.py
That is correct! Here's your flag: picoCTF{3qu4l1ty_n0t_4551gnm3nt_e8814d03}
```

**Flag:** `picoCTF{3qu4l1ty_n0t_4551gnm3nt_e8814d03}`

---

## Alternative Solve — Windows Side (Python on Windows)

If you have the file in `C:\Users\ASUS\Downloads\`:

```
PS C:\Users\ASUS\Downloads> python fixme2.py
That is correct! Here's your flag: picoCTF{3qu4l1ty_n0t_4551gnm3nt_e8814d03}
```

You can also fix the file in any text editor (Notepad++, VS Code, nano, vim) — just change `=` to `==` on line 22.

---

## What Happened Internally

1. The challenge author planted an `=` (assignment) inside an `if` statement
2. Python's parser rejects this because `if` expects a **boolean expression**, not an assignment
3. Python is smart enough to suggest using `==` (the equality operator) instead of `=` (the assignment operator)
4. Once the `=` is changed to `==`, the script runs:
   - The `str_xor` function XORs the encrypted flag with the key `'enkidu'` to recover the plaintext
   - The `if flag == "":` check correctly evaluates to `False` (since the flag is not empty)
   - The `else` branch runs and prints the flag

The flag content `3qu4l1ty_n0t_4551gnm3nt` is leet-speak for **"equality not assignment"** — a direct reference to the bug we just fixed!

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| python3 | Run the script after fixing | Easy |
| nano / any text editor | Change `=` to `==` | Easy |
| wget | Download the challenge file | Easy |

---

## Key Takeaways

- Python uses `=` for **assignment** and `==` for **equality comparison** — they are NOT the same
- Using `=` inside an `if` is a `SyntaxError` in Python (unlike C/JavaScript which would silently accept it)
- Python's error messages often **suggest the fix** directly — always read them carefully
- The flag wordplay: `3qu4l1ty_n0t_4551gnm3nt` = "equality not assignment" — exactly the lesson the challenge teaches
- The `str_xor` function was a red herring — the bug was a one-character typo, not a logic error
