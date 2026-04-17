# Time Machine — picoCTF Writeup

**Challenge:** Time Machine  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{t1m3m@ch1n3_e8c98b3a}`

---

## Description

> What was I last working on? I remember writing a note to help me remember...
> Download the challenge files here: challenge.zip

**Hint shown in challenge:** `The \`cat\` command will let you read a file, but that won't help you here!`

---

## Background Knowledge (Read This First!)

### What is git reflog?

`git reflog` shows the **history of all commits** made in a repository, including the commit messages. In this challenge, the flag was stored inside the commit message of the initial commit.

The hint "Time Machine" refers to Git being like a time machine — you can look back at all previous commits and changes.

---

## Solution — Step by Step

### Step 1 — Download and Extract the File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip challenge.zip
└─$ cd drop-in
```

### Step 2 — Look at the Files

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ ls -la
drwxr-xr-x  3 zham zham 4096 .
drwxr-xr-x 29 zham zham 4096 ..
drwxr-xr-x  8 zham zham 4096 .git
-rw-r--r--  1 zham zham   87 message.txt
```

I noticed a `.git` folder — this means the directory is a **Git repository**!

### Step 3 — Read message.txt

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ cat message.txt
This is what I was working on, but I'd need to look at my commit history to know why...
```

The hint says `cat` won't help — the message tells us to look at commit history!

### Step 4 — Check Git Commit History

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git reflog
712314f (HEAD -> master) HEAD@{0}: commit (initial): picoCTF{t1m3m@ch1n3_e8c98b3a}
```

The flag was in the **first commit message**! 🎯

---

## Alternative Method

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git log --oneline
712314f (HEAD -> master) picoCTF{t1m3m@ch1n3_e8c98b3a}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the zip file |
| `cat` | Read message.txt |
| `git reflog` | View commit history and find the flag |
| `git log` | Alternative way to view commit history |

---

## Key Takeaways

- Always check for a `.git` folder — it means the directory is a Git repository
- Git stores **all commit messages** in its history — never put sensitive info in commit messages
- `git reflog` and `git log` let you look back at all previous commits
- The challenge name "Time Machine" was hinting at going back in time through Git history
- The hint "cat won't help you" was pointing away from reading files and toward Git commands
