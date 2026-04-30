# Magikarp Ground Mission — picoCTF Writeup

**Challenge:** Magikarp Ground Mission  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{xxsh_0ut_0f_//4t3r_0b24fc4f}`  

---

## Description

> Do you know how to move between directories and read files in the shell? Start the container, ssh to it, and then `ls` once connected to begin.
> Login via `ssh` as `ctf-player` with the password `8c606eb1` on the host `wily-courier.picoctf.net` and port `51432`.

**Hint 1:** `Finding a cheatsheet for bash would be really helpful!`

---

## Background Knowledge (Read This First!)

### What is `cd`?

`cd` (change directory) is the Linux command for navigating between folders. Think of it like double-clicking a folder in a file manager, but in the terminal.

```bash
cd /some/folder    # go to an absolute path
cd ~               # go to your home directory
cd /               # go to the root directory
```

### What is the root directory `/`?

On Linux, `/` is the **root** of the entire filesystem — the top-level directory that contains everything else. Every file and folder on the system lives somewhere under `/`. It's like the `C:\` drive on Windows.

### What is the home directory `~`?

`~` is a shortcut for the current user's **home directory**. For `ctf-player`, that's `/home/ctf-player`. It's where you land when you first SSH in.

### What is the flag split into 3 parts?

The challenge hides the flag across **3 different locations** on the filesystem — home directory, root `/`, and back home again. Each location also has an instructions file telling you where to go next. You must follow the trail to collect all 3 pieces and assemble the complete flag.

### ⚠️ Important Note

The password `8c606eb1` is invisible as you type it. You may need to type it carefully multiple times — just keep trying if it says "Permission denied."

---

## Solution — Step by Step

### Step 1 — SSH into the server

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 51432 ctf-player@wily-courier.picoctf.net
```

Type `yes` to accept the fingerprint, then enter password `8c606eb1`:

```
ctf-player@pico-chall$
```

### Step 2 — Look around and read Part 1

```
ctf-player@pico-chall$ ls
1of3.flag.txt  instructions-to-2of3.txt

ctf-player@pico-chall$ cat 1of3.flag.txt
picoCTF{xxsh_

ctf-player@pico-chall$ cat instructions-to-2of3.txt
Next, go to the root of all things, more succinctly `/`
```

**Part 1:** `picoCTF{xxsh_`

### Step 3 — Go to root `/` and read Part 2

```
ctf-player@pico-chall$ cd /

ctf-player@pico-chall$ ls
2of3.flag.txt  bin  boot  challenge  dev  etc  home  instructions-to-3of3.txt ...

ctf-player@pico-chall$ cat 2of3.flag.txt
0ut_0f_//4t3r_

ctf-player@pico-chall$ cat instructions-to-3of3.txt
Lastly, ctf-player, go home... more succinctly `~`
```

**Part 2:** `0ut_0f_//4t3r_`

### Step 4 — Go home `~` and read Part 3

```
ctf-player@pico-chall$ cd ~

ctf-player@pico-chall$ cat 3of3.flag.txt
0b24fc4f}
```

**Part 3:** `0b24fc4f}`

### Step 5 — Assemble the flag

Combine all 3 parts:

```
picoCTF{xxsh_  +  0ut_0f_//4t3r_  +  0b24fc4f}
= picoCTF{xxsh_0ut_0f_//4t3r_0b24fc4f}
```

✅ Got the flag! 🎯

---

## The Flag Trail

| Location | Command | File | Content |
|----------|---------|------|---------|
| Home (`~`) | `cd ~` | `1of3.flag.txt` | `picoCTF{xxsh_` |
| Root (`/`) | `cd /` | `2of3.flag.txt` | `0ut_0f_//4t3r_` |
| Home (`~`) | `cd ~` | `3of3.flag.txt` | `0b24fc4f}` |

---

## Alternative — One-liner to read all 3 parts

Once you know the locations, you can collect the entire flag in one command:

```bash
cat ~/1of3.flag.txt && cat /2of3.flag.txt && cat ~/3of3.flag.txt
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `ssh` | Connect to the remote challenge server | ⭐ Easy |
| `ls` | List files in the current directory | ⭐ Easy |
| `cat` | Read file contents | ⭐ Easy |
| `cd /` | Navigate to the root directory | ⭐ Easy |
| `cd ~` | Navigate to the home directory | ⭐ Easy |

---

## Key Takeaways

- **`cd /`** goes to the root of the filesystem — the top of everything
- **`cd ~`** goes to your home directory — a shortcut every Linux user should know
- **Always `ls` after `cd`** — it shows what files are available in the new location
- **Flags can be split across multiple locations** — follow the instruction files as a trail
- The flag `xxsh_0ut_0f_//4t3r` → "xxsh out of water" — a pun on "fish out of water" and the `ssh` command. Magikarp is a fish Pokémon known for flopping on land — perfectly themed!
