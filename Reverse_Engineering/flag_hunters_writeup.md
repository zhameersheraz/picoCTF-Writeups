# Flag Hunters - picoCTF Writeup

**Challenge:** Flag Hunters  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{70637h3r_f0r3v3r_836f0788}`

---

## Description

Lyrics jump from verses to the refrain kind of like a subroutine call. There's a hidden refrain this program doesn't print by default. Can you get it to print it? There might be something in it for you.

Connect to the program with netcat:
```
$ nc verbal-sleep.picoctf.net 49675
```

---

## Hints

1. This program can easily get into undefined states. Don't be shy about Ctrl-C.

---

## Solution

### Step 1: Download and Read the Source Code

I downloaded the source code using `wget`:

```bash
wget https://challenge-files.picoctf.net/.../lyric-reader.py
```

Then I read the contents using `cat`:

```bash
cat lyric-reader.py
```

### Step 2: Understand the Source Code

After reading the code, I found two important things:

**1. The flag is hidden in a section called `secret_intro`**

This section contains the flag but it runs BEFORE `[VERSE1]`. Since the program starts at `[VERSE1]`, this section never gets printed normally:

```python
secret_intro = \
'''Pico warriors rising, puzzles laid bare,
Solving each challenge with precision and flair.
With unity and skill, flags we deliver,
The ether's ours to conquer, ''' + flag + '\n'
```

**2. The CROWD input uses `;` to split commands**

When the program hits a `CROWD` line, it asks for user input. It then processes that input using `split(';')` to separate commands:

```python
elif re.match(r"CROWD.*", line):
    crowd = input('Crowd: ')
    song_lines[lip] = 'Crowd: ' + crowd
    lip += 1
```

This means if we type `;RETURN 1` as our crowd input, the program will:
- Ignore the empty part before `;`
- Execute `RETURN 1` which jumps back to **line 1** of the song
- Line 1 is the hidden `secret_intro` that contains the flag!

### Step 3: First Attempt (Failed)

I connected to the server and tried typing `RETURN 1` as crowd input:

```bash
nc verbal-sleep.picoctf.net 49675
```

**Crowd input:** `RETURN 1`

This did not work because the input was treated as plain text, not a command.

### Step 4: Successful Exploit

I reconnected and this time added a semicolon before the command:

```bash
nc verbal-sleep.picoctf.net 49675
```

**Crowd input:** `;RETURN 1`

**Output:**
```
Solving each challenge with precision and flair.
With unity and skill, flags we deliver,
The ether's ours to conquer, picoCTF{70637h3r_f0r3v3r_836f0788}
```

Got the flag! 🎯

---

## Why This Works

The program splits each line using `;` as a separator. So when we enter `;RETURN 1`, it becomes:

```
['Crowd: ', 'RETURN 1']
```

- `Crowd: ` gets printed as normal text
- `RETURN 1` gets executed as a command, jumping to line 1

Line 1 is the `secret_intro` section which contains the flag!

### Simple Breakdown

```
Type: ;RETURN 1
        |
        v
Program splits by ";"
        |
        v
"RETURN 1" is treated as a command
        |
        v
Jumps to line 1 (secret_intro)
        |
        v
Flag is printed!
```

---

## Alternative Solution

Instead of typing manually, you can automate it using `echo` and pipe it directly to netcat:

```bash
echo ";RETURN 1" | nc verbal-sleep.picoctf.net 49675
```

This sends `;RETURN 1` automatically as the crowd input without having to type it manually. The flag will appear in the output.

---

## Commands Used

```bash
# Download source code
wget https://challenge-files.picoctf.net/.../lyric-reader.py

# Read source code
cat lyric-reader.py

# Connect to server
nc verbal-sleep.picoctf.net 49675

# When asked for Crowd input, type:
;RETURN 1
```

---

## Flag

```
picoCTF{70637h3r_f0r3v3r_836f0788}
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
- The flag was hidden in a section that never gets printed by default
- Adding `;` before a command lets you inject it through the crowd input
- `RETURN` in this program works like a jump, it goes to any line number you give it
- User input that gets processed as a command is a common vulnerability to look for
