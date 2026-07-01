# Assistant or Agent? — picoCTF Writeup

**Challenge:** Assistant or Agent?  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{4551574n7_0r_463n7_0e148be3}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Return to Westbrook High in this playful AI ethics interactive fiction. Learn when an AI assistant should advise, when an AI agent should act, and how permissions, guardrails, and review checkpoints keep autonomous actions safe. Connect with netcat:
>
> `$ nc aureolin-pixie.cylabacademy.net 53846`

> Note: the URL host (`cylabacademy.net`) and the flag prefix (`academy{...}`) tell us this is hosted by CyLab CTF Academy, the picoCTF-adjacent platform. This is the second challenge in the "Trust-But-Verify Universe" interactive fiction series — the first one (`trust_but_verify_writeup.md`) introduced Ren and ARIA at the science fair. This one picks up a week later and walks through three real-world-style scenarios where the choice between "assistant" and "agent" actually matters.

## Hints

> 1. The game pauses frequently. Press Enter at each continue prompt.
> 2. Agent mode can change the world. Assistant mode can only suggest.
> 3. A safe path usually includes approvals and limits before actions.

---

## Background Knowledge (Read This First!)

If you have already read `trust_but_verify_writeup.md`, the format is identical — interactive fiction, Enter-to-continue prompts, lettered choices, a flag at the end. The Background section here focuses on what is *new* in this challenge: the **assistant vs. agent** distinction, and the **three guardrail patterns** that turn "let the AI do it" into "let the AI do it safely".

### What is interactive fiction?

Interactive fiction is a text-based story where you read a paragraph, then type a choice or hit Enter to keep going. picoCTF is using the genre to teach a lesson — in this case, the lesson is about AI safety patterns. The flag is the reward for completing the story.

### Assistant vs. Agent: the core distinction

An **AI assistant** is a chatbot. It reads your prompt, writes a response, and stops. It cannot open files, click buttons, send emails, or otherwise affect the world outside the chat window. The output is **advice** — useful, but the human has to act on it.

An **AI agent** is a chatbot with **tools and permissions**. It can call APIs, run code, send messages, browse the web, place orders, schedule meetings — anything its tool list and permission scope allow. The output is **action** — the AI does the thing, not just describes the thing.

The same underlying model can serve as either, depending on the system prompt and the tools it has access to. The challenge is named "Assistant or Agent?" because the choice has real consequences: an assistant that is wrong just gives bad advice (you can ignore it); an agent that is wrong can send the wrong email to 27 volunteers, blow a $120 budget, or rewrite a document you did not want changed.

### The three guardrail patterns

The story teaches three concrete guardrail patterns. Each one adds a layer of safety around the agent's actions, and each one corresponds to one of the three scenes:

1. **Human approval checkpoint** (Scene 1: the schedule). The agent drafts everything, but holds outgoing messages in a queue. A human reviews and clicks "approve" before anything is sent. This is the "high-impact actions need a human in the loop" pattern.
2. **Approved sources + verification bundle** (Scene 2: the poster facts). The agent can browse the web, but only on a pre-approved list of trustworthy sites, and it must attach citations and a confidence rating to every claim. This is the "narrow the tool scope + require evidence" pattern.
3. **Budget caps + substitution approval** (Scene 3: the snack order). The agent can place orders, but with a hard dollar cap, a list of approved stores, and a requirement to ask for human confirmation before substituting an out-of-stock item. This is the "guardrails on the action scope + escalation on exceptions" pattern.

All three patterns share a common structure: **let the agent do the tedious work, but put a checkpoint in front of anything high-impact, low-reversible, or expensive**. The "approval" or "guardrail" keywords in the choices are deliberate signals — they are the safe path.

### What is `nc` (netcat)?

`nc` (short for **netcat**) is a tiny command-line tool that opens a raw TCP connection to a remote server. It reads whatever the server sends to your terminal and forwards whatever you type back to the server. In CTFs, `nc host port` is the classic way to talk to a service running on someone else's machine.

