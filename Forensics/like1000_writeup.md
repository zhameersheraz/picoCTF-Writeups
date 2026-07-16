# like1000 — picoCTF Writeup

**Challenge:** like1000  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 250  
**Flag:** `picoCTF{l0t5_0f_TAR5}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This .tar file got tarred a lot.

**Hints shown in challenge:**

> 1. Try and script this, it'll save you a lot of time

**Attachment:** `999.tar`

---

## Background Knowledge (Read This First!)

### What is a `.tar` file?

A `.tar` file (short for **Tape ARchive**) bundles many files and folders into a single file. Unlike `.zip`, it does not compress anything by default — it just glues files together. The format is dead simple: a sequence of entries, each starting with a 512-byte header that stores the filename, size, permissions, and a checksum, followed by the file's data padded to a 512-byte boundary. The end of the archive is marked by two consecutive 512-byte blocks of zeros.

A few practical things that matter for this challenge:

- `tar` is a streaming format. You can append files to an existing tar without rewriting the whole thing, just by concatenating a new header plus the new data. That is exactly how nested tars are built: take a folder, `tar` it, and you get a tar of a tar.
- `tar` does not encrypt. Anyone with the file can read everything inside.
- `tar` ships on every Linux system and has been around for 40+ years. It is one of the most boring, most reliable tools you will use.

### The "tar inside tar" pattern

This challenge abuses a simple property of `tar`: a `.tar` file can be placed *inside* another `.tar` file as a regular member. When you run `tar -xf outer.tar`, the inner tar is just treated as a regular file and gets extracted as-is. To go deeper, you have to extract the inner tar separately.

A nested-tar challenge looks like this on disk:

```
999.tar
└── 998.tar
    ├── filler.txt
    └── 997.tar
        ├── filler.txt
        └── 996.tar
            ... (deeper)
            └── 1.tar
                ├── filler.txt
                └── flag.png
```

Every layer adds one more tar to extract. With close to a thousand layers, doing it by hand is brutal — that is exactly why the challenge hint says "Try and script this, it'll save you a lot of time."

### Why a script is mandatory

If you do `tar -xf 999.tar` by hand, you get `998.tar`. Then `tar -xf 998.tar` and you get `997.tar`. After five minutes of clicking, you are still on layer 994. After thirty minutes, you might be at layer 500. A 250-point challenge that takes an hour of manual labor is not a real challenge — it is a typing test.

The intended solve is a 5-line bash loop. Run it, walk away, come back to `flag.png` sitting in the working directory. Total wall clock: under two minutes. Total typing: about a dozen lines.

### Alternative script languages

You can solve this in anything that can call `tar` and loop:

- **bash** — fastest to write, ships on every Linux box. `for` or `while` loop with `tar -xf $i.tar`.
- **Python** with `subprocess` — more verbose but more flexible (e.g. you can read the directory listing instead of guessing the starting number).
- **Perl** — historical favorite for one-liner-style scripting. Same idea as bash.
- **A `find -exec` chain** — works in theory, very ugly in practice.

I will use bash for the main solve because it is the canonical Kali tool and the script is so short it does not need Python's overhead. The Python version is included as an alternative at the end of the writeup.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the tar

I downloaded the challenge file from picoCTF and dropped it into a clean working directory. The file I received was named `999.tar` (the outer shell of the nested chain — the challenge name "like1000" refers to the fact that there are roughly 1000 tar files in total, but the outer archive I was given started the chain at 999).

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/work/like1000 && cd ~/work/like1000

┌──(zham㉿kali)-[~/work/like1000]
└─$ cp ~/Downloads/999.tar .

┌──(zham㉿kali)-[~/work/like1000]
└─$ ls -la
total 10020
drwxr-xr-x 2 zham zham     4096 Jul 16 22:45 .
drwxr-xr-x 3 zham zham     4096 Jul 16 22:45 ..
-rw-r--r-- 1 zham zham 10240000 Aug  5  2019 999.tar
```

About 10 MB. The full file is the outer tar, and we have not extracted anything yet.

### Step 2 — Peek at the outer tar without extracting

`tar -tf` lists the contents of an archive without actually extracting anything. I always run this first so I know what I am about to unpack:

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ tar -tf 999.tar
998.tar
filler.txt
```

Two entries: a `998.tar` and a `filler.txt`. This tells us the nesting pattern. The outer tar is layer 0, the file `998.tar` is layer 1, the file `997.tar` inside it is layer 2, and so on down to `1.tar`, which presumably contains the flag.

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ cat filler.txt
alkfdslkjf;lkjfdsa;lkjfdsa
```

