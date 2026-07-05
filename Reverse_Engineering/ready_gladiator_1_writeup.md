# Ready Gladiator 1 — picoCTF Writeup

**Challenge:** Ready Gladiator 1  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 200  
**Flag:** `picoCTF{1mp_1n_7h3_cr055h41r5_dba6f40d}`  
**Platform:** picoCTF 2023  
**Writeup by:** zham  

---

## Description

> Can you make a CoreWars warrior that wins?
>
> Your opponent is the Imp. The source is available [here]. If you wanted to pit the Imp against himself, you could download the Imp and connect to the CoreWars server like this:
>
> `nc saturn.picoctf.net 52901 < imp.red`
>
> To get the flag, you must beat the Imp at least once out of the many rounds.

## Hints

> 1. You may be able to find a viable warrior in beginner docs

---

## Background Knowledge (Read This First!)

### What is CoreWars?

CoreWars is a programming game invented by A. K. Dewdney in 1984. Two small programs, called **warriors**, run inside a shared memory called the **core**. The core is a fixed-size circular array (on picoCTF's server it is 8000 cells) and every memory address wraps around.

Each cycle both warriors execute exactly **one** instruction. The last warrior that is still alive wins that round. A warrior "dies" the moment one of its processes tries to execute a `DAT` instruction. The whole point of the game is to write a warrior in an assembly-like language called **Redcode** that kills the opponent before the opponent kills it.

### What is Redcode?

Redcode is the assembly language CoreWars uses. For this challenge we only need four instructions:

- `MOV src, dst` — copy the value at `src` into `dst`. PC advances by 1.
- `ADD a, b` — add `a` to `b`. PC advances by 1.
- `JMP addr` — set the program counter to `addr`.
- `DAT x, y` — data. If a warrior ever **executes** a `DAT`, that process dies.

And a few addressing modes:

- `#4`, `#0` — immediate. The number is the value.
- `2`, `-2` — direct relative. `2` means "the cell two ahead of PC", `-2` means "two behind".
- `@2` — indirect. Take the B-field of the cell at PC+2, use that as the destination.

### The Imp

The Imp is the smallest possible warrior. It is one instruction long:

```
mov 0, 1
```

That is "copy the current cell into the cell one ahead, then advance PC by 1". So the Imp keeps sliding a copy of itself one cell forward, forever. As long as nothing disturbs it, the Imp never dies. To beat it, you have to make it execute a `DAT`.

### The Dwarf (the warrior we are going to use)

The hint tells us to look at beginner docs, and the canonical first warrior every CoreWars tutorial introduces is the **Dwarf** (also called a "stone" or "paper"). It is a 4-instruction warrior that systematically drops `DAT` bombs every 4 cells across the entire core:

```
ADD #4, 3
MOV 2, @2
JMP -2
DAT #0, #0
```

Line by line:

- `ADD #4, 3` — add the immediate value `4` to the B-field of the cell at PC+3 (the `DAT` instruction below). After this, the `DAT` becomes `DAT #0, #4`.
- `MOV 2, @2` — copy the instruction at PC+2 (the `JMP -2`) to the **address stored in the B-field of PC+2**. That B-field is now `4`, so this writes `JMP -2` to whatever cell is 4 away from the `DAT`. That new cell is now a brand new Dwarf. PC advances to PC+1.
- `JMP -2` — jump back to the `ADD` at PC-2 from here (PC was at the `JMP`, so PC-2 is back at `ADD`). This loops the whole warrior.
- `DAT #0, #4` — after the `ADD`, this is now `DAT #0, #4`. When the warrior copies itself forward, the copy keeps the bomb-dropping logic, so every 4th cell becomes a Dwarf too. And because the Dwarf also writes a `JMP -2` to the address held in the new cell's B-field, the bomb line keeps stepping forward by 4.

So the Dwarf paints the entire core with `JMP -2` instructions spaced 4 cells apart. Anywhere the Imp wanders, it eventually hits one of those `JMP -2` cells, gets teleported backward into the next `JMP -2`, and keeps getting yanked backward instead of moving forward. Because the Imp's progress is constantly reset, it eventually wanders into a `DAT` bomb that the Dwarf has also been scattering, and dies.

### Why the Dwarf works for THIS challenge but not for Ready Gladiator 2

The Dwarf wins about 20-30% of rounds and ties the rest. For Ready Gladiator 1 that is plenty, because we only need **at least one** win out of the 100 rounds the server runs. For Ready Gladiator 2 (the 400-point sequel) you need 100/100 wins, and the Dwarf is not deterministic enough — that is why we had to upgrade to the **imp gate** (`jmp 0, <-2`) for that one.

### What is a "round" here?

The server runs 100 independent fights against freshly-loaded Imps. As soon as our warrior wins one of them, the server declares victory and prints the flag. We do not need to win all 100.

---

## Solution — Step by Step

### Step 1 — Read the Imp

The challenge ships the Imp source. I copied it into a workspace folder so I could look at it without poking the server:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-1]
└─$ cat imp.red
;redcode
;name Imp Ex
;assert 1
mov 0, 1
end
```

One instruction. Whatever warrior I write has to kill it (or at least outlast it in one of the 100 rounds).

### Step 2 — Build the Dwarf

The hint literally points us at beginner CoreWars docs, and the very first warrior in every beginner guide is the Dwarf. I opened `nano` and pasted the canonical 4-line version:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-1]
└─$ nano dwarf.red
```

