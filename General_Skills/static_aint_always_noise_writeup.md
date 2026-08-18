# Static ain't always noise — picoCTF Writeup

**Challenge:** Static ain't always noise  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{d15a5m_t34s3r_20335e41}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Can you look at the data in this binary? The bash script might help!

**Attachments:** `static` (16 KB ELF binary), `ltdis.sh` (bash script that runs `objdump` and `strings`)

---

## Hints

(No hints)

---

## Background Knowledge (Read This First!)

### What is `strings`?

`strings` is a Unix tool that scans a binary file and prints every "run" of printable characters it finds (by default, runs of 4+ bytes terminated by a null or newline). It is one of the first things you should run on any binary you do not recognize. Even before you disassemble anything, `strings` will tell you the human-readable bits the program has stashed inside: error messages, log strings, URLs, embedded data, and the occasional flag.

### What is `objdump`?

`objdump` is a binary analysis tool that can disassemble (turn machine code back into assembly) and dump section headers, symbol tables, and more. The most common usage is `objdump -d <file>` (or `-D` to disassemble everything, not just known code sections). The `-j .text` flag limits the output to the `.text` section, which is where the actual executable code lives.

### Why "static"?

The challenge is called "Static ain't always noise" because the binary is statically analyzable — you do not have to *run* it to learn what it does. `strings` and `objdump` are static analysis tools: they work on the bytes of the file without executing anything. The flag is just sitting there in the `.rodata` section (or similar) waiting to be read. The lesson is: a binary is not opaque; it is a structured file full of readable strings.

### Why the bash script is overkill for this challenge

The `ltdis.sh` script runs both `objdump -Dj .text` (full disassembly of the `.text` section) and `strings -a -t x` (strings with hex file offsets). For a 16 KB binary with a flag literally sitting in the data section, only the `strings` half is needed. The script is teaching you the *full* static-analysis workflow, but for *this particular challenge* the `strings` half is enough to find the flag in 1 second.

---

## Solution

### Step 1: Examine the attachments

I downloaded both files and looked at the bash script first:

```bash
$ cat ltdis.sh
#!/bin/bash
echo "Attempting disassembly of $1 ..."
objdump -Dj .text $1 > $1.ltdis.x86_64.txt
if [ -s "$1.ltdis.x86_64.txt" ]
then
    echo "Disassembly successful! ..."
    strings -a -t x $1 > $1.ltdis.strings.txt
    echo "Any strings found in $1 have been written to $1.ltdis.strings.txt with file offset"
else
    echo "Disassembly failed!"
fi
```

The script:
1. Disassembles the binary with `objdump -Dj .text` (full disassembly of the `.text` section).
2. If that succeeds, runs `strings -a -t x` (extract printable strings, with hex file offsets) into a `.ltdis.strings.txt` file.

For the flag hunt, only the `strings` step matters. I do not need to disassemble to find it.

### Step 2: Run `strings` on the binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings static | grep -i pico
picoCTF{d15a5m_t34s3r_20335e41}
```

One match. The flag is in the binary's printable strings.

A slower but more thorough alternative is to run the full script:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ chmod +x ltdis.sh
└─$ ./ltdis.sh static
Attempting disassembly of static ...
Disassembly successful! Available at: static.ltdis.x86_64.txt
Ripping strings from binary with file offsets...
Any strings found in static have been written to static.ltdis.strings.txt with file offset
└─$ grep -i pico static.ltdis.strings.txt
   2908 picoCTF{d15a5m_t34s3r_20335e41}
```

The number `2908` is the hex file offset — the flag lives at byte `0x2908` of the binary.

**Flag:** `picoCTF{d15a5m_t34s3r_20335e41}`

---

## What Happened Internally (Timeline)

| Step | What I did | What happened |
|---|---|---|
| 1 | Read `ltdis.sh` | Saw that the script runs `objdump` + `strings`. |
| 2 | Ran `strings static \| grep pico` | Found the flag in one line. |
| 3 | Optionally ran the full `ltdis.sh` | Confirmed the flag at file offset `0x2908`. |

The whole solve is one command (or two if you also want the disassembly).

