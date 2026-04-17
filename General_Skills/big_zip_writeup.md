# Big Zip — picoCTF Writeup

**Challenge:** Big Zip  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{gr3p_15_m4g1c_ef8790dc}`  

---

## Description

> Unzip this archive and find the flag.
> Download the zip file.

**Hint shown in challenge:** `Can grep be instructed to look at every file in a directory and its subdirectories?`

---

## Background Knowledge (Read This First!)

### What is grep?

`grep` is a Linux command used to **search for text inside files**. By default it searches
one file at a time — but with the right flag, it can search through thousands of files
and folders automatically.

```
grep "word" file.txt        ← searches one file
grep -r "word" folder/      ← searches ALL files in folder and subfolders
```

### Why does this matter here?

The zip contains **9090 files** spread across deeply nested folders. Opening them one by
one would take hours. `grep -r` scans all of them in seconds — that's the entire point
of this challenge.

### ⚠️ Note

No extra tools needed — only `unzip` and `grep`, both pre-installed on Kali Linux.

---

## Solution — Step by Step

### Step 1 — Examine the Zip File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ exiftool big-zip-files.zip
File Name                       : big-zip-files.zip
File Size                       : 3.2 MB
File Type                       : ZIP
Warning : [minor] Use the Duplicates option to extract tags for all 9090 files
```

✅ Confirmed ZIP file containing **9090 files**. Way too many to open manually.

---

### Step 2 — Extract the Zip

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip -o big-zip-files.zip -d big-zip-files/
```

- `-o` = overwrite existing files automatically (no y/n prompts)
- `-d big-zip-files/` = extract into this folder

Wait for it to finish extracting all 9090 files — this takes about a minute.

✅ All files extracted successfully.

---

### Step 3 — Search All Files at Once with grep

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep -r "picoCTF" big-zip-files/
big-zip-files/big-zip-files/folder_pmbymkjcya/folder_cawigcwvgv/folder_ltdayfmktr/folder_fnpfclfyee/whzxrpivpqld.txt:
information on the record will last a billion years. Genes and brains and books encode picoCTF{gr3p_15_m4g1c_ef8790dc}
```

Got the flag! 🎯

What `-r` does here:
- Without `-r` → grep only checks one file
- With `-r` → grep dives into **every folder and subfolder** automatically
- Found the flag buried **4 folders deep** inside a random `.txt` file among 9090 files

---

## Alternative Methods

**Method 1 — grep with filename only (cleaner output)**
```bash
grep -rl "picoCTF" big-zip-files/
```
`-l` prints just the filename instead of the full line — useful when you just need to
locate which file contains the flag.

**Method 2 — find + grep combo**
```bash
find big-zip-files/ -type f -exec grep -l "picoCTF" {} +
```
Uses `find` to list all files, then passes each one to `grep`.

**Method 3 — grep with line number**
```bash
grep -rn "picoCTF" big-zip-files/
```
`-n` shows the line number inside the file where the flag was found.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `exiftool` | Check file type and number of files in zip |
| `unzip` | Extract all 9090 files from the archive |
| `grep -r` | Recursively search all files for the flag |

---

## Key Takeaways

- `grep -r` is one of the most powerful tools in CTF — it searches thousands of files
  instantly without opening a single one manually
- The hint directly pointed to `grep -r` — always read hints carefully!
- When unzipping large archives, use `-o` flag to avoid being prompted for every file
- The flag name says it all — **gr3p 15 m4g1c** (grep is magic)
- File location doesn't matter when you have `grep -r` — even 4 folders deep is found
  in seconds
