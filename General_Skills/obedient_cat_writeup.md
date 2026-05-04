# Obedient Cat — picoCTF Writeup

**Challenge:** Obedient Cat  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{s4n1ty_v3r1f13d_9b8fa0bc}`  

---

## Description

> This file has a flag in plain sight (aka "in-the-clear").
> Download: `flag`

**Hint 1:** `Any hints about entering a command into the Terminal (such as the next one), will start with a '$'... everything after the dollar sign will be typed (or copy and pasted) into your Terminal.`

---

## Background Knowledge (Read This First!)

### What is `cat`?

**`cat`** (concatenate) is one of the most fundamental Linux commands. It reads the contents of a file and prints them to the terminal. Despite its simple name, you'll use it constantly in CTFs.

```bash
cat filename
```

### What does "in-the-clear" mean?

"In-the-clear" (or "in the clear") means the data is **unencrypted and unencoded** — completely readable as plain text. No decoding, no cracking, no reversing needed. The flag is just sitting there waiting to be read.

### What is the `$` in terminal commands?

As Hint 1 explains, the `$` symbol represents the **terminal prompt** — it's not part of the command. Everything **after** the `$` is what you type. So `$ cat flag` means type `cat flag` into your terminal.

---

## Solution — Step by Step

### Step 1 — Read the file with `cat`

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat flag
picoCTF{s4n1ty_v3r1f13d_9b8fa0bc}
```

✅ Got the flag! 🎯 That's it — one command.

---

## Alternative Methods

### Alternative 1 — Open the file with any text editor

```bash
nano flag
# or
gedit flag
```

Since the file is plain text, any text editor will show the flag.

### Alternative 2 — Use `strings`

```bash
strings flag
```

`strings` extracts all readable text from a file — works on plain text files too.

### Alternative 3 — Use `less` or `more`

```bash
less flag
# or
more flag
```

Both are file viewers that let you read file contents page by page. Press `q` to quit.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `cat` | Read the plain text file | ⭐ Easy |

---

## Key Takeaways

- **`cat filename`** is the most basic way to read any file in Linux — use it constantly in CTFs
- **"In-the-clear"** means no encoding or encryption — the flag is exactly what it looks like
- **The `$` is the prompt, not part of the command** — a key convention to understand when following terminal instructions
- This is a **sanity check challenge** — its purpose is to confirm you know how to download a file and read it
- The flag `s4n1ty_v3r1f13d` → "sanity verified" — you've proven you can handle the basics!