---

## Alternative Solve Methods

### Method 1: The intended path — run `ltdis.sh`

```
$ chmod +x ltdis.sh
$ ./ltdis.sh static
```

This produces two text files: the full disassembly and the strings dump. The flag is in the strings file. This is the workflow the author wanted you to learn, but it is heavier than the challenge actually requires.

### Method 2: `strings` directly + `grep`

```
$ strings static | grep pico
```

Skips the disassembly step entirely. Same answer, half the work.

### Method 3: `strings` with file offsets

```
$ strings -a -t x static | grep pico
   2908 picoCTF{d15a5m_t34s3r_20335e41}
```

The `-t x` flag tells `strings` to print the hex file offset of each match. Useful if you want to know *where* in the binary the flag lives (handy when comparing two binaries to see what changed, for example).

### Method 4: `objdump -s` (raw section dump)

If you want to look at the data section directly without going through `strings`:

```
$ objdump -s -j .rodata static
```

`-s` dumps raw bytes of the section, `-j .rodata` filters to the read-only data section (where string literals live). You will see the flag as raw bytes.

### Method 5: PowerShell / Python on Windows

If you do not have `strings` on Windows (the default install of Git Bash does not include it):

```python
import re
data = open(r'C:\Users\ASUS\Downloads\static', 'rb').read()
for s in re.findall(rb'[\x20-\x7e]{4,}', data):
    if b'pico' in s.lower():
        print(s.decode('latin-1'))
```

A regex that matches runs of 4+ printable bytes, then a `grep`-style filter for `pico`. Same logic, no external dependencies.

Or in PowerShell:

```powershell
$bytes = [System.IO.File]::ReadAllBytes('C:\Users\ASUS\Downloads\static')
$text = [System.Text.Encoding]::Latin1.GetString($bytes)
[regex]::Matches($text, '[\x20-\x7e]{4,}') | ForEach-Object { $_.Value } | Select-String -Pattern 'pico'
```

The `Latin1` encoding preserves every byte as a code point, so we can search for printable runs without mangling the data.

### Method 6: Just `grep` the binary directly

```
$ grep -a pico static
```

`-a` forces `grep` to treat the file as text. It will print every line containing `pico`, including the one with the flag.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `strings` | Extract printable strings from the binary |
| `grep` | Filter the strings output to find the flag |
| `objdump` (alt) | Disassemble the binary, or dump raw section data |
| `bash` / `ltdis.sh` (alt) | Run the author's helper script end-to-end |
| Python `re` (alt) | Windows-native way to find printable runs |
| PowerShell `[regex]` (alt) | Windows-native way to find printable runs |

---

## Key Takeaways

- **`strings` is the first move on any binary.** If you have a binary and you do not know what it is, run `strings` first. 90% of the time, the answer to "where is the flag in this binary?" is "in the strings."
- **`objdump` is the second move.** If `strings` does not show the flag, the flag is encoded, encrypted, or constructed at runtime. `objdump -d` (or `-D`) lets you read the assembly and see how the data is built.
- **Static analysis > dynamic analysis for a lot of CTF work.** You do not need to run the binary to read its strings, see its imports, or understand its structure. That makes static analysis safer (no risk of executing malware) and faster (no setup).
- **`grep -a` is a fallback for "I don't have `strings`."** Any environment with `grep` and `-a` (force text mode) can extract printable runs from a binary, just clumsier than `strings` proper.
- **Wordplay, as always:** the challenge name "Static ain't always noise" is a riff on the *Sniper* movie line ("Static ain't what it used to be"). The lesson is the same: even in a binary, where you expect to see only machine code, there are human-readable strings mixed in. The flag decodes from leetspeak as **disassembler** plus a hex suffix `20335e41`. The author is naming the tool that reveals the flag (`objdump` = disassembler) and showing you that you do not actually need the heavy tool for this one — `strings` alone finds the flag because the challenge author literally put the flag string in the binary. The "hard" version of this challenge would XOR-encrypt the flag in `.rodata` so `strings` could not find it, and you would have to actually run the disassembler to reconstruct it. We got the easy version. The lesson still applies: know how to use both.
