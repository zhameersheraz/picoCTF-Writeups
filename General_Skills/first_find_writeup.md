# First Find — picoCTF Writeup

**Challenge:** First Find  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{f1nd_15_f457_ab443fd1}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Unzip this archive and find the file named 'uber-secret.txt'

**Attachment:** `files.zip`

---

## Hints

(No hints)

---

## Background Knowledge (Read This First!)

### What is `find`?

`find` is the Unix tool for searching a directory tree. The basic incantation is:

```
find <path> -name <filename>
```

It walks every directory under `<path>` (recursively, by default) and prints the full path of every file whose name matches. It is the right answer whenever a CTF challenge says "find the file named X" or "where is X in this mess of folders."

For this challenge, the file is hidden 6 directories deep inside a zip archive. Typing out the path by hand is exactly the kind of tedious thing `find` was invented to avoid.

### Why dotfile names matter

`find -name 'uber-secret.txt'` will only find files literally named `uber-secret.txt`. If the filename has a different case or includes spaces, you need to match exactly. For "fuzzy" searches, `find -iname` (case-insensitive) or `find -name '*uber*'` (glob) is your friend. For this challenge the filename is fixed, so the literal match works.

---

## Solution

### Step 1: Unzip the archive

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip files.zip -d files_extracted
Archive:  files.zip
   creating: files_extracted/files/...
```

The archive expands to a directory tree with about 22 files and folders.

### Step 2: Run `find` from the top

The filename is `uber-secret.txt`. I pointed `find` at the extracted directory and asked for it by name:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ find files_extracted -name 'uber-secret.txt'
files_extracted/files/adequate_books/more_books/.secret/deeper_secrets/deepest_secrets/uber-secret.txt
```

One match, one line. The path is six directories deep (and the first one — `.secret` — is a hidden folder, which is why `ls` would not have shown it on the first pass).

### Step 3: Read the file

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat files_extracted/files/adequate_books/more_books/.secret/deeper_secrets/deepest_secrets/uber-secret.txt
picoCTF{f1nd_15_f457_ab443fd1}
```

**Flag:** `picoCTF{f1nd_15_f457_ab443fd1}`

---

## What Happened Internally (Timeline)

| Step | What I did | What happened |
|---|---|---|
| 1 | `unzip files.zip` | Extracted 22 files and directories into a tree. |
| 2 | `find . -name 'uber-secret.txt'` | Walked the tree, found one match. |
| 3 | `cat <the path>` | Printed the flag. |

The whole solve is two commands. The point of the challenge is to teach you that `find` is the right tool when you do not know where a file is, and that hidden directories (`.secret`) are still found by `find` — they are not hidden from `find` because `find` walks everything by default.

---

## Alternative Solve Methods

### Method 1: The intended path (just `find`)

```
$ find . -name 'uber-secret.txt'
```

That is the whole point. Use it.

### Method 2: `find` with a glob, in case the name is fuzzy

```
$ find . -name '*uber*'
```

If you are not sure of the exact filename, a glob like `*uber*` matches anything containing the word "uber". For this challenge the exact name is given, so this is overkill, but it is a useful reflex.

### Method 3: `grep -r` for the flag content

If you know the flag starts with `picoCTF{`, you can skip the file-name search entirely:

```
$ grep -r 'picoCTF' files_extracted/
files_extracted/files/.../uber-secret.txt:picoCTF{f1nd_15_f457_ab443fd1}
```

`grep -r` walks the directory tree and prints every line that contains the pattern. Same one-line answer, different mental model — you search for the *content* instead of the *name*.

### Method 4: `find ... -exec cat`

If you want to skip step 3 entirely, ask `find` to run `cat` on every match:

```
$ find files_extracted -name 'uber-secret.txt' -exec cat {} +
picoCTF{f1nd_15_f457_ab443fd1}
```

`{}` is the placeholder for the matched path, and `+` says "build a single command line out of all the matches" (more efficient than `\;` which runs `cat` once per match). The output is the flag, no further step needed.

### Method 5: PowerShell

On Windows:

```powershell
Get-ChildItem -Path C:\Users\ASUS\Downloads\files_extracted -Recurse -Filter 'uber-secret.txt' |
    ForEach-Object { Get-Content $_.FullName }
```

`Get-ChildItem -Recurse` walks the tree, `-Filter` matches the filename, and `Get-Content` reads the file. Three lines, but each one is doing the equivalent of one of the bash steps.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `unzip` | Extract the zip archive |
| `find` | Locate the file by name in the directory tree |
| `cat` | Print the file's contents |
| `grep -r` (alt) | Search by content instead of name |
| `find ... -exec cat` (alt) | One-liner that finds and reads in one go |
| PowerShell `Get-ChildItem` (alt) | Windows-native equivalent |

---

## Key Takeaways

- **`find` is the right tool for "where is this file?"** Memorize `find <path> -name '<filename>'`. The default behaviour is recursive, so it walks every subdirectory for you.
- **`find` sees hidden files.** A directory named `.secret` looks empty to `ls` (without `-a`), but `find` walks it without complaint. This is by design: `find` is a tool for *finding* things, and hidden files are still findable.
- **The intended path is the most boring one.** This challenge is literally "use `find`." The author wants you to walk away with the reflex. Do not overthink it.
- **`grep -r <pattern> <dir>` is the complement:** when you do not know the file name but you know what is *inside* it, recursive grep is your answer. `find` searches by metadata (name, size, mtime, permissions); `grep -r` searches by content. Knowing which one fits the question is the skill.
- **Wordplay, as always:** the flag decodes from leetspeak as **find is fast** — and the author is right, `find` is *blazing* fast on a tree of 22 files. The `ab443fd1` suffix is the per-instance hex identifier. The author wants the muscle memory: when you see "find a file by name in a tree," your fingers should already be typing `find -name`.

