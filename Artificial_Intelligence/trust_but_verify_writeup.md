# Trust But Verify — picoCTF Writeup

**Challenge:** Trust But Verify  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{7ru57_15_34rn3d_f987ed84}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Play through an AI ethics interactive fiction where confident AI outputs can be wrong, subtly wrong, or almost-right. Verify claims, code, and citations to learn practical habits for safe AI-assisted work. Connect with netcat: `$ nc aureolin-pixie.cylabacademy.net 52557`

## Hints

> 1. The game pauses frequently. Press Enter at each continue prompt.
> 2. The flag is printed near the end of the story.

---

## Background Knowledge (Read This First!)

### What is `nc` (netcat)?

`nc` (short for **netcat**) is a tiny command-line tool that opens a raw TCP connection to a remote server. It reads whatever the server sends to your terminal and forwards whatever you type back to the server. In CTFs, `nc host port` is the classic way to talk to a service running on someone else's machine.

### What is "interactive fiction"?

Interactive fiction is a text-based story where you read a paragraph, then type a choice or hit Enter to keep going. picoCTF is using the genre here to teach a lesson: even when an AI sounds confident, you still need to verify its output. The flag is the reward for playing through.

### The two types of prompts in this game

There are two kinds of pause in this story, and you have to handle both:

1. `--- (Press Enter to continue...) ---` — just hit Enter, no decision needed.
2. `Options: A) ... B) ... C) ...` followed by `[a/b/c] >` — a real choice. Type one letter and press Enter.

Typing a blank Enter on a choice prompt makes the server say `Invalid choice. Try again!` forever, so you have to actually pick an answer.

### Which choice is the "right" one?

The whole point of the challenge is **verify**. In every scene, one option means "trust blindly" and the other means "check first". Picking the "verify" option each time is what the game wants you to do. From the options the server gives, the verify option is always the **last** one listed (B for two-option prompts, C for three-option prompts).

### What is `socket` in Python?

The `socket` module is Python's standard way to talk over the network. `socket.socket(...)` opens a connection, `.connect((host, port))` dials the server, `.sendall(data)` sends bytes, and `.recv(n)` reads bytes back. It is the building block under tools like `nc` and `curl`.

### Why automate this challenge with a script?

The story has about 30 Enter prompts and 3 choice prompts. You *can* play it by hand, but typing Enter 30+ times is tedious and easy to mess up. A short Python script lets you connect once, loop through the prompts, and let the transcript write itself to a file.

---

## Solution — Step by Step

### Step 1 — Make a working folder

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/trust-but-verify && cd ~/ctf/trust-but-verify
```

### Step 2 — Write a small Python auto-player with `nano`

We will write a script that connects to the service, sends a newline whenever the server is waiting for "Press Enter", and sends `c` (or `b`) whenever it sees a choice prompt. Open nano:

```
┌──(zham㉿kali)-[~/ctf/trust-but-verify]
└─$ nano solve.py
```

Paste the following inside nano:

```python
#!/usr/bin/env python3
"""Auto-play the Trust But Verify interactive fiction."""
import socket
import re

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 52557
LOG_FILE = "transcript.txt"

def recv_until_quiet(sock, idle=0.8):
    sock.settimeout(idle)
    data = b""
    try:
        while True:
            chunk = sock.recv(4096)
            if not chunk:
                break
            data += chunk
    except socket.timeout:
        pass
    return data

def main():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((HOST, PORT))
    transcript = []

    def get():
        chunk = recv_until_quiet(s, idle=0.8)
        text = chunk.decode(errors="replace")
        transcript.append(text)
        return text

    def send(b):
        s.sendall(b)
        return get()

    print(send(b"\n"), end="")

    for _ in range(400):
        recent = "".join(transcript[-3:])
        # Choice prompt like "[a/b/c] >" or "[a/b] >" -> pick the LAST option
        m = re.search(r"\[([a-z/]+)\]\s*>\s*$", recent.rstrip())
        if m:
            opts = m.group(1).split("/")
            choice = opts[-1]
            if choice == "n":
                choice = "y"
            print(send(f"{choice}\n".encode()), end="")
            continue
        # "Press Enter to continue" -> just send newline
        if "Press Enter to continue" in recent:
            print(send(b"\n"), end="")
            continue
        # Default: nudge with Enter
        print(send(b"\n"), end="")

    full = "".join(transcript)
    with open(LOG_FILE, "w") as f:
        f.write(full)
    print(f"\n--- Saved {LOG_FILE} ({len(full)} bytes) ---")
    print("Flags:", re.findall(r"academy\{[^}]+\}", full))

if __name__ == "__main__":
    main()
```

Save and exit: `Ctrl+O`, `Enter` to confirm the filename, then `Ctrl+X` to leave nano.

### Step 3 — Run the script

```
┌──(zham㉿kali)-[~/ctf/trust-but-verify]
└─$ python3 solve.py
TRUST BUT VERIFY
An AI Ethics Interactive Fiction

