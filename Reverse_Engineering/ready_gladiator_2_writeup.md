# Ready Gladiator 2 — picoCTF Writeup

**Challenge:** Ready Gladiator 2   
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 400  
**Flag:** `picoCTF{d3m0n_3xpung3r_47037b25}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Can you make a CoreWars warrior that wins every single round?
>
> Your opponent is the Imp. The source is available [here]. If you wanted to pit the Imp against himself, you could download the Imp and connect to the CoreWars server like this:
>
> `nc saturn.picoctf.net 54793 < imp.red`
>
> To get the flag, you must beat the Imp all 100 rounds.

## Hints

> 1. If your warrior is close, try again, it may work on subsequent tries... why is that?

---

## Background Knowledge (Read This First!)

### What is CoreWars?

CoreWars is a programming game invented by A. K. Dewdney in 1984. Two small programs, called **warriors**, run at the same time inside a shared memory called the **core**. The core is a fixed-size circular array (on picoCTF's server it is 8000 cells) and every memory address wraps around.

Each cycle both warriors execute exactly **one** instruction, and the last warrior that is still alive wins the round. Anything that kills the other warrior's process is fair game. The whole point of the game is that you write a warrior in an assembly-like language called **Redcode** and watch it fight.

### What is Redcode?

Redcode is the assembly language CoreWars uses. The instructions you will see in this challenge are:

- `MOV src, dst` — copy the value at `src` into `dst`. PC advances by 1.
- `JMP addr` — set the program counter to `addr`. PC jumps there directly.
- `DAT x, y` — data. If a warrior ever tries to **execute** a `DAT` instruction, that process dies instantly. `DAT` is the universal "kill" instruction.

There are also a bunch of **addressing modes** that change how operands are resolved:

- `0`, `1`, `-1`, … — direct relative. `0` means "the current cell", `1` means "one cell after this one".
- `#0`, `#1`, … — immediate. The number is the value, no memory reference.
- `$5` — direct relative, addressed from the current PC.
- `<-2` — **pre-decrement indirect**. Before reading the address, decrement the cell at the relative location; then use that cell's value as an *indirect* pointer.

You can ignore most of those. For this challenge we only need direct relative (`0`, `1`, `-2`) and the pre-decrement indirect trick (`<-2`).

### The Imp

The Imp is the smallest possible warrior. It is one instruction long:

```
mov 0, 1
```

That is "copy the current cell into the cell one ahead, then advance PC by 1". So the Imp keeps sliding a copy of itself one cell forward, forever. It looks like this over time:

```
cycle 1: [mov 0, 1]  _              _              _
cycle 2: [mov 0, 1] [mov 0, 1]      _              _
cycle 3: [mov 0, 1] [mov 0, 1] [mov 0, 1]         _
cycle 4: [mov 0, 1] [mov 0, 1] [mov 0, 1] [mov 0, 1]
```

The Imp walks forward forever. As long as nothing disturbs it, it never dies. To beat it, you have to somehow make it execute a `DAT`.

### The Imp Gate (the trick we are going to use)

The classic anti-Imp weapon is the **imp gate**. The idea is brutally simple:

1. You write a single `JMP` instruction.
2. The **B-operand** of that `JMP` is a `<-N` addressing mode. Even though `JMP` only reads its A-operand, the addressing mode still fires and **decrements** whatever cell sits at PC-N.
3. The Imp spirals forward, wraps around the core, and eventually lands on the cell you are decrementing.
4. By the time the Imp arrives, that cell is no longer the clean `mov 0, 1` value the Imp needs to keep replicating, so the Imp tries to copy a corrupted instruction and dies.

The shorter the distance `N`, the faster the kill — but if `N` is too short the Imp can stall right next to the gate and never die. `N = 2` and `N = 5` both work reliably; `N = 1` ties 100 out of 100 rounds for exactly that reason (we will prove it below).

### What is a "round" here?

The server runs 100 independent fights. You win the flag only if your warrior wins all 100. Most warriors beat the Imp sometimes but lose or tie the rest. The whole challenge is finding a warrior that is **deterministically** better than the Imp.

---

## Solution — Step by Step

### Step 1 — Download and Read the Imp

