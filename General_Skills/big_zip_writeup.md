# Big Zip — picoCTF Writeup

**Challenge:** Big Zip  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{gr3p_15_m4g1c_ef8790dc}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Unzip this archive and find the flag.

**Attachment:** `big-zip-files.zip` (3.1 MB, lots of nested directories)

---

## Hints

1. Can grep be instructed to look at every file in a directory and its subdirectories?

---

## Background Knowledge (Read This First!)

### Recursive `grep`

By default, `grep` only looks at the files you pass it on the command line. To make it walk a whole directory tree, you add `-r` (or `--recursive`):

```
grep -r 'pattern' directory/
```

This is the right answer whenever the flag could be in *any* file under a folder and you do not know which one. The hint in the challenge literally spells this out.

### `grep -r` vs `find ... -exec grep`

Two equivalent ways to do the same thing:

```
# Style 1: grep walks the tree
grep -r 'pattern' dir/

# Style 2: find finds files, grep scans each one
find dir/ -type f -exec grep -l 'pattern' {} +
```

Style 1 is shorter and what you should reach for first. Style 2 is useful when you need more control over *which* files to scan (e.g. only `.txt` files, or only files modified in the last day). For this challenge, the simple `grep -r` is the right tool.

### Why this challenge exists

The author picked a 3 MB archive with hundreds of files in a deep directory tree. The flag is in a single, randomly-named `.txt` file at the bottom of the tree. Manually walking the tree would take forever. The whole point of the challenge is to teach you to reach for `grep -r` before doing anything else.

---

## Solution

### Step 1: Unzip the archive

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip big-zip-files.zip -d big_zip_extracted
```

This is a big zip (~3 MB), so it takes a few seconds. The extracted tree has hundreds of files scattered across many directories.

### Step 2: Recursive `grep` for the flag pattern

I used `grep -r 'picoCTF' .` to search the whole tree at once. Each match prints the path and the line, so I get the file location and the content in one shot:

```
┌──(zham㉿kali)-[/media/sf_downloads/big_zip_extracted]
└─$ grep -r 'picoCTF' .
./big-zip-files/folder_pmbymkjcya/folder_cawigcwvgv/folder_ltdayfmktr/folder_fnpfclfyee/whzxrpivpqld.txt:Genes and brains and books encode picoCTF{gr3p_15_m4g1c_ef8790dc}
```

One match, one line. The flag is wrapped in a sentence that is part of a longer quote, but the `picoCTF{...}` substring is right there.

### Step 3: Extract the flag

```
picoCTF{gr3p_15_m4g1c_ef8790dc}
```

**Flag:** `picoCTF{gr3p_15_m4g1c_ef8790dc}`

---

## What Happened Internally (Timeline)

| Step | What I did | What happened |
|---|---|---|
| 1 | `unzip big-zip-files.zip` | Extracted the tree (hundreds of files). |
| 2 | `grep -r 'picoCTF' .` | Walked every file under `.` and printed lines containing `picoCTF`. |
| 3 | Eyeballed the output, copied the `picoCTF{...}` substring | Got the flag. |

The whole solve is two commands. The challenge is teaching you one reflex: when faced with a "find the flag in a tree" problem, reach for `grep -r` first.

---

## Alternative Solve Methods

### Method 1: The intended `grep -r`

```
grep -r 'picoCTF' big_zip_extracted/
```

That is the answer the author wants. Done.

### Method 2: `find` + `xargs grep`

For when you want to filter by file type or extension:

```
$ find big_zip_extracted/ -type f -name '*.txt' -exec grep -H 'picoCTF' {} +
```

This only searches `.txt` files. The `-H` flag makes grep print the filename even when invoked through `-exec`.

### Method 3: `grep -rl` (just filenames)

If the flag is long and you want only the file path first, then read it:

```
$ grep -rl 'picoCTF' big_zip_extracted/
big_zip_extracted/.../whzxrpivpqld.txt
$ cat big_zip_extracted/.../whzxrpivpqld.txt
```

`-l` (lowercase L) tells grep to print only the names of files that contain a match, instead of the matching lines. Useful when you want a quick "which files have this pattern" answer.

### Method 4: `rg` (ripgrep) if you have it

```
$ rg 'picoCTF' big_zip_extracted/
```

`rg` is a modern alternative to `grep -r` that is faster and has saner defaults. If you do enough CTF work, install it; you will not go back.

### Method 5: PowerShell on Windows

If you are on Windows, the equivalent is a `Get-ChildItem -Recurse` + `Select-String` loop:

```powershell
Get-ChildItem -Path C:\Users\ASUS\Downloads\big_zip_extracted -Recurse -File |
    ForEach-Object {
        $m = Select-String -Path $_.FullName -Pattern 'picoCTF' -ErrorAction SilentlyContinue
        if ($m) { Write-Host $_.FullName; $m | Select-Object -ExpandProperty Line }
    }
```

That is what I used on the Windows side. It is wordier than the bash version, but the logic is the same: walk every file, search each one, print matches.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `unzip` | Extract the big zip archive |
| `grep -r` | Recursive search across the whole extracted tree |
| `find ... -exec grep` (alt) | Same idea, more control over which files to scan |
| `grep -rl` (alt) | List matching files instead of matching lines |
| `rg` (alt) | Modern, faster recursive grep |
| PowerShell `Get-ChildItem -Recurse` + `Select-String` (alt) | Windows-native equivalent |

---

## Key Takeaways

- **`grep -r` is the answer to "find a string in a tree of files."** It is one of the most-used commands in CTFs and in real-world incident response. Memorize the shape: `grep -r 'pattern' directory/`.
- **The hint is a literal how-to.** "Can grep be instructed to look at every file in a directory and its subdirectories?" is the author spelling out the answer. Take the hint.
- **`-l` lists only file names, `-c` counts matches, `-i` is case-insensitive, `--include='*.txt'` filters by glob.** Knowing the four or five most useful flags turns `grep` from "I hope this works" into a precision tool.
- **Manual `cd` + `cat` will not scale.** When a challenge says "find the flag in a tree" and the tree is bigger than 10 files, do not start clicking. The first reflex is `grep -r`.
- **Wordplay, as always:** the flag decodes from leetspeak as **grep is magic** — and once you have used `grep -r` on a deep tree a few times, you will agree. Manually walking that 3 MB zip would have taken an hour; `grep -r` finds it in milliseconds. The `ef8790dc` suffix is the per-instance hex identifier. The author wants you to feel the magic, not just read about it.