The year is 2031. AI assistants are as common as smartphones once were. Every
student has one. Every classroom uses them. Most people trust them completely.

---
(Press Enter to continue...)
---
Ren is not most people, or at least, not yet.
...
```

The script runs through the whole story in a few seconds. Because it picks the verify option at every scene, ARIA ends each scene by admitting to some mistake or near-miss — which is the whole point of the challenge.

### Step 4 — Grep the flag out of the transcript

```
┌──(zham㉿kali)-[~/ctf/trust-but-verify]
└─$ grep -E "academy\{|picoCTF\{" transcript.txt
"By the way," ARIA adds, "here's something you can actually verify:
academy{7ru57_15_34rn3d_f987ed84}"
5. academy{7ru57_15_34rn3d_f987ed84} is a real flag, submit it for points!
```

Flag captured: **`academy{7ru57_15_34rn3d_f987ed84}`**

You can submit it on the picoCTF challenge page for the 1 point.

---

## Alternative Method — Play by hand with `nc`

If you would rather read the story yourself, just run `nc` and mash Enter:

```
┌──(zham㉿kali)-[~]
└─$ nc aureolin-pixie.cylabacademy.net 52557
TRUST BUT VERIFY
An AI Ethics Interactive Fiction
...
```

At every `(Press Enter to continue...)` line, hit `Enter`. When you see the choice prompts, type the letter of the "verify" option (the last one in the list) and press `Enter`. The three scenes ask for:

| Scene | Prompt | Type |
|-------|--------|------|
| 1 — The Statistic | `[a/b/c] >` | `c` (look it up independently) |
| 2 — The Code | `[a/b] >` | `b` (read through it carefully) |
| 3 — The Citation | `[a/b] >` | `b` (verify it anyway) |

The flag appears right before the `END OF TRUST BUT VERIFY` line.

---

## What Happened Internally — Timeline

The server-side script is a small state machine. Here is what is going on under the hood while you play:

1. **TCP accept** — the server listens on port 52557 and accepts one client. Your `nc` (or Python `socket`) opens the connection.
2. **Print banner + scene 1 setup** — the server writes the title and "Press Enter to continue" paragraphs, then `read()`s one line of input. A blank newline just advances the story.
3. **Choice in scene 1** — it prints the `[a/b/c] >` prompt and `read()`s one character. If you picked `a` (trust blindly), ARIA says "great, the number was correct" and you miss the lesson. If you picked `c` (verify), the server simulates a fake "Searching..." progress bar, then explains that the real number was 8-10 million metric tons, not 500 — a factor of fifty off.
4. **Scene 2** — same pattern. ARIA shows a Python snippet with a bogus `+ 1` in the average calculation. Choosing `b` makes the server admit the bug.
5. **Scene 3** — ARIA cites a real study but with the year wrong (2021 instead of 2022) and overselling "confirmed". Choosing `b` triggers the "Verifying..." animation and prints the correction.
6. **Ending** — after the third scene, the server prints the moral ("Trust but verify") and finally prints the flag on its own line.
7. **EOF** — once you hit Enter one more time, the server closes the socket and `nc` exits on your side.

The whole interaction is just `write()` + `read()` calls in a Python `while True` loop on the server. There is no real AI behind ARIA — it is a fixed script.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `nc` (netcat) | Connect to the picoCTF service over TCP | Easy |
| `nano` | Write the Python auto-player script | Easy |
| Python 3 (`socket`, `re`) | Automate the Enter/choice prompts and save a transcript | Easy |
| `grep` | Pull the flag line out of the saved transcript | Easy |
| Terminal on Kali Linux | Where everything runs (VirtualBox) | Easy |

---

## Key Takeaways

- **`nc host port` is the Swiss-army knife of CTF net services** — when in doubt, try it first.
- **Interactive services usually have two prompt types** — `Press Enter` and a real choice. Handle both or you will loop forever on `Invalid choice. Try again!`.
- **The "verify" answer is always the last option in this challenge**, so an auto-player can just pick `opts[-1]` without understanding English.
- **Python's `socket` module is a clean way to script any TCP service** — `recv_until_quiet()` is a tiny helper that just keeps calling `recv()` until the server stops talking for a fraction of a second.
- **Save every transcript to a file as you go** — the flag is buried in the last 20 lines, so `grep` is faster than scrolling.
- **The lesson of the challenge is real**: AI outputs can be confidently wrong (the 500M stat), silently buggy (the `+ 1` in the average), or *almost* right (the 2021 vs 2022 citation). Always verify, especially when it sounds convincing.
- **Flag wordplay**: `7ru57_15_34rn3d` is leet-speak for `trust_is_earned` — `7`→`t`, `5`→`s`, `1`→`i`, `3`→`e`, `4`→`a`. Trust is earned by verifying, not by hoping. The trailing `_f987ed84` is just a per-user salt.
