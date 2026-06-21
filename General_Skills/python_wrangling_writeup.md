# Python Wrangling — picoCTF Writeup

**Challenge:** Python Wrangling  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 10  
**Flag:** `picoCTF{4p0110_1n_7h3_h0us3_9c5f9bcf}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Python scripts are invoked kind of like programs in the Terminal...
> Can you run `ende.py` using `password.txt` to get `flag.txt.en`?

**Hint 1:** `Get the Python script accessible in your shell by entering the following command in the Terminal prompt: $ wget followed by a link to the script. The link can be copied from the details section.`

**Hint 2:** `$ man python`

---

## Background Knowledge (Read This First!)

### What does "invoked like a program" actually mean?

Most Python scripts you write yourself you'd run with `python3 script.py` and nothing else. But scripts can also accept **command-line arguments** the same way a real terminal program does — extra words typed after the script name, which Python collects into a list called `sys.argv`. `sys.argv[0]` is always the script's own filename; everything after that is whatever the user typed next.

### Reading `ende.py`'s argument logic

```python
if sys.argv[1] == "-e":      # encrypt mode
    ...
elif sys.argv[1] == "-d":    # decrypt mode
    if len(sys.argv) < 4:
        sim_sala_bim = input("Please enter the password:")
    else:
        sim_sala_bim = sys.argv[3]
```

`sys.argv[1]` needs to be `-e` or `-d` to pick a mode, `sys.argv[2]` is the file to act on, and `sys.argv[3]` is an *optional* password — if you don't supply it as a fourth word on the command line, the script just asks for it interactively instead.

### What is Fernet, and why base64-encode the password?

**Fernet** (from Python's `cryptography` library) is a symmetric encryption scheme — the same key both encrypts and decrypts. Fernet specifically requires its key to be a base64-encoded representation of exactly 32 raw bytes. Our password file holds a 32-character string, so the script does exactly one conversion to make it usable:

```python
ssb_b64 = base64.b64encode(sim_sala_bim.encode())
c = Fernet(ssb_b64)
```

`sim_sala_bim.encode()` turns the 32-character password into 32 raw bytes, and `base64.b64encode()` re-encodes those 32 bytes into the base64 format Fernet expects. That's the entire trick — the password file already contains a value of exactly the right length for this to work.

---

## Solution — Step by Step

### Step 1 — Get all three files in one folder

Place `ende.py`, `password.txt`, and `flag.txt.en` together in the same directory.

### Step 2 — Check the usage message

```
┌──(zham㉿kali)-[~/pythonwrangling]
└─$ python3 ende.py
Usage: ende.py (-e/-d) [file]
```

Confirms the script needs a mode flag and a filename at minimum.

### Step 3 — Check what's in the password file

```
┌──(zham㉿kali)-[~/pythonwrangling]
└─$ cat password.txt
720b6ad346f84cd483c60c7464dd95d4
```

### Step 4 — Decrypt, feeding the password file in as input

```
┌──(zham㉿kali)-[~/pythonwrangling]
└─$ python3 ende.py -d flag.txt.en < password.txt
Please enter the password:picoCTF{4p0110_1n_7h3_h0us3_9c5f9bcf}
```

Redirecting `password.txt` into the script's stdin answers the `input()` prompt automatically, with no copy-pasting needed.

---

## Alternative Method — Pass the password directly as the fourth argument

Since `ende.py` already supports taking the password as `sys.argv[3]`, you can skip the interactive prompt and redirection entirely:

```
┌──(zham㉿kali)-[~/pythonwrangling]
└─$ python3 ende.py -d flag.txt.en 720b6ad346f84cd483c60c7464dd95d4
picoCTF{4p0110_1n_7h3_h0us3_9c5f9bcf}
```

One line, no extra prompt to answer — this is the script's own built-in shortcut, just read directly from its argument-handling code.

---

## What Happened Internally

```
Timeline:
1. ende.py expects sys.argv[1] = mode (-e/-d), sys.argv[2] = filename,
   optionally sys.argv[3] = password
2. python3 ende.py with no arguments — prints usage message, confirms the
   script needs at least a mode and a file
3. cat password.txt — reveals the 32-character password string
4. -d flag.txt.en < password.txt — redirects the password file as stdin,
   automatically answering the input() prompt
5. base64.b64encode(password) converts the 32-byte password into a valid
   Fernet key
6. Fernet.decrypt() reverses the encryption on flag.txt.en's contents
7. Flag printed directly to stdout
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `python3` | Run the provided script | Easy |
| `cat` | Read the password file's contents | Easy |
| `< file` (stdin redirection) | Feed a file's contents into a script's `input()` prompt automatically | Easy |
| `cryptography.fernet.Fernet` (read, not implemented) | The actual symmetric decryption used by the script | Easy |

---

## Key Takeaways

- **Command-line scripts often hide their full feature set behind argument count, not just flags** — `ende.py` accepts the password as either an interactive prompt or a direct argument; reading the source is the only way to know both options exist
- **Stdin redirection (`< file`) can answer any `input()` prompt automatically** — useful any time a script asks a question but you already know the answer is sitting in a file
- **Symmetric encryption schemes like Fernet often have strict key-format requirements** — base64-encoding a password isn't obfuscation here, it's a required formatting step to satisfy the library's exact 32-byte key requirement
- **The simplest tasks in a CTF are often just "read the usage message, then read the source"** — no exploitation needed, just understanding what the tool in front of you was built to do
- The flag `4p0110_1n_7h3_h0us3` reads as "Apollo in the house" — a lighthearted callback to nothing in particular within the challenge itself, just a fun, energetic way to close out one of the very first scripting challenges in picoCTF's General Skills track
