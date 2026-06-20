# useless — picoCTF Writeup

**Challenge:** useless  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{us3l3ss_ch4ll3ng3_3xpl0it3d_8504}`  
**Platform:** picoCTF  
**Writeup by:** zham  

---

## Description

> There's an interesting script in the user's home directory
> The work computer is running SSH. We've been given a script which performs some basic calculations, explore the script and find a flag.
> Hostname: `saturn.picoctf.net` Port: `55433` Username: `picoplayer` Password: `password`

**Tag:** `man`

---

## Background Knowledge (Read This First!)

### What is a man page?

`man` (manual) is the built-in documentation system on Linux. Every standard command has one, accessed with:

```bash
man <command>
```

Custom scripts can also have their own man pages if a `.1` (or similar) manual file is installed for them. These man pages can contain anything — including an Authors section, examples, notes, or in this case, the flag.

### What is bash arithmetic command substitution?

Inside `$(( ))`, bash evaluates a mathematical expression. Normally, a plain `$(command)` inside arithmetic context fails with a syntax error because bash treats the substitution as a literal value, not something to re-evaluate.

However, **array subscripts get their own separate expansion pass**:

```bash
$(( x[$(touch /tmp/pwned)] ))
```

The command inside `x[...]` executes as a side effect during array index evaluation — this is a known bash arithmetic injection technique. It doesn't always lead anywhere useful (commands run with the same privileges as the user invoking the script), but it's an instructive vulnerability worth knowing about.

### The `man` tag on the challenge page

picoCTF tags challenges with the tool or technique most relevant to solving it. This challenge was tagged `man` — a direct hint that the solution involves the manual page system, not necessarily code execution.

---

## Step 1 — Connect via SSH

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 55433 picoplayer@saturn.picoctf.net
picoplayer@saturn.picoctf.net's password: password
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 6.8.0-1053-aws x86_64)
picoplayer@challenge:~$
```

---

## Step 2 — Find and read the script

```
picoplayer@challenge:~$ ls
useless

picoplayer@challenge:~$ file useless
useless: Bourne-Again shell script, ASCII text executable

picoplayer@challenge:~$ cat useless
#!/bin/bash
# Basic mathematical operations via command-line arguments
if [ $# != 3 ]
then
  echo "Read the code first"
else
        if [[ "$1" == "add" ]]
        then
          sum=$(( $2 + $3 ))
          echo "The Sum is: $sum"
        elif [[ "$1" == "sub" ]]
        then
          sub=$(( $2 - $3 ))
          echo "The Substract is: $sub"
        elif [[ "$1" == "div" ]]
        then
          div=$(( $2 / $3 ))
          echo "The quotient is: $div"
        elif [[ "$1" == "mul" ]]
        then
          mul=$(( $2 * $3 ))
          echo "The product is: $mul"
        else
          echo "Read the manual"
        fi
fi
```

A simple calculator script. Notice the `else` branch says **"Read the manual"** — that's the actual hint, pointing directly at `man`, not at the script's logic.

---

## Failed Rabbit Hole — Arithmetic Injection

Before noticing the "Read the manual" hint, the natural instinct is to look for command injection in `$(( $2 + $3 ))` since user input flows directly into bash arithmetic.

```
picoplayer@challenge:~$ ./useless add 1 '$(whoami)'
./useless: line 10: 1 + $(whoami) : syntax error: operand expected
```

Plain command substitution fails inside arithmetic context. But array subscript expansion bypasses this:

```
picoplayer@challenge:~$ ./useless add 1 'x[$(touch /tmp/pwned)]'
The Sum is: 1

picoplayer@challenge:~$ ls -la /tmp/pwned
-rw-rw-r-- 1 picoplayer picoplayer 0 Jun 20 12:37 /tmp/pwned
```

Confirmed code execution. However, the script is **not SUID** (`-rwxr-xr-x root root`, no `s` bit), so any command run this way executes with `picoplayer`'s own privileges — not root. Extensive searching through `/root`, `/home`, `/etc/passwd`, SUID binaries, capabilities, cron jobs, and a filesystem-wide grep for `picoCTF{` all turned up nothing. This path was a dead end — the vulnerability is real, but it isn't the intended solution.

---

## Step 3 — Read the manual (the actual solution)

```
picoplayer@challenge:~$ man useless

useless
     useless, — This is a simple calculator script

SYNOPSIS
     useless, [add sub mul div] number1 number2

DESCRIPTION
     Use the useless, macro to make simple calulations like addition,
     subtraction, multiplication and division.

Examples
     ./useless add 1 2
       This will add 1 and 2 and return 3
     ...

Authors
     This script was designed and developed by Cylab Africa

     picoCTF{us3l3ss_ch4ll3ng3_3xpl0it3d_8504}
```

The flag was sitting in the man page's Authors section the entire time.

---

## What Happened Internally

```
Timeline:
1. Connect via SSH, find "useless" script in home directory
2. Read script — notice "Read the manual" in the else branch
3. (Rabbit hole) Test bash arithmetic injection via array subscripts — works, but
   script isn't SUID so it grants no privilege escalation
4. Run `man useless` — the custom man page reveals the flag in its Authors section
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `ssh` | Connect to the challenge box | Easy |
| `cat` / `file` | Read and identify the script | Easy |
| Bash arithmetic injection | Tested but ultimately a dead end (no SUID) | Medium |
| `man` | Read the script's custom manual page — the actual solution | Easy |

---

## Key Takeaways

- **Read every hint literally** — the script's own output, "Read the manual," was a direct pointer to the solution; the `man` tag on the challenge page confirmed it
- **Not every vulnerability is the intended path** — bash arithmetic injection via array subscripts is real and worth knowing, but without SUID permissions it grants no extra privilege and was a dead end here
- **Custom man pages can hide information** — `man <scriptname>` is easy to overlook when a script seems self-contained, but custom manual pages are simple to create and just as simple to check
- **Don't overthink medium-difficulty challenges** — sometimes the "exploit" angle is a deliberate distraction, and the real solution is far simpler than it looks
- The flag `us3l3ss_ch4ll3ng3_3xpl0it3d` is "useless challenge exploited" — slightly ironic, since the actual solve didn't require exploiting anything
