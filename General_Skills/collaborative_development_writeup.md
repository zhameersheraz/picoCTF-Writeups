# Collaborative Development — picoCTF Writeup

**Challenge:** Collaborative Development  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{t3@mw0rk_m@k3s_th3_dr3@m_w0rk_6c06cec1}`

---

## Description

> My team has been working very hard on new features for our flag printing program! I wonder how they'll work together?
> Download the challenge file: `challenge.zip`

**Hint shown in challenge:** `` `git branch -a` will let you see available branches ``

---

## Background Knowledge (Read This First!)

### What are Git Branches?

Git branches allow developers to work on different features simultaneously without affecting each other's code. Each branch is an independent line of development. In this challenge, the flag was **split across three separate branches**.

---

## Solution — Step by Step

### Step 1 — Unzip the Challenge File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ unzip challenge.zip
└─$ cd drop-in
```

### Step 2 — Fix the Safe Directory Error (if needed)

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git config --global --add safe.directory /media/sf_downloads/drop-in
```

### Step 3 — Check the Branches

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git branch -a
  feature/part-1
  feature/part-2
  feature/part-3
  main
```

### Step 4 — Read Each Branch

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git show feature/part-1
```
```python
print("picoCTF{t3@mw0rk_", end='')
```

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git show feature/part-2
```
```python
print("m@k3s_th3_dr3@m_", end='')
```

```
┌──(zham㉿kali)-[/media/sf_downloads/drop-in]
└─$ git show feature/part-3
```
```python
print("w0rk_6c06cec1}")
```

### Step 5 — Combine the Parts

```
picoCTF{t3@mw0rk_  +  m@k3s_th3_dr3@m_  +  w0rk_6c06cec1}
= picoCTF{t3@mw0rk_m@k3s_th3_dr3@m_w0rk_6c06cec1}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `unzip` | Extract the challenge zip |
| `git branch -a` | List all branches |
| `git show` | View changes in each branch |

---

## Key Takeaways

- `git branch -a` shows all local and remote branches — always run this first on git challenges
- `git show <branch>` displays the latest commit and diff on that branch
- The `fatal: detected dubious ownership` error is common on Kali shared folders — fix with `git config --global --add safe.directory <path>`
- Flags split across multiple branches is a classic picoCTF git challenge pattern
- Always read ALL branches — the flag is often distributed across them
