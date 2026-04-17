# My Git — picoCTF Writeup

**Challenge:** My Git  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{1mp3rs0n4t4_g17_345y_e522152d}`  

---

## Description

> I have built my own Git server with my own rules! You can clone the challenge repo using the command below.
>
> `git clone ssh://git@foggy-cliff.picoctf.net:50858/git/challenge.git`
>
> Here's the password: `d9df7038`
>
> Check the README to get your flag!

**Hint 1:** `How do you specify your Git username and email?`

**Tags:** `git`, `impersonation`, `General Skills`

---

## Background Knowledge (Read This First!)

### What is `git config`?

Every git commit records **who made it** — a name and an email address. You set these with:

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

Critically, git **does not verify** these values. You can set them to anything — any name, any email — and git will happily use them. This is the core vulnerability this challenge exploits.

### What is git impersonation?

Because git identity is self-reported and unverified by default, anyone can commit under any name or email they choose. In real-world projects, **signed commits** (using GPG keys) exist to cryptographically prove a commit actually came from who it claims. Without signing, git author identity is purely on the honor system.

### What are server-side git hooks?

A **git hook** is a script that runs automatically on the server when certain git events happen — like receiving a push. In this challenge, the server has a hook that inspects every incoming commit's author name and email. If they match `root:root@picoctf`, it rewards you with the flag in the push response.

### What is SSH git cloning?

Git repos can be cloned over SSH using the format:

```
ssh://user@host:port/path/to/repo.git
```

When prompted about a host fingerprint, type `yes` to accept it. The password is entered invisibly — nothing appears on screen as you type, which is completely normal.

---

## Solution — Step by Step

### Step 1 — Clone the repository

```
┌──(zham㉿kali)-[~]
└─$ git clone ssh://git@foggy-cliff.picoctf.net:50858/git/challenge.git
Cloning into 'challenge'...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
git@foggy-cliff.picoctf.net's password:
remote: Enumerating objects: 3, done.
remote: Total 3 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (3/3), done.
```

Type `yes` at the fingerprint prompt, then enter `d9df7038` as the password (nothing appears as you type — that's normal).

### Step 2 — Read the README

```
┌──(zham㉿kali)-[~]
└─$ cd challenge

┌──(zham㉿kali)-[~/challenge]
└─$ cat README.md
# MyGit
### If you want the flag, make sure to push the flag!
Only flag.txt pushed by `root:root@picoctf` will be updated with the flag.
GOOD LUCK!
```

The server is telling us exactly what it wants: a push of `flag.txt` where the git author is `root` with email `root@picoctf`. Since git doesn't verify identity, we can simply set our local config to match.

### Step 3 — Impersonate root using `git config`

```
┌──(zham㉿kali)-[~/challenge]
└─$ git config user.name "root"

┌──(zham㉿kali)-[~/challenge]
└─$ git config user.email "root@picoctf"
```

These two commands set our local git identity to exactly what the server expects. This only affects this repository (no `--global` flag), so our real identity elsewhere is unchanged.

### Step 4 — Create, commit, and push `flag.txt`

```
┌──(zham㉿kali)-[~/challenge]
└─$ echo "flag" > flag.txt

┌──(zham㉿kali)-[~/challenge]
└─$ git add flag.txt

┌──(zham㉿kali)-[~/challenge]
└─$ git commit -m "flag"
[master 973face] flag
 1 file changed, 1 insertion(+)
 create mode 100644 flag.txt

┌──(zham㉿kali)-[~/challenge]
└─$ git push
git@foggy-cliff.picoctf.net's password:
Writing objects: 100% (3/3), 264 bytes | 264.00 KiB/s, done.
remote: Author matched and flag.txt found in commit...
remote: Congratulations! You have successfully impersonated the root user
remote: Here's your flag: picoCTF{1mp3rs0n4t4_g17_345y_e522152d}
To ssh://foggy-cliff.picoctf.net:50858/git/challenge.git
   b4df14d..973face  master -> master
```

✅ Got the flag! 🎯

> **Note:** The flag is delivered in the **remote's push response** — not written back into `flag.txt`. Always read the full output of `git push` — servers can communicate via those `remote:` lines.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `git clone` | Download the remote repository over SSH |
| `cat README.md` | Read the challenge instructions |
| `git config user.name / user.email` | Set a fake author identity to match the server's expectation |
| `git add / commit / push` | Stage, commit, and push the file to trigger the server-side hook |

---

## Key Takeaways

- **Git does not verify identity** — `user.name` and `user.email` are self-reported. Anyone can set them to anything without any authentication challenge.
- **Server-side git hooks can check author metadata** — the challenge server used a post-receive hook to validate the committer's identity and respond conditionally.
- **GPG-signed commits exist for this exact reason** — in production environments, signed commits cryptographically prove authorship. Unsigned commits can be trivially impersonated as shown here.
- **The flag came from the remote's push response** — not from the file itself. Always read the full output of `git push`; servers can communicate through `remote:` lines.
- **The challenge name says it all** — "impersonata git" (`1mp3rs0n4t4_g17`) is exactly what we did: we impersonated another user's git identity to trick the server.
