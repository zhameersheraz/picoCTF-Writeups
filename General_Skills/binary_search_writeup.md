# Binary Search - picoCTF Writeup

**Challenge:** Binary Search  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{g00d_gu355_ee8225d0}`

---

## Description

Want to play a game? As you use more of the shell, you might be interested in how they work! Binary search is a classic algorithm used to quickly find an item in a sorted list. Can you find the flag? You'll have 1000 possibilities and only 10 guesses.

Connect using:
```
ssh -p 49696 ctf-player@atlas.picoctf.net
```
Password: `83dcefb7`

---

## Hints

1. Have you ever played hot or cold? Binary search is a bit like that.

---

## Solution

### Step 1: Download and Inspect the File

I downloaded and extracted the challenge zip:

```bash
wget https://artifacts.picoctf.net/c_atlas/4/challenge.zip
unzip challenge.zip
```

This extracted `guessing_game.sh` — a shell script for the binary search game.

### Step 2: Connect via SSH

```bash
ssh -p 49696 ctf-player@atlas.picoctf.net
```

- Accepted the fingerprint with `yes`
- Entered the password `83dcefb7` (note: password is hidden when typing)

### Step 3: Play the Binary Search Game

The game picks a number between **1 and 1000** and I have **10 guesses** to find it.

The strategy is **Binary Search** — always guess the middle of the remaining range:

```
Guess 500 → Lower! (range: 1-499)
Guess 400 → Lower! (range: 1-399)
Guess 300 → Higher! (range: 301-399)
Guess 350 → Higher! (range: 351-399)
Guess 390 → Lower! (range: 351-389)
Guess 388 → Lower! (range: 351-387)
Guess 360 → Higher! (range: 361-387)
Guess 370 → Lower! (range: 361-369)
Guess 365 → Correct!
```

**Output:**
```
Congratulations! You guessed the correct number: 365
Here's your flag: picoCTF{g00d_gu355_ee8225d0}
```

Got the flag! 🎯

---

## Why This Works

### What is Binary Search?

**Binary Search** is an algorithm that finds a target number by repeatedly cutting the search range in half.

Instead of guessing 1, 2, 3, 4... one by one (which could take up to 1000 guesses), binary search only needs at most **10 guesses** for a range of 1000 because:

```
2^10 = 1024 > 1000
```

### The Strategy

Always guess the **middle** of your current range:

```
Start:        1 -------- 500 -------- 1000
Too high:     1 ---- 250 ---- 500
Too low:           250 ---- 375 ---- 500
And so on...
```

Each guess cuts the remaining possibilities in half, making it very efficient.

### Simple Breakdown

```
Connect via SSH
      |
      v
Game gives range: 1 to 1000
      |
      v
Always guess the middle of the remaining range
      |
      v
"Higher" = number is above your guess
"Lower"  = number is below your guess
      |
      v
Narrow down until correct
      |
      v
Flag is revealed!
```

---

## Alternative Method: Script It

You can automate the guessing with a Python script:

```python
import subprocess

low, high = 1, 1000
while low <= high:
    mid = (low + high) // 2
    print(mid)
    response = input()
    if "Congratulations" in response:
        break
    elif "Lower" in response:
        high = mid - 1
    elif "Higher" in response:
        low = mid + 1
```

---

## Commands Used

```bash
# Download and extract
wget https://artifacts.picoctf.net/c_atlas/4/challenge.zip
unzip challenge.zip

# Connect via SSH
ssh -p 49696 ctf-player@atlas.picoctf.net
# Password: 83dcefb7
```

---

## Flag

```
picoCTF{g00d_gu355_ee8225d0}
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` | Download the zip file |
| `unzip` | Extract the zip file |
| `ssh` | Connect to the game server |

---

## Key Takeaways

- Binary search cuts the search range in half with every guess
- With 10 guesses you can find any number between 1 and 1000
- Always guess the **middle** of the remaining range
- "Higher" means the answer is above your guess, "Lower" means it is below
- SSH passwords are hidden when typing — just type it and press Enter
