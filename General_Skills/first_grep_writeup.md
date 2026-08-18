# First Grep — picoCTF Writeup

**Challenge:** First Grep  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{grep_is_good_to_find_things_eb80073D}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> Can you find the flag in the file? This would be really tedious to look through manually, something tells me there is a better way.

**Attachment:** `file` (14 KB, mostly noise, flag hidden in the middle)

---

## Hints

1. `grep` tutorial

---

## Background Knowledge (Read This First!)

### What is `grep`?

`grep` (Global Regular Expression Print) is one of the most-used Unix tools. It scans files line by line and prints every line that matches a pattern. The basic syntax is:

```
grep 'pattern' file
```

`grep` is a "search the haystack for the needle" tool. It is the right answer whenever a CTF challenge says "find X in a huge file" or "filter lines that contain Y."

### Why we need it here

The challenge attachment is a single 14 KB line of garbage characters — random printable ASCII with no whitespace, no formatting, no obvious structure. The flag is hidden somewhere in the middle. Trying to find it by scrolling is a fool's errand. One `grep` command and the flag pops out instantly.

### A common beginner gotcha

`grep` by default treats files as text, but on modern systems it will detect a file with too many non-printable bytes and refuse to print matches, saying `binary file matches`. The fix is to add the `-a` flag (`--text`), which forces grep to treat the file as text. We will see this in the alternative solutions.

---

## Solution

### Step 1: Download the file and look at it (or don't)

The attachment is a 14 KB file with one giant line of noise. I peeked at the first few bytes:

```
$ head -c 200 file
apgqq8v;o!PW3b!fJ=-~9p&.-9jPRc|U8J@~Fr6UM)wQ>|I4a.!gJ	@waMDUkj/r...
```

It is one long run of printable characters, no line breaks, no obvious structure. Trying to spot the flag by eye would take a while. Let me skip that.

### Step 2: `grep` for `picoCTF`

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep 'picoCTF' file
...picoCTF{grep_is_good_to_find_things_eb80073D}...
```

One line, one match, the flag printed inline with all the noise around it. From there, copy the `picoCTF{...}` substring and submit.

**Flag:** `picoCTF{grep_is_good_to_find_things_eb80073D}`

---

## What Happened Internally (Timeline)

| Step | What I did | What happened |
|---|---|---|
| 1 | Looked at the file briefly | Saw one long line of noise. |
| 2 | Ran `grep 'picoCTF' file` | grep scanned the line and printed the matching substring. |
| 3 | Extracted the `picoCTF{...}` from the output | Submitted the flag. |

The whole solve is one command. There is nothing to install, no script to write, just `grep`.

---

## Alternative Solve Methods

### Method 1: `grep -a` (force text mode)

Some versions of `grep` will refuse to print the match if the file has too many non-printable bytes, and instead print `Binary file file matches`. Force text mode with `-a` (or `--text`):

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep -a 'picoCTF' file
...picoCTF{grep_is_good_to_find_things_eb80073D}...
```

`-a` tells grep to process the file as if it were plain text. This is the workaround whenever grep gets fussy about binary content.

### Method 2: `strings | grep`

If you prefer to first strip non-printable bytes:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ strings file | grep picoCTF
picoCTF{grep_is_good_to_find_things_eb80073D}
```

`strings` is overkill here (the file is already printable) but it is a useful reflex for binary files.

### Method 3: PowerShell `Select-String`

On Windows:

```powershell
Get-Content C:\Users\ASUS\Downloads\file | Select-String -Pattern 'picoCTF'
```

`Select-String` is the PowerShell equivalent of `grep`. It walks the file line by line and prints matches.

### Method 4: Python one-liner

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
import re
data = open('file').read()
m = re.search(r'picoCTF\{[^}]+\}', data)
print(m.group(0) if m else 'no match')
"
picoCTF{grep_is_good_to_find_things_eb80073D}
```

A regex that matches `picoCTF{anything-but-}` is the most general approach and works even if the flag is split across lines (rare, but possible).

### Method 5: Highlight the match in your editor

If you are using `vim`, `nano`, or VS Code:
- vim: `/picoCTF` then `<Enter>` to jump to the next match
- VS Code: `Ctrl+F` and search for `picoCTF`

Visual editors highlight matches as you type, which is nice when you also want to see the noise around the flag.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `grep` | Search for the `picoCTF` pattern in the file |
| `strings` + `grep` (alt) | Same thing via a different path |
| `grep -a` (alt) | Force text mode if grep complains about binary content |
| `Select-String` (alt) | PowerShell equivalent |
| Python `re` (alt) | Regex approach for any environment with Python |

---

## Key Takeaways

- **`grep` is your first tool for "find X in a file."** Memorize `grep 'pattern' file`. Add `-i` for case-insensitive, `-a` to force text mode, `-r` to recurse into directories, `-n` to print line numbers, `-v` to invert the match.
- **For files with weird characters, `grep -a` is the safety net.** The default behaviour of "I think this is a binary file" is helpful in shell scripting, annoying in CTFs.
- **For binary blobs, `strings | grep` is the classic pipeline.** `strings` extracts printable ASCII runs (4+ bytes by default), `grep` filters to the line you want. The two together handle a huge range of "find the flag in this mystery file" challenges.
- **Huge single-line files defeat `less` and `cat` ergonomically.** `grep` (or any other search) is the only way. The whole point of this challenge is to teach you to reach for it before scrolling.
- **Wordplay, as always:** the flag content `grep_is_good_to_find_things` is the entire thesis statement of the challenge. The author is not being subtle: the lesson is that grep is good for finding things, full stop. The `eb80073D` suffix is the per-instance hex identifier. The author wants you to walk away with a reflex, not just a flag.
