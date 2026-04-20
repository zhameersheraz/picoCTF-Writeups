# First Find — picoCTF Writeup

**Challenge:** First Find  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{f1nd_15_f457_ab443fd1}`  

---

## Description

> Unzip this archive and find the file named 'uber-secret.txt'
> Download zip file

**Hints:** None

---

## Background Knowledge (Read This First!)

### What is a ZIP archive?

A **ZIP file** is a compressed archive that bundles multiple files and folders into one. The `unzip` command extracts everything inside it back into the original folder structure.

### What are hidden folders?

On Linux, any file or folder whose name **starts with a dot** (`.`) is hidden. For example `.secret` is a hidden folder. Hidden files and folders do not show up when you run a plain `ls` command — you need `ls -a` (show all) to see them.

This is a common trick in CTF challenges — the flag is buried inside a hidden folder that beginners might miss.

### What is the `find` command?

`find` is a powerful Linux command that searches through an entire directory tree looking for files matching a given name or pattern. Instead of manually opening every folder, you can let `find` locate the target file instantly — no matter how deep or hidden it is.

```bash
find <starting_folder> -name "filename.txt"
```

This is exactly what the challenge name is hinting at — **First Find** → use the `find` command!

---

## Solution — Step by Step

### Step 1 — Extract the zip

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip files.zip
Archive:  files.zip
   creating: files/
   creating: files/satisfactory_books/
   creating: files/satisfactory_books/more_books/
  inflating: files/satisfactory_books/more_books/37121.txt.utf-8
  ...
   creating: files/adequate_books/more_books/.secret/
   creating: files/adequate_books/more_books/.secret/deeper_secrets/
   creating: files/adequate_books/more_books/.secret/deeper_secrets/deepest_secrets/
 extracting: files/adequate_books/more_books/.secret/deeper_secrets/deepest_secrets/uber-secret.txt
  ...
```

The zip extracts into a `files/` folder with many subdirectories. Notice that the extraction output reveals a hidden folder called `.secret` buried deep inside — but you would never find it by browsing manually.

### Step 2 — Read the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat files/adequate_books/more_books/.secret/deeper_secrets/deepest_secrets/uber-secret.txt
picoCTF{f1nd_15_f457_ab443fd1}
```

✅ Got the flag! 🎯

---

## The Intended Solution — Using `find`

The challenge name **"First Find"** is a direct hint to use the `find` command. Instead of reading the unzip output carefully, the proper approach is:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ find files/ -name "uber-secret.txt"
files/adequate_books/more_books/.secret/deeper_secrets/deepest_secrets/uber-secret.txt
```

`find` searches the entire `files/` directory tree and instantly returns the full path to `uber-secret.txt` — no matter how deeply nested or hidden it is.

Then read it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat files/adequate_books/more_books/.secret/deeper_secrets/deepest_secrets/uber-secret.txt
picoCTF{f1nd_15_f457_ab443fd1}
```

You can also combine both steps into one command:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat $(find files/ -name "uber-secret.txt")
picoCTF{f1nd_15_f457_ab443fd1}
```

---

## Why Was the File Hard to Find Manually?

The folder structure was deliberately designed to hide the file:

```
files/
├── satisfactory_books/
├── acceptable_books/
└── adequate_books/
    └── more_books/
        └── .secret/          ← hidden folder (starts with .)
            └── deeper_secrets/
                └── deepest_secrets/
                    └── uber-secret.txt   ← flag is here
```

Two things make it tricky:
1. **`.secret` is a hidden folder** — `ls` won't show it; you need `ls -a`
2. **It's deeply nested** — 5 levels deep inside the zip

The `find` command bypasses both problems in one shot.

---

## Alternative Methods

### Alternative 1 — Browse hidden folders with `ls -a`

If you want to manually explore, use `ls -a` to reveal hidden folders:

```bash
ls -a files/adequate_books/more_books/
# shows: .secret  1023.txt.utf-8  ...

ls files/adequate_books/more_books/.secret/deeper_secrets/deepest_secrets/
# shows: uber-secret.txt
```

### Alternative 2 — Use `find` with `-type f` to list all files

```bash
find files/ -type f
```

This lists every single file in the entire archive — `uber-secret.txt` will appear in the list and stands out immediately among `.txt.utf-8` book files.

### Alternative 3 — grep through everything

```bash
grep -r "picoCTF" files/
```

This searches every file's content for the flag format and prints the matching line along with the filename.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `unzip` | Extract the zip archive | ⭐ Easy |
| `find` | Locate a file by name anywhere in a directory tree | ⭐ Easy |
| `cat` | Read the flag file | ⭐ Easy |
| `ls -a` (optional) | Reveal hidden folders | ⭐ Easy |
| `grep -r` (optional) | Search file contents recursively | ⭐ Easy |

---

## Key Takeaways

- **The challenge name is the hint** — "First Find" → use `find`
- **Hidden folders start with `.`** — always use `ls -a` when browsing CTF directories so you don't miss hidden items
- **`find <dir> -name "filename"`** is the fastest way to locate any file regardless of how deep or hidden it is
- **`cat $(find . -name "target.txt")`** combines finding and reading into one command — a useful one-liner for CTFs
- The flag confirms it: `f1nd_15_f457` → "find is fast" — the whole point of the challenge is to learn the `find` command
