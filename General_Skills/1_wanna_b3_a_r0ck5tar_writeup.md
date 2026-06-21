# 1_wanna_b3_a_r0ck5tar — picoCTF Writeup

**Challenge:** 1_wanna_b3_a_r0ck5tar  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 350  
**Flag:** `picoCTF{BONJOVI}`  
**Platform:** picoCTF (2019)  
**Writeup by:** zham

---

## Description

> I wrote you another song. Put the flag in the picoCTF{} flag format.

A `.txt` file named `lyrics.txt` is provided via the challenge link.

---

## Hints

No hints provided for this challenge.

---

## Background Knowledge (Read This First!)

### What is Rockstar?

Rockstar is an esoteric programming language created by Dylan Beattie. The goal is for valid programs to also read like dramatic rock song lyrics. Every instruction maps to a real programming statement written in rock-and-roll-style poetry.

You can run Rockstar programs at: https://codewithrockstar.com/online

Or install the interpreter via npm:

```
npm install -g rockstar
```

### Poetic Number Literals

The most important concept in this challenge. When you write:

```
Variable is [some words]
```

Rockstar counts the **number of letters in each word** (mod 10), then joins those digits left to right to form a number.

Example:

```
Tommy is playing rock
```

- "playing" = 7 letters → digit 7
- "rock" = 4 letters → digit 4
- Tommy = **74**

### Output Commands

`Say`, `Shout`, `Scream`, and `Whisper` all print a value. Numbers print as decimal integers.

### Pronouns

"it", "they", "him", "her", etc. refer to the **last variable mentioned** in the program. So `Shout it!` prints whatever the most recently assigned variable holds.

### Input

`Listen to X` reads a value from stdin and stores it in variable X.

### Arithmetic

"without" means subtraction. So `rhythm without Music` = rhythm − Music.

### Conditionals

`If X is Y` is a standard equality check. `If X is nothing` checks if X equals zero.

---

## Step-by-Step Solution

### Step 0: Get the file

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/CTF/picoCTF_2019/General_Skills/1_wanna_b3_a_r0ck5tar && cd $_

┌──(zham㉿kali)-[~/CTF/picoCTF_2019/General_Skills/1_wanna_b3_a_r0ck5tar]
└─$ wget https://artifacts.picoctf.net/c/1_wanna_b3_a_r0ck5tar/lyrics.txt

┌──(zham㉿kali)-[~/CTF/picoCTF_2019/General_Skills/1_wanna_b3_a_r0ck5tar]
└─$ cat lyrics.txt
Rocknroll is right
Silence is wrong
A guitar is a six-string
Tommy's been down
Music is a billboard-burning razzmatazz!
Listen to the music
If the music is a guitar
Say "Keep on rocking!"
Listen to the rhythm
If the rhythm without Music is nothing
Tommy is rockin guitar
Shout Tommy!
Music is amazing sensation
Jamming is awesome presence
Scream Music!
Scream Jamming!
Tommy is playing rock
Scream Tommy!
They are dazzled audiences
Shout it!
Rock is electric heaven
Scream it!
Tommy is jukebox god
Say it!
Break it down
Shout "Bring on the rock!"
Else Whisper "That ain't it, Chief"
Break it down
```

This is a program written in the **Rockstar** esoteric programming language.

---

### Step 1: Figure out the input gates

The program has two `Listen` calls that pause and wait for stdin:

```
Listen to the music          ← stores input into Music
If the music is a guitar     ← if Music == guitar, print "Keep on rocking!"

Listen to the rhythm         ← stores input into rhythm
If the rhythm without Music is nothing   ← if (rhythm − Music) == 0, run main block
```

I need to know the value of `guitar` first:

```
A guitar is a six-string
```

Counting letters per word (mod 10):
- "a" = 1
- "six" = 3
- "string" = 6

**guitar = 136**

So I enter **136** for both inputs. The first satisfies `If the music is a guitar` (136 == 136), and the second satisfies `rhythm − Music = 136 − 136 = 0`, which opens the main output block.

---

### Step 2: Install the Rockstar interpreter

```
┌──(zham㉿kali)-[~/CTF/picoCTF_2019/General_Skills/1_wanna_b3_a_r0ck5tar]
└─$ sudo npm install -g rockstar
```

---

### Step 3: Run the program

```
┌──(zham㉿kali)-[~/CTF/picoCTF_2019/General_Skills/1_wanna_b3_a_r0ck5tar]
└─$ echo -e "136\n136" | rockstar lyrics.txt
Keep on rocking!
66
79
78
74
79
86
73
```

"Keep on rocking!" fires because our first input (136) equals guitar (136). The seven numbers that follow come from the Shout/Scream/Say calls inside the second if block.

---

### Step 4: Convert decimal output to ASCII

```
┌──(zham㉿kali)-[~/CTF/picoCTF_2019/General_Skills/1_wanna_b3_a_r0ck5tar]
└─$ python3 -c "print(''.join(chr(n) for n in [66,79,78,74,79,86,73]))"
BONJOVI
```

Flag: `picoCTF{BONJOVI}`

---

## Alternative: Manual Decode (No Interpreter Needed)

I can skip the interpreter entirely and read the values straight from the source. Each `Shout`/`Scream`/`Say` line inside the second if block prints the current value of a variable. I count letters in the words of each assignment:

| Assignment line               | Word lengths | Value | chr() |
|-------------------------------|-------------|-------|-------|
| `Tommy is rockin guitar`      | 6, 6        | 66    | B     |
| `Music is amazing sensation`  | 7, 9        | 79    | O     |
| `Jamming is awesome presence` | 7, 8        | 78    | N     |
| `Tommy is playing rock`       | 7, 4        | 74    | J     |
| `They are dazzled audiences`  | 7, 9        | 79    | O     |
| `Rock is electric heaven`     | 8, 6        | 86    | V     |
| `Tommy is jukebox god`        | 7, 3        | 73    | I     |

B O N J O V I → **BONJOVI**

Or use a script to do it faster:

```
┌──(zham㉿kali)-[~/CTF/picoCTF_2019/General_Skills/1_wanna_b3_a_r0ck5tar]
└─$ nano decode.py
```

Paste this:

```python
#!/usr/bin/env python3

