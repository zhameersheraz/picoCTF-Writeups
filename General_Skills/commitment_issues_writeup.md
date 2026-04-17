# Commitment Issues — picoCTF Writeup

**Challenge:** Commitment Issues  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{s@n1t1z3_cf09a485}`

---

## Description

> Ever had to keep your commits... secret?
> Download the challenge files here: challenge.zip

---

## Background Knowledge (Read This First!)

### Git Never Truly Deletes Anything

Even when someone removes a file or overwrites content in a commit, **Git keeps the full history** of all previous commits. You can always go back to any previous state using `git checkout` with the commit hash.

In this challenge, the developer thought they were hiding the flag by making a new commit. But the original commit with the flag was still stored in Git history!

---

## Solution — Step by Step

### Step 1 — Download and Extract the File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip challenge.zip
└─$ cd drop-in
```

### Step 2 — Read message.txt

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ cat message.txt
TOP SECRET
```

The file just says "TOP SECRET" — the flag was removed!

### Step 3 — Check Git Commit History

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git reflog
ef0b7cc (HEAD -> master) HEAD@{0}: commit: remove sensitive info
ea859bf HEAD@{1}: commit (initial): create flag
```

Two commits found — the first one originally created the flag!

### Step 4 — Go Back to the First Commit

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git checkout ea859bf
```

### Step 5 — Read message.txt Again

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ cat message.txt
picoCTF{s@n1t1z3_cf09a485}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the zip file |
| `cat` | Read message.txt |
| `git reflog` | View all commits and their hashes |
| `git checkout` | Travel back to a previous commit |

---

## Key Takeaways

- Git stores the **full history** of all commits — deleting content in a new commit does not erase old commits
- `git reflog` shows every commit ever made, including ones that removed files
- `git checkout <hash>` lets you travel back to any previous commit
- Never store sensitive info like flags, passwords or API keys in Git — even deleted commits can be recovered
- Properly sanitizing Git history requires tools like `git filter-branch` or `git filter-repo`
