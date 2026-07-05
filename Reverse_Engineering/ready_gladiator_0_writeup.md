# Ready Gladiator 0 — picoCTF Writeup

**Challenge:** Ready Gladiator 0  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{h3r0_t0_z3r0_4m1r1gh7_f1e207c4}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Can you make a CoreWars warrior that always loses, no ties?
>
> Your opponent is the Imp. The source is available [here]. If you wanted to pit the Imp against himself, you could download the Imp and connect to the CoreWars server like this:
>
> `nc saturn.picoctf.net 57187 < imp.red`

## Hints

> 1. CoreWars is a well-established game with a lot of docs and strategy
> 2. Experiment with input to the CoreWars handler or create a self-defeating bot

---

## Background Knowledge (Read This First!)

### What is CoreWars?

CoreWars is a programming game invented by A. K. Dewdney in 1984. Two small programs, called **warriors**, run inside a shared memory called the **core**. The core is a fixed-size circular array (on picoCTF's server it is 8000 cells) and every memory address wraps around.

Each cycle both warriors execute exactly **one** instruction. The last warrior that is still alive wins the round. A warrior "dies" the moment one of its processes tries to execute a `DAT` instruction. The whole point of the game is to write a warrior in an assembly-like language called **Redcode** that outlasts the opponent.

### What is Redcode?

Redcode is the assembly language CoreWars uses. For this challenge we only need two instructions:

- `MOV src, dst` — copy the value at `src` into `dst`. PC advances by 1.
- `DAT x, y` — data. If a warrior ever **executes** a `DAT`, that process dies instantly.

### The Imp

The Imp is the smallest possible warrior. It is one instruction long:

```
mov 0, 1
```

That is "copy the current cell into the cell one ahead, then advance PC by 1". So the Imp keeps sliding a copy of itself one cell forward, forever. As long as nothing disturbs it, the Imp never dies.

### Why this challenge is the opposite of Ready Gladiator 1 and 2

The three "Ready Gladiator" challenges are a trilogy that walk you through the win condition spectrum:

- **Ready Gladiator 0** (100 pts) — make your warrior **always lose**, with no ties.
- **Ready Gladiator 1** (200 pts) — make your warrior **win at least once** out of 100 rounds.
- **Ready Gladiator 2** (400 pts) — make your warrior **win every round**.

Same opponent (the Imp), same server protocol, completely different objective. The trick to challenge 0 is to write a warrior that dies on the very first cycle.

### How to make a warrior that always loses

The server loads our warrior starting at PC=0. The very first thing it does is execute whatever instruction lives at cell 0. If that instruction is a `DAT`, our process dies instantly, the Imp lives forever, and we lose the round 100 out of 100 times.

The simplest possible "self-defeating bot" is therefore a single `DAT`:

```redcode
;redcode
;name Self DAT
;assert 1
dat 0, 1
end
```

The `dat 0, 1` does not actually matter — the `1` could be anything, even `0`. The point is that the server tries to execute it and our process is gone before the Imp even takes its first step.

---

## Solution — Step by Step

### Step 1 — Read the Imp

The challenge gives us the Imp source. I copied it into a workspace folder so I could look at it without poking the server:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-0]
└─$ cat imp.red
;redcode
;name Imp Ex
;assert 1
mov 0, 1
end
```

One instruction, no defensive logic. The Imp is going to live forever unless we deliberately make our own warrior die.

### Step 2 — Build the Self-Defeating Warrior

The hint suggests either "experiment with input" or "create a self-defeating bot". The second option is easier: just write a warrior that crashes on cycle 1. A single `DAT` does that.

I opened `nano` and pasted the shortest possible warrior:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-0]
└─$ nano warrior.red
```

In `nano` I pasted this and saved with `Ctrl+O`, `Enter`, `Ctrl+X`:

```redcode
;redcode
;name Self DAT
;assert 1
dat 0, 1
end
```

