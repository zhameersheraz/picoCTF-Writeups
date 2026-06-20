# Special — picoCTF Writeup

**Challenge:** Special  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{5p311ch3ck_15_7h3_w0r57_f906e25a}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Don't power users get tired of making spelling mistakes in the shell? Not anymore! Enter Special, the Spell Checked Interface for Affecting Linux. Now, every word is properly spelled and capitalized... automatically and behind-the-scenes! Be the first to test Special in beta, and feel free to tell us all about how Special streamlines every development process that you face. When your co-workers see your amazing shell interface, just tell them: That's Special (TM)
> Start your instance to see connection details.
> `ssh -p 65298 ctf-player@saturn.picoctf.net`. The password is `3f39b042`

**Hint 1:** `Experiment with different shell syntax`

---

## Background Knowledge (Read This First!)

### What does "spell-checked shell" actually mean?

This challenge drops you into a wrapper program (not real Bash) that reads whatever you type, runs it through a spellchecker, capitalizes the first letter, and only then executes it. So if you type a command that isn't a recognized English dictionary word, the wrapper silently "fixes" it into the nearest real word before running it — turning `ls` or `pwd` into garbage you never typed.

### The actual source code

The wrapper script lives at `/usr/local/Special.py`. Its logic is:

```python
#!/usr/bin/python3
import os
from spellchecker import SpellChecker

spell = SpellChecker()

while True:
  cmd = input("Special$ ")

  if cmd == 'exit':
    break
  elif 'sh' in cmd:
    print('Why go back to an inferior shell?')
    continue
  elif cmd[0] == '/':
    print('Absolutely not paths like that, please!')
    continue

  # Spellcheck every word
  spellcheck_cmd = ''
  for word in cmd.split():
    fixed_word = spell.correction(word)
    if fixed_word is None:
      fixed_word = word
    spellcheck_cmd += fixed_word + ' '

  # Capitalize the first letter
  fixed_cmd = list(spellcheck_cmd)
  first_letter = spellcheck_cmd.split()[0][0]
  if ord(first_letter) >= 97 and ord(first_letter) <= 122:
    fixed_cmd[0] = chr(ord(spellcheck_cmd[0]) - 0x20)
  fixed_cmd = ''.join(fixed_cmd)

  os.system(fixed_cmd)
```

### Why this is breakable — three filters, three blind spots