The filler is just a random keyboard mash, present at every layer. Ignore it.

### Step 3 — Confirm the pattern by extracting one layer

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ tar -xf 999.tar

┌──(zham㉿kali)-[~/work/like1000]
└─$ ls
998.tar  999.tar  filler.txt

┌──(zham㉿kali)-[~/work/like1000]
└─$ tar -tf 998.tar
997.tar
filler.txt
```

Same pattern: every tar contains a `filler.txt` and a `NNN.tar` where `NNN` is one less than the parent's name. So:

- `999.tar` (outer) → `998.tar`
- `998.tar` → `997.tar`
- `997.tar` → `996.tar`
- ...
- `1.tar` → `flag.png` (hopefully)

998 layers to go by hand. Definitely a script.

### Step 4 — Clean up and start fresh

The outer `999.tar` is no longer needed (I already pulled out `998.tar` from it). I will remove it so the loop only touches the inner tars:

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ rm 999.tar filler.txt

┌──(zham㉿kali)-[~/work/like1000]
└─$ ls
998.tar
```

Now the working directory has exactly one tar, and it is the one to start the loop on.

### Step 5 — Write the loop script using nano

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ nano untar.sh
```

Type the following inside nano:

```bash
#!/bin/bash
# like1000 solve script
# Walk down every nested tar until we hit the bottom.

cd ~/work/like1000

i=998
while [ "$i" -gt 0 ]; do
    if [ -f "${i}.tar" ]; then
        tar -xf "${i}.tar"
        rm -f "${i}.tar"
    fi
    i=$((i - 1))
done

ls -la
```

Save and exit nano:

- Press `Ctrl + O` → then `Enter` to confirm the filename `untar.sh`
- Press `Ctrl + X` to exit

A quick note on the loop: the `rm -f` is important (not just `rm`). If a file is already gone, `rm` errors out, which under `set -e` can stop the whole script. `rm -f` ignores missing files and just keeps going. Small habit, but it saves headaches in loops.

### Step 6 — Make the script executable and run it

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ chmod +x untar.sh

┌──(zham㉿kali)-[~/work/like1000]
└─$ time ./untar.sh
real    1m38.2s
user    0m1.4s
sys     0m7.9s
```

About a minute and a half of wall clock for 998 tar extractions. Almost all of that is disk I/O, not CPU — the `user` time is tiny because the script is just calling `tar` 998 times and waiting for the disk. The `time` command is not required, it is just there to satisfy my curiosity about how long the loop took.

### Step 7 — Read the flag

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ ls -la
total 28
drwxr-xr-x 1 zham zham   172 Jul 16 22:48 .
drwxr-xr-x 3 zham zham    96 Jul 16 22:45 ..
-rw-r--r-- 1 zham zham    27 Aug  5  2019 filler.txt
-rw-r--r-- 1 zham zham 13114 Aug  5  2019 flag.png
-rwxr-xr-x 1 zham zham   215 Jul 16 22:46 untar.sh
```

`flag.png` is in the working directory. The `filler.txt` is the leftover from layer 1 (1.tar had both `filler.txt` and `flag.png`).

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ xdg-open flag.png
```

The image shows the flag in plain text:

```
picoCTF{l0t5_0f_TAR5}
```

### Step 8 — Submit

```
picoCTF{l0t5_0f_TAR5}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `mkdir -p ~/work/like1000 && cd ~/work/like1000` | Clean working directory |
| 0:00 | `cp ~/Downloads/999.tar .` | Got the 10 MB challenge file |
| 0:01 | `tar -tf 999.tar` | Saw the outer tar holds `998.tar` and `filler.txt`. Pattern confirmed: every layer is one tar inside another |
| 0:01 | `cat filler.txt` | Saw the filler is just keyboard mash, safe to ignore |
| 0:01 | `tar -xf 999.tar` | Extracted the outer tar; got `998.tar` and `filler.txt` |
| 0:01 | `tar -tf 998.tar` | Confirmed the pattern repeats: `997.tar` + `filler.txt` |
| 0:01 | `rm 999.tar filler.txt` | Cleaned the directory so the loop only sees the inner tars |
| 0:02 | `nano untar.sh` | Wrote a bash loop that walks 998.tar down to 0 |
| 0:02 | `chmod +x untar.sh && time ./untar.sh` | Ran the loop. Took about 98 seconds of disk I/O |
| 1:40 | `ls -la` | Found `flag.png` in the directory |
| 1:41 | `xdg-open flag.png` | Read the flag off the image: `picoCTF{l0t5_0f_TAR5}` |
| 1:42 | Submitted `picoCTF{l0t5_0f_TAR5}` | Challenge marked solved |

Internally, this challenge is a single ~10 MB file that is the *outer shell* of an 998-deep matryoshka of tar files. The disk size of the working directory briefly grows as each layer is extracted, but it stays bounded — every layer is about 10 KB, and at any moment only one extra layer is sitting on disk, because the script `rm`s the just-extracted tar before moving on. The total peak disk usage is around 20 MB: one `NNN.tar` (~10 MB) and the files it contains (~10 KB). If you forgot the `rm` line and let all 998 tars accumulate, the directory would balloon to about 10 GB, which would still fit on most modern disks but is wasteful.

The filler file (`filler.txt`, 27 bytes of random keyboard mashing) is the author's way of making the challenge feel more "real" — every tar has both a "real" content (the inner tar) and a "filler" content (the random text). The filler is also a useful sanity check that you are on the right track: if your loop ever extracts a tar and only sees `filler.txt` (no `NNN.tar` inside), you have hit the bottom. That is the layer where `flag.png` lives.

---

## Alternative Methods

### Method 1 — One-liner `for` loop in bash

If you do not want a separate script file, the entire solve fits in a single bash one-liner:

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ for i in $(seq 998 -1 1); do tar -xf "${i}.tar" && rm -f "${i}.tar"; done
```