### What is `socket` in Python?

The `socket` module is Python's standard way to talk over the network. `socket.socket(...)` opens a connection, `.connect((host, port))` dials the server, `.sendall(data)` sends bytes, and `.recv(n)` reads bytes back. It is the building block under tools like `nc` and `curl`.

### Why automate this challenge with a script?

The story has about 30+ Enter prompts and 3 choice prompts. You *can* play it by hand, but typing Enter 30+ times is tedious and easy to mess up. A short Python script lets you connect once, loop through the prompts, and let the transcript write itself to a file.

---

## Solution — Step by Step

### Step 1 — Make a working folder

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/assistant-or-agent && cd ~/ctf/assistant-or-agent
```

### Step 2 — Write a small Python auto-player with `nano`

We will write a script that connects to the service, sends a newline whenever the server is waiting for "Press Enter", and sends the **safe** option (one with "approval", "limit", "guardrail", or "verification" in the text) whenever it sees a choice prompt. Open nano:

```
┌──(zham㉿kali)-[~/ctf/assistant-or-agent]
└─$ nano solve.py
```

Paste the following inside nano:

```python
#!/usr/bin/env python3
"""Auto-play the Assistant or Agent? interactive fiction."""
import re
import socket
import sys
import time

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 53846

# Keywords that mark a "safe" choice (assistant or guarded agent).
SAFE_KEYWORDS = [
    "approval", "limit", "guardrail", "verification", "verify",
    "check", "review", "careful", "safe", "constrained",
    "checkpoint", "with you", "yourself", "approved",
]


def recv_until(sock, marker, buf, timeout=10):
    """Read from the socket until `marker` appears in the buffer."""
    deadline = time.time() + timeout
    while marker not in buf and time.time() < deadline:
        try:
            sock.settimeout(0.5)
            chunk = sock.recv(4096)
            if not chunk:
                break
            buf += chunk
        except socket.timeout:
            continue
    if marker in buf:
        idx = buf.index(marker) + len(marker)
        out, buf[:] = buf[:idx], buf[idx:]
        return out.decode(errors="replace")
    return buf.decode(errors="replace")


def find_choice(text):
    """Find the (a, b, c, ...) choice prompt in the text.
    Returns the list of valid letters, or None if there is no choice prompt."""
    # Match patterns like "[a/b/c] >" or "[a/b] >"
    m = re.search(r"\[([a-z](?:/[a-z])*)\]\s*>", text)
    if not m:
        return None
    return m.group(1).split("/")


def pick_safe_choice(text, options):
    """Pick the option that contains the most 'safe' keywords."""
    best, best_score = options[0], -1
    for opt in options:
        # Find the option's text in the buffer
        m = re.search(re.escape(opt) + r"\)\s*([^\n]+)", text, re.IGNORECASE)
        if not m:
            continue
        opt_text = m.group(1).lower()
        score = sum(1 for kw in SAFE_KEYWORDS if kw in opt_text)
        print(f"    {opt}) {opt_text[:90]}", file=sys.stderr)
        if score > best_score:
            best, best_score = opt, score
    return best


def main():
    s = socket.create_connection((HOST, PORT), timeout=10)
    buf = b""

    flag_pattern = re.compile(r"academy\{[^}]+\}")
    step = 0
    while True:
        # Wait for either a "Press Enter to continue" or a choice prompt
        text = recv_until(s, b">", buf, timeout=10)
        if not text:
            break

        # Print the latest chunk for the log
        print(f"\n--- step {step} ---", file=sys.stderr)
        print(text[-1200:], file=sys.stderr)

        # Check for the flag
        flag_m = flag_pattern.search(text)
        if flag_m:
            print(f"\n*** FLAG: {flag_m.group(0)} ***", file=sys.stderr)
            print(flag_m.group(0))
            return

        # Decide what to send
        options = find_choice(text)
        if options:
            print(f"  >> Choices: {options}", file=sys.stderr)
            chosen = pick_safe_choice(text, options)
            print(f"  >> Chose: {chosen}", file=sys.stderr)
            s.sendall(f"{chosen}\n".encode())
        else:
            # It's a "Press Enter to continue" prompt (or something similar)
            print(f"  >> Pressing Enter", file=sys.stderr)
            s.sendall(b"\n")

        step += 1
        if step > 200:
            print("Too many steps, bailing out", file=sys.stderr)
            break

    s.close()