1. **`'sh' in cmd`** blocks any input containing the substring `"sh"` *anywhere* — this is why typing `bash` fails (it's blocked directly), but it only checks the raw string, nothing smarter.
2. **`cmd[0] == '/'`** only checks the very first character of the *entire line* — not the first character of each individual word. A line that starts with anything other than `/` sails straight past this check, even if a `/` shows up later.
3. **The spellchecker** runs `spell.correction(word)` on each word. If a word isn't close enough to any real English word, `correction()` returns `None`, and the original word is kept untouched. Words wrapped in single quotes, like `'/bin/ls'`, are never real dictionary words — so they always survive untouched.

Putting all three together: **wrap every command and every argument in single quotes.** A quoted token defeats the spellchecker (it's never a real word) and the line no longer starts with `/` (it starts with `'`). The script then hands the whole string to `os.system()`, which passes it to a real shell — and that real shell strips the quotes and runs your command exactly as written.

---

## Solution — Step by Step

### Step 1 — Connect via SSH

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 65298 ctf-player@saturn.picoctf.net
ctf-player@saturn.picoctf.net's password: 3f39b042
Special$
```

### Step 2 — Find the home directory with a quoted `pwd`

```
Special$ 'pwd'
'pwd'
/home/ctf-player
```

The quotes stop the spellchecker from mangling `pwd`, and the script echoes back the exact command it's about to run before executing it.

### Step 3 — List files with a quoted absolute path

```
Special$ '/bin/ls'
'/bin/ls'
blargh
```

Quoting `/bin/ls` dodges both the leading-slash filter and the spellchecker in one move. One directory found: `blargh`.

### Step 4 — List inside `blargh`

```
Special$ '/bin/ls' 'blargh'
'/bin/ls' 'blargh'
flag.txt
```

Each argument gets its own pair of quotes — `flag.txt` is sitting right there.

### Step 5 — Read the flag with a quoted `cat`

```
Special$ '/bin/cat' '/home/ctf-player/blargh/flag.txt'
'/bin/cat' '/home/ctf-player/blargh/flag.txt'
picoCTF{5p311ch3ck_15_7h3_w0r57_f906e25a}
```

Flag captured.

---

## Alternative Methods

### The `((command))` trick — exploiting `os.system`'s real shell

`os.system()` on most Linux systems doesn't call Bash — it calls `/bin/sh`, which on Debian/Ubuntu is `dash`. Bash treats `((...))` as a special arithmetic-evaluation block, but dash has no such feature — it just sees two nested subshells: `( (cat) )`. Since `cat` with no arguments reads from stdin, this line:

```
Special$ ((cat)) < blargh/flag.txt
```

ends up behaving exactly like `cat < blargh/flag.txt` — same result as `cat blargh/flag.txt` — and it sails past every filter: no `/` at the start, no `sh` substring, and `((cat))` isn't a real word so the spellchecker leaves it alone.

### Full escape — drop into a real Python shell

The `cmd[0] == '/'` check only looks at the very first character. Prefixing the line with a fake environment-variable assignment (valid shell syntax: `VAR=value command`) moves the slash later in the string:

```
Special$ placeholder=abc /usr/bin/python3
Python 3.8.10 (default, Nov 14 2022, 12:59:47)
>>> import os
>>> os.system("cat blargh/flag.txt")
picoCTF{5p311ch3ck_15_7h3_w0r57_f906e25a}
```

This drops you into an actual unrestricted Python interpreter — from there, `os.system()` runs anything, no quoting gymnastics needed at all.

---

## What Happened Internally

```
Timeline:
1. Connect via SSH — dropped into "Special$", a Python wrapper that spellchecks + capitalizes every input
2. Plain commands like ls/pwd get auto-corrected into real dictionary words and break
3. 'pwd' — single quotes aren't a dictionary word, spellcheck leaves it alone, and the line no longer starts with "/"
4. '/bin/ls' — same trick on a full path, reveals the "blargh" directory
5. '/bin/ls' 'blargh' — same trick with two quoted arguments, reveals flag.txt
6. '/bin/cat' '/home/ctf-player/blargh/flag.txt' — quoted cat reads the file untouched
7. os.system() hands the (still-quoted) string to a real shell, which strips the quotes and executes it normally
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `ssh` | Connect to the challenge server | Easy |
| Single-quoting (`'cmd'`) | Defeats the spellchecker and the leading-slash filter at once | Medium |
| `pyspellchecker` (read, not used by us) | The library being exploited — only corrects words it recognizes | Medium |
| `os.system()` (read, not used by us) | What actually runs the command — hands it to a real shell that strips quotes | Medium |

---

## Key Takeaways

- **A filter that checks `cmd[0]` only ever sees the first character of the whole line** — not the first character of each word. Anything that moves the "dangerous" character later in the string slips right past
- **Spellcheckers only "fix" words they almost recognize** — wrapping a command in quotes, or otherwise making it not resemble a real word, guarantees it's left alone
- **`os.system()` always re-parses your string through a real shell** — any quoting you add survives that hand-off and gets correctly stripped, so you can use normal shell quoting to smuggle a command through a filter built on top of it
- **`/bin/sh` isn't always Bash** — on Debian/Ubuntu it's `dash`, and constructs that mean one thing in Bash (`((...))` as arithmetic) can mean something completely different in `dash` (nested subshells), which is exactly the gap the `((cat))` trick lives in
- The flag `5p311ch3ck_15_7h3_w0r57` decodes to "spellcheck is the worst" — the exact complaint that, per Specialer's description, made the developers rip the spellchecker out entirely for the sequel challenge
