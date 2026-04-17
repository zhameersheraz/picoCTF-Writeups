# Flag Hunters — picoCTF Writeup

**Challenge:** Flag Hunters  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Flag:** `picoCTF{70637h3r_f0r3v3r_836f0788}`

---

## Description

> Lyrics jump from verses to the refrain kind of like a subroutine call. There's a hidden refrain this program doesn't print by default. Can you get it to print it?
> Connect: `nc verbal-sleep.picoctf.net 49675`

**Hint shown in challenge:** `This program can easily get into undefined states. Don't be shy about Ctrl-C.`

---

## Background Knowledge (Read This First!)

### How does the program work?

The source code uses `;` to split commands on each line. When the program hits a `CROWD` line, it asks for user input and processes that input split by `;`.

If we type `;RETURN 1` as crowd input, the program will:
- Ignore the empty part before `;`
- Execute `RETURN 1` which jumps back to **line 1** of the song
- Line 1 is the hidden `secret_intro` that contains the flag!

---

## Solution — Step by Step

### Step 1 — Download and Read the Source Code

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ wget https://challenge-files.picoctf.net/.../lyric-reader.py
└─$ cat lyric-reader.py
```

### Step 2 — Find the Hidden Section

Inside the code, the flag is in a section called `secret_intro` that runs BEFORE `[VERSE1]`. Since the program starts at `[VERSE1]`, this section never gets printed normally:

```python
secret_intro = '''Pico warriors rising, puzzles laid bare,
Solving each challenge with precision and flair.
With unity and skill, flags we deliver,
The ether's ours to conquer, ''' + flag + '\n'
```

### Step 3 — Connect and Exploit

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nc verbal-sleep.picoctf.net 49675
```

When asked for Crowd input, I typed:

```
Crowd: ;RETURN 1
```

**Output:**
```
Solving each challenge with precision and flair.
With unity and skill, flags we deliver,
The ether's ours to conquer, picoCTF{70637h3r_f0r3v3r_836f0788}
```

Got the flag! 🎯

---

## Alternative — One-liner

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ echo ";RETURN 1" | nc verbal-sleep.picoctf.net 49675
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the source code file |
| `cat` | Read and analyze the source code |
| `nc` (netcat) | Connect to the remote program |

---

## Key Takeaways

- Always read the source code first to understand how the program works
- The flag was in a section that never gets printed by default
- Adding `;` before a command lets you inject it through the crowd input
- `RETURN` in this program works like a jump to any line number you give it
- User input that gets processed as a command is a common vulnerability to look for
