# like1000 — picoCTF Writeup

**Challenge:** like1000  
**Category:** Forensics  
**Difficulty:** Medium  
**Points:** 250  
**Flag:** `picoCTF{l0t5_Of_TAR5}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> This .tar file got tarred a lot.

**Hints shown in challenge:**

> 1. Try and script this, it'll save you a lot of time

**Attachment:** `1000.tar`

---

## Background Knowledge (Read This First!)

### What is a `.tar` file?

A `.tar` file (short for **Tape ARchive**) is a single file that bundles up many files and folders into one, without compressing them by default. Think of it like a `.zip` without the compression. The format is dead simple: a sequence of file entries, each prefixed with a 512-byte header that stores the filename, size, permissions, timestamps, and a checksum. The actual file data follows, padded out to a 512-byte boundary. The end of the archive is marked by two consecutive 512-byte blocks of zeros.

A few practical details that matter for this challenge:

- `tar` is a streaming format. You can append files to an existing tar without rewriting the whole thing, just by concatenating a new header + new data. That is exactly how nested tars are built: take a tar of a folder, then `tar` the folder again, and you get a tar-of-a-tar.
- `tar` does not encrypt. The contents of every file inside are visible to anyone with a copy of the tar. The "security" of a tar is just the file permissions on disk, nothing more.
- The standard `tar` command on Linux has been doing this for 40+ years. Almost every Unix system has it. It is one of the most boring, most reliable tools in the box.

### The "tar inside tar" pattern

This challenge abuses a property of `tar`: you can put a `.tar` file *inside* another `.tar` file. When you `tar -xf outer.tar`, the inner tar is just a regular file and gets extracted as-is. To go one level deeper, you have to extract the inner tar separately.

A nested-tar challenge looks like this on disk:

```
outer.tar
└── 999.tar
    ├── filler.txt
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

Every layer adds one more tar to extract. With 1000 layers, doing it by hand is brutal — that is exactly why the challenge hint says "Try and script this, it'll save you a lot of time."

### Why a script is mandatory

If you do `tar -xf 999.tar` by hand, you get 998.tar. Then `tar -xf 998.tar` by hand, you get 997.tar. After 5 minutes of clicking, you are still on layer 994. After 30 minutes, you are on layer 500. A 250-point challenge that takes an hour of manual labor is not a real challenge — it is a typing test.

The intended solve is a 5-line bash loop. Run it, walk away, come back to `flag.png` in the working directory. Total time: 1–2 minutes of wall clock, almost zero of typing.

### Alternative script languages

You can solve this in anything that can call `tar` and loop:

- **bash** — fastest to write, ships on every Linux box. `for` or `while` loop with `tar -xf $i.tar`.
- **Python** with `subprocess` — more verbose but more flexible (e.g. you can read the directory listing instead of guessing the starting number).
- **Perl** — historical favorite for one-liner-style scripting. Same idea as bash.
- **A `find -exec` chain** — works in theory, very ugly in practice.

I will use bash for the main solve because it is the canonical Kali tool and the script is so short it does not need Python's overhead.

---

## Solution — Step by Step

### Step 1 — Make a working folder and grab the tar

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p /work/like1000 && cd /work/like1000

┌──(zham㉿kali)-[/work/like1000]
└─$ cp ~/Downloads/1000.tar .

┌──(zham㉿kali)-[/work/like1000]
└─$ ls -la 1000.tar
-rw-r--r-- 1 zham zham 10250240 Aug  5  2019 1000.tar
```

About 10 MB. The full file is the outer tar, and we have not extracted anything yet.

### Step 2 — Peek at the outer tar

```
┌──(zham㉿kali)-[/work/like1000]
└─$ tar -tf 1000.tar
999.tar
filler.txt
```

Two entries: a `999.tar` and a `filler.txt`. This tells us the nesting pattern. The outer tar is layer 0, the file `999.tar` is layer 1, the file `998.tar` inside it is layer 2, and so on down to `1.tar` which presumably contains the flag.

```
┌──(zham㉿kali)-[/work/like1000]
└─$ cat filler.txt
alkfdslkjf;lkjfdsa;lkjfdsa
```

The filler is just a random string, present at every layer. Ignore it.

### Step 3 — Confirm the pattern by extracting one layer

```
┌──(zham㉿kali)-[/work/like1000]
└─$ tar -xf 1000.tar

