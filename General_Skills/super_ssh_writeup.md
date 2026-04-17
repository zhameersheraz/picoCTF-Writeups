# Super SSH — picoCTF Writeup

**Challenge:** Super SSH  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{s3cur3_c0nn3ct10n_8306c99d}`

---

## Description

> Using a Secure Shell (SSH) is going to be pretty important. Can you ssh as `ctf-player` to `titan.picoctf.net` at port `51030` to get the flag?
> Password: `1ad5be0d`

**Hint shown in challenge:** `https://linux.die.net/man/1/ssh`

---

## Background Knowledge (Read This First!)

### What is SSH?

**SSH (Secure Shell)** is a protocol used to securely connect to a remote computer over a network. It encrypts all communication between your machine and the server.

The basic SSH command format:
```bash
ssh -p <port> <username>@<hostname>
```

| Part | Value in this challenge |
|------|------------------------|
| `-p 51030` | Connect on port 51030 (default SSH port is 22) |
| `ctf-player` | The username to log in as |
| `titan.picoctf.net` | The server address |

---

## Solution — Step by Step

### Step 1 — Connect via SSH

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ssh -p 51030 ctf-player@titan.picoctf.net
```

- Accepted the fingerprint by typing `yes`
- Entered the password `1ad5be0d` (note: password is hidden when typing)

**Output:**
```
Welcome ctf-player, here's your flag: picoCTF{s3cur3_c0nn3ct10n_8306c99d}
```

Got the flag! 🎯

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `ssh` | Connect securely to the remote server |

---

## Key Takeaways

- SSH is used to connect to remote servers securely
- Use `-p` to specify a non-default port
- SSH passwords are hidden when typing — just type and press Enter
- When connecting to a new server, SSH asks you to accept a fingerprint — type `yes` to continue
- The fingerprint is a way to verify you are connecting to the right server
