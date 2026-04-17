# Binary Search — picoCTF Writeup

**Challenge:** Binary Search  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{g00d_gu355_ee8225d0}`

---

## Description

> Want to play a game? Binary search is a classic algorithm used to quickly find an item in a sorted list. Can you find the flag? You'll have 1000 possibilities and only 10 guesses.
> Connect using: `ssh -p 49696 ctf-player@atlas.picoctf.net`
> Password: `83dcefb7`

**Hint shown in challenge:** `Have you ever played hot or cold? Binary search is a bit like that.`

---

## Background Knowledge (Read This First!)

### What is Binary Search?

**Binary Search** finds a target number by repeatedly cutting the search range in half. Instead of guessing 1, 2, 3... one by one (up to 1000 guesses), binary search only needs at most **10 guesses** for a range of 1000:

```
2^10 = 1024 > 1000
```

### The Strategy

Always guess the **middle** of your current range:
```
Start:     1 -------- 500 -------- 1000
Too high:  1 ---- 250 ---- 500
Too low:        250 ---- 375 ---- 500
```

Each guess cuts the remaining possibilities in half.

---

## Solution — Step by Step

### Step 1 — Connect via SSH

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ ssh -p 49696 ctf-player@atlas.picoctf.net
```

- Accepted the fingerprint with `yes`
- Entered the password `83dcefb7`

### Step 2 — Play the Binary Search Game

The game picks a number between **1 and 1000** and I have **10 guesses**:

```
Guess 500 → Lower! (range: 1-499)
Guess 250 → Higher! (range: 251-499)
Guess 375 → Higher! (range: 376-499)
Guess 437 → Lower! (range: 376-436)
Guess 406 → Lower! (range: 376-405)
Guess 390 → Higher! (range: 391-405)
Guess 398 → Higher! (range: 399-405)
Guess 402 → Lower! (range: 399-401)
Guess 400 → Higher! (range: 401-401)
Guess 401 → Correct!
```

**Output:**
```
Congratulations! You guessed the correct number: 401
Here's your flag: picoCTF{g00d_gu355_ee8225d0}
```

Got the flag! 🎯

---

## Alternative Method — Python Script

```python
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

## Tools Used

| Tool | Purpose |
|------|---------|
| `ssh` | Connect to the game server |

---

## Key Takeaways

- Binary search cuts the search range in half with every guess
- With 10 guesses you can find any number between 1 and 1000
- Always guess the **middle** of the remaining range
- "Higher" means the answer is above your guess, "Lower" means it is below
- SSH passwords are hidden when typing — just type it and press Enter
