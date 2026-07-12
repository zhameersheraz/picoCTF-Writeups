# Operation Oni — picoCTF Writeup

**Challenge:** Operation Oni  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{k3y_5l3u7h_b5066e83}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Download this disk image, find the key and log into the remote machine.
>
> Note: if you are using the webshell, download and extract the disk image into /tmp not your home directory.
>
> Download disk image
>
> Remote machine: `ssh -i key_file -p 61365 ctf-player@saturn.picoctf.net`

## Hints

The challenge page did not display a hint block when I solved it. The challenge is its own hint: the word "key" and the example command both point at an SSH key. I just had to find one inside the image and use it.

---

## Background Knowledge (Read This First!)

### What is an SSH key?

SSH supports two authentication methods:

- **Passwords** — you type a string and the server checks it.
- **Public-key cryptography** — the server holds a *public* key, you hold the matching *private* key, and SSH uses them in a handshake to prove who you are without ever sending a secret over the wire.

A key pair is just two files. By convention, the private key has no extension (e.g. `id_ed25519`) and the public key is the same name with `.pub` (e.g. `id_ed25519.pub`). The private key is the secret. Anyone who reads it can impersonate you. The public key is meant to be shared: you paste it into the server's `~/.ssh/authorized_keys` to grant yourself access.

Modern OpenSSH defaults to the **Ed25519** algorithm. The private key file is plaintext ASCII that starts with:

```
-----BEGIN OPENSSH PRIVATE KEY-----
```

and ends with the matching footer. If you ever see a file that looks like that, treat it like a password.

### Where do SSH keys live on Linux?

In a normal Linux install:

- A user's keys live in `~/.ssh/` — usually `/root/.ssh/` for root, `/home/alice/.ssh/` for alice, and so on.
- The system's host keys (used by the server to identify itself to clients) live in `/etc/ssh/` — `ssh_host_ed25519_key`, `ssh_host_rsa_key`, etc.

For a forensics challenge like this, the user keys are what I care about. The host keys are useless — they identify the *server*, not the *client*.

### Why `chmod 600`?

SSH refuses to use a private key that is readable by anyone other than its owner. If the key sits in your home directory and the permissions are `0644`, ssh prints:

```
Permissions 0644 for 'id_ed25519' are too open.
```

and exits. The fix is `chmod 600 id_ed25519` (or stricter). On a CTF, that is the second thing I always do after extracting a key from a disk image.

### The Plan

1. Open the image with Sleuth Kit, the same as Operation Orchid.
2. Walk into `/root/.ssh/` and pull out the private key.
3. Make sure the permissions are tight.
4. SSH into the per-user instance, get the flag, and shut it down.

---

## Solution — Step by Step

### Step 1 — Download and extract the image

Same as before: curl the `.gz`, then `gunzip -k` so I keep both copies.

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ curl -L -o disk.img.gz \
    "https://artifacts.picoctf.net/c/70/disk.img.gz"
  % Total    % Received % Xferd   Average Speed   Time    Current
                                 Dload  Upload   Total   Spent    Speed
100 45.9M  100 45.9M    0     0  74.2M      0  74.2M

┌──(zham㉿kali)-[~/operation-oni]
└─$ gunzip -k disk.img.gz
```

The extracted image is `disk.img`, about 230 MB.

### Step 2 — Inspect the partition table

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ fdisk -l disk.img
Disk disk.img: 230 MiB, 241172480 bytes, 471040 sectors
Units: sectors of 1 * 512 = 512 bytes
...
Device     Boot  Start    End Sectors  Size Id Type
disk.img1  *      2048 206847  204800  100M 83 Linux
disk.img2       206848 471039  264192  129M 83 Linux
```

Two partitions this time, both ext filesystems. Partition 1 is the boot partition (kernel, initramfs, bootloader — same as last time). Partition 2 is the main filesystem, starting at sector `206848`.

### Step 3 — Find the SSH key in the filesystem

The challenge tells me to look for a *key*. I start with a recursive listing filtered to anything key-shaped.

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ fls -r -o 206848 disk.img 2>&1 | grep -iE "ssh|id_rsa|id_ed25519" | head -20
+ d/d 14:	ssh
++ r/r 15:	ssh_host_ed25519_key
++ r/r 16:	ssh_host_ed25519_key.pub
++ r/r 17:	ssh_host_ecdsa_key
...
++ d/d 2048:	key
++ d/d 2066:	keys
```

Most of those are system paths under `/etc/`, `/usr/lib/`, and `/lib/`. I do not need any of them. I need the *user's* key, which almost always lives in a `~/.ssh/` directory. Let me check `/root`:

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ fls -o 206848 disk.img 470
r/r 2344:	.ash_history
d/d 3916:	.ssh
```