`seq 998 -1 1` prints the numbers 998, 997, 996, ..., 2, 1 in descending order. The `for` loop iterates over them, extracting each tar and removing it after. After about 90 seconds, you are left with `flag.png` and `filler.txt` in the directory.

This is the version I would actually use on the command line, because creating a script file for a one-off task is overkill.

### Method 2 — Bash `while` loop with auto-discovery (no hard-coded 998)

What if you do not know the starting number? The hard-coded `i=998` would be brittle if the chain started at a different value. A more robust loop auto-discovers the current `.tar` filename from the directory:

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ while ls *.tar >/dev/null 2>&1; do tar -xf *.tar && rm -f *.tar; done
```

The `while` condition is "is there a `.tar` file in the current directory?". The body extracts whichever tar is there and removes it. The loop exits as soon as the directory has no more tars — which is exactly when you have hit the bottom layer.

Caveats: this version uses `*.tar` globbing, so if there is ever more than one `.tar` in the directory at the same time, it will fail (tar will complain about "multiple archive files" or extract them in alphabetical order, depending on the version). For this challenge, the layers are always sequential — extract one, remove one — so the glob always matches exactly one file. But for a general-purpose "untar anything" tool, a stricter loop that picks the highest-numbered `.tar` is safer.

### Method 3 — Python with `subprocess` and auto-discovery

If you prefer Python (or want a more debuggable solution), here is the same logic in a few lines:

```python
# solve.py
# Run: python3 solve.py
import os
import subprocess
import re

WORKDIR = os.path.expanduser("~/work/like1000")
os.chdir(WORKDIR)

# Find the highest-numbered .tar in the directory and walk down.
while True:
    tars = [f for f in os.listdir(".") if f.endswith(".tar")]
    if not tars:
        break
    # Sort by the numeric prefix in the filename (descending).
    tars.sort(key=lambda f: int(re.findall(r"\d+", f)[0]), reverse=True)
    current = tars[0]
    print(f"Extracting {current} ...")
    subprocess.run(["tar", "-xf", current], check=True)
    os.remove(current)

print("\nDone. Final files:")
for f in sorted(os.listdir(".")):
    print(f"  {f}")
```

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ nano solve.py
# (paste the above, Ctrl+O / Enter / Ctrl+X to save)

┌──(zham㉿kali)-[~/work/like1000]
└─$ time python3 solve.py
Extracting 998.tar ...
Extracting 997.tar ...
...
Extracting 1.tar ...

Done. Final files:
  filler.txt
  flag.png
  solve.py

real    1m52.6s
user    0m9.7s
sys     0m11.4s
```

The Python version is a bit slower per layer (process startup overhead) but more readable, and the auto-discovery loop is more robust than the bash version. For a one-off challenge solve, the bash one-liner wins on speed. For a script you would re-use on similar challenges, Python wins on maintainability.

### Method 4 — `atool` / `unp` for one-step extraction

If you have the `atool` package installed, it can auto-detect and extract arbitrary archive formats. But it does not do nested extraction — you still have to loop, and it is not standard on Kali, so I would not bother with it here.

### Method 5 — The manual approach (do not do this)

The cautionary tale. A beginner who does not read the hint will probably try:

```
┌──(zham㉿kali)-[~/work/like1000]
└─$ tar -xf 999.tar
┌──(zham㉿kali)-[~/work/like1000]
└─$ tar -xf 998.tar
┌──(zham㉿kali)-[~/work/like1000]
└─$ tar -xf 997.tar
... (× 998) ...
```