def rock_num(words):
    digits = [str(len(w) % 10) for w in words.split()]
    return int(''.join(digits))

assignments = [
    "rockin guitar",
    "amazing sensation",
    "awesome presence",
    "playing rock",
    "dazzled audiences",
    "electric heaven",
    "jukebox god",
]

nums = [rock_num(a) for a in assignments]
flag = ''.join(chr(n) for n in nums)
print(f"Numbers : {nums}")
print(f"Flag    : picoCTF{{{flag}}}")
```

Ctrl+O, Enter, Ctrl+X to save and exit.

```
┌──(zham㉿kali)-[~/CTF/picoCTF_2019/General_Skills/1_wanna_b3_a_r0ck5tar]
└─$ python3 decode.py
Numbers : [66, 79, 78, 74, 79, 86, 73]
Flag    : picoCTF{BONJOVI}
```

---

## What Happened Internally

```
[1] Setup variables.
    guitar = 136 (from "a six-string" → 1, 3, 6).
    Other variables defined but unused or immediately overwritten by input.

[2] Listen to the music → Music = 136 (our input).
    If Music == guitar → 136 == 136 → true.
    Prints: "Keep on rocking!"

[3] Listen to the rhythm → rhythm = 136 (our input).
    If (rhythm − Music) == 0 → 136 − 136 = 0 → true.
    Main output block executes.

[4] Tommy = rockin(6) guitar(6) = 66     → Shout Tommy   → prints 66  → 'B'
    Music = amazing(7) sensation(9) = 79  → Scream Music  → prints 79  → 'O'
    Jamming = awesome(7) presence(8) = 78 → Scream Jamming → prints 78 → 'N'
    Tommy = playing(7) rock(4) = 74       → Scream Tommy  → prints 74  → 'J'
    They = dazzled(7) audiences(9) = 79, "it" = They → Shout it → prints 79 → 'O'
    Rock = electric(8) heaven(6) = 86, "it" = Rock → Scream it → prints 86 → 'V'
    Tommy = jukebox(7) god(3) = 73, "it" = Tommy → Say it → prints 73 → 'I'

[5] Break it down → exits block.
    The "Bring on the rock!" / "That ain't it, Chief" branch is a separate
    conditional that does not affect the flag output.

[6] 66 79 78 74 79 86 73 → ASCII → BONJOVI → picoCTF{BONJOVI}
```

---

## Tools Used

| Tool            | Purpose                                        |
|-----------------|------------------------------------------------|
| Rockstar (npm)  | Execute lyrics.txt as a Rockstar program       |
| Python 3        | Convert decimal output to ASCII characters     |
| Manual analysis | Count word lengths to extract values by hand   |

---

## Key Takeaways

- **Esoteric languages in CTFs** — Not all code looks like code. When a challenge gives you poetry or song lyrics as a file, think Rockstar, Brainfuck, Piet, or similar esolangs.
- **Poetic number literals** — Rockstar encodes numbers as word-length counts. Count letters per word, take each mod 10, join as digits left to right.
- **Input gates** — If a program has `Listen to X` + `If X is Y`, figure out Y's value first, then pass it via stdin (`echo -e "val\nval" | rockstar file`).
- **Pronouns** — "it" always refers to the last named variable. Track this carefully when reading `Shout it!` or `Scream it!` statements.

**Flag wordplay decode:** `1_wanna_b3_a_r0ck5tar` in leet = "I wanna be a rockstar." The answer hiding inside the lyrics is **BON JOVI** — one of the biggest rock acts of all time. The song wrote the flag itself.
