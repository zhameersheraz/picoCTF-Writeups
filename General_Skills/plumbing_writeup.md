# Plumbing — picoCTF Writeup

**Challenge:** plumbing  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{digital_plumb3r_00da27CC}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham

---

## Description

> Sometimes you need to handle process data outside of a file. Can you find a way to keep the output from this program and search for the flag?
> Connect to `fickle-tempest.picoctf.net 60256`.

## Hints

> Remember the flag format is picoCTF{XXXX}

> What's a pipe? No not that kind of pipe... This kind

---

## Background Knowledge (Read This First!)

**What's a "pipe" in the terminal?** In Linux, the `|` character connects the output of one program directly to the input of another, without ever touching a file on disk. The data "flows through the pipe" from the first command into the second, live, as it's produced.

**Why does that matter here?** The challenge description says it wants me to "handle process data outside of a file." That's a direct nudge toward piping. The naive approach would be: run the program, watch a wall of text scroll by, manually scan for the flag. The pipe approach is: run the program and immediately hand its output to a filter (like `grep`) that does the scanning for me, in real time, without me ever saving a `.txt` file first.

**What `grep` does here.** `grep picoCTF` reads whatever text is fed to it line by line and only prints the lines containing the string `picoCTF`. Since I already know the flag format is `picoCTF{...}`, I can let `grep` do the searching instead of reading through everything myself.

---

## Solution

### Step 1 — Connect and pipe straight into grep

Instead of running `nc` on its own and scrolling through the output by eye, I chain it directly into `grep`:

```
┌──(zham㉿kali)-[~]
└─$ nc fickle-tempest.picoctf.net 60256 | grep picoCTF
picoCTF{digital_plumb3r_00da27CC}
```

The `nc` command connects to the server and the program on the other end starts dumping its output. That output is piped straight into `grep picoCTF`, which throws away every line except the one containing the flag.

Flag obtained — no file ever touched the disk.

### Alternative method — capture to a file first, then search

If `grep` had come back empty (for example, if the connection closed before the flag line printed, or `grep` needed to see the whole output before matching), the fallback is to actually save the output and search the saved copy:

```
┌──(zham㉿kali)-[~]
└─$ nc fickle-tempest.picoctf.net 60256 | tee output.txt
```

`tee` does something useful here: it prints the output to my screen *and* writes a copy to `output.txt` at the same time, so I get to watch it live while also keeping a permanent copy.

Once the connection finishes, search the saved file:

```
┌──(zham㉿kali)-[~]
└─$ grep picoCTF output.txt
picoCTF{digital_plumb3r_00da27CC}
```

Same result, just with a file as a safety net along the way.

---

## What Happened Internally

1. I connect to the challenge server with `nc`, which opens a TCP connection and starts streaming whatever the remote program prints to its standard output.
2. Normally that output would just scroll across my terminal. The shell's `|` operator instead redirects it: instead of going to my screen, the program's stdout becomes `grep`'s stdin.
3. `grep picoCTF` reads that stream line by line, checking each line against the pattern `picoCTF`.
4. The moment a line matching that pattern arrives — the flag line — `grep` prints it and lets everything else through unmatched (and therefore silently discarded).
5. No intermediate file is created; the data moves directly from one process's output to another process's input, which is exactly the "process data outside of a file" the challenge description was pointing at.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nc` | Connect to the remote challenge server and receive its output |
| `grep` | Filter the live output stream for the flag pattern |
| `tee` | (Alternative) capture output to a file while still viewing it live |

---

## Key Takeaways

- A pipe (`|`) lets you chain commands so one program's output becomes another's input, live, with nothing written to disk.
- When a challenge tells you the output format in advance (`picoCTF{XXXX}`), that's a direct invitation to filter for it with `grep` instead of reading manually.
- `tee` is the right tool when you want to both watch output live and keep a saved copy — useful when you're not sure a single pipe will catch everything on the first try.
- Flag wordplay: `digital_plumb3r` is literally the skill being tested — using Unix "pipes" to plumb data from one process straight into another, the same way a real plumber routes water through pipes instead of carrying it by hand.
