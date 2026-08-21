# fixme1.py — picoCTF Writeup

**Challenge:** fixme1.py  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** 100 pts  
**Flag:** `picoCTF{1nd3nt1ty_cr1515_6a476c8f}`  
**Platform:** picoMini 2022  
**Writeup by:** zham

---

## Description

> Fix the syntax error in this Python script to print the flag.

**Attachment:** fixme1.py (Python source code)

---

## Hints

> 1. Indentation is very meaningful in Python
> 2. To view the file in the webshell, do: `$ nano fixme1.py`
> 3. To exit nano, press Ctrl and x and follow the on-screen prompts.
> 4. The str_xor function does not need to be reverse engineered for this challenge.

---

## Background Knowledge (Read This First!)

### Python Indentation Rules

Python uses **indentation** (spaces or tabs) to define code blocks instead of curly braces `{}` like C, Java, or JavaScript. Every line inside a function, loop, if-statement, or class must be indented to the same level.

A common mistake beginners make is mixing tabs and spaces, or accidentally adding extra spaces at the start of a line. Python raises an `IndentationError` when:

- A line has unexpected leading whitespace
- The indentation level does not match what Python expects (e.g., a top-level statement with indentation)
- Tabs and spaces are mixed

### Reading Python Errors

When Python encounters a syntax error, it prints a traceback that tells you:

1. The **file name** and **line number** where the error occurred
2. The **type of error** (e.g., `IndentationError`, `SyntaxError`)
3. An **arrow** (`^`) pointing to the problematic character

Always read the traceback from bottom to top: the last line is where the error happened, the lines above show the call stack that led there.

---

## Solution

### Step 1 — Download the challenge file

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ wget <challenge_url> -O fixme1.py
└─$ cat fixme1.py
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


flag_enc = chr(0x15) + chr(0x07) + chr(0x08) + chr(0x06) + chr(0x27) + chr(0x21) + chr(0x23) + chr(0x15) + chr(0x5a) + chr(0x07) + chr(0x00) + chr(0x46) + chr(0x0b) + chr(0x1a) + chr(0x5a) + chr(0x1d) + chr(0x1d) + chr(0x2a) + chr(0x06) + chr(0x1c) + chr(0x5a) + chr(0x5c) + chr(0x55) + chr(0x40) + chr(0x3a) + chr(0x58) + chr(0x0a) + chr(0x5d) + chr(0x53) + chr(0x43) + chr(0x06) + chr(0x56) + chr(0x0d) + chr(0x14)

  
flag = str_xor(flag_enc, 'enkidu')
  print('That is correct! Here\'s your flag: ' + flag)
```

Notice the last line has **2 extra spaces** at the beginning — that is the bug.

### Step 2 — Try to run the script and see the error

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fixme1.py
  File "fixme1.py", line 20
    print('That is correct! Here\'s your flag: ' + flag)
    ^
IndentationError: unexpected indent
```

Python points at line 20 with the `^` under the `p` in `print`. The two leading spaces make Python think this line belongs inside a code block, but no block is open at the top level.

### Step 3 — Fix the indentation

Open the file in nano and remove the two extra spaces on line 20:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano fixme1.py
```

In nano:
- Use the **arrow keys** to move the cursor to the start of line 20
- Press **Backspace** twice to remove the two leading spaces
- Press **Ctrl+O**, then **Enter** to save
- Press **Ctrl+X** to exit

The fixed line should look like:

```python
print('That is correct! Here\'s your flag: ' + flag)
```

(no leading spaces)

### Step 4 — Run the fixed script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fixme1.py
That is correct! Here's your flag: picoCTF{1nd3nt1ty_cr1515_6a476c8f}
```

**Flag:** `picoCTF{1nd3nt1ty_cr1515_6a476c8f}`

---

## Alternative Solve — Windows Side (Python on Windows)

If you have the file in `C:\Users\ASUS\Downloads\`:

```
PS C:\Users\ASUS\Downloads> python fixme1.py
That is correct! Here's your flag: picoCTF{1nd3nt1ty_cr1515_6a476c8f}
```

You can also fix the file in any text editor (Notepad++, VS Code, nano, vim) — just remove the 2 leading spaces on the `print` line.

---

## What Happened Internally

1. The challenge author planted an `IndentationError` on the `print` line of `fixme1.py`
2. The two extra spaces look harmless but they tell Python this line is inside a nested code block, which does not exist at the top level
3. Python stops execution immediately and prints a traceback showing the exact line and column of the problem
4. Once the leading whitespace is removed, the script runs normally
5. The `str_xor` function XORs the encrypted flag with the key `'enkidu'` to recover the plaintext flag
6. The script prints the recovered flag with a confirmation message

The flag content `1nd3nt1ty_cr1515` is leet-speak for **"indentity crisis"** — a pun on Python's strict indentation rules.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| python3 | Run the script after fixing | Easy |
| nano / any text editor | Remove the 2 extra spaces | Easy |
| wget | Download the challenge file | Easy |

---

## Key Takeaways

- Python uses **indentation** to define code blocks — a single stray space can break a script
- Always read the **full traceback**: it tells you the file, line number, and exact character that caused the error
- The `^` arrow in the traceback points to the problematic indentation, making it easy to spot
- For this challenge, no reverse engineering of the `str_xor` function was needed — the bug was a simple whitespace fix
- The flag wordplay: `1nd3nt1ty_cr1515` = "indentity crisis" — a nod to Python's strict indentation rules
