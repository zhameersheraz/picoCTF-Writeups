# fixme2.py — picoCTF Writeup

**Challenge:** fixme2.py  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{3qu4l1ty_n0t_4551gnm3nt_4863e11b}`  

---

## Description

> Fix the syntax error in the Python script to print the flag.
> Download Python script

**Hint 1:** `Are equality and assignment the same symbol?`

---

## Background Knowledge (Read This First!)

### `=` vs `==` — The Most Common Python Mistake

This is one of the most classic bugs in programming:

| Symbol | Name | What it does | Example |
|--------|------|--------------|---------|
| `=` | Assignment | Stores a value into a variable | `x = 5` |
| `==` | Equality check | Compares two values, returns True/False | `x == 5` |

Using `=` inside an `if` statement is a **syntax error** in Python because `if` expects a comparison (True/False), not an assignment. Hint 1 pointed directly at this — "Are equality and assignment the same symbol?"

### What is a Syntax Error?

A **syntax error** means the code is grammatically wrong — Python cannot even parse it, so the script refuses to run at all. The error appears before any output:

```
  File "fixme2.py", line X
    if flag = "":
            ^
SyntaxError: invalid syntax
```

---

## Finding the Bug

```python
# BROKEN - uses = (assignment) inside if statement
if flag = "":
  print('String XOR encountered a problem, quitting.')
else:
  print('That is correct! Here\'s your flag: ' + flag)
```

The fix is simple — change `=` to `==`:

```python
# FIXED - uses == (equality comparison)
if flag == "":
    print('String XOR encountered a problem, quitting.')
else:
    print('That is correct! Here\'s your flag: ' + flag)
```

---

## Solution — Step by Step

### Step 1 — Try running the broken script first

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fixme2.py
  File "/media/sf_downloads/fixme2.py", line 18
    if flag = "":
            ^
SyntaxError: invalid syntax
```

Python immediately shows the syntax error on the `if flag = ""` line.

### Step 2 — Fix the bug using nano

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano fixme2.py
```

Find this line:
```python
if flag = "":
```

Change it to:
```python
if flag == "":
```

Press `Ctrl+X` → `Y` → `Enter` to save.

### Step 3 — Run the fixed script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 fixme2.py
That is correct! Here's your flag: picoCTF{3qu4l1ty_n0t_4551gnm3nt_4863e11b}
```

✅ Got the flag! 🎯

---

## Alternative Method — Fix and run in one command

Instead of editing the file, patch it inline with `sed` and pipe directly to Python:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ sed 's/if flag = ""/if flag == ""/' fixme2.py | python3
That is correct! Here's your flag: picoCTF{3qu4l1ty_n0t_4551gnm3nt_4863e11b}
```

`sed 's/old/new/'` replaces the first occurrence of `old` with `new` in the file output, then pipes the corrected code straight into Python without modifying the original file.

---

## fixme1.py vs fixme2.py

Both challenges in the fixme series involve fixing a Python syntax error:

| | fixme1.py | fixme2.py |
|--|-----------|-----------|
| Bug type | Indentation error | Wrong operator (`=` vs `==`) |
| Error shown | `IndentationError` | `SyntaxError` |
| Fix | Add correct indentation | Change `=` to `==` in `if` statement |

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `python3 fixme2.py` | Run the script and see the error | ⭐ Easy |
| `nano fixme2.py` | Edit the file to fix the bug | ⭐ Easy |
| `sed` (optional) | Patch and run without editing the file | ⭐⭐ Medium |

---

## Key Takeaways

- **`=` is assignment, `==` is comparison** — confusing these is one of the most common beginner mistakes in Python
- **Python's `SyntaxError` tells you exactly which line is broken** — always read the error message carefully, it points right at the problem
- **`if` statements require a boolean expression** — using `=` (assignment) inside `if` is always a `SyntaxError` in Python
- The flag says it all: `3qu4l1ty_n0t_4551gnm3nt` → "equality not assignment" — the fix in one phrase