There it is. `/root/.ssh/` exists. Inside:

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ fls -o 206848 disk.img 3916
r/r 2345:	id_ed25519
r/r 2346:	id_ed25519.pub
```

An ed25519 keypair. The challenge mentioned "key" (singular) and SSH — this matches. There is also a `.ash_history` file, which is the same one I saw in Operation Orchid. This challenge is from the same series, and the author used the same Alpine base image.

### Step 4 — Sanity check with `.ash_history`

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ icat -o 206848 disk.img 2344
ssh-keygen -t ed25519
ls .ssh/
halt
```

Three lines. The operator generated an ed25519 key, listed `.ssh/` to confirm it landed there, and shut down. That confirms: the key in `/root/.ssh/id_ed25519` is the one I want, and it is the one the challenge wants me to use.

I also peek at the public key to be thorough:

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ icat -o 206848 disk.img 2346
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGCtd7hso2E7OQItY6aTjMMyKZb1FVmeBfnVjyHcGYos root@localhost
```

The comment `root@localhost` is just a label the generator wrote in. It is the same key as the private one, just on the public side.

### Step 5 — Extract the private key and lock it down

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ icat -o 206848 disk.img 2345 > id_ed25519
┌──(zham㉿kali)-[~/operation-oni]
└─$ chmod 600 id_ed25519
┌──(zham㉿kali)-[~/operation-oni]
└─$ head -1 id_ed25519
-----BEGIN OPENSSH PRIVATE KEY-----
┌──(zham㉿kali)-[~/operation-oni]
└─$ ls -la id_ed25519
-rw------- 1 zham zham 411 Jul 12 14:53 id_ed25519
```

The `-rw-------` (octal `600`) permissions are what OpenSSH wants. With looser permissions ssh would refuse to use the key.

### Step 6 — SSH into the per-user instance

The challenge gives me the exact command. The `-i` flag points at the key, `-p 61365` is the port, and the user/host come at the end. I add `-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null` so the first-time host key prompt does not get in the way — the picoCTF servers recycle, and I do not want a stale fingerprint to block me.

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ ssh -i id_ed25519 \
       -o StrictHostKeyChecking=no \
       -o UserKnownHostsFile=/dev/null \
       -p 61365 ctf-player@saturn.picoctf.net \
       "cat flag.txt"
Warning: Permanently added '[saturn.picoctf.net]:61365' (ED25519) to the list of known hosts.
picoCTF{k3y_5l3u7h_b5066e83}
```

I passed `cat flag.txt` as the remote command, so ssh runs it on the server and prints the result. I do not have to drop into an interactive shell just to read a single file.

```
Flag: picoCTF{k3y_5l3u7h_b5066e83}
```

Done.

### Step 7 — Make sure the flag is really there

If I want to double-check, I can ssh in interactively (no remote command) and look around. The flag file is in the home directory of `ctf-player`:

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ ssh -i id_ed25519 -p 61365 ctf-player@saturn.picoctf.net
$ pwd
/home/ctf-player
$ ls -la
total 4
drwxr-xr-x 1 ctf-player ctf-player 20 Jul 12 14:53 .
drwx------ 2 ctf-player ctf-player 34 Jul 12 14:53 .cache
drwxr-xr-x 2 ctf-player ctf-player 29 Aug  4  2023 .ssh
-rw-r--r-- 1 root       root       28 Aug  4  2023 flag.txt
$ cat flag.txt
picoCTF{k3y_5l3u7h_b5066e83}
$ exit
```

Two things worth noticing:

- The flag file is owned by `root`, not `ctf-player`. That is fine — it is world-readable (`-rw-r--r--`), so any user can read it. picoCTF usually does this so the SSH login itself is the gate, not the file permissions.
- The home directory is otherwise empty — no puzzles, no chained commands, no rabbit holes. The whole challenge is "get the key, log in, read the file".

---

## Alternative Solve — `ssh-agent` and a One-Line Config

If I were going to use this key repeatedly (or wanted a cleaner shell), I would load it into the SSH agent and let it pick the key automatically.

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ eval "$(ssh-agent -s)"
Agent pid 1234

┌──(zham㉿kali)-[~/operation-oni]
└─$ ssh-add id_ed25519
Identity added: id_ed25519 (root@localhost)