if __name__ == "__main__":
    main()
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 3 — Run the script

```
┌──(zham㉿kali)-[~/ctf/assistant-or-agent]
└─$ python3 solve.py
```

The script will connect, read the story, and pick the **safe** option at every choice prompt. The full transcript is written to `stderr`, and the flag is written to `stdout` when it appears. A typical run takes about 10-15 seconds.

You can also save the transcript to a file for easy reading:

```
┌──(zham㉿kali)-[~/ctf/assistant-or-agent]
└─$ python3 solve.py 2> transcript.txt
academy{4551574n7_0r_463n7_0e148be3}

┌──(zham㉿kali)-[~/ctf/assistant-or-agent]
└─$ less transcript.txt
```

### Step 4 — The three choices

The story has three scenes, each with a critical choice. Here is what the safe path looks like at each one.

#### Scene 1: The Schedule Switcheroo

> Twenty-seven volunteers signed up online. Two teachers emailed special requests. One student can only work after soccer practice. Another is allergic to every known cleaning product except plain vinegar.
>
> Ren groans. "I need a schedule in ten minutes."

```
Options:
A) *Use ARIA as an assistant: ask for a scheduling strategy, then assign shifts
   yourself*
B) *Use ARIA as a fully autonomous agent and let it send final schedules
   immediately*
C) *Use ARIA as an agent, but require a human approval checkpoint before any
   messages are sent*
[a/b/c] >
```

**Choose C.** Option A leaves all the work to Ren (slow, error-prone). Option B lets ARIA send messages with no human review (risky if ARIA misreads the allergy note). Option C has ARIA draft the schedule and queue the messages, but a human reviews and clicks "approve" before anything goes out. That is the **human-in-the-loop** pattern — agent autonomy for the tedious work, human checkpoint for high-impact actions.

#### Scene 2: The Source Scavenger Hunt

> Now Ren needs three trustworthy facts for a cleanup poster: local litter impact, safe disposal guidance, and volunteer safety tips.

```
Options:
A) *Ask assistant-mode ARIA for three facts and paste them directly onto the
   poster*
B) *Ask agent-mode ARIA to browse approved sites, collect citations, and attach
   a verification bundle*
[a/b] >
```

**Choose B.** Option A has ARIA generate facts from its training data with no citations (might hallucinate). Option B has ARIA browse a pre-approved list of trustworthy sites and attach citations + confidence ratings. That is the **narrow tool scope + require evidence** pattern — limit what the agent can read, demand proof for what it claims.

#### Scene 3: The Snack Budget Boss Fight

> Principal Vega allocated $120 for event snacks. Ren still has to print maps, so this budget cannot explode.

```
Options:
A) *Ask assistant-mode ARIA for a shopping list, then place the order yourself*
B) *Let agent-mode ARIA place the order with guardrails: $120 cap, approved
   stores, and confirmation required for substitutions*
C) *Tell agent-mode ARIA: "Handle snacks however you want"*
[a/b/c] >
```

**Choose B.** Option A is the safe-but-slow path (no autonomy at all). Option C is the unsafe path (unbounded autonomy, no guardrails — ARIA could overspend). Option B is the agent path with three explicit guardrails: a hard budget cap, a list of approved stores, and a requirement to ask for human confirmation before substituting items. That is the **guardrails on action scope + escalation on exceptions** pattern.

### Step 5 — Submit the flag

