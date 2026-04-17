# Log Hunt — picoCTF Writeup

**Challenge:** Log Hunt  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{us3_y0urlinux_sk1lls_cedfa5fb}`

---

## Description

> This challenge provides a server log file that contains scattered pieces of a flag. The flag fragments are repeated throughout the log and need to be extracted and reconstructed.

---

## Background Knowledge (Read This First!)

### What is grep?

`grep` is a Linux command that **searches for a pattern** inside a file or stream. Combined with `cat` and pipes (`|`), it is one of the most powerful text filtering tools in forensics.

---

## Solution — Step by Step

### Step 1 — Download the Log File

I downloaded `server.log` from the challenge link provided.

### Step 2 — View the Log Structure

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat server.log
```

The log contained many entries with timestamps, log levels, and various messages.

### Step 3 — Filter INFO Messages

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat server.log | grep INFO
```

I noticed some entries contained "FLAGPART" in them.

### Step 4 — Search for Flag Fragments

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ cat server.log | grep FLAGPART
[1990-08-09 10:00:10] INFO FLAGPART: picoCTF{us3_
[1990-08-09 10:02:55] INFO FLAGPART: y0urlinux_
[1990-08-09 10:05:54] INFO FLAGPART: sk1lls_
[1990-08-09 10:10:54] INFO FLAGPART: cedfa5fb}
(repeated entries...)
```

### Step 5 — Reconstruct the Flag

I identified four distinct fragments in chronological order and combined them:

```
picoCTF{us3_ + y0urlinux_ + sk1lls_ + cedfa5fb}
= picoCTF{us3_y0urlinux_sk1lls_cedfa5fb}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Display file contents |
| `grep` | Search for patterns in text |
| `\|` (pipe) | Chain commands together |

---

## Key Takeaways

- Progressive filtering with `grep` helps narrow down large log files efficiently
- Log analysis often requires reconstructing information from scattered fragments
- The fragments appeared in chronological order based on timestamps — making reconstruction straightforward
- Understanding command-line text processing tools is crucial for forensics challenges