The challenge gives us the Imp source. I copied it locally so I could look at it without poking the server:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-2]
└─$ cat imp.red
;redcode
;name Imp Ex
;assert 1
mov 0, 1
end
```

That is the entire Imp. One instruction, no defensive logic whatsoever. Whatever warrior I write has to be the one that lives through 100 rounds.

### Step 2 — Build the Winning Warrior (Imp Gate)

I opened `nano` and wrote a single-instruction warrior: an imp gate that decrements the cell two positions behind itself every cycle.

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-2]
└─$ nano warrior.red
```

In `nano` I pasted this and saved with `Ctrl+O`, `Enter`, `Ctrl+X`:

```redcode
;redcode
;name Imp Gate
;assert 1
jmp 0, <-2
end
```

Let me unpack the only line that matters:

- `jmp 0, <-2`
- `0` (A-operand, direct relative) — "jump to the current cell". That is an infinite loop on purpose; we never want to leave.
- `<-2` (B-operand, pre-decrement indirect) — before the jump resolves, the core silently decrements the cell at PC-2.

PC-2 is two cells behind our warrior. We are not using that value for anything; the **side effect** is what kills the Imp. Every cycle our warrior ticks, that back cell loses 1.

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-2]
└─$ cat warrior.red
;redcode
;name Imp Gate
;assert 1
jmp 0, <-2
end
```

### Step 3 — Send It to the Server

The server speaks plain TCP. The hint says `nc saturn.picoctf.net 54793 < imp.red`, but I am sending my own warrior instead of the Imp. The server reads our warrior from stdin until it sees the literal line `end`.

I do not have `netcat` installed in my sandbox, so I used a tiny inline `python3` script instead — it does the exact same thing as `nc file`:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-2]
└─$ python3 -c "
import socket
s = socket.create_connection(('saturn.picoctf.net', 54793), timeout=15)
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
;name Imp Gate
;assert 1
jmp 0, <-2
end
end
Warrior1:
;redcode
;name Imp Gate
;assert 1
jmp 0, <-2
end

Rounds: 100
Warrior 1 wins: 100
Warrior 2 wins: 0
Ties: 0
You did it!
picoCTF{d3m0n_3xpung3r_47037b25}
```

All 100 rounds won, 0 losses, 0 ties. The server printed the flag straight back:

```
picoCTF{d3m0n_3xpung3r_47037b25}
```

---

## Alternative — Dewdney's `JMP 0, <-5`

The original Dewdney / Ilmari Karonen paper uses a 5-cell offset. Same idea, slightly bigger back-step. It also wins every round:

```redcode
;redcode
;name Imp Gate v2
;assert 1
jmp 0, <-5
end
```

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-2]
└─$ python3 -c "...same snippet, swapping warrior.red for alt1.red..."
Warrior 1 wins: 100
Warrior 2 wins: 0
Ties: 0
You did it!
picoCTF{d3m0n_3xpung3r_47037b25}
```

Use `<-5` if you want to be safe on a server with a smaller core size where the Imp might wrap into the gate's decrement zone too soon.

## Alternative — The Classic Dwarf (Paper)

The other famous "always beats Imp" warrior is the **Dwarf**, a paper that drops `DAT` bombs every 4 cells:

```redcode
;redcode
;name Dwarf
;assert 1
ADD #4, 3
MOV 2, @2
JMP -2
DAT #0, #0
end
```

I tried it on the same server to compare:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-2]
└─$ python3 -c "...same snippet, swapping warrior.red for alt3_dwarf.red..."
Warrior 1 wins: 20
Warrior 2 wins: 0
Ties: 80
Try again. Your warrior (warrior 1) must win 100 times.
```

The Dwarf only won 20 of 100 and tied the other 80, so it is **not** enough on this server. That matches the challenge hint: "If your warrior is close, try again, it may work on subsequent tries". The Dwarf is close, but it is not deterministic, so it does not guarantee 100 wins. The imp gate is what guarantees it.

## Alternative — Why `JMP 0, <-1` Is a Trap

Out of curiosity I also tried `<-1`. It is the obvious-looking "smallest possible gate":

