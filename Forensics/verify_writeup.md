# Verify — picoCTF Writeup

**Challenge:** Verify  
**Category:** Forensics  
**Difficulty:** Easy  
**Flag:** `picoCTF{trust_but_verify_451fd69b}`  

---

## Description

> People keep trying to trick my players with imitation flags. I want to make sure they get the real thing! I'm going to provide the SHA-256 hash and a decrypt script to help you know that my flags are legitimate.
>
> `ssh -p 63262 ctf-player@rhea.picoctf.net`
> Using the password `6abf4a82`. Accept the fingerprint with `yes`, and `ls` once connected to begin. Remember, in a shell, passwords are hidden!

**Checksum:** `b09c99c555e2b39a7e97849181e8996bc6a62501f0149c32447d8e65e205d6d2`  
**Hint 1:** `Checksums let you tell if a file is complete and from the original distributor. If the hash doesn't match, it's a different file.`

**Tags:** `grep`, `checksum`, `browser_webshell_solvable`

---

## Background Knowledge (Read This First!)

### What is SHA-256?

**SHA-256** is like a **fingerprint for a file**. You feed any file into it, and it spits out a unique 64-character string — called a **hash** or **checksum**. For example:

```
b09c99c555e2b39a7e97849181e8996bc6a62501f0149c32447d8e65e205d6d2
```

Two important rules about SHA-256:
- The **same file always produces the same hash** — no matter where or when you run it
- **Even changing a single byte** of the file produces a completely different hash

This is why checksums are used for verification. If someone gives you a file and says "the SHA-256 should be `b09c99...`", you can run `sha256sum` on it yourself. If the hash matches → the file is authentic. If it doesn't → someone tampered with it, or it's the wrong file entirely.

### What is `sha256sum`?

`sha256sum` is a Linux command that computes the SHA-256 hash of a file. You give it a file (or many files), and it prints the hash next to the filename:

```
b09c99c555e2b39a7e97849181e8996bc6a62501f0149c32447d8e65e205d6d2  files/451fd69b
```

Left side = the hash. Right side = the filename.

You can also use a wildcard `*` to hash **every file in a folder at once**:

```bash
sha256sum files/*
```

This prints one line per file — hundreds of lines in this challenge.

### What is `grep`?

`grep` is a Linux command that **searches text for a pattern** and prints only the matching lines. Think of it like Ctrl+F but for the terminal.

When you pipe (`|`) `sha256sum` output into `grep`, you're saying:
> "Compute the hash of every file, then show me only the line containing this specific hash."

```bash
sha256sum files/* | grep b09c99c...
```

The `|` symbol connects two commands — the output of the left command becomes the input of the right command.

### What is SSH?

**SSH (Secure Shell)** lets you remotely log into another computer over the internet and use its terminal as if you were sitting in front of it. The challenge runs on a remote server at `rhea.picoctf.net`, and you connect to it using SSH.

### ⚠️ Two Important Notes Before Starting

**Note 1 — Your password is invisible when you type it**  
When SSH asks for a password, nothing appears on screen as you type — not even dots or asterisks. This is normal and intentional. Just type `6abf4a82` and press Enter even though you can't see it.

**Note 2 — The instance has a countdown timer**  
The challenge server shuts down after the timer expires. If the connection suddenly closes mid-solve, just click "Restart Instance" on the challenge page and SSH back in.

---

## Solution — Step by Step

### Step 1 — Connect to the challenge server

Open your terminal and run:

```
┌──(zham㉿kali)-[~]
└─$ ssh -p 63262 ctf-player@rhea.picoctf.net
```

Breaking this command down:
- `ssh` — the remote login tool
- `-p 63262` — connect on port 63262 (not the default port)
- `ctf-player@rhea.picoctf.net` — username `ctf-player` at the server address

You'll see a fingerprint warning — type `yes` and press Enter:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

Then type the password `6abf4a82` and press Enter (nothing will appear as you type):

```
ctf-player@rhea.picoctf.net's password:
Welcome to Ubuntu 20.04.3 LTS...
ctf-player@pico-chall$
```

You are now inside the remote server.

### Step 2 — Look around with `ls`

```
ctf-player@pico-chall$ ls
checksum.txt  decrypt.sh  files
```

There are three things here:

