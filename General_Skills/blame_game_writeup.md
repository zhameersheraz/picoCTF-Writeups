# Blame Game — picoCTF Writeup

**Challenge:** Blame Game  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{@sk_th3_1nt3rn_e9957ce1}`  

---

## Description

> Someone's commits seems to be preventing the program from working. Who is it?
> Download the challenge files here: `challenge.zip`

**Hint 1:** `In collaborative projects, many users can make many changes. How can you see the changes within one file?`

**Tags:** `git`, `browser_webshell_solvable`

---

## Background Knowledge (Read This First!)

### What is Git?

**Git** is a version control system — it tracks every change ever made to a project's files. Every time someone saves a change (called a **commit**), Git records what changed, when it changed, and **who made the change**. This lets teams collaborate on code and trace the history of every line.

### What is `git log`?

`git log` shows the full history of commits in a repository — most recent first. Each entry shows a commit hash, author, date, and message. In this challenge, every commit is suspiciously named `"important business work"` — clearly not helpful.

### What is `git blame`?

`git blame` is the key command here. It annotates every single line of a file and shows:
- **Who last changed that line** (the author)
- **When** it was changed
- **Which commit** introduced it

It's called "blame" because it tells you exactly who to blame for each line of code. In CTFs, the author field is a common hiding spot for flags.

### What is `challenge.zip`?

The zip contains a folder called `drop-in/` which is a full Git repository — it has a `.git/` folder (the hidden directory where Git stores all its history) and a single Python file called `message.py`.

---

## Solution — Step by Step

### Step 1 — Extract the zip

```
┌──(zham㉿kali)-[~]
└─$ unzip challenge.zip
```

This creates a folder called `drop-in/` containing the git repository.

```
┌──(zham㉿kali)-[~]
└─$ cd drop-in
```

### Step 2 — Check the git log

```
┌──(zham㉿kali)-[/drop-in]
└─$ git log --oneline
ee09a4c important business work
7d196f4 important business work
4d6ef56 important business work
f901ef0 important business work
...
```

Every single commit has the same message: `"important business work"`. There are many commits and nothing stands out from the messages alone. The hint says to look at changes **within one file** — that points to `git blame`.

### Step 3 — Run git blame on message.py

```
┌──(zham㉿kali)-[/drop-in]
└─$ git blame message.py
```

Output:

```
fadeca94 (picoCTF{@sk_th3_1nt3rn_e9957ce1} 2024-03-09 21:09:11 +0000 1) print("Hello, World!"
```

The flag is sitting right there — hidden as the **author name** of the commit that introduced line 1 of `message.py`.

✅ Got the flag! 🎯

---

## Breaking Down the Output

```
fadeca94  (picoCTF{@sk_th3_1nt3rn_e9957ce1}  2024-03-09 21:09:11 +0000  1)  print("Hello, World!"
│          │                                   │                           │   │
commit     author name ← FLAG IS HERE          timestamp                  line  content
hash
```

The challenge author set the Git **username** to the flag itself before making the commit. Since `git blame` always shows the author name, the flag was always just one command away.

---

## Alternative Methods

### Alternative 1 — Search the entire git log for the flag

If you didn't know which file to blame, you can search all commit metadata at once:

```bash
git log --all --format="%an %s" | grep picoCTF
```

- `--all` — searches every branch
- `--format="%an %s"` — prints author name (`%an`) and commit message (`%s`)
- `grep picoCTF` — filters for the flag format

### Alternative 2 — Search all git objects directly

For a brute-force approach that searches everything in the `.git` folder:

```bash
git log --all --format="%H %an %ae %s" | grep -i pico
```

Or even raw:
```bash
grep -r "picoCTF" .git/
```

This searches every raw git object file for the flag string.

### Alternative 3 — Use git shortlog

```bash
git shortlog -s -n
```

This lists all authors and how many commits they made. A suspicious author name (like one containing `picoCTF{`) would immediately stand out.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `unzip` | Extract the challenge archive | ⭐ Easy |
| `git log` | View commit history | ⭐ Easy |
| `git blame` | Show who last changed each line | ⭐ Easy |
| `grep` (optional) | Search git metadata for flag pattern | ⭐ Easy |

---

## Key Takeaways

- **`git blame` shows the author of every line** — not just the content. Author names, emails, and commit messages are all places flags can hide in git challenges
- **Git stores everything permanently** — every commit, every author, every message is preserved in the `.git/` folder forever, even if the file contents change later
- When a CTF challenge involves git and asks "who did it?", `git blame` is almost always the answer
- **`git log --format="%an"`** is a quick way to dump all author names when you're not sure which file to blame
- The flag name `@sk_th3_1nt3rn` → "ask the intern" — a nod to blaming the intern for breaking things, which is exactly what `git blame` is for