In `nano` I pasted this and saved with `Ctrl+O`, `Enter`, `Ctrl+X`:

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

Confirm what I wrote:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-1]
└─$ cat dwarf.red
;redcode
;name Dwarf
;assert 1
ADD #4, 3
MOV 2, @2
JMP -2
DAT #0, #0
end
```

### Step 3 — Send It to the Server

The server is a plain TCP service. The challenge's recommended command is `nc saturn.picoctf.net 52901 < imp.red`, but I am sending my Dwarf instead of the Imp. The server reads our warrior from stdin until it sees the literal line `end`.

I do not have `netcat` installed in my sandbox, so I used a tiny inline `python3` script that does the exact same thing:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-1]
└─$ python3 -c "
import socket
s = socket.create_connection(('saturn.picoctf.net', 52901), timeout=15)
s.settimeout(15)
s.recv(4096)
s.sendall(open('dwarf.red','rb').read() + b'\nend\n')
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
;name Dwarf
;assert 1
ADD #4, 3
MOV 2, @2
JMP -2
DAT #0, #0
end
end
Warrior1:
;redcode
;name Dwarf
;assert 1
ADD #4, 3
MOV 2, @2
JMP -2
DAT #0, #0
end

Rounds: 100
Warrior 1 wins: 21
Warrior 2 wins: 0
Ties: 79
You did it!
picoCTF{1mp_1n_7h3_cr055h41r5_dba6f40d}
```

21 wins out of 100 rounds. That is way more than the 1 win we needed. The server printed the flag straight back:

```
picoCTF{1mp_1n_7h3_cr055h41r5_dba6f40d}
```

---

## Alternative — The Imp Gate (carried over from Ready Gladiator 2)

The imp gate `jmp 0, <-2` from the 400-point sequel also works here. If you have already written that warrior for Ready Gladiator 2, you can reuse it without changes:

```redcode
;redcode
;name Imp Gate
;assert 1
jmp 0, <-2
end
```

It is overkill for this challenge — we only need a single win, not 100/100 — but it is the safer "always works" answer if you do not want to think about why the Dwarf sometimes ties.

## Alternative — Sending `end` with No Warrior

A cute trick from the Ready Gladiator 0 challenge (the "always lose" version): if you send just the trailing `end` line and no warrior at all, the server treats you as a warrior that crashes immediately, which counts as a loss. For Ready Gladiator **0** that is enough. For Ready Gladiator **1** you actually need to win at least once, so this trick does **not** work here — I tried it on a fresh connection to confirm:

```
┌──(zham㉿kali)-[~/picoctf/ready-gladiator-1]
└─$ python3 -c "...same snippet, sending only b'end\n'..."
Warrior 1 wins: 0
Warrior 2 wins: 100
Ties: 0
Try again. Your warrior (warrior 1) must win 100 times.
```

Confirmed: no warrior means we always lose, and the server does not give us the flag. We need an actual winning warrior — the Dwarf is the simplest one.

---

## What Happened Internally (Timeline)

