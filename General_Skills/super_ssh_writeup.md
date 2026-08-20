# Super SSH — picoCTF Writeup  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** 200  
**Flag:** `picoCTF{s3cur3_c0nn3ct10n_07a987ac}`  
**Platform:** picoCTF 2024  
**Writeup by:** zham  

## Description
> Using a Secure Shell (SSH) is going to be pretty important.  
>   
> Can you `ssh` as `ctf-player` to `titan.picoctf.net` at port `53988` to get the flag?  
>   
> You'll also need the password `84b12bae`. If asked, accept the fingerprint with `yes`.  
>   
> If your device doesn't have a shell, you can use: `https://webshell.picoctf.org`  
>   
> If you're not sure what a shell is, check out our Primer: `https://primer.picoctf.com/#_the_shell`

## Hints
> 1. `https://linux.die.net/man/1/ssh`  
> 2. You can try logging in 'as' someone with `<user>@titan.picoctf.net`  
> 3. How could you specify the port?  
> 4. Remember, passwords are hidden when typed into the shell

## Background Knowledge
**SSH (Secure Shell)** is a network protocol for logging into a remote machine and running commands on it. Everything sent over the wire (after the initial key exchange) is encrypted, which is why it's preferred over `telnet` or `rlogin`.

The basic syntax is:
```
ssh -p <port> <user>@<host>
```

When you connect to a host for the first time, SSH asks you to verify the server's fingerprint (a hash of its public key). Typing `yes` adds the host to `~/.ssh/known_hosts` so you don't get asked again.

The password prompt accepts input but doesn't echo the characters back to the screen (that's hint 4). That's why you might think your keyboard isn't working - it is, the shell just isn't showing what you type.

## Solution

### Step 1 — Open a shell
We need a terminal. On Kali, the shell is built-in. The user is on Kali Linux on VirtualBox, so the shell is ready to go.

```bash
┌──(zham㉿kali)-[~]
└─$ whoami
zham
```

### Step 2 — SSH into the box

```bash
┌──(zham㉿kali)-[~]
└─$ ssh -p 53988 ctf-player@titan.picoctf.net
The authenticity of host '[titan.picoctf.net]:53988 ([3.131.60.8]:53988)' can't be established.
ED25519 key fingerprint is SHA256:4dc0eBq5nyB4NTc5dK2L8F4nK1jAd4Ld3vDOQR1MNAA.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[titan.picoctf.net]:53988' (ED25519) to the list of known hosts.
ctf-player@titan.picoctf.net's password: 84b12bae
```

The password is typed **invisibly** - you won't see anything as you type `84b12bae`. Just hit Enter after typing it.

The login banner is the flag:

```
Welcome ctf-player, here's your flag: picoCTF{s3cur3_c0nn3ct10n_07a987ac}
Connection to titan.picoctf.net closed.
```

The server prints the flag on login and immediately closes the connection. Done.

### Alternative: scripted SSH with Python (Windows-friendly)

If you don't have a TTY (Windows native shell, no Git Bash, etc.), use Python with `paramiko`:

```bash
┌──(zham㉿kali)-[~]
└─$ pip install paramiko
```

```python
#!/usr/bin/env python3
import paramiko

c = paramiko.SSHClient()
c.set_missing_host_key_policy(paramiko.AutoAddPolicy())
c.connect('titan.picoctf.net', port=53988,
          username='ctf-player', password='84b12bae',
          look_for_keys=False, allow_agent=False, timeout=10)
stdin, stdout, stderr = c.exec_command('cat flag* 2>/dev/null; cat /flag* 2>/dev/null')
print(stdout.read().decode())
c.close()
```

```bash
┌──(zham㉿kali)-[~]
└─$ python3 super_ssh.py
Welcome ctf-player, here's your flag: picoCTF{s3cur3_c0nn3ct10n_07a987ac}
```

### Step 3 — Submit

```bash
┌──(zham㉿kali)-[~]
└─$ echo "picoCTF{s3cur3_c0nn3ct10n_07a987ac}"
picoCTF{s3cur3_c0nn3ct10n_07a987ac
```

Pasted into the picoCTF flag box. Accepted.

## What Happened Internally
1. The picoCTF challenge server is running an SSH daemon on TCP port 53988. It accepts password authentication for the user `ctf-player` with password `84b12bae`.
2. On successful login, the server's shell profile (likely `~/.bashrc` or a system-level drop-in) prints the flag and then closes the session - so you don't actually get an interactive shell, just a banner dump.
3. The server's SSH host key is a standard ED25519 key with a fixed fingerprint (`SHA256:4dc0eBq5nyB4NTc5dK2L8F4nK1jAd4Ld3vDOQR1MNAA`). Accepting it adds the host to `~/.ssh/known_hosts` so future connections are silent.
4. The password is sent over the encrypted channel, so the "hidden" behavior in hint 4 is a local TTY thing, not a security thing. The server still sees the cleartext.

## Tools Used
| Tool | Purpose |
|------|---------|
| `ssh` (OpenSSH client) | The whole challenge |
| `paramiko` (Python) | Alternative on Windows without a real TTY |
| `pip` | Install paramiko if needed |

## Key Takeaways
- **SSH syntax is `ssh -p <port> <user>@<host>`.** The `-p` flag is the most commonly missed - without it, SSH defaults to port 22.
- **Host fingerprints are how you trust a server.** First-time connection asks `yes/no` - typing `yes` saves the host's public key to `~/.ssh/known_hosts`. If a future connection's fingerprint doesn't match, SSH refuses to connect (possible MITM).
- **Passwords are hidden by the TTY, not by the protocol.** What you see on screen is decided by your local terminal, not the remote server. SSH itself is fine with showing the password if the local TTY is configured to echo.
- **Bash login banners are an easy CTF pattern.** The server's `~/.bashrc` or `/etc/motd` (message of the day) prints a welcome string and exits. You don't need to run any commands - just log in and read.
- **Wordplay decode**: "Super SSH" - the flag is `s3cur3_c0nn3ct10n` = "secure connection" in 1337speak (3->e, 0->o, 1->i). The "super" is just marketing - SSH is the right tool for any remote shell job.

## Alternative Solve Methods
1. **picoCTF's webshell** (`https://webshell.picoctf.org`). Browser-based terminal, the challenge's intended fallback for users without a local shell. Same commands, just a different host.
2. **PuTTY** (Windows). Classic SSH client with a GUI. Host=titan.picoctf.net, Port=53988, Connection type=SSH.
3. **Windows Terminal + OpenSSH** (Windows 10+ has OpenSSH built in). `ssh -p 53988 ctf-player@titan.picoctf.net` works the same as on Linux.
4. **`nc` won't work** - SSH is not raw TCP, you need an actual SSH client to do the key exchange and protocol negotiation.