```redcode
;redcode
;name Imp Gate v3
;assert 1
jmp 0, <-1
end
```

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-2]
└─$ python3 -c "...same snippet, swapping warrior.red for alt2.red..."
Warrior 1 wins: 0
Warrior 2 wins: 0
Ties: 100
Try again. Your warrior (warrior 1) must win 100 times.
```

100 ties. What happened? With `<-1` the gate sits one cell directly in front of the warrior. The Imp reaches it almost immediately, the Imp's `mov 0, 1` overwrites that cell with its own code, and both processes then sit there in lockstep forever copying `mov 0, 1` back and forth without ever running a `DAT`. So the gate target has to be far enough away that the Imp has already passed it before the Imp copies over it. `<-2` is the smallest offset that still works reliably.

---

## What Happened Internally (Timeline)

1. I sent my warrior to `saturn.picoctf.net:54793`. The server echoed it back as "Warrior 1" and paired it against the Imp as "Warrior 2".
2. Both warriors were loaded into the same 8000-cell circular core with their PCs set to 0.
3. Each cycle, the server executed one instruction from each warrior, in warrior order. Our `JMP 0, <-2` did two things: (a) it pre-decremented the value in cell PC-2, and (b) it set PC back to itself. The Imp did `mov 0, 1`: copied its own instruction into PC+1 and advanced to PC+1.
4. After cycle 1 the core looked roughly like:
   - cell 0: `jmp 0, <-2` (our warrior, unchanged)
   - cell 1: `mov 0, 1` (the Imp's fresh copy)
   - cell -2 (a.k.a. cell 7998): some value, now 1 less than whatever the loader put there.
5. The Imp kept spiraling forward: 2, 3, 4, 5, …. Our warrior kept decrementing cell 7998 in place. The Imp never went backwards, so it never touched our warrior.
6. Eventually the Imp wrapped past cell 7999 and landed on or near cell 7998. By that time cell 7998 had been decremented thousands of times, so it was no longer a clean `mov 0, 1`. When the Imp tried to copy itself onto that corrupted cell, the resulting instruction was invalid and the Imp process died.
7. With the Imp gone, only our `JMP` was still running. The server declared us the winner of that round.
8. The server repeated the whole fight 100 times against freshly-loaded Imps. We won every single time, so it printed the flag.

The whole attack boils down to: the Imp's survival depends on its own instructions staying byte-perfect, and we silently corrupt one byte per cycle until the Imp walks into the trap.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the leaked Imp source |
| `nano` | Write `warrior.red` and the alternative warriors |
| `python3` (stdlib `socket`) | Talk to the picoCTF server because `netcat` was not installed in the sandbox |
| `nc` (the hint's recommended command) | The same thing — pipe a `.red` file into the server until `end` |
| The picoCTF CoreWars server at `saturn.picoctf.net:54793` | The actual opponent; runs 100 rounds of Redcode and emits the flag |

---

## Key Takeaways

- **CoreWars is a real game from the 1980s**, and picoCTF uses its 1994 "Redcode" dialect. Knowing it exists is half the battle; the other half is knowing the imp gate pattern.
- **Side effects on unused operands are a thing in Redcode.** `JMP` only reads the A-operand, but the addressing mode on the B-operand still fires. That is the entire trick.
- **Distance matters.** `<-2` and `<-5` win 100/100. `<-1` ties 100/100 because the Imp stalls right on top of the gate. If you write your own gate, sanity-check it on a tie count before assuming it works.
- **The hint about "trying again if close" is real.** The Dwarf paper wins about 20-30 rounds out of 100 deterministically and the rest are ties, which looks "almost there". That is the trap the hint is warning about. A true 100/100 winner exists (the imp gate); the Dwarf is just close.
- **One-line warriors are fine.** The whole solve is one line of Redcode plus the file boilerplate. Do not over-engineer this challenge.
- **`nc` is just `python3 -c "import socket; ..."` if you ever need a substitute.** That fallback came in handy here because my sandbox did not have netcat installed.

### Flag Wordplay Decode

The flag is:

```
picoCTF{d3m0n_3xpung3r_47037b25}
```

Decoded, it reads:

> **demon expunger** + suffix `47037b25`

with the usual leet substitutions:

- `d3m0n` — "demon" with `3` → `e`
- `3xpung3r` — "expunger" with `3` → `e` on both sides
- `47037b25` — a unique 8-character hex suffix per flag instance, not part of the wordplay

The lesson is right in the flag: the Imp is the "demon" that walks forever through memory, and the imp gate is the trick that **expunges** it. The challenge name "Ready Gladiator 2" is just the arena; the real opponent is the demon we are deleting.
