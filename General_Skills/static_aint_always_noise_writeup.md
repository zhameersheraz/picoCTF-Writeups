# Static ain't always noise — picoCTF Writeup

**Challenge:** Static ain't always noise  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{d15a5m_t34s3r_20335e41}`  

---

## Description

> Can you look at the data in this binary? The bash script might help!
> Downloads: `static`, `ltdis.sh`

**Hints:** None

---

## Background Knowledge (Read This First!)

### What is a binary file?

A **binary** is a compiled executable program — the kind of file you run directly in the terminal. Unlike Python scripts which are plain text, binaries are machine code that the CPU executes directly. They look like gibberish when opened in a text editor.

### What is `strings`?

**`strings`** is a Linux tool that scans a binary file and extracts any readable text sequences it finds — things like error messages, URLs, or hardcoded text like flags. It's one of the first tools used in binary analysis because it's fast and requires zero reverse engineering knowledge.

```bash
strings filename
```

The `-a` flag scans the entire file, and `-t x` prints the file offset (position) of each string in hex.

### What is `objdump`?

**`objdump`** is a disassembler — it converts machine code back into human-readable assembly language. The `ltdis.sh` script uses it to disassemble the `.text` section (the code section) of the binary.

### What does `ltdis.sh` do?

The provided script does two things automatically:
1. **Disassembles** the binary with `objdump` → saves to `static.ltdis.x86_64.txt`
2. **Extracts strings** with `strings` → saves to `static.ltdis.strings.txt`

The flag is hidden as a plaintext string inside the binary — `strings` finds it instantly.

---

## Solution — Step by Step

### Step 1 — Make the files executable

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ chmod +x static ltdis.sh
```

### Step 2 — Run `ltdis.sh` on the binary

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ bash ltdis.sh static
Attempting disassembly of static ...
Disassembly successful! Available at: static.ltdis.x86_64.txt
Ripping strings from binary with file offsets...
Any strings found in static have been written to static.ltdis.strings.txt with file offset
```

This creates two output files.

### Step 3 — Search the strings file for the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep picoCTF static.ltdis.strings.txt
   3020 picoCTF{d15a5m_t34s3r_20335e41}
```

The flag is at file offset `0x3020` inside the binary. ✅ Got the flag! 🎯

---

## Alternative Method — Skip the script and use `strings` directly

The script is helpful but not required. You can extract the flag in one command:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings static | grep picoCTF
picoCTF{d15a5m_t34s3r_20335e41}
```

`strings` scans the binary and `grep picoCTF` filters for lines containing the flag format.

---

## What the Two Output Files Contain

| File | Tool used | Contents |
|------|-----------|----------|
| `static.ltdis.x86_64.txt` | `objdump` | Disassembled x86-64 assembly code |
| `static.ltdis.strings.txt` | `strings` | All readable text found in binary with hex offsets |

For this challenge only the strings file is needed. The disassembly file would be useful for deeper reverse engineering challenges.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `chmod +x` | Make files executable | ⭐ Easy |
| `bash ltdis.sh static` | Run the provided helper script | ⭐ Easy |
| `grep picoCTF` | Filter output for the flag | ⭐ Easy |
| `strings` (alternative) | Extract readable text from binary directly | ⭐ Easy |

---

## Key Takeaways

- **`strings binary | grep picoCTF`** is the fastest way to find a hardcoded flag in any binary — memorize this combo for CTFs
- **Binaries can contain plaintext strings** — flags, passwords, and messages are often stored as readable text inside compiled programs
- **`ltdis.sh` automates the analysis** — always read helper scripts provided in challenges, they reveal the intended approach
- The challenge title "Static ain't always noise" — static (as in the binary's static data section) contains useful readable data, not just noise
- The flag `d15a5m_t34s3r` → "disasm teaser" — a teaser introduction to binary disassembly, the deeper skill used in harder reverse engineering challenges