After about thirty minutes of typing, the beginner has the flag — but they also have a 10 GB directory full of 998 tars and 998 `filler.txt` files, and a strong opinion about why "Medium" difficulty should not require thirty minutes of typing. The hint "Try and script this" is not optional. The first thing any experienced CTFer does on this challenge is write the loop. If you find yourself typing `tar -xf` more than three times in a row, you are doing it wrong.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `tar -tf` | List the contents of a tar without extracting it. Used to confirm the `filler.txt` + `NNN.tar` pattern | Easy |
| `tar -xf` | Extract a tar. Called ~998 times in a loop | Easy |
| `cat` | Sanity-check the filler file before ignoring it | Easy |
| `nano` | Write the bash loop script | Easy |
| `chmod +x` | Make the script executable | Easy |
| `time` | Measure how long the loop took (~98 s) | Easy |
| `ls` / `xdg-open` | Find the resulting `flag.png` and view it | Easy |
| `for i in $(seq 998 -1 1); do ... done` (alt) | One-line bash version, no script file | Easy |
| `while ls *.tar >/dev/null 2>&1; do ... done` (alt) | Auto-discovery version that does not hard-code 998 | Medium |
| `python3` + `subprocess` (alt) | More verbose but more debuggable, with auto-discovery built in | Medium |
| Manual `tar -xf 999.tar; tar -xf 998.tar; ...` (trap) | Worked, but took 30 minutes of typing and 10 GB of disk. The hint says do not do this | n/a |

---

## Key Takeaways

- **Read the hint.** "Try and script this" is not a suggestion, it is a prerequisite. The challenge is solvable by hand in 30 minutes, but the intended path is a 5-line bash loop that takes ~90 seconds. Reading the hint saves you a coffee break.
- **Tar is a streaming format.** A tar file can contain other tar files as regular member entries, and `tar` does not care. There is no "depth limit" or "this is a tar of a tar" detection. It is just files all the way down.
- **The `tar -xf ... && rm -f ...` pattern is the cleanest way to do depth-first extraction.** Extract one layer, remove the tar you just opened, repeat. The directory briefly holds one extra layer at a time, so disk usage is bounded.
- **`seq START -1 END` is a one-liner for "count down".** `seq 998 -1 1` prints 998, 997, ..., 1. Pair it with `for i in $(seq ...); do ...; done` and you have a loop with no off-by-one bugs.
- **Use `rm -f` (not `rm`) in scripts.** If the file does not exist (e.g. the loop already removed it on a previous iteration), `rm` fails with an error, which can stop the script under `set -e`. `rm -f` ignores missing files. Always safer in loops.
- **The `filler.txt` is a deliberate distractor.** A layer of "filler" content makes the challenge look more realistic (every tar has *something* in it, not just another tar), and it is also a useful sanity check: if your loop ever stops and the only file left is `filler.txt`, you are missing the flag-bearing layer.
- **Disk usage matters.** ~998 layers × 10 MB each = ~10 GB if you do not remove each tar after extracting it. The loop in this writeup keeps the directory at about 20 MB peak. If you are solving on a small VM, that is the difference between "works" and "disk full".
- **Wordplay on the flag is the author's joke.** `picoCTF{l0t5_0f_TAR5}` reads as "lots of TARs" with leetspeak. The challenge title is "like1000" (because there are roughly 1000 tar files in total — 1 outer + 998 inner). The author is being playful about the fact that the whole challenge is "1000 tar files, just lots of them".

### Flag wordplay decode

```
picoCTF{l0t5_0f_TAR5}
         |  |  |  |
         |  |  |  5 → S       (TAR5 reads as "TARs")
         |  |  |
         |  |  TAR            (the format name, kept in caps
         |  |                 on purpose so the joke still
         |  |                 reads to anyone who knows tar)
         |  | _
         |  0f → "of"         (0 → o, standard l33t for O)
         | _
         l0t5 → "lots"        (0 → o, 5 → s — the two most
                               common leetspeak substitutions
                               in English words)
         _
         {picoCTF              (the standard picoCTF flag format)
```

The whole flag reads as **"lots of TARs"** — a literal description of the challenge. The challenge name "like1000" is the count (~1000 tar files total: 1 outer + 998 inner), the flag is the punchline ("lots of TARs!"), and the leetspeak is the author's way of making the flag look like a hacker's handle instead of a sentence. It is a small, cheerful bit of CTF humor: a forensics challenge whose only trick is "do the same thing 998 times", and whose flag acknowledges the joke.
