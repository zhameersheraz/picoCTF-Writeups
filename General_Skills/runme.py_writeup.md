# runme.py — picoCTF Writeup

**Challenge:** runme.py  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{run_s4n1ty_run}`  

---

## Description

> Run the `runme.py` script to get the flag. Download the script with your browser or with `wget` in the webshell.

**Hint 1:** `If you have Python on your computer, you can download the script normally and run it. Otherwise, use the wget command in the webshell.`

**Tag:** `Python`

---

## Background Knowledge (Read This First!)

### What is Python?

**Python** is one of the most popular programming languages in the world. Python scripts are plain text files ending in `.py` that you run with the `python3` command. Unlike compiled languages, Python scripts are human-readable — you can open a `.py` file and read exactly what it does before running it.

### What is `python3`?

`python3` is the command that executes a Python script. On Kali Linux (and most modern systems), Python 3 is pre-installed. You run a script like this:

```bash
python3 script.py
```

### What does this script actually do?

```python
flag = 'picoCTF{run_s4n1ty_run}'
print(flag)
```

It stores the flag in a variable and prints it. That's it — the simplest possible Python program. The challenge is just teaching you how to run a Python script.

### ⚠️ Important Note

Make sure you are in the correct directory before running the script, or provide the full path to the file.

---

## The Script

```python
#!/usr/bin/python3
################################################################################
# Python script which just prints the flag
################################################################################

flag = 'picoCTF{run_s4n1ty_run}'
print(flag)
```

Opening the file shows the flag is stored directly in the script. All you need to do is run it.

---

## Solution — Step by Step

### Step 1 — Navigate to the downloads folder

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Run the script

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 runme.py
picoCTF{run_s4n1ty_run}
```

✅ Got the flag! 🎯

---

## Alternative Methods

### Alternative 1 — Read the flag directly with `cat`

Since the flag is stored as a plaintext string inside the script, you can just read the file without running it:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat runme.py
```

The flag is visible in the source code on the `flag =` line.

### Alternative 2 — Use `grep` to extract just the flag

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ grep picoCTF runme.py
flag ='picoCTF{run_s4n1ty_run}'
```

### Alternative 3 — Download and run using `wget` (webshell method)

If you are using the picoCTF webshell instead of your own Kali machine:

```bash
wget <download_link> -O runme.py
python3 runme.py
```

`wget` downloads the file from the URL and `-O runme.py` saves it with that filename.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `python3` | Execute the Python script | ⭐ Easy |
| `cat` (optional) | Read the script contents directly | ⭐ Easy |
| `grep` (optional) | Extract just the flag line | ⭐ Easy |
| `wget` (optional) | Download the script in the webshell | ⭐ Easy |

---

## Key Takeaways

- **`python3 script.py`** is the basic command to run any Python script on Linux
- **Always read scripts before running them** — opening a `.py` file with `cat` shows exactly what it does, which in CTFs often reveals the flag directly
- Python scripts are plain text — unlike compiled programs, you can always inspect the source code
- The flag name says it all: `run_s4n1ty_run` → "run sanity run" — this is literally a sanity check challenge to make sure you know how to execute a Python script