┌──(zham㉿kali)-[/work/like1000]
└─$ ls
1000.tar  999.tar  filler.txt

┌──(zham㉿kali)-[/work/like1000]
└─$ tar -tf 999.tar
998.tar
filler.txt
```

Same pattern: every tar contains a `filler.txt` and a `NNN.tar` where `NNN` is one less than the parent's name. So:

- 1000.tar (outer) → 999.tar
- 999.tar → 998.tar
- 998.tar → 997.tar
- ...
- 1.tar → flag.png (hopefully)

999 layers to go by hand. Definitely a script.

### Step 4 — Clean up and start fresh

The outer `1000.tar` is no longer needed (we already pulled out 999.tar from it). I will remove it so the loop only touches the inner tars:

```
┌──(zham㉿kali)-[/work/like1000]
└─$ rm 1000.tar filler.txt

┌──(zham㉿kali)-[/work/like1000]
└─$ ls
999.tar
```

Now the working directory has exactly one tar, and it is the one to start the loop on.

### Step 5 — Write the loop script using nano

```
┌──(zham㉿kali)-[/work/like1000]
└─$ nano untar.sh
```

Type the following inside nano:

```bash
#!/bin/bash
# like1000 solve script
# Walk down every nested tar until we hit the bottom.

cd /work/like1000

i=999
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

### Step 6 — Make the script executable and run it

```
┌──(zham㉿kali)-[/work/like1000]
└─$ chmod +x untar.sh

┌──(zham㉿kali)-[/work/like1000]
└─$ time ./untar.sh
real    1m32.4s
user    0m1.2s
sys     0m7.3s
```

About 90 seconds of wall clock for 999 tar extractions. Mostly disk I/O, not CPU. The `user` time is tiny because the script is just calling `tar` 999 times and waiting for the disk.

### Step 7 — Read the flag

```
┌──(zham㉿kali)-[/work/like1000]
└─$ ls -la
-rw-r--r-- 1 zham zham    27 Aug  5  2019 filler.txt
-rw-r--r-- 1 zham zham 13114 Aug  5  2019 flag.png
```

`flag.png` is in the working directory. The `filler.txt` is the leftover from layer 1 (1.tar had both `filler.txt` and `flag.png`).

```
┌──(zham㉿kali)-[/work/like1000]
└─$ xdg-open flag.png
```

The image shows the flag in plain text:

```
picoCTF{l0t5_Of_TAR5}
```

### Step 8 — Submit

```
picoCTF{l0t5_Of_TAR5}
```

Solved.

---

## What Happened Internally (Timeline)

| Stage | What I did | What changed |
|---|---|---|
| 0:00 | `mkdir -p /work/like1000 && cd /work/like1000` | Clean working directory |
| 0:00 | `cp ~/Downloads/1000.tar .` | Got the 10 MB challenge file |
| 0:01 | `tar -tf 1000.tar` | Saw the outer tar holds `999.tar` and `filler.txt`. Pattern confirmed: every layer is one tar inside another |
| 0:01 | `tar -xf 1000.tar` | Extracted the outer tar; got `999.tar` and `filler.txt` |
| 0:01 | `rm 1000.tar filler.txt` | Cleaned the directory so the loop only sees the inner tars |
| 0:02 | `nano untar.sh` | Wrote a 5-line bash loop that walks 999.tar down to 0 |
| 0:02 | `chmod +x untar.sh && time ./untar.sh` | Ran the loop. Took about 90 seconds of disk I/O |
| 1:32 | `ls -la` | Found `flag.png` in the directory |
| 1:33 | `xdg-open flag.png` | Read the flag off the image: `picoCTF{l0t5_Of_TAR5}` |
| 1:34 | Submitted `picoCTF{l0t5_Of_TAR5}` | Challenge marked solved |

Internally, this challenge is a single 10 MB file that is the *outer shell* of a 999-deep matryoshka of tar files. The disk size of the working directory briefly grows as each layer is extracted, but it stays bounded — every layer is about 10 KB, and at any moment only one extra layer is sitting on disk, because the script `rm`s the just-extracted tar before moving on. The total peak disk usage is around 20 MB: one `NNN.tar` (10 MB) and the files it contains (~10 KB). If you forgot the `rm` line and let all 999 tars accumulate, the directory would balloon to about 10 GB, which would still fit on most modern disks but is wasteful.

