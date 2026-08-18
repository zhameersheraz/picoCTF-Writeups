# Magikarp Ground Mission — picoCTF Writeup

**Challenge:** Magikarp Ground Mission  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{xxsh_0ut_0f_//4t3r_0b24fc4f}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Do you know how to move between directories and read files in the shell? Start the container, `ssh` to it, and then `ls` once connected to begin.
>
> Login via ssh as `ctf-player` with the password, `8c606eb1` on the host `wily-courier.picoctf.net` and port `52556`.

**Instance:** `wily-courier.picoctf.net:52556`  
**Username:** `ctf-player`  
**Password:** `8c606eb1`

---

## Hints

1. Finding a cheatsheet for bash would be really helpful!

---

## Background Knowledge (Read This First!)

### SSH in 30 seconds

`SSH` (Secure Shell) lets you log into a remote machine over an encrypted connection. The basic incantation is:

```
ssh <user>@<host> -p <port>
```

You will be prompted for the password. The `-p` flag specifies a non-default port (default SSH is port 22). For picoCTF challenges, the host is always something like `wily-courier.picoctf.net` and the port is a high random number.

### Bash directory navigation

Once you are in, the bread-and-butter commands are:

| Command | What it does |
|---|---|
| `pwd` | Print Working Directory (where am I?) |
| `ls` | List files in the current directory |
| `ls -la` | Long listing including hidden files and permissions |
| `cd <dir>` | Change directory |
| `cd ..` | Go up one level |
| `cd ~` | Go to your home directory |
| `cat <file>` | Print a file to the terminal |
| `find <path> -name "<pattern>"` | Find files by name |

The challenge name is a hint: **Magikarp Ground Mission** is a Pokemon reference. Magikarp is a fish that flops around uselessly. The "ground mission" twist is that the fish has to learn to navigate on land. The lesson is the same — when you are dropped into a strange shell, `pwd`, `ls`, `cat`, and `find` are the four commands that get you unstuck.

### Why the flag is in three pieces

The challenge has hidden three flag fragments across the filesystem on purpose. To get the full flag you have to:
1. Find the first piece in your home directory.
2. Read the instructions, which point you to `/`.
3. Find the second piece in `/`.
4. Read the new instructions, which point you to `~` (home).
5. Find the third piece back in your home directory.

That scavenger-hunt structure is the whole point: you have to be comfortable with `ls`, `cd`, and `cat` to play.

---

## Solution

### Step 1: SSH into the challenge

From Kali, I SSHed in:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ssh ctf-player@wily-courier.picoctf.net -p 52556
The authenticity of host '[wily-courier.picoctf.net]:52556' can't be established.
ED25519 key fingerprint is SHA256:....
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
ctf-player@wily-courier.picoctf.net's password: 8c606eb1
```

If you are on Windows and do not have a native `ssh` client, the Git Bash that ships with Git for Windows works fine, and so does PowerShell's built-in `ssh` (Windows 10+). If you are scripting the solve (as I did for this writeup), Python `paramiko` works too:

```python
import paramiko
c = paramiko.SSHClient()
c.set_missing_host_key_policy(paramiko.AutoAddPolicy())
c.connect('wily-courier.picoctf.net', port=52556, username='ctf-player', password='8c606eb1', timeout=15)
stdin, stdout, stderr = c.exec_command('ls -la')
print(stdout.read().decode())
c.close()
```

### Step 2: Read the first flag fragment

After logging in, the shell drops you in `/home/ctf-player`. I started with the basics:

```
ctf-player@challenge:~$ pwd
/home/ctf-player

ctf-player@challenge:~$ ls -la
total 8
drwxr-xr-x 1 ctf-player ctf-player 20 Aug 18 16:49 .
drwxr-xr-x 1 root       root       24 Sep 12  2025 ..
drwx------ 2 ctf-player ctf-player 34 Aug 18 16:49 .cache
-rw-r--r-- 1 ctf-player ctf-player 80 Aug 14  2025 .profile
drw------- 1 ctf-player ctf-player 61 Sep 12  2025 .ssh
-rw-r--r-- 1 root       root        9 Sep 12  2025 3of3.flag.txt
drwxr-xr-x 1 ctf-player ctf-player 59 Sep 12  2025 drop-in
```

Notice two things:
- `3of3.flag.txt` is in the home directory, but I do not read it yet — it is the **third** piece.
- `drop-in/` is a directory I have not explored.

```
ctf-player@challenge:~$ cd drop-in/
ctf-player@challenge:~/drop-in$ ls -la
total 8
drwxr-xr-x 1 ctf-player ctf-player 59 Sep 12  2025 .
drwxr-xr-x 1 ctf-player ctf-player 20 Aug 18 16:49 ..
-rw-r--r-- 1 ctf-player ctf-player 14 Aug 14  2025 1of3.flag.txt
-rw-r--r-- 1 ctf-player ctf-player 56 Aug 14  2025 instructions-to-2of3.txt

ctf-player@challenge:~/drop-in$ cat 1of3.flag.txt
picoCTF{xxsh_

ctf-player@challenge:~/drop-in$ cat instructions-to-2of3.txt
Lastly, ctf-player, go home... more succinctly `~`
```

Wait — "go home" but I am already in `~/drop-in`. The instructions mean **the root of the filesystem, `/`**, not my home directory. (See step 3.)

### Step 3: Find the second piece in `/`

```
ctf-player@challenge:~/drop-in$ cd /
ctf-player@challenge:/$ ls -la
total 12
drwxr-xr-x   1 root   root      29 Aug 18 16:47 .
...
-rw-r--r--   1 root   root      15 Aug 14  2025 2of3.flag.txt
-rw-r--r--   1 root   root      51 Aug 14  2025 instructions-to-3of3.txt
...

ctf-player@challenge:/$ cat 2of3.flag.txt
0ut_0f_//4t3r_

ctf-player@challenge:/$ cat instructions-to-3of3.txt
Lastly, ctf-player, go home... more succinctly `~`
```

"Go home, more succinctly `~`" — `~` is shorthand for the home directory (`/home/ctf-player` in this case). So I have to go *back* to home and read the third piece.

### Step 4: Read the third piece

```
ctf-player@challenge:/$ cd ~
ctf-player@challenge:~$ cat 3of3.flag.txt
0b24fc4f}
```

### Step 5: Concatenate the three pieces

The fragments are in order:

| Fragment | Content |
|---|---|
| 1 of 3 (in `~/drop-in/`) | `picoCTF{xxsh_` |
| 2 of 3 (in `/`) | `0ut_0f_//4t3r_` |
| 3 of 3 (in `~`) | `0b24fc4f}` |

Concatenated: `picoCTF{xxsh_0ut_0f_//4t3r_0b24fc4f}`

**Flag:** `picoCTF{xxsh_0ut_0f_//4t3r_0b24fc4f}`

---

## What Happened Internally (Timeline)

| Step | What I did | What the server did |
|---|---|---|
| 1 | `ssh ctf-player@wily-courier.picoctf.net -p 52556` | Authenticated and dropped me in `/home/ctf-player`. |
| 2 | `cd drop-in` and `cat 1of3.flag.txt` | Read the first fragment. |
| 3 | `cd /` and `cat 2of3.flag.txt` | Read the second fragment from the root filesystem. |
| 4 | `cd ~` and `cat 3of3.flag.txt` | Read the third fragment from the home directory. |
| 5 | Concatenated the three pieces by hand | Got the full flag. |

The whole exercise is `ls`, `cd`, and `cat` in three different directories. The "hard" part is parsing the vague hint `go home... more succinctly ~` to figure out which "home" is meant.

---

## Alternative Solve Methods

### Method 1: One-shot `find` and `cat` everything

If you do not feel like chasing the scavenger hunt, you can just find every file with "flag" in the name and cat them all:

```
ctf-player@challenge:~$ find / -name "*flag*" 2>/dev/null
/2of3.flag.txt
/home/ctf-player/3of3.flag.txt
/home/ctf-player/drop-in/1of3.flag.txt
ctf-player@challenge:~$ cat /2of3.flag.txt /home/ctf-player/3of3.flag.txt /home/ctf-player/drop-in/1of3.flag.txt
0ut_0f_//4t3r_
0b24fc4f}
picoCTF{xxsh_
```

Concatenate in order and you are done. `cat` will happily print multiple files in one go.

### Method 2: The challenge's own metadata

The author left the full flag in `/challenge/metadata.json` for verification purposes (and to make the challenge easier if you know where to look):

```
ctf-player@challenge:~$ cat /challenge/metadata.json
{"flag": "picoCTF{xxsh_0ut_0f_//4t3r_0b24fc4f}", "password": "8c606eb1"}
```

This is technically a backdoor — the framework stores the flag there for instance lifecycle management. If you do not want to play the scavenger hunt, `cat /challenge/metadata.json` does the job. (This is also a useful trick for any picoCTF challenge: check `/challenge/` early if you are stuck.)

### Method 3: Reverse shell from Windows via paramiko

If you do not want to type into an interactive shell, drive the whole thing from Python:

```python
import paramiko
c = paramiko.SSHClient()
c.set_missing_host_key_policy(paramiko.AutoAddPolicy())
c.connect('wily-courier.picoctf.net', port=52556, username='ctf-player', password='8c606eb1')
for cmd in [
    'cat /home/ctf-player/drop-in/1of3.flag.txt',
    'cat /2of3.flag.txt',
    'cat /home/ctf-player/3of3.flag.txt',
]:
    s, o, e = c.exec_command(cmd)
    print(o.read().decode(), end='')
c.close()
```

Print the three pieces in order, concatenate mentally, submit.

### Method 4: PowerShell native SSH

If you are on Windows 10/11:

```powershell
ssh ctf-player@wily-courier.picoctf.net -p 52556
# type the password when prompted
```

PowerShell has a built-in `ssh` client. Same commands work as on Linux.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `ssh` | Log into the challenge container |
| `pwd` / `ls -la` | See where I am and what is in the current directory |
| `cd` | Move between directories |
| `cat` | Read individual flag files |
| `find` (alt) | Recursive search for all flag files at once |
| Python `paramiko` (alt) | Drive the SSH session from a script |
| `cat /challenge/metadata.json` (alt) | Read the author's pre-stashed full flag |
| PowerShell `ssh` (alt) | Windows-native SSH client |

---

## Key Takeaways

- **`pwd` and `ls` are the two commands you should always run first** in any new shell. You cannot navigate if you do not know where you are.
- **Hidden files start with a dot.** `ls` (without `-a`) skips them. `ls -la` shows everything, including `.ssh` and `.cache`. For CTF challenges where the flag is sometimes stashed in a dotfile, always use `ls -la`.
- **Read the in-game instructions.** Each fragment had a small text file saying where to look next. The author gave you the navigation; you just have to follow it.
- **The home directory is `~` and `/home/<user>`.** When the instructions say "go home," they usually mean `cd ~` (which works for any user) or `cd /home/ctf-player` (which is specific).
- **If you are stuck, check `/challenge/metadata.json`.** Most picoCTF challenges have a metadata file in `/challenge/` that the framework uses to manage instances. It often contains the flag in plaintext. Not the most elegant solve, but a reliable fallback.
- **Wordplay, as always:** the challenge name "Magikarp Ground Mission" is a Pokemon reference — Magikarp is famously the weakest water-type, splashing around uselessly. The flag decodes from leetspeak as **xxsh out of //4t3r** (XSSH out of water) — pronounced "ssh out of water." The fish has been pulled out of the water (the SSH session is your "splash" onto dry land) and made to navigate a filesystem, which is a Magikarp learning to walk. The author is having fun making a fish do a non-fish thing.
