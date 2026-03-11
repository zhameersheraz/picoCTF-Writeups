# Collaborative Development - picoCTF 2024 Writeup

**Challenge:** Collaborative Development  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** Not specified  
**Flag:** `picoCTF{t3@mw0rk_m@k3s_th3_dr3@m_w0rk_6c06cec1}`

---

## Description

My team has been working very hard on new features for our flag printing program! I wonder how they'll work together? Download the challenge file: `challenge.zip`

---

## Hints

1. `git branch -a` will let you see available branches

---

## Solution

### Step 1: Unzip the Challenge File

```bash
unzip challenge.zip
cd drop-in
```

The zip contained a git repository with a `flag.py` file and multiple branches.

### Step 2: Fix the Safe Directory Error

When trying to run git commands, I got a `fatal: detected dubious ownership` error because the repo was extracted to a shared folder (`/media/sf_downloads`). Fixed it with:

```bash
git config --global --add safe.directory /media/sf_downloads/drop-in
```

### Step 3: Check the Branches

```bash
git branch -a
```

The repo had three feature branches:
```
feature/part-1
feature/part-2
feature/part-3
main
```

### Step 4: Read Each Branch

Each feature branch added a part of the flag to `flag.py`. I used `git show` to read each branch's changes:

**feature/part-1:**
```bash
git show feature/part-1
```
```python
print("picoCTF{t3@mw0rk_", end='')
```

**feature/part-2:**
```bash
git show feature/part-2
```
```python
print("m@k3s_th3_dr3@m_", end='')
```

**feature/part-3:**
```bash
git show feature/part-3
```
```python
print("w0rk_6c06cec1}")
```

### Step 5: Combine the Parts

Putting all three print statements together in order:

```
picoCTF{t3@mw0rk_  +  m@k3s_th3_dr3@m_  +  w0rk_6c06cec1}
= picoCTF{t3@mw0rk_m@k3s_th3_dr3@m_w0rk_6c06cec1}
```

Got the flag! 🎯

---

## Why This Works — Git Branches

### What are Git Branches?

Git branches allow developers to work on different features simultaneously without affecting each other's code. Each branch is an independent line of development. In this challenge, the flag was split across three separate branches — simulating real collaborative development where teammates each add their own piece.

### Visual of the Attack

```
main branch:        print("Printing the flag...")
                              +
feature/part-1:     print("picoCTF{t3@mw0rk_", end='')
                              +
feature/part-2:     print("m@k3s_th3_dr3@m_", end='')
                              +
feature/part-3:     print("w0rk_6c06cec1}")
                              ↓
Full flag:   picoCTF{t3@mw0rk_m@k3s_th3_dr3@m_w0rk_6c06cec1}  🎯
```

---

## Commands Used

```bash
# Unzip and navigate
unzip challenge.zip
cd drop-in

# Fix ownership error (if extracting to shared/external folder)
git config --global --add safe.directory /media/sf_downloads/drop-in

# List all branches
git branch -a

# View each branch's changes
git show feature/part-1
git show feature/part-2
git show feature/part-3
```

---

## Key Takeaways

- **`git branch -a`** shows all local and remote branches — always run this first on git challenges
- **`git show <branch>`** displays the latest commit and diff on that branch
- The `fatal: detected dubious ownership` error is common on Kali shared folders — fix with `git config --global --add safe.directory <path>`
- Flags split across multiple branches is a classic picoCTF git challenge pattern
- Always read ALL branches — the flag is often distributed across them
