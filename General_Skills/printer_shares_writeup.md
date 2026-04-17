# Printer Shares — picoCTF Writeup

**Challenge:** Printer Shares  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{5mb_pr1nter_5h4re5_ac4c227e}`  

---

## Description

> Oops! Someone accidentally sent an important file to a network printer—can you retrieve it from the print server?  
> The printer is on `60867`.  
> you can try `$ nc -vz mysterious-sea.picoctf.net 60867`

**Hint 1:** `knowing how SMB protocol works would be helpful!`

**Tags:** `SMB`, `network`, `file-sharing`, `browser_webshell_solvable`

---

## Background Knowledge (Read This First!)

### What is SMB?

**SMB (Server Message Block)** is a network protocol used for **sharing files, printers, and other resources** between computers on a network. You've probably used it without knowing — it's what powers Windows file sharing (the `\\server\share` paths you see in corporate environments).

In this challenge, the print server is running **Samba**, which is the Linux implementation of SMB. Someone accidentally sent a file to a shared network printer, and that file is sitting on the server waiting to be retrieved.

### What is `smbclient`?

`smbclient` is a Linux command-line tool that lets you interact with SMB shares — think of it like an FTP client, but for SMB. You can use it to:

- **List shares** available on a server (`-L` flag)
- **Connect to a specific share** and browse its files
- **Download files** with the `get` command

### What is `nc` (netcat)?

`nc` is short for **netcat** — a raw networking utility often called the "Swiss Army knife of networking." In this challenge, it's used to verify that the server is reachable and the port is open before attempting a full SMB connection.

The `-v` flag means verbose (show details) and `-z` means just check if the port is open without sending data.

### ⚠️ Important Notes Before Starting

**Note 1 — `smb: \>` commands are typed INSIDE smbclient**  
When you connect with `smbclient`, you enter an interactive shell. Commands like `ls`, `get`, and `exit` are typed there — not in your regular Kali terminal. Many beginners make the mistake of typing them in the wrong place.

**Note 2 — The instance has a countdown timer**  
The challenge server shuts down after the timer expires. If the connection drops mid-solve, click "Restart Instance" on the picoCTF challenge page and reconnect.

---

## Solution — Step by Step

### Step 1 — Verify the server is reachable with `nc`

```
┌──(zham㉿kali)-[~]
└─$ nc -vz mysterious-sea.picoctf.net 60867
DNS fwd/rev mismatch: mysterious-sea.picoctf.net != ec2-3-130-79-223.us-east-2.compute.amazonaws.com
mysterious-sea.picoctf.net [3.130.79.223] 60867 (?) open
```

Breaking this down:
- `nc` — netcat, the raw networking utility
- `-vz` — verbose mode + just check port (don't send data)
- `mysterious-sea.picoctf.net 60867` — the server address and port

The important output is `open` — this confirms the server is alive and accepting connections on port 60867. The DNS mismatch warning is harmless; it's just an AWS hostname quirk.

### Step 2 — List available SMB shares

```
┌──(zham㉿kali)-[~]
└─$ smbclient -L //mysterious-sea.picoctf.net -p 60867 -N
        Sharename       Type      Comment
        ---------       ----      -------
        shares          Disk      Public Share With Guests
        IPC$            IPC       IPC Service (Samba 4.19.5-Ubuntu)
```

Breaking this down:
- `-L` — list all available shares on the server
- `-p 60867` — use port 60867 (not the default SMB port 445)
- `-N` — no password (anonymous/guest access)

We can see two shares:

| Share | Type | Meaning |
|-------|------|---------|
| `shares` | Disk | A public file share — **this is what we want** |
| `IPC$` | IPC | Internal inter-process communication, not useful here |

> **Why not `print$`?** That's the default printer driver share. In this challenge, the print server stores the file in a regular disk share called `shares` instead.

### Step 3 — Connect to the `shares` share

```
┌──(zham㉿kali)-[~]
└─$ smbclient //mysterious-sea.picoctf.net/shares -p 60867 -N
Try "help" to get a list of possible commands.
smb: \>
```

You are now inside the smbclient interactive shell. The `smb: \>` prompt means you're browsing the share.

### Step 4 — List files on the share

```
smb: \> ls
  .                                   D        0  Fri Mar  6 15:25:40 2026
  ..                                  D        0  Fri Mar  6 15:25:40 2026
  dummy.txt                           N     1142  Wed Feb  4 16:22:17 2026
  flag.txt                            N       37  Fri Mar  6 15:25:40 2026

                65536 blocks of size 1024. 58232 blocks available
```

Two files are present:

- **`dummy.txt`** — a decoy file (1142 bytes, older date)
- **`flag.txt`** — this is clearly what we want (37 bytes, matches the size of a picoCTF flag)

### Step 5 — Download the flag file

```
smb: \> get flag.txt
getting file \flag.txt of size 37 as flag.txt (0.0 KiloBytes/sec) (average 0.0 KiloBytes/sec)
```

The `get` command downloads the file to your **local** working directory (your Kali home folder). 

### Step 6 — Exit smbclient and read the flag

```
smb: \> exit

┌──(zham㉿kali)-[~]
└─$ cat flag.txt
picoCTF{5mb_pr1nter_5h4re5_ac4c227e}
```

✅ Got the flag! 🎯

> **Fun detail:** The flag itself contains `5mb_pr1nter_5h4re5` — leet-speak for "SMB printer shares," the exact protocol and scenario used in this challenge.

---

## Alternative Methods

### Alternative 1 — Download all files at once

If you're not sure which file contains the flag, download everything in one go:

```bash
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```

- `recurse ON` — include subdirectories
- `prompt OFF` — don't ask for confirmation on each file
- `mget *` — download all files matching `*` (everything)

Then exit and search all downloaded files:

```bash
grep -r "picoCTF" .
```

### Alternative 2 — Use `smbmap` for reconnaissance

`smbmap` is another tool that gives a cleaner overview of shares and their permissions:

```bash
smbmap -H mysterious-sea.picoctf.net -P 60867
```

It shows read/write permissions per share — useful for larger engagements where you need to quickly identify accessible shares.

### Alternative 3 — Read the file without downloading

You can pipe a file directly to stdout without saving it locally using `more`:

```bash
smb: \> more flag.txt
```

This prints the file contents directly inside the smbclient session — handy for quick reads.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` | Verify the server port is open | ⭐ Easy |
| `smbclient -L` | List available SMB shares | ⭐ Easy |
| `smbclient` | Connect to the share and browse files | ⭐ Easy |
| `get` | Download the flag file | ⭐ Easy |
| `cat` | Read the downloaded flag | ⭐ Easy |

---

## Key Takeaways

- **SMB is a file-sharing protocol** — it's everywhere in corporate networks and CTF challenges involving network printers or file servers
- **`smbclient -L //host -p PORT -N`** is your go-to for listing shares anonymously; always try guest/anonymous access first
- **`smb: \>` commands run inside smbclient** — don't confuse them with your regular terminal
- **Spotting the right file** — `flag.txt` was obvious here, but in real engagements, look at file sizes, dates, and names for clues
- **Port numbers matter** — SMB normally runs on port 445, but CTF challenges often use custom ports; always check with `-p`