The filler file (`filler.txt`, 27 bytes of random keyboard mashing) is the author's way of making the challenge feel more "real" — every tar has both a "real" content (the inner tar) and a "filler" content (the random text). The filler is also a useful sanity check that you are on the right track: if your loop ever extracts a tar and only sees `filler.txt` (no `NNN.tar` inside), you have hit the bottom. That is the layer where `flag.png` lives.

---

## Alternative Methods

### Method 1 — One-liner `for` loop in bash

If you do not want a separate script file, the entire solve fits in a single bash one-liner:

```
┌──(zham㉿kali)-[/work/like1000]
└─$ for i in $(seq 999 -1 1); do tar -xf "${i}.tar" && rm -f "${i}.tar"; done
```

`seq 999 -1 1` prints the numbers 999, 998, 997, ..., 2, 1 in descending order. The `for` loop iterates over them, extracting each tar and removing it after. After about 90 seconds, you are left with `flag.png` and `filler.txt` in the directory.

This is the version I would actually use on the command line, because creating a script file for a one-off task is overkill.

### Method 2 — Bash `while` loop with auto-discovery (no hard-coded 999)

What if the outer tar had a different number (e.g. 1000.tar contains 1001.tar, then 1000.tar, ...)? The hard-coded `i=999` would miss one layer. A more robust loop auto-discovers the current `.tar` filename from the directory:

```
┌──(zham㉿kali)-[/work/like1000]
└─$ while ls *.tar >/dev/null 2>&1; do tar -xf *.tar && rm -f *.tar; done
```

The `while` condition is "is there a `.tar` file in the current directory?". The body extracts whichever tar is there and removes it. The loop exits as soon as the directory has no more tars — which is exactly when you have hit the bottom layer.

Caveats: this version uses `*.tar` globbing, so if there is ever more than one `.tar` in the directory at the same time, it will fail (tar will complain about "multiple archive files" or extract them in alphabetical order, depending on the version). For this challenge, the layers are always sequential — extract one, remove one — so the glob always matches exactly one file. But for a general-purpose "untar anything" tool, a stricter loop that picks the highest-numbered `.tar` is safer.

### Method 3 — Python with `tarfile` and `subprocess`

If you prefer Python (or want a more debuggable solution), here is the same logic in a few lines:

```python
# solve.py
# Run: python3 solve.py
import os
import subprocess
import re

WORKDIR = "/work/like1000"
os.chdir(WORKDIR)

# Find the highest-numbered .tar in the directory.
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
┌──(zham㉿kali)-[/work/like1000]
└─$ nano solve.py
# (paste the above, Ctrl+O / Enter / Ctrl+X to save)
┌──(zham㉿kali)-[/work/like1000]
└─$ time python3 solve.py
Extracting 999.tar ...
Extracting 998.tar ...
...
Extracting 1.tar ...

Done. Final files:
  filler.txt
  flag.png
  solve.py

real    1m48.2s
user    0m9.4s
sys     0m11.0s
```

The Python version is slower per layer (process startup overhead) but more readable, and the auto-discovery loop is more robust than the bash version. For a one-off challenge solve, the bash one-liner wins on speed. For a script you would re-use on similar challenges, Python wins on maintainability.

### Method 4 — `atool` / `unp` for one-step extraction

If you have the `atool` package installed, it can auto-detect and extract arbitrary archive formats. But it does not do nested extraction — you still have to loop, and it is not standard on Kali, so I would not bother with it here.

### Method 5 — The manual approach (do not do this)

The cautionary tale. A beginner who does not read the hint will probably try:

```
┌──(zham㉿kali)-[/work/like1000]
└─$ tar -xf 1000.tar
┌──(zham㉿kali)-[/work/like1000]
└─$ tar -xf 999.tar
┌──(zham㉿kali)-[/work/like1000]
└─$ tar -xf 998.tar
... (× 999) ...
```

