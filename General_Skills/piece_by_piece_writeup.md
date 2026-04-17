# Piece by Piece — picoCTF Writeup

**Challenge:** Piece by Piece  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{z1p_and_spl1t_f1l3s_4r3_fun_81362b37}`  

---

## Description

> After logging in, you will find multiple file parts in your home directory. These parts need to be combined and extracted to reveal the flag.
>
> SSH to `dolphin-cove.picoctf.net:60410` and login as `ctf-player` with password `f3b61b38`.

**Tags:** `ssh`, `cat`, `zip`, `file-parts`, `browser_webshell_solvable`

---

## Background Knowledge (Read This First!)

### What is SSH?

**SSH (Secure Shell)** lets you remotely log into another computer over the internet and use its terminal as if you were sitting right in front of it. The challenge runs on a remote server, and you connect to it using SSH.

The format `dolphin-cove.picoctf.net:60410` means:
- **hostname:** `dolphin-cove.picoctf.net`
- **port:** `60410`

In the SSH command, you specify the port with the `-p` flag:

```bash
ssh ctf-player@dolphin-cove.picoctf.net -p 60410
```

> **Note:** When SSH prompts for a password, nothing appears on screen as you type — not even dots. This is normal. Just type `f3b61b38` and press Enter.

### What are split file parts?

Large files can be **split into smaller chunks** using Linux's `split` command. The chunks are usually named sequentially like:

```
part_aa  part_ab  part_ac  part_ad  part_ae
```

The `aa`, `ab`, `ac`... suffix is alphabetical ordering — just like how files are split. To reconstruct the original file, you **combine all the parts back in order** using `cat` and redirect the output to a new file:

```bash
cat part_a* > combined_flag
```

The `*` is a wildcard that matches everything starting with `part_a` in alphabetical order, which is exactly the order the parts need to be joined.

### What is a ZIP file?

A **ZIP file** is a compressed archive that can contain one or more files inside it. After combining the parts, the result is a ZIP file. You can extract it using the `unzip` command.

If the ZIP is **password-protected**, you pass the password with the `-P` flag:

```bash
unzip -P <password> <zipfile>
```

### What is the `file` command?

The `file` command tells you what **type** a file is based on its contents — not its name. This is useful when a file has no extension or an unknown extension:

```bash
file combined_flag
```

Output example:
```
combined_flag: Zip archive data, at least v1.0 to extract
```

This tells you the combined file is a ZIP, so you know to use `unzip` next.

---

## Solution — Step by Step

### Step 1 — Connect to the challenge server

Open your terminal and run:

```
┌──(zham㉿kali)-[~]
└─$ ssh ctf-player@dolphin-cove.picoctf.net -p 60410
```

Breaking this command down:
- `ssh` — the remote login tool
- `ctf-player@dolphin-cove.picoctf.net` — username `ctf-player` at the server address
- `-p 60410` — connect on port 60410 (not the default SSH port)

You'll see a fingerprint warning — type `yes` and press Enter:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

Then type the password `f3b61b38` and press Enter (nothing will appear as you type):

```
ctf-player@dolphin-cove.picoctf.net's password:
Welcome to Ubuntu 20.04.3 LTS...
ctf-player@pico-chall$
```

You are now inside the remote server.

### Step 2 — Look around with `ls`

```
ctf-player@pico-chall$ ls
instructions.txt  part_aa  part_ab  part_ac  part_ad  part_ae
```

There are several things here:

- **`instructions.txt`** — a hint file explaining what to do
- **`part_aa` through `part_ae`** — five file fragments that need to be combined

Let's read the instructions:

```
ctf-player@pico-chall$ cat instructions.txt
Hint:
- The flag is split into multiple parts as a zipped file.
- Use Linux commands to combine the parts into one file.
- The zip file is password protected. Use this "supersecret" password to extract the zip file.
- After unzipping, check the extracted text file for the flag.
```

The instructions tell us everything we need:
1. Combine the parts into one file
2. Unzip it using the password `supersecret`
3. Read the extracted text file

### Step 3 — Combine the file parts

Use `cat` with a wildcard to join all parts in order:

```
ctf-player@pico-chall$ cat part_a* > combined_flag
```

What happens here:

1. `cat part_a*` — reads `part_aa`, `part_ab`, `part_ac`, `part_ad`, `part_ae` in alphabetical order and streams them all to standard output
2. `> combined_flag` — redirects that output into a new file called `combined_flag`

Now verify what type of file we got:

```
ctf-player@pico-chall$ file combined_flag
combined_flag: Zip archive data, at least v1.0 to extract
```

It's a ZIP file.

### Step 4 — Unzip with the password

```
ctf-player@pico-chall$ unzip -P supersecret combined_flag
Archive:  combined_flag
 extracting: flag.txt
```

The `-P supersecret` provides the password directly in the command. The archive extracts a file called `flag.txt`.

### Step 5 — Read the flag

```
ctf-player@pico-chall$ cat flag.txt
picoCTF{z1p_and_spl1t_f1l3s_4r3_fun_81362b37}
```

✅ Got the flag! 🎯

---

## Full Terminal Session (Clean View)

```bash
$ ssh ctf-player@dolphin-cove.picoctf.net -p 60410
# (type yes, then password f3b61b38)

ctf-player@pico-chall$ ls
instructions.txt  part_aa  part_ab  part_ac  part_ad  part_ae

ctf-player@pico-chall$ cat instructions.txt
# (reveals the hint and password "supersecret")

ctf-player@pico-chall$ cat part_a* > combined_flag

ctf-player@pico-chall$ file combined_flag
combined_flag: Zip archive data, at least v1.0 to extract

ctf-player@pico-chall$ unzip -P supersecret combined_flag
Archive:  combined_flag
 extracting: flag.txt

ctf-player@pico-chall$ cat flag.txt
picoCTF{z1p_and_spl1t_f1l3s_4r3_fun_81362b37}
```

---

## Common Mistakes to Avoid

### ❌ Using `nc` instead of `ssh`

```bash
# WRONG
nc dolphin-cove.picoctf.net 60410

# RIGHT
ssh ctf-player@dolphin-cove.picoctf.net -p 60410
```

The challenge explicitly says "SSH to..." — `nc` (netcat) is for raw TCP connections, not SSH sessions.

### ❌ Wrong wildcard pattern

```bash
# WRONG — no file named flag.txt.part* exists
cat flag.txt.part* > combined_flag

# RIGHT — match the actual filenames
cat part_a* > combined_flag
```

Always `ls` first to see the exact filenames before using wildcards.

### ❌ Forgetting the `-P` flag for passwords

```bash
# WRONG — will prompt for password interactively (fine, but less smooth)
unzip combined_flag

# RIGHT — pass password directly
unzip -P supersecret combined_flag
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `ssh` | Connect to the remote challenge server | ⭐ Easy |
| `ls` | List files to understand the environment | ⭐ Easy |
| `cat` | Read files and combine parts using wildcard | ⭐ Easy |
| `file` | Identify the type of the combined file | ⭐ Easy |
| `unzip -P` | Extract the password-protected ZIP archive | ⭐ Easy |

---

## Key Takeaways

- **Split files are rejoined with `cat part_* > output`** — the wildcard ensures alphabetical order, which matches the original split order
- **Always run `file` on unknown files** — file extensions can be misleading or absent; `file` reads the actual content to identify the type
- **Read the hints** — `instructions.txt` in this challenge literally told us the password. Don't skip files named `instructions`, `readme`, or `hint`
- **SSH uses `-p` for port** — unlike `nc` which takes the port as a positional argument, SSH uses `-p <port>` as a flag
