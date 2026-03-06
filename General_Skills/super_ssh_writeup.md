# Super SSH - picoCTF Writeup

**Challenge:** Super SSH  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{s3cur3_c0nn3ct10n_8306c99d}`

---

## Description

Using a Secure Shell (SSH) is going to be pretty important. Can you ssh as `ctf-player` to `titan.picoctf.net` at port `51030` to get the flag? You'll also need the password `1ad5be0d`. If asked, accept the fingerprint with `yes`.

---

## Hints

1. https://linux.die.net/man/1/ssh

---

## Solution

### Step 1: Connect via SSH

I connected to the server using the provided credentials:

```bash
ssh -p 51030 ctf-player@titan.picoctf.net
```

- Accepted the fingerprint by typing `yes`
- Entered the password `1ad5be0d` (note: password is hidden when typing)

**Output:**
```
Welcome ctf-player, here's your flag: picoCTF{s3cur3_c0nn3ct10n_8306c99d}
```

Got the flag! 🎯

---

## Why This Works

### What is SSH?

**SSH (Secure Shell)** is a protocol used to securely connect to a remote computer over a network. It encrypts all communication between your machine and the server.

The basic SSH command format is:

```bash
ssh -p <port> <username>@<hostname>
```

| Part | Value |
|------|-------|
| `-p 51030` | Connect on port 51030 (default SSH port is 22) |
| `ctf-player` | The username to log in as |
| `titan.picoctf.net` | The server address |

### Simple Breakdown

```
Run ssh command with correct port, username and host
        |
        v
Accept the fingerprint with "yes"
        |
        v
Enter the password (1ad5be0d)
        |
        v
Server prints the flag and closes connection!
```

---

## Commands Used

```bash
ssh -p 51030 ctf-player@titan.picoctf.net
# Accept fingerprint: yes
# Password: 1ad5be0d
```

---

## Flag

```
picoCTF{s3cur3_c0nn3ct10n_8306c99d}
```

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
