# SansAlpha — picoCTF Writeup

**Challenge:** SansAlpha  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{7h15_mu171v3r53_15_m4dn355_640b6add}`  
**Platform:** picoCTF 2025  
**Writeup by:** zham  

---

## Description

> The Multiverse is within your grasp! Unfortunately, the server that contains the secrets of the multiverse is in a universe where keyboards only have numbers and (most) symbols.
> `ssh -p 56730 ctf-player@mimas.picoctf.net`
> Use password: `f3b61b38`

**Hint 1:** `Where can you get some letters?`

---

## Background Knowledge (Read This First!)

### What is SansAlpha?

SansAlpha is a restricted shell that blocks any input containing alphabetic characters (`a-z`, `A-Z`). Every command you type is checked, and if it contains any letter, it is rejected with `SansAlpha: Unknown character detected`.

This means:
- `ls` → blocked
- `cat flag.txt` → blocked
- Even `echo *` → blocked

### How do we get letters without typing them?

Bash has special variables that **already contain letters** — and we can reference positions inside them using the `${VAR:offset:length}` syntax:

```bash
${VAR:5:1}   # extract 1 character starting at position 5 of $VAR
```

Since we are only typing `$`, `{`, `_`, `:`, numbers, and `}` — no letters — this passes the filter. The letters come from the variable's value, not our input.

**Key variables:**

| Variable | Value | Why useful |
|---|---|---|
| `$-` | `himBHs` | Contains `h`, `i`, `m`, `s` |
| `$_` | last argument used | Set by running `~` → becomes `/home/ctf-player` |
| `~` | `/home/ctf-player` | Tilde expands without letters, sets `$_` |

### Letter map of `/home/ctf-player`

Once `$_` = `/home/ctf-player`, we can extract any letter we need:

```
/ h o m e / c  t  f  -  p  l  a  y  e  r
0 1 2 3 4 5 6  7  8  9  10 11 12 13 14 15
```

- `${_:6:1}` = `c`
- `${_:7:1}` = `t`
- `${_:11:1}` = `l`
- `${_:12:1}` = `a`
- `${_:13:1}` = `y`
- `${-:5:1}` = `s` (from `$-` = `himBHs`)

So:
- `ls` = `${_:11:1}${-:5:1}`
- `cat` = `${_:6:1}${_:12:1}${_:7:1}`

---

## Step 1 — Connect via SSH

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 56730 ctf-player@mimas.picoctf.net
ctf-player@mimas.picoctf.net's password: f3b61b38
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 6.8.0-1053-aws x86_64)
SansAlpha$
```

We are dropped into the SansAlpha restricted shell. Any input containing letters is blocked.

---

## Step 2 — Discover letters in `$-`

```
SansAlpha$ ${-}
bash: himBHs: command not found
```

`$-` contains the current shell option flags. Its value is `himBHs` — the "command not found" error reveals the full string. This gives us:
- pos 5 = `s`
- pos 0 = `h`
- pos 1 = `i`

---

## Step 3 — Set `$_` to `/home/ctf-player` using `~`

```
SansAlpha$ ~
bash: /home/ctf-player: Is a directory
```

`~` expands to the home directory path without any letters in our input. After this fails as a command, bash sets `$_` (last argument) to `/home/ctf-player`. Now we have a full alphabet map inside `$_`.

---

## Step 4 — Run `ls` using extracted letters

Using `${_:11:1}` = `l` and `${-:5:1}` = `s`:

```
SansAlpha$ ${_:11:1}${-:5:1}
blargh    on-calastran.txt
```

We can see two items — a directory called `blargh` and a file `on-calastran.txt`.

---

## Step 5 — Read all files with `cat */*`

`$_` changes after each command, so we need to reset it with `~` before building `cat`. We chain both in one line with `;`:

Using `${_:6:1}` = `c`, `${_:12:1}` = `a`, `${_:7:1}` = `t`:

```
SansAlpha$ ~ ; ${_:6:1}${_:12:1}${_:7:1} */*
bash: /home/ctf-player: Is a directory
cat: blargh: Is a directory
return 0 picoCTF{7h15_mu171v3r53_15_m4dn355_640b6add}
...
```

The `*/*` glob expands to all files inside all subdirectories — no letters needed. The flag was inside `blargh/`.

---

## What Happened Internally

```
Timeline:
1. Connect via SSH — SansAlpha shell blocks all letter input
2. ${-} → reveals $- = "himBHs" → we get 's' at position 5
3. ~ → sets $_ = "/home/ctf-player" → maps c(6) t(7) l(11) a(12) y(13)
4. ${_:11:1}${-:5:1} → "ls" → reveals blargh/ and on-calastran.txt
5. ~ resets $_ to /home/ctf-player again
6. ${_:6:1}${_:12:1}${_:7:1} */* → "cat */*" → reads flag inside blargh/
```

---

## Full Command Summary

```bash
${-}                                          # reveals $- = himBHs
~                                             # sets $_ = /home/ctf-player
${_:11:1}${-:5:1}                             # ls
~ ; ${_:6:1}${_:12:1}${_:7:1} */*            # cat */*
```

Four commands. No letters typed.

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `ssh` | Connect to the restricted shell | Easy |
| `$-` | Source of letters `h`, `i`, `m`, `s` without typing them | Medium |
| `~` | Set `$_` to `/home/ctf-player` to extract `c`, `t`, `l`, `a` | Medium |
| `${VAR:N:1}` | Extract single characters from variable values | Medium |
| `*/*` glob | Read all files in subdirectories without typing any letters | Easy |

---

## Key Takeaways

- **Bash special variables contain letters** — `$-` (shell flags) and `$_` (last argument) hold strings we can slice into individual characters using `${VAR:offset:length}`
- **`~` is your best friend in letter-restricted shells** — it expands to the full home directory path, giving you a rich set of letters to extract without typing a single one
- **`$_` changes after every command** — always reset it with `~` before building a command that depends on it, and chain with `;` to use it immediately before it changes again
- **Globs replace filenames** — `*` and `*/*` let you reference files and directories without typing their names, essential when filenames contain letters
- The flag `7h15_mu171v3r53_15_m4dn355` is "this multiverse is madness" — fitting for a challenge where you cannot type letters
