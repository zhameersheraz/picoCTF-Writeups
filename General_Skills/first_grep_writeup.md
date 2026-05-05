# First Grep — picoCTF Writeup

**Challenge:** First Grep  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{grep_is_good_to_find_things_eb80073D}`  

---

## Description

> Can you find the flag in the file? This would be really tedious to look through manually, something tells me there is a better way.
> The flag is in this file.

**Hint 1:** `grep tutorial`

---

## Background Knowledge (Read This First!)

### What is `grep`?

**`grep`** is a Linux command that searches through text for a specific pattern and prints only the matching lines. It's one of the most essential tools for CTF challenges and real-world Linux use.

```bash
grep "pattern" filename
```

The file in this challenge contains thousands of lines of random characters — finding `picoCTF{...}` manually would take forever. `grep` finds it instantly.

---

## Solution — Step by Step

### Step 1 — Download and search the file

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep picoCTF file
picoCTF{grep_is_good_to_find_things_eb80073D}
```

One command. Done. ✅ Got the flag! 🎯

---

## Alternative Methods

### Alternative 1 — Case-insensitive search

```bash
grep -i "picoctf" file
```

The `-i` flag makes grep case-insensitive — useful when you're not sure about capitalisation.

### Alternative 2 — Show line number

```bash
grep -n picoCTF file
```

`-n` shows which line number the flag is on.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `grep` | Search the file for the flag pattern | ⭐ Easy |

---

## Key Takeaways

- **`grep pattern filename`** is the fastest way to find a specific string in any file
- **Never scroll through large files manually** — `grep` finds what you need in milliseconds
- The flag `grep_is_good_to_find_things` — the lesson in the flag itself