Confirm what I wrote:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-0]
└─$ cat warrior.red
;redcode
;name Self DAT
;assert 1
dat 0, 1
end
```

### Step 3 — Send It to the Server

The server is a plain TCP service. The challenge's recommended command is `nc saturn.picoctf.net 57187 < imp.red`, but I am sending my own self-defeating warrior instead of the Imp. The server reads our warrior from stdin until it sees the literal line `end`.

I do not have `netcat` installed in my sandbox, so I used a tiny inline `python3` script that does the exact same thing:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-0]
└─$ python3 -c "
import socket
s = socket.create_connection(('saturn.picoctf.net', 57187), timeout=15)
s.settimeout(15)
s.recv(4096)
s.sendall(open('warrior.red','rb').read() + b'\nend\n')
data = b''
while True:
    chunk = s.recv(4096)
    if not chunk: break
    data += chunk
    if b'did it' in data or b'Try again' in data:
        try:
            s.settimeout(1.5); more = s.recv(4096)
            if more: data += more
        except Exception: pass
        break
print(data.decode('latin-1', errors='replace'))"
;redcode
;name Self DAT
;assert 1
dat 0, 1
end
end
Warrior1:
;redcode
;name Self DAT
;assert 1
dat 0, 1
end

Rounds: 100
Warrior 1 wins: 0
Warrior 2 wins: 100
Ties: 0
You did it!
picoCTF{h3r0_t0_z3r0_4m1r1gh7_f1e207c4}
```

0 wins, 0 ties, 100 losses for us, 100 wins for the Imp. Exactly what the challenge asked for. The server printed the flag straight back:

```
picoCTF{h3r0_t0_z3r0_4m1r1gh7_f1e207c4}
```

---

## Alternative — Empty Submission (Just `end`)

The challenge hint also mentions "experiment with input to the CoreWars handler". If you send just the trailing `end` line and no warrior at all, the server has nothing to load for Warrior 1. From the Imp's perspective, Warrior 1 just never existed, so the Imp is the last process standing and Warrior 1 loses by default. I tried this on a fresh connection to confirm:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-0]
└─$ python3 -c "
import socket
s = socket.create_connection(('saturn.picoctf.net', 57187), timeout=15)
s.settimeout(15)
s.recv(4096)
s.sendall(b'end\n')
data = b''
while True:
    chunk = s.recv(4096)
    if not chunk: break
    data += chunk
    if b'did it' in data or b'Try again' in data or b'wins:' in data:
        try:
            s.settimeout(1.5); more = s.recv(4096)
            if more: data += more
        except Exception: pass
        break
print(data.decode('latin-1', errors='replace'))"
end
Warrior1:

Rounds: 100
Warrior 1 wins: 0
Warrior 2 wins: 100
Ties: 0
You did it!
picoCTF{h3r0_t0_z3r0_4m1r1gh7_f1e207c4}
```

Same flag, same result. The empty-submission trick is even shorter than the `dat 0, 1` warrior, and it works because the server does not require a warrior to be present. Either approach is valid; the explicit `DAT` warrior is what the hint calls a "self-defeating bot", and the empty submission is the "experiment with input" path.

## Alternative — `mov 0, 0` (a non-Imp that also loses)

Another "self-defeating bot" variant is `mov 0, 0`. That instruction copies the current cell onto itself, so it is a no-op that keeps our PC at the same cell forever. While we are stuck looping on `mov 0, 0`, the Imp spirals forward and never dies. Eventually the round limit hits and the server calls it a tie. Ties are NOT allowed for this challenge, so this one fails on the second condition even though we never "win":

```redcode
;redcode
;name Tie Bot
;assert 1
mov 0, 0
end
```

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-0]
└─$ python3 -c "...same snippet, swapping warrior.red for mov00.red..."
Warrior 1 wins: 0
Warrior 2 wins: 0
Ties: 100
Try again. Your warrior (warrior 1) must lose 100 times with no ties.
```

Exactly as predicted. This is a good way to confirm the rule: the server really does check that there are zero ties, not just zero wins.

---

## What Happened Internally (Timeline)

