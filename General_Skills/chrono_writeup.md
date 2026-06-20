# chrono — picoCTF Writeup

**Challenge:** chrono  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{Sch3DUL7NG_T45K3_L1NUX_5b7059d0}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> How to automate tasks to run at intervals on linux servers?
> Use ssh to connect to this server:
> Server: `saturn.picoctf.net` Port: `50129` Username: `picoplayer` Password: `kZx-HVJKu8`

No hint was provided for this challenge — the title itself is the only clue.

---

## Background Knowledge (Read This First!)

### What is cron?

**Cron** is the standard Linux daemon for running tasks automatically on a schedule — daily backups, log rotation, periodic cleanup scripts, anything that needs to happen "every X minutes/hours/days" without a human triggering it. The word "chrono" (Greek for "time") in the challenge title is the giveaway pointing straight at this.

### Where cron jobs actually live

There are two separate places cron jobs get defined, and it's easy to check the wrong one first:

| Location | Scope | How to view it |
|---|---|---|
| `/var/spool/cron/crontabs/<user>` | Jobs for one specific user | `crontab -l` (only shows *your own* jobs) |
| `/etc/crontab` | System-wide jobs, run as whichever user is specified in the line | Readable directly with `cat` |

If you try `crontab -l` as a low-privilege user with no personal cron jobs set up, you'll just get `no crontab for picoplayer` — a dead end. The real target here is the **system-wide** file, `/etc/crontab`, which is a plain text file and, on most systems, readable by any logged-in user by default — no special permissions needed.

---

## Solution — Step by Step

### Step 1 — Connect via SSH

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 50129 picoplayer@saturn.picoctf.net
picoplayer@saturn.picoctf.net's password: kZx-HVJKu8
picoplayer@challenge:~$
```

### Step 2 — Read the system-wide crontab directly

```
picoplayer@challenge:~$ cat /etc/crontab
# picoCTF{Sch3DUL7NG_T45K3_L1NUX_5b7059d0}
```

The flag was placed as a comment line inside the system crontab — no scheduling logic to actually parse, no privilege escalation needed, just knowing which file to look at.

---

## Alternative Method — Browse `/etc/cron.d/`

If `/etc/crontab` had come up empty, the next place to check is the `/etc/cron.d/` directory, which holds additional system-wide cron job definitions split across multiple files (commonly used by installed packages to register their own scheduled tasks):

```
picoplayer@challenge:~$ ls /etc/cron.d/
picoplayer@challenge:~$ cat /etc/cron.d/*
```

Worth knowing as a fallback for real-world cron investigation, even though it wasn't needed here.

---

## What Happened Internally

```
Timeline:
1. Connect via SSH as picoplayer, a regular unprivileged user
2. Title "chrono" + description about "automating tasks at intervals" points to cron
3. cron jobs split across two locations: per-user (crontab -l) and system-wide (/etc/crontab)
4. cat /etc/crontab — world-readable by default, no permission errors
5. Flag found sitting as a plain comment line inside the file
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `ssh` | Connect to the challenge server | Easy |
| `cat` | Read the world-readable system crontab directly | Easy |
| `crontab -l` (background knowledge) | Would show per-user jobs only — not where the flag lives | Easy |

---

## Key Takeaways

- **"Automate tasks at intervals on Linux" is always pointing at cron** — recognizing that phrase immediately narrows down where to look
- **There isn't one single crontab file** — per-user jobs and system-wide jobs live in completely different locations, and checking only one can send you down a dead end
- **`/etc/crontab` is plain text and often world-readable** — it's worth a direct `cat` early in recon on any box, since system configuration files frequently leak more than intended
- **Not every challenge needs an exploit** — sometimes the entire "vulnerability" is just knowing which standard Linux file to check
- The flag `Sch3DUL7NG_T45K3_L1NUX` decodes to "scheduling tasks linux" — a literal, on-the-nose description of exactly what cron does and exactly what this challenge tested