- **`checksum.txt`** — contains the target SHA-256 hash we need to match
- **`decrypt.sh`** — a script that decrypts a file and reveals the flag
- **`files/`** — a folder full of candidate files

Let's peek inside the `files/` folder:

```
ctf-player@pico-chall$ ls files/
0QFPjDGl  3uK74LSS  8KKJnhNd  BS2al2Fb  FCJb9D8b  451fd69b ...
```

There are **hundreds of files** with random-looking names like `0QFPjDGl`, `451fd69b`, `BS2al2Fb`, and so on. Only **one** of them is the real file. The rest are decoys.

We can't open them one by one — we need to find the one whose SHA-256 hash matches the checksum given in the challenge:

```
b09c99c555e2b39a7e97849181e8996bc6a62501f0149c32447d8e65e205d6d2
```

### Step 3 — Find the matching file using `sha256sum` + `grep`

Run this single command:

```
ctf-player@pico-chall$ sha256sum files/* | grep b09c99c555e2b39a7e97849181e8996bc6a62501f0149c32447d8e65e205d6d2
```

What happens here:

1. `sha256sum files/*` — computes the SHA-256 hash of **every single file** in the `files/` folder and prints hundreds of lines like:
   ```
   a3f1c2d4...  files/0QFPjDGl
   99ab12fe...  files/3uK74LSS
   b09c99c5...  files/451fd69b   ← this is the one
   ...
   ```

2. `| grep b09c99c5...` — from all those hundreds of lines, shows only the one containing our target hash

The result:

```
b09c99c555e2b39a7e97849181e8996bc6a62501f0149c32447d8e65e205d6d2  files/451fd69b
```

The matching file is **`files/451fd69b`**.

> **Where does `451fd69b` come from?**  
> It's simply the **filename** of one of the files that was already sitting in the `files/` folder — you can see it in the `ls` output from Step 2. The `sha256sum | grep` command didn't create it; it just *identified* it as the one whose hash matched. You would never have known which file it was without running the checksum comparison.

### Step 4 — Decrypt the flag

Now pass that filename to the decrypt script:

```
ctf-player@pico-chall$ ./decrypt.sh files/451fd69b
picoCTF{trust_but_verify_451fd69b}
```

✅ Got the flag! 🎯

> **Fun detail:** The challenge author named the file `451fd69b`, and the flag also ends in `451fd69b`. That's intentional — it's a small nod saying "you found the right file."

---

## Alternative Methods

### Alternative 1 — Read the checksum from the file instead of the description

Instead of copying the hash from the challenge description page, you can read it directly from the server:

```bash
cat checksum.txt
```

This prints the target hash. Use that value in your grep command. Both give the same result.

### Alternative 2 — `sha256sum --check` (automated verification)

If `checksum.txt` contains the hash in the correct format (`hash  filename`), you can use the built-in check mode and skip grep entirely:

```bash
sha256sum --check checksum.txt
```

SHA-256 automatically compares every hash in the file and tells you which ones pass or fail:

```
files/451fd69b: OK
```

### Alternative 3 — Brute-force loop (for learning purposes)

A longer but educational approach using a shell loop:

```bash
for f in files/*; do
    if [ "$(sha256sum $f | awk '{print $1}')" = "b09c99c555e2b39a7e97849181e8996bc6a62501f0149c32447d8e65e205d6d2" ]; then
        echo "Match found: $f"
    fi
done
```

This loops through every file, computes its hash, and compares it to the target manually. It works but is much slower than the one-liner. Good for understanding what's happening under the hood.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `ssh` | Connect to the remote challenge server | ⭐ Easy |
| `ls` | List files to understand the environment | ⭐ Easy |
| `sha256sum` | Compute SHA-256 hashes of all files | ⭐ Easy |
| `grep` | Filter output to find the matching file | ⭐ Easy |
| `./decrypt.sh` | Decrypt and reveal the flag | ⭐ Easy |

---

## Key Takeaways

- **SHA-256 is a fingerprint for files** — if the hash matches, the file is authentic; if it doesn't, something is wrong
- **`sha256sum files/* | grep <hash>`** is the fastest way to find one real file hidden among hundreds of fakes
- **Piping commands with `|`** is one of the most powerful habits in Linux — it lets you chain simple tools together to solve complex problems in one line
- **The challenge name says it all** — "trust but verify" is a real security principle. Never blindly trust a file; always check its hash
