# strings it — picoCTF Writeup

**Challenge:** strings it  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{5tRIng5_1T_dB2CEA76}`  

---

## Description

> Can you find the flag in file without running it?

**Hint 1:** `strings`

---

## Background Knowledge (Read This First!)

### What is `strings`?

**`strings`** is a Linux tool that scans any file — binary or text — and extracts sequences of readable ASCII characters. Since compiled binaries contain machine code that looks like gibberish, `strings` filters out all the noise and shows only the human-readable parts like error messages, URLs, and hardcoded text like flags.

The challenge says "without running it" — `strings` lets you inspect a binary's embedded text without executing it at all.

---

## Solution — Step by Step

### Step 1 — Run strings and grep for the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings strings | grep picoCTF
picoCTF{5tRIng5_1T_dB2CEA76}
```

✅ Got the flag! 🎯

---

## Alternative — strings without grep

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings strings | less
```

Use `/picoCTF` inside `less` to search. But `grep` is faster.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `strings` | Extract readable text from the binary | ⭐ Easy |
| `grep` | Filter for the flag pattern | ⭐ Easy |

---

## Key Takeaways

- **`strings filename | grep picoCTF`** is one of the most-used combos in binary CTF challenges — memorise it
- Binaries can contain plaintext flags hardcoded as strings — `strings` exposes them without any reverse engineering
- The flag `5tRIng5_1T` → "strings it" — the tool name is the solve
