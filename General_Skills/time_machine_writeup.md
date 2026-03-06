# Time Machine - picoCTF Writeup

**Challenge:** Time Machine  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{t1m3m@ch1n3_e8c98b3a}`

---

## Description

What was I last working on? I remember writing a note to help me remember...

Download the challenge files here: challenge.zip

---

## Hints

1. The `cat` command will let you read a file, but that won't help you here!

---

## Solution

### Step 1: Download and Extract the File

```bash
wget https://artifacts.picoctf.net/c_titan/162/challenge.zip
unzip challenge.zip
cd drop-in
```

### Step 2: Look at the Files

```bash
ls -la
```

**Output:**
```
drwxr-xr-x  3 zham zham 4096 .
drwxr-xr-x 29 zham zham 4096 ..
drwxr-xr-x  8 zham zham 4096 .git
-rw-r--r--  1 zham zham   87 message.txt
```

I noticed a `.git` folder — this means the folder is a **Git repository**!

### Step 3: Read message.txt

```bash
cat message.txt
```

**Output:**
```
This is what I was working on, but I'd need to look at my commit history to know why...
```

The hint says `cat` won't help — and the message tells us to look at commit history!

### Step 4: Check Git Commit History

```bash
git reflog
```

**Output:**
```
712314f (HEAD -> master) HEAD@{0}: commit (initial): picoCTF{t1m3m@ch1n3_e8c98b3a}
```

The flag was in the **first commit message**! 🎯

---

## Why This Works

### What is `git reflog`?

`git reflog` shows the **history of all commits** made in a repository, including the commit messages. In this challenge, the developer stored the flag inside the commit message of the initial commit.

The hint "Time Machine" refers to Git being like a time machine — you can look back at all previous commits and changes.

### Simple Breakdown

```
Extract zip file
      |
      v
Notice .git folder inside
      |
      v
message.txt says to look at commit history
      |
      v
Run git reflog
      |
      v
Flag is in the first commit message!
```

---

## Alternative Method

You can also use `git log` to see commit history:

```bash
git log --oneline
```

**Output:**
```
712314f (HEAD -> master) picoCTF{t1m3m@ch1n3_e8c98b3a}
```

---

## Commands Used

```bash
# Download and extract
wget https://artifacts.picoctf.net/c_titan/162/challenge.zip
unzip challenge.zip
cd drop-in

# Read the message file
cat message.txt

# Check git commit history
git reflog

# Alternative
git log --oneline
```

---

## Flag

```
picoCTF{t1m3m@ch1n3_e8c98b3a}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the zip file |
| `unzip` | Extract the zip file |
| `cat` | Read message.txt |
| `git reflog` | View commit history and find the flag |

---

## Key Takeaways

- Always check for a `.git` folder — it means the directory is a Git repository
- Git stores **all commit messages** in its history — never put sensitive info in commit messages
- `git reflog` and `git log` let you look back at all previous commits
- The challenge name "Time Machine" was hinting at going back in time through Git history
- The hint "cat won't help you" was pointing away from reading files and toward Git commands
