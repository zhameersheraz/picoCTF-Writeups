# strings it — picoCTF Writeup

**Challenge:** strings it  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{5tRIng5_1T_d6306c19}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> Can you find the flag in `file` without running it?

**Attachment:** `strings` (784 KB ELF binary)

---

## Hints

1. strings

---

## Background Knowledge (Read This First!)

### The `strings` Unix tool

`strings` is one of the most-used tools in any CTF player's belt. It scans a file (binary or text) and prints every "run" of printable ASCII characters it finds — by default, runs of 4+ characters terminated by a null byte or newline. It is your first move on any unknown binary because:

- Compilers store string literals (error messages, log strings, embedded URLs, etc.) in the `.rodata` section. These are exactly the runs `strings` extracts.
- Flags inside binaries almost always start with a known prefix (`picoCTF{` in this case), so a one-line `strings | grep` finds them in milliseconds.
- It is *static* — `strings` does not execute the binary, so it is safe to use on suspicious files.

### When `strings` is not enough

If the flag is encoded, encrypted, or constructed at runtime, `strings` will not find it — you have to look at the disassembly (`objdump -d`) or run the binary in a sandbox. For "easy" challenges like this one, the flag is almost always a literal in the data section, so `strings` is enough.

### The challenge is named for the tool

"strings it" is a direct instruction: the challenge wants you to run `strings` on the file. The hint "strings" is literally the tool name. The author is teaching the muscle memory.

---

## Solution

### Step 1: Download and look at the file

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file strings
strings: ELF 64-bit LSB executable, x86-64, ...
```

It is a 784 KB ELF binary. Trying to run it could be dangerous (no telling what it does without analysis) — but the question is whether we even need to.

### Step 2: Run `strings` and grep for the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings strings | grep -i pico
picoCTF{5tRIng5_1T_d6306c19}
```

One line, one match. The flag is sitting in the binary's printable strings.

**Flag:** `picoCTF{5tRIng5_1T_d6306c19}`

---

## What Happened Internally (Timeline)

| Step | What I did | What happened |
|---|---|---|
| 1 | `file strings` | Confirmed it is an ELF binary. |
| 2 | `strings strings \| grep pico` | Scanned for printable runs, filtered to the flag prefix, printed one match. |
| 3 | Submitted the match | Got the flag. |

The whole solve is two commands, and the binary is never executed. The flag is a literal in the binary's `.rodata` section.

---

## Alternative Solve Methods

### Method 1: The intended path — `strings | grep`

```
$ strings strings | grep pico
```

That is the answer. Done.

### Method 2: `strings -a` to be sure

If for some reason `strings` misses the flag (long runs only, weird encoding), add `-a` to force it to scan the whole file:

```
$ strings -a strings | grep pico
```

`strings` defaults to scanning only the initialized data sections of the object file. `-a` (or `--all`) tells it to scan the whole file. For an ELF binary the default usually works, but `-a` is the safety net.

### Method 3: `strings` with file offsets

```
$ strings -a -t x strings | grep pico
   1d6e0 picoCTF{5tRIng5_1T_d6306c19}
```

The `-t x` flag prints the hex file offset of each match. Useful if you want to know *where* in the binary the flag lives, e.g. for cross-referencing with `objdump -d` output or for comparing two binaries.

### Method 4: `grep -a` directly (no `strings`)

If you do not have `strings` installed:

```
$ grep -a pico strings
```

`-a` forces `grep` to treat the file as text. It will print every line that contains `pico`, even though the file is mostly binary. Slower than `strings` (because `grep` walks the file linearly) but it gets the job done.

### Method 5: PowerShell on Windows

```powershell
$bytes = [System.IO.File]::ReadAllBytes('C:\Users\ASUS\Downloads\strings')
$text = [System.Text.Encoding]::Latin1.GetString($bytes)
[regex]::Matches($text, '[\x20-\x7e]{4,}') | ForEach-Object { $_.Value } | Select-String -Pattern 'pico'
```

A regex that matches runs of 4+ printable bytes, filtered to `pico`. The `Latin1` encoding preserves every byte as a code point so we can search the binary as if it were a string.

### Method 6: Python one-liner

```python
import re
data = open(r'C:\Users\ASUS\Downloads\strings', 'rb').read()
matches = [s.decode('latin-1') for s in re.findall(rb'[\x20-\x7e]{4,}', data) if b'pico' in s]
for m in matches:
    print(m)
```

Same idea, in Python. Useful when you want to script the search or chain it with other processing.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `strings` | Extract printable ASCII runs from the binary |
| `grep` | Filter the output to find `pico` |
| `file` (alt) | Confirm the file type before doing anything else |
| `grep -a` (alt) | Fallback when `strings` is not available |
| Python `re` (alt) | Cross-platform way to do the same thing |
| PowerShell `[regex]` (alt) | Windows-native equivalent |

---

## Key Takeaways

- **`strings` is a reflex.** Any time you see a binary file in a CTF, run `strings` first. The flag is almost always a literal string in the data section, and `strings` is the fastest way to find it.
- **Pair `strings` with `grep`.** The combination `strings file | grep <pattern>` is so common in CTFs it is practically a single command in your muscle memory. The pattern is usually a flag prefix (`picoCTF{`, `CTF{`, `flag{`, `HTB{`).
- **`-a` is the safety net for `strings`.** When in doubt, `strings -a <file>` scans the whole file, not just the initialized data sections. For ELF files the default usually works, but `-a` is the "be safe" version.
- **`-t x` is great for cross-referencing.** If you want to see the file offset of the match (handy for jumping to it in a hex editor, or comparing two binaries to see what changed), `-t x` gives you hex offsets inline.
- **Wordplay, as always:** the challenge name "strings it" is literally the answer. The flag decodes from leetspeak as **strings it** (with the `1` standing in for the lowercase `i`). The author is not subtle: the tool name, the challenge name, the hint, and the flag content all say the same thing. This is a one-joke challenge, and the joke is "use the tool named in the title." The `d6306c19` suffix is the per-instance hex identifier. Spot the pun: the answer was in the challenge name all along.