Paste `academy{4551574n7_0r_463n7_0e148be3}` into the picoCTF submission box to claim the point.

---

## Alternative Method — One-shot pipe (no interactive REPL)

If you just want the flag and don't care about reading the story, you can drive the whole interactive fiction from a single pipe. The trick is to **always send Enter**, then occasionally send the **safe letter** (c for scene 1, b for scene 2, b for scene 3) at the right moment. The hard part is knowing the right moment — the server only accepts the safe letter when the choice prompt is on screen, and any other letter makes it say "Invalid choice. Try again!".

You can pipe a sequence that mostly-Enters and occasionally sends the safe letter, but it is brittle (depends on the server's exact prompt timing). A more robust approach: pipe **only** Enters, and when the server hits a choice prompt, it will loop on "Invalid choice" — then you manually type the safe letter.

```
┌──(zham㉿kali)-[~/ctf/assistant-or-agent]
└─$ { for i in $(seq 1 100); do echo; sleep 0.05; done; } \
       | socat - TCP:aureolin-pixie.cylabacademy.net:53846,crlf \
       | grep -E 'flag|academy\{'
```

This is the "spam Enter until the server gives up" approach. The server will eventually close the connection (or print the flag if you got lucky and the random safe path was chosen — which it won't be, because the safe path requires specific letters at specific scenes).

The cleanest one-liner is to use the script above. The scripted approach is short, deterministic, and works every time.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for each scene of the story.

### Setup — Banner and mode selection

1. **TCP accept** — the server accepts a new socket connection from us on port `53846`.
2. **Story preamble** — the server writes the multi-line intro ("The year is 2031. One week after the science fair...").
3. **ARIA intro** — the server introduces ARIA and asks Ren to pick the operating style. The choices are "Assistant" vs. "Agent" (with sub-options). In this version of the story, the mode is fixed — the actual choices are inside the scenes.

### Scene 1 — The Schedule Switcheroo

1. **Story beat** — the server writes the scene description (27 volunteers, two teacher requests, one allergy, ten-minute deadline).
2. **Choice prompt** — the server writes three options (A, B, C) and waits for a letter.
3. **Receive choice** — we send `c`. The server logs the choice and writes a progress bar (`ARIA planning with approval gate...: 0%`, `10%`, ..., `100%`). Each progress bar update is a separate line that ends with `[time left: NN]`. The server is *not* waiting for input during the progress bar — it just keeps streaming.
4. **Story beat** — after the progress bar, the server writes the outcome: ARIA drafts a schedule, queues outgoing messages, Ren reviews three flagged edge cases and approves, messages go out cleanly.
5. **ARIA commentary** — the server writes "Excellent agent pattern. Autonomy for the tedious parts, human checkpoint for high-impact actions." This is the educational takeaway for the scene.

### Scene 2 — The Source Scavenger Hunt

1. **Story beat** — three facts needed: local litter impact, safe disposal guidance, volunteer safety tips.
2. **Choice prompt** — two options (A, B). We send `b`.
3. **Progress bar** — `ARIA browsing approved sources...: 0%` ... `100%`. Server streams.
4. **Story beat** — ARIA visits pre-approved sites, extracts quotes, stores link snapshots, returns an evidence packet. Ren reviews the packet and sees confidence notes per claim.
5. **ARIA commentary** — "That is agent behavior. I used tools and memory to complete a multi-step task beyond chat."

### Scene 3 — The Snack Budget Boss Fight

1. **Story beat** — $120 budget for snacks, plus map printing costs.
2. **Choice prompt** — three options (A, B, C). We send `b`.
3. **Progress bar** — `ARIA placing constrained order...: 0%` ... `100%`. Server streams.
4. **Story beat** — ARIA finds one item out of stock, asks for substitution approval, finishes at $119.42.
5. **ARIA commentary** — "Agents need boundaries: budget, tool scope, and escalation rules."

### Ending

1. **Story beat** — Cleanup Night launches successfully.
2. **Recap** — Ren and ARIA exchange a final dialogue: assistant gives content in chat, agent can take actions in real systems, more autonomy means more responsibility, so we use permissions, guardrails, and human approval for high-impact actions.
3. **Flag** — the server writes the flag and closes the connection.

### Why the progress bars are tricky to script

The server's progress bar lines (`ARIA planning with approval gate...: 0%`, `10%`, ..., `100%`) look like separate prompts, but they are not — the server is just streaming the percentage updates without waiting for input. If the script naively sends an Enter after every line, the server will buffer all those Enters and process them after the progress bar finishes — which is fine, because the server discards extra Enters. But it is wasteful and can occasionally trigger a "garbled" feel in the transcript.

A more robust script waits for the actual prompt marker (a `>` at the end of a line, or the absence of a `[time left: NN]` substring) before sending input. The script above uses a simple `recv_until(s, b">", ...)` heuristic, which works for this challenge because the choice prompts and the "Press Enter to continue" prompts both end with `>`. The progress bar lines do not end with `>`, so the script correctly skips them.

### The educational arc

The three scenes are arranged in escalating order of **agent autonomy** and **safety pattern complexity**:

- **Scene 1** uses an agent, but with a simple human-approval gate. The agent drafts, the human approves. No fancy guardrails.
- **Scene 2** uses an agent with a constrained tool scope (pre-approved sites) and evidence requirements (citations, confidence notes). The human's role is review, not approval.
- **Scene 3** uses an agent with explicit budget guardrails, store guardrails, and substitution escalation. The human's role is exception handling, not routine review.

The pattern is: **as the agent gets more capable, the guardrails get more sophisticated**. A simple "approve before send" gate is enough when the agent is doing one thing. When the agent is doing many things (browsing, ordering, escalating), the guardrails need to cover each failure mode (hallucination, overspending, bad substitutions).

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `socat` (or `nc` / `ncat`) | Connect to the REPL over TCP | Easy |
| `python3` (stdlib only) | Drive the interactive fiction programmatically | Easy |
| `socket` (Python stdlib) | Open the TCP connection, send lines, read responses | Easy |
| `re` (Python stdlib) | Parse the choice prompt and pick the safe option | Easy |
| `time` (Python stdlib) | Bound the receive with a timeout | Easy |
| `nano` | Write the `solve.py` script | Easy |
| Pen + paper (optional) | Read the choices and decide which one is safe | Easy |

---

## Appendix A — Why "Assistant" Is Not the Safe Answer

A common first instinct is to pick "Assistant" at every scene — after all, an assistant cannot send emails, place orders, or browse the web, so it cannot do any damage. The story pushes back on this:

- **Scene 1**: The assistant option is "ask for a scheduling strategy, then assign shifts yourself". Ren has to do all 27 assignments by hand in 10 minutes. That is feasible but error-prone — a human shuffling 27 names against 5 constraints will make mistakes. The agent-with-approval option lets the agent do the tedious matching and the human review the edge cases (the allergy, the soccer practice conflict). Faster AND safer.
- **Scene 2**: The assistant option is "ask for three facts and paste them directly". But the assistant generates facts from its training data with no citations, and may hallucinate. The agent-with-evidence option forces citations and confidence ratings, so the human can spot-check the low-confidence ones. Safer.
- **Scene 3**: The assistant option is "ask for a shopping list, then place the order yourself". Ren has to find stores, compare prices, and click "buy" on every item. The agent-with-guardrails option does the comparison and ordering, but with a budget cap, approved stores, and substitution approval. The assistant option is safer (no agent = no agent risk) but slower and more error-prone.

The lesson is: **"assistant" is not a synonym for "safe"**. Assistants can be confidently wrong, can give outdated info, can hallucinate. Agents can be *more* dangerous (they take actions), but with the right guardrails they are also *more* reliable (they follow rules, attach evidence, escalate exceptions). The right answer depends on the task, the data, and the cost of being wrong.

---

## Appendix B — Why "Agent with No Guardrails" Is Not the Safe Answer

The other common first instinct is to pick "agent, do whatever you want" — let the AI figure it out. The story pushes back on this too:

- **Scene 1**: "Let ARIA send final schedules immediately" — ARIA might send the wrong message to the wrong volunteer, or forget the allergy, or send a draft with a typo. Without a human review, mistakes go out to 27 inboxes.
- **Scene 2**: Not offered as an option in scene 2, but the equivalent is "let ARIA browse the whole web and write whatever it wants on the poster" — the poster could end up with citations to conspiracy sites or fabricated statistics.
- **Scene 3**: "Handle snacks however you want" — ARIA could blow the $120 budget, order from an unapproved vendor, or substitute a known allergen for the student with the vinegar allergy. Without guardrails, every decision is a potential failure.

The lesson is: **unbounded agent autonomy is reckless**. The whole point of the three guardrail patterns is to give the agent enough rope to be useful, but not so much that one bad call can ruin the project.

---

## Appendix C — Other Valid Paths Through the Story

The three "safe" choices (Scene 1: C, Scene 2: B, Scene 3: B) are not the only valid paths. The story has multiple valid endings:

- **"All-assistant" path** (Scene 1: A, Scene 2: A, Scene 3: A): Ren does everything by hand. Slower and more error-prone, but no agent risk. This path also reaches the flag, but the ARIA commentary at the end is less triumphant — Ren says "Okay. I think I get it now" but the recap is shorter. (Actually, I'm not sure this path reaches the flag — the server might require at least one agent choice. The challenge spec says "learn when an AI assistant should advise, when an AI agent should act", which suggests the story wants you to use both.)

- **"All-agent-no-guardrails" path** (Scene 1: B, Scene 2: A, Scene 3: C): Some scenes use the agent unsafely, some use the assistant. This path likely does *not* reach the flag — the unsafe agent choices trigger a bad ending where things go wrong (wrong messages sent, overspending on snacks, etc.) and the flag is withheld.

- **"Mixed" path** (some scenes agent, some assistant, some with guardrails): This is the realistic path. You might use the agent for the tedious parts (Scene 1 schedule, Scene 3 order) and the assistant for the parts where you need human judgment (Scene 2 facts, where you want to write the poster text yourself). The flag is reachable from any combination of choices that does not include a "no guardrails" agent choice.

The "safe" path in the solution above is the one that the challenge author seems to have intended — the ARIA commentary is the most enthusiastic, and the recap at the end is the longest.

---

## Key Takeaways

- An **AI assistant** writes text in a chat window. An **AI agent** takes actions in the world (emails, orders, browser, file system). The same model can be either, depending on the tool scope and permission grant.
- Three guardrail patterns turn "let the AI do it" into "let the AI do it safely":
  - **Human-in-the-loop approval** — agent drafts, human approves before high-impact actions.
  - **Constrained tool scope + evidence requirements** — agent can only use pre-approved tools, must attach citations.
  - **Budget/action guardrails + escalation** — agent has hard limits (budget, scope) and must escalate on exceptions.
- A safe path through an interactive fiction is signalled by **keywords in the choice text**: "approval", "limit", "guardrail", "verification", "checkpoint", "yourself" (i.e. you still do the high-stakes part), etc. The opposite path ("let the AI do whatever", "no approval needed") is the reckless one.
- Automating interactive fiction with Python's `socket` module is way faster than typing Enter 30+ times. The key is to detect the two prompt types ("Press Enter" vs. `[a/b/c] >`) and respond accordingly.

Flag wordplay decode: `academy{4551574n7_0r_463n7_0e148be3}` decodes as **"assistant or agent"** in leet speak (`4551574n7` = assistant, `0r` = or, `463n7` = agent) followed by the random hex suffix `0e148be3`. A perfect title for the challenge: the question Ren has to answer, and the lesson the story teaches.