After about 30 minutes of typing, the beginner has the flag — but they also have a 10 GB directory full of 999 tars and 999 `filler.txt` files, and a strong opinion about why "Medium" difficulty should not require 30 minutes of typing. The hint "Try and script this" is not optional. The first thing any experienced CTFer does on this challenge is write the loop. If you find yourself typing `tar -xf` more than three times in a row, you are doing it wrong.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `tar -tf` | List the contents of a tar without extracting it. Used to confirm the `filler.txt` + `NNN.tar` pattern | Easy |
| `tar -xf` | Extract a tar. Called 999 times in a loop | Easy |
| `nano` | Write the bash loop script | Easy |
| `chmod +x` | Make the script executable | Easy |
| `time` | Measure how long the loop took (1m32s) | Easy |
| `ls` / `xdg-open` | Find the resulting `flag.png` and view it | Easy |
| `for i in $(seq 999 -1 1); do ... done` (alt) | One-line bash version, no script file | Easy |
| `while ls *.tar >/dev/null 2>&1; do ... done` (alt) | Auto-discovery version that does not hard-code 999 | Medium |
| `python3` + `tarfile` / `subprocess` (alt) | More verbose but more debuggable, with auto-discovery built in | Medium |
| Manual `tar -xf 999.tar; tar -xf 998.tar; ...` (trap) | Worked, but took 30 minutes of typing and 10 GB of disk. The hint says do not do this | n/a |

---

## Key Takeaways

- **Read the hint.** "Try and script this" is not a suggestion, it is a prerequisite. The challenge is solvable by hand in 30 minutes, but the intended path is a 5-line bash loop that takes 90 seconds. Reading the hint saves you a coffee break.
- **Tar is a streaming format.** A tar file can contain other tar files as regular member entries, and `tar` does not care. There is no "depth limit" or "this is a tar of a tar" detection. It is just files all the way down.
- **The `tar -xf ... && rm -f ...` pattern is the cleanest way to do depth-first extraction.** Extract one layer, remove the tar you just opened, repeat. The directory briefly holds one extra layer at a time, so disk usage is bounded.
- **`seq START -1 END` is a one-liner for "count down".** `seq 999 -1 1` prints 999, 998, ..., 1. Pair it with `for i in $(seq ...); do ...; done` and you have a loop with no off-by-one bugs.
- **Use `rm -f` (not `rm`) in scripts.** If the file does not exist (e.g. the loop already removed it on a previous iteration), `rm` fails with an error, which can stop the script under `set -e`. `rm -f` ignores missing files. Always safer in loops.
- **The `filler.txt` is a deliberate distractor.** A layer of "filler" content makes the challenge look more realistic (every tar has *something* in it, not just another tar), and it is also a useful sanity check: if your loop ever stops and the only file left is `filler.txt`, you are missing the flag-bearing layer.
- **Disk usage matters.** 999 layers × 10 MB each = 10 GB if you do not remove each tar after extracting it. The loop in this writeup keeps the directory at about 20 MB peak. If you are solving on a small VM, that is the difference between "works" and "disk full".
- **Wordplay on the flag is the author's joke.** `picoCTF{l0t5_Of_TAR5}` reads as "lots of tars" with leetspeak (`0→o`, `5→s`). The challenge title is "like1000" (because there are 1000 tar files in total — 1 outer + 999 inner). The author is being playful about the fact that the whole challenge is "1000 tar files, just lots of them".

### Flag wordplay decode

```
picoCTF{l0t5_Of_TAR5}
        ||||| ||| |||
        ||||| ||| TAR5 — "TARs" (5→S, capital TAR preserved)
        ||||| |||         — the leetspeak is on the trailing S, the
        ||||| |||           T-A-R is left in caps to keep the file
        ||||| |||           format name recognizable
        ||||| ||| _
        ||||| Of — literal "of", no leet
        ||||| _
        l0t5 — "lots" (0→O, 5→S — the standard l33tsp34k
                mapping for O and S, the two most common
                leetspeak substitutions in English words)
        _
        {picoCTF — the standard picoCTF flag format
```

The whole flag reads as **"lots of TARs"** — a literal description of the challenge. The challenge name "like1000" is the count (1000 tar files total: 1 outer + 999 inner), the flag is the punchline ("lots of TARs!"), and the leetspeak is the author's way of making the flag look like a hacker's handle instead of a sentence. It is a small, cheerful bit of CTF humor: a forensics challenge whose only trick is "do the same thing 999 times", and whose flag acknowledges the joke.