┌──(zham㉿kali)-[~/operation-oni]
└─$ ssh -p 61365 ctf-player@saturn.picoctf.net "cat flag.txt"
picoCTF{k3y_5l3u7h_b5066e83}
```

I do not need `-i` anymore because `ssh-agent` already knows about the key. The `eval "$(ssh-agent -s)"` line starts a background agent and prints the environment variables I need to set, all in one go.

For an even shorter command, I can put a block in `~/.ssh/config`:

```
┌──(zham㉿kali)-[~]
└─$ nano ~/.ssh/config
```

```
Host oni
    HostName saturn.picoctf.net
    Port 61365
    User ctf-player
    IdentityFile ~/operation-oni/id_ed25519
    IdentitiesOnly yes
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`. Now I can connect with one word:

```
┌──(zham㉿kali)-[~/operation-oni]
└─$ ssh oni "cat flag.txt"
picoCTF{k3y_5l3u7h_b5066e83}
```

`IdentitiesOnly yes` tells ssh to use *only* the key in `IdentityFile`, not any other keys in my agent — that matters if you have lots of keys loaded, because ssh would otherwise try each one and get rate-limited by the server.

---

## What Happened Internally

| Step | What happened |
|------|---------------|
| 1 | The challenge author built a tiny Alpine VM, dropped into a root shell, and ran `ssh-keygen -t ed25519` with no passphrase. OpenSSH created `/root/.ssh/id_ed25519` and the matching `.pub`. |
| 2 | The author copied the *public* half of the key into the per-user instance, into `/home/ctf-player/.ssh/authorized_keys`. That is what makes the per-user container trust the private key. |
| 3 | The author dumped the entire disk to a raw image, gzipped it, and uploaded it to the artifact server. |
| 4 | I downloaded the image, found the key in `/root/.ssh/id_ed25519`, and SSHed into the instance as `ctf-player`. From the server's point of view, my SSH client offered the matching public key, the server looked it up in `authorized_keys`, and the handshake completed. |
| 5 | The flag was sitting at `/home/ctf-player/flag.txt`, world-readable, waiting for me to log in and read it. |

The puzzle boils down to "find a private key on a disk and use it for the obvious purpose". The forensics skills I needed were the same as Operation Orchid: fls + icat + an understanding of where SSH stores things.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Download the disk image archive | Easy |
| `gunzip` | Decompress `.img.gz` to `.img` | Easy |
| `fdisk` | Inspect the partition table | Easy |
| `fls` (Sleuth Kit) | Walk the filesystem looking for `.ssh/` | Easy |
| `icat` (Sleuth Kit) | Pull the private key out of the image | Easy |
| `chmod 600` | Set safe permissions on the key | Easy |
| `ssh` | Authenticate to the per-user instance | Easy |
| `ssh-agent` + `ssh-add` (alternative) | Cache the key so I do not have to pass `-i` every time | Medium |
| `~/.ssh/config` (alternative) | Save connection details so I can type `ssh oni` instead of the full command | Medium |

---

## Key Takeaways

- **SSH keys are files on disk, and disk images are easy to walk.** If an attacker can read a disk, they can read every private key on it. That is exactly why production servers use full-disk encryption, key passphrases, and hardware tokens — a flat disk image is a sitting duck.
- **Permissions matter even for files you just extracted.** `chmod 600` is two characters; the "Permissions too open" error from ssh is the price of forgetting it.
- **Always check the obvious directories first.** For SSH, that is `~/.ssh/`. For shell history, that is `~/.bash_history` or `~/.ash_history`. For cron jobs, that is `/etc/crontab` and `/var/spool/cron/`. A five-second `fls -r` over the image saves you from a long hunt.
- **The challenge's wording is the hint.** "find the key" + `ssh -i key_file` in the example command was telling me the key is the whole challenge. When the description and the example command agree, the design is rarely hidden.
- **Add `-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null` on first contact.** picoCTF re-uses hostnames and IPs across instances, so the fingerprint of the last challenge's server is almost always wrong. Disabling the prompt saves a `yes` keystroke and keeps the terminal log clean.
- **Use `ssh ... "remote command"` for one-off reads.** You do not need an interactive shell to grab a flag. Single quotes around the command keep `$` and backticks from being expanded on your side.
- **Flag wordplay.** `picoCTF{k3y_5l3u7h_b5066e83}` decodes as `key_sleuth_<hash>`. `5l3u7h` is leetspeak for `sleuth` (`5`→`s`, `3`→`e`, `7`→`t`, the `l`, `u`, `h` stay themselves). The whole flag is a self-describing title: the challenge is about being a *key sleuth* — someone who hunts SSH private keys on a captured disk.