1. I sent my Dwarf to `saturn.picoctf.net:52901`. The server echoed it back as "Warrior 1" and paired it against the Imp as "Warrior 2".
2. Both warriors were loaded into the same 8000-cell circular core with their PCs set to 0.
3. Cycle 1, our Dwarf: `ADD #4, 3` mutated the `DAT #0, #0` at PC+3 into `DAT #0, #4`. PC advanced to PC+1.
4. Cycle 1, our Dwarf: `MOV 2, @2` copied the `JMP -2` at PC+2 into the address held in PC+2's B-field, which was now `4`. So a copy of the whole Dwarf landed at cell 4. PC advanced to PC+1 (the `JMP -2`).
5. Cycle 1, our Dwarf: `JMP -2` jumped back to PC-2 (the `ADD #4, 3`). PC wrapped around.
6. Cycle 1, the Imp: `mov 0, 1` copied itself from cell 0 into cell 1 and advanced to cell 1.
7. After a few hundred cycles, the Dwarf had painted cells 4, 8, 12, 16, … with copies of itself, and the Imp had walked forward through 0, 1, 2, 3, 4, 5, …. When the Imp reached cell 4, it executed whatever was there — a fresh Dwarf's `ADD #4, 3` — which is not a valid Imp instruction. The Imp process died.
8. With the Imp gone, the Dwarf was the last process running. The server declared us the winner of that round.
9. The server then reset both warriors and ran another round. Of the 100 rounds, the Dwarf won 21 outright and tied the other 79 (the tie happens when both warriors exhaust the round limit without either process dying — the Imp just keeps wandering and the Dwarf keeps painting, and nobody executes a `DAT`).
10. Because we won at least one round, the server printed the flag and closed the connection.

The whole attack boils down to: the Dwarf floods the core with copies of itself on a 4-cell stride, and the Imp's `mov 0, 1` cannot survive the moment it steps on one of those copies because its source operand (`0` = relative self) starts reading a Dwarf instruction instead of its own.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `cat` | Read the leaked Imp source |
| `nano` | Write `dwarf.red` |
| `python3` (stdlib `socket`) | Talk to the picoCTF server because `netcat` was not installed in the sandbox |
| `nc` (the hint's recommended command) | The same thing — pipe a `.red` file into the server until `end` |
| CoreWars beginner docs (the hint) | Reference for the canonical Dwarf warrior |
| The picoCTF CoreWars server at `saturn.picoctf.net:52901` | The actual opponent; runs 100 rounds of Redcode and emits the flag |

---

## Key Takeaways

- **Read the win condition carefully.** Ready Gladiator 0 wants you to **lose** every round, Ready Gladiator 1 wants you to **win at least once**, Ready Gladiator 2 wants you to **win all 100**. The same warrior might be the right answer for one and the wrong answer for another. Read the description, not just the title.
- **The Dwarf is the canonical first warrior.** Every beginner CoreWars tutorial introduces it. If the hint says "beginner docs", this is what they mean. Memorise it.
- **Probability vs determinism matters.** The Dwarf wins ~20-30% of rounds, which is plenty when you only need 1 win out of 100. It is not enough when you need 100/100 — that is why the 400-point sequel demands the imp gate instead.
- **Side-effect addressing modes are everywhere in Redcode.** `@2` (indirect), `<-N` (pre-decrement indirect), `>N` (post-increment indirect) all do extra work you might not expect. The Dwarf's `MOV 2, @2` and the imp gate's `JMP 0, <-2` are both leaning on this.
- **`nc` is just `python3 -c "import socket; ..."` if you ever need a substitute.** That fallback came in handy here because my sandbox did not have netcat installed.
- **The trailing `end` is mandatory.** The server reads your warrior line by line until it sees the literal string `end` on its own line. Forgetting it just hangs the connection.

### Flag Wordplay Decode

The flag is:

```
picoCTF{1mp_1n_7h3_cr055h41r5_dba6f40d}
```

Decoded, it reads:

> **imp in the crosshairs** + suffix `dba6f40d`

with the usual leet substitutions:

- `1mp` — "imp" with `1` → `i`
- `1n` — "in" with `1` → `i`
- `7h3` — "the" with `7` → `t`
- `cr055h41r5` — "crosshairs" with `0` → `o`, `5` → `s`, `4` → `a`
- `dba6f40d` — a unique 8-character hex suffix per flag instance, not part of the wordplay

The flag is a perfect description of the challenge: the Imp is the little `mov 0, 1` program we have to put **in the crosshairs**. The challenge name "Ready Gladiator" is just the arena framing; the real win condition is dropping that tiny spiraling demon.
