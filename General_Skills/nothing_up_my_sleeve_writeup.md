# Nothing Up My Sleeve — picoCTF Writeup

**Challenge:** Nothing Up My Sleeve  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 1  
**Flag:** `picoCTF{c0ngr4ts_0n_y0ur_s4n1ty}`  
**Platform:** picoCTF 2025 *(year unconfirmed — no original release year could be verified; flag this if the actual year is known)*  
**Writeup by:** zham  

---

## Description

> Let's check that your internet connection is working. This flag is 'in-the-clear', I promise!
> Download [flag.txt]

**Hint 1:** `If all you had access to was a shell, you could use wget to download the file at the URL above!`

---

## Background Knowledge (Read This First!)

### What does "in-the-clear" mean?

A flag described as **in-the-clear** means it's stored as plain, unencrypted, unencoded text — no decryption, no decoding, no privilege escalation required. The entire "challenge" here is just successfully downloading a file and reading it.

### Why this challenge exists at all

This is a **sanity check** — a deliberately trivial task that exists purely to confirm your tools work before you tackle anything harder. Specifically, it confirms that `wget` (or any file-fetching tool) on your system can actually reach the internet and pull down a file from a URL. If this step fails, every later challenge that needs file downloads will fail for the same underlying reason, so it's worth ruling out first.

### Why `wget` specifically

`wget` is a command-line tool built for exactly this: fetching a file from a URL and saving it to disk, with no browser or GUI needed. It's the standard tool for this in any restricted shell or webshell environment where there's no point-and-click way to download anything.

---

## Solution — Step by Step

### Step 1 — Download the file

```
┌──(zham㉿kali)-[~]
└─$ wget https://artifacts.picoctf.net/<challenge-path>/flag.txt
```

(Use whichever exact download link the challenge page provides next to "Download flag.txt" — it's unique per instance.)

### Step 2 — Read it

```
┌──(zham㉿kali)-[~]
└─$ cat flag.txt
picoCTF{c0ngr4ts_0n_y0ur_s4n1ty}
```

That's the whole challenge.

---

## Alternative Method — Skip the terminal entirely

Since the file is provided as a direct download link on the challenge page itself, you can just click it in your browser and open the downloaded `flag.txt` in any text editor or even your browser's own file preview — no terminal, `wget`, or shell access required at all. The hint specifically frames `wget` as what you'd use *if all you had was a shell* — implying a normal browser download works just as well.

---

## What Happened Internally

```
Timeline:
1. Challenge provides a direct download link to flag.txt
2. wget (or a plain browser download) retrieves the file over HTTP(S)
3. cat (or any text viewer) displays its contents
4. The flag was never encoded, encrypted, or hidden — confirming
   internet/file-retrieval tooling works correctly
```

---

## Tools Used

| Tool | Purpose | Level |
|---|---|---|
| `wget` | Download the file from its URL via the command line | Easy |
| `cat` | Display the plain-text flag | Easy |
| Browser download (alternative) | Same result with no shell access needed | Easy |

---

## Key Takeaways

- **Not every challenge has a "trick"** — some exist purely to confirm your environment and tools are working before you move on to harder problems
- **`wget <url>` is the standard way to pull a file into a shell with no browser available** — a core skill that comes up constantly in later, harder challenges
- **"In-the-clear" is a precise term, not just a hint** — it tells you directly there's no decoding step coming, so don't waste time looking for one
- The flag `c0ngr4ts_0n_y0ur_s4n1ty` reads as "congrats on your sanity" — a literal, friendly confirmation that this sanity-check challenge passed exactly as intended
