# fixme1.py — picoCTF Writeup

**Challenge:** fixme1.py  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{1nd3nt1ty_cr1515_6a476c8f}`  

---

## Description

> Fix the syntax error in this Python script to print the flag.
> Download Python script

**Hint 1:** `Indentation is very meaningful in Python`

---

## Background Knowledge (Read This First!)

### What is indentation in Python?

**Indentation** is how Python groups code into blocks. Unlike most languages that use `{}` curly braces, Python uses **spaces or tabs at the beginning of a line** to define structure. Every line inside a function, `if`, `for`, or `while` block must be indented consistently.

```python
def my_function():
    print("I am indented - I belong to the function")

print("I am NOT indented - I am outside the function")
```

### What is an IndentationError?

When a line has unexpected indentation — spaces at the start when there shouldn't be any — Python raises an `IndentationError`:

```
IndentationError: unexpected indent
```

This means Python found indentation where it didn't expect any, which breaks the program before it even runs.

### The Bug in fixme1.py

Looking at the broken code:

```python
flag = str_xor(flag_enc, 'enkidu')
  print('That is correct! Here\'s your flag: ' + flag)
```

The `print` line has **2 extra spaces** at the beginning. Since it's not inside any function, `if`, or loop, that indentation is unexpected and illegal — Python doesn't know what block it belongs to.

The fix is simple — remove the leading spaces from the `print` line:

```python
flag = str_xor(flag_enc, 'enkidu')
print('That is correct! Here\'s your flag: ' + flag)
```

---

## Solution — Step by Step

### Step 1 — Try running the broken script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fixme1.py
  File "/media/sf_downloads/fixme1.py", line 18
    print('That is correct! Here\'s your flag: ' + flag)
    ^
IndentationError: unexpected indent
```

Python pinpoints exactly which line has the bad indentation.

### Step 2 — Fix the indentation using nano

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano fixme1.py
```

Find this line (notice the 2 leading spaces):
```
  print('That is correct! Here\'s your flag: ' + flag)
```

Remove the leading spaces so it looks like:
```
print('That is correct! Here\'s your flag: ' + flag)
```

Press `Ctrl+X` → `Y` → `Enter` to save.

### Step 3 — Run the fixed script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fixme1.py
That is correct! Here's your flag: picoCTF{1nd3nt1ty_cr1515_6a476c8f}
```

✅ Got the flag! 🎯

---

## Alternative Method — Fix with sed and run directly

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sed 's/^  print/print/' fixme1.py | python3
That is correct! Here's your flag: picoCTF{1nd3nt1ty_cr1515_6a476c8f}
```

`sed 's/^  print/print/'` strips the two leading spaces from the `print` line — `^` means "start of line" — then pipes the corrected code straight into Python without modifying the original file.

---

## fixme1.py vs fixme2.py — Side by Side

| | fixme1.py | fixme2.py |
|--|-----------|-----------|
| Bug | Extra indentation on `print` | `=` instead of `==` in `if` |
| Error type | `IndentationError` | `SyntaxError` |
| Fix | Remove leading spaces | Change `=` to `==` |
| Hint | "Indentation is very meaningful in Python" | "Are equality and assignment the same symbol?" |
| Flag | `1nd3nt1ty_cr1515` | `3qu4l1ty_n0t_4551gnm3nt` |

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `python3 fixme1.py` | Run the script and see the IndentationError | ⭐ Easy |
| `nano fixme1.py` | Edit the file to remove the extra indentation | ⭐ Easy |
| `sed` (optional) | Fix and run without editing the original file | ⭐⭐ Medium |

---

## Key Takeaways

- **Indentation is syntax in Python** — unlike most languages, Python uses spaces to define code structure, so extra or missing spaces break the program
- **`IndentationError: unexpected indent`** means a line has leading spaces where none are expected — always check the line Python points to
- **The error message tells you exactly where the bug is** — reading it carefully is faster than scanning the whole file
- The flag says it all: `1nd3nt1ty_cr1515` → "indentity crisis" — a pun on "identity crisis" and Python indentation