1. I sent my self-defeating warrior to `saturn.picoctf.net:57187`. The server echoed it back as "Warrior 1" and paired it against the Imp as "Warrior 2".
2. Both warriors were loaded into the same 8000-cell circular core with their PCs set to 0.
3. Cycle 1, Warrior 1: the server tried to execute the instruction at PC=0, which was `dat 0, 1`. Because `DAT` is the universal "kill" instruction, our process died immediately. PC became undefined (the process is gone).
4. Cycle 1, Warrior 2: the Imp executed `mov 0, 1`, copied itself from cell 0 into cell 1, and advanced to cell 1.
5. Cycles 2 through N: Warrior 1 had no live process, so nothing happened on our side. The Imp just kept spiraling forward forever, exactly the way it does when nothing is fighting back.
6. The server hit the round limit (some fixed number of cycles). With Warrior 1 dead and Warrior 2 alive, Warrior 2 (the Imp) was declared the winner of that round.
7. The server then reset both warriors and ran another round. Because our warrior was still `dat 0, 1`, the same thing happened: we died on cycle 1, the Imp lived forever. 100 times in a row.
8. With 0 wins for us, 0 ties, and 100 wins for the Imp, the server printed the flag and closed the connection.

The whole "attack" boils down to: the only winning move is to deliberately lose, and the cleanest way to lose is to commit suicide on cycle 1 by handing the server a `DAT` as our very first instruction.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the leaked Imp source |
| `nano` | Write `warrior.red` (the single `dat 0, 1` line) |
| `python3` (stdlib `socket`) | Talk to the picoCTF server because `netcat` was not installed in the sandbox |
| `nc` (the hint's recommended command) | The same thing — pipe a `.red` file (or just `end`) into the server |
| The picoCTF CoreWars server at `saturn.picoctf.net:57187` | The actual opponent; runs 100 rounds of Redcode and emits the flag |

---

## Key Takeaways

- **Read the win condition carefully.** "Always loses, no ties" is the opposite of every other CoreWars challenge. Most CTF players' first instinct is to try to win, which is the wrong move here.
- **`DAT` is the kill instruction in Redcode.** Anything that the PC ever points at and tries to execute as a `DAT` is a dead process. This is useful offensively (in the Dwarf from Ready Gladiator 1) but it is also useful defensively — for *yourself*, when the goal is to die as fast as possible.
- **The server checks for ties.** A `mov 0, 0` warrior technically "loses" (0 wins) but it also produces 100 ties, which the server rejects. The flag requires 0 ties, not just 0 wins.
- **An empty submission works.** Sending just `end` with no warrior is functionally identical to submitting a `DAT`. The server treats an empty Warrior 1 as "no live process", which counts as a loss. This is the "experiment with input" hint in action.
- **`nc` is just `python3 -c "import socket; ..."` if you ever need a substitute.** That fallback came in handy here because my sandbox did not have netcat installed.
- **The trilogy teaches you to invert the goal.** Challenge 0 wants you to lose, Challenge 1 wants you to barely win, Challenge 2 wants you to deterministically win. Same opponent, three different win conditions. Pay attention to the description.

### Flag Wordplay Decode

The flag is:

```
picoCTF{h3r0_t0_z3r0_4m1r1gh7_f1e207c4}
```

Decoded, it reads:

> **hero to zero amiright** + suffix `f1e207c4`

with the usual leet substitutions:

- `h3r0` — "hero" with `3` → `e`
- `t0` — "to" with `0` → `o`
- `z3r0` — "zero" with `3` → `e`
- `4m1r1gh7` — "amiright" with `4` → `a`, `1` → `i`, `7` → `t`
- `f1e207c4` — a unique 8-character hex suffix per flag instance, not part of the wordplay

The flag is a tongue-in-cheek nod to the challenge: we went from being a **hero** (the warrior entering the arena) to being **zero** (dead on cycle 1), and the flag author is joking "amiright?" about how absurdly easy the challenge is once you realise the goal is to lose. The challenge name "Ready Gladiator 0" plays the same joke — the gladiator is ready, but to immediately fall over.
