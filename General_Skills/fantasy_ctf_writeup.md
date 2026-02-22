# FANTASY CTF - picoCTF Writeup

**Challenge:** FANTASY CTF  
**Category:** General Skills  
**Difficulty:** Easy  
**Points:** (not specified)  
**Flag:** `picoCTF{m1113n1um_3d1710n_1e2b417a}`

---

## Description

Play this short game to get familiar with terminal applications and some of the most important rules in scope for picoCTF.

Connect to the program with netcat:
```
nc verbal-sleep.picoctf.net 52341
```

---

## Hints

1. When a choice is presented like [a/b/c], choose one, for example: `a` and then press Enter.

---

## Solution

### Step 1: Connect to the Server

I used netcat to connect to the challenge:
```bash
nc verbal-sleep.picoctf.net 52341
```

### Step 2: Read the Story

The challenge presents an interactive fiction game set in the year 3025. The story follows Eibhilin, a student registering for picoCTF with her AI companion Nyx.

I pressed **Enter** to progress through the story dialogue.

### Step 3: First Choice - Account Registration

The first decision appeared:
```
Options:
A) *Register multiple accounts*
B) *Share an account with a friend*
C) *Register a single, private account*
[a/b/c] >
```

I chose **`c`** (Register a single, private account).

**Why?** Nyx warns that registering multiple accounts has been grounds for disqualification for 1,000 years! This teaches an important rule: **never create multiple accounts in CTF competitions**.

### Step 4: Continue Reading

The story continues with an introductory message about using hacking skills responsibly: "With great power, comes great responsibility!"

I continued pressing **Enter** to advance through the dialogue.

### Step 5: Second Choice - Finding the Flag

Another decision appeared:
```
Options:
A) *Play the game*
B) *Search the Ether for the flag*
[a/b] >
```

I chose **`a`** (Play the game).

**Why?** Nyx says "You never want to share flags or artifact downloads." This teaches another important rule: **don't cheat by searching for or sharing flags**.

### Step 6: Get the Flag

After playing the game, a progress bar appears:
```
Playing the Game: 100%|████████████████ [time left: 00:00]
Playing the Game completed successfully!
```

The story concludes with Eibhilin finding the flag and Nyx reminding her to wait until after the competition to publish writeups.

**Final dialogue:**
```
"Thanks, Nyx! Here's the flag I found: picoCTF{m1113n1um_3d1710n_1e2b417a}"
```

---

## Why This Works

* This is an **educational challenge** teaching CTF ethics and rules
* It uses **interactive fiction** to make learning engaging
* The correct choices reflect **real picoCTF rules**:
  - Don't register multiple accounts
  - Don't share accounts
  - Don't share flags with others
  - Don't cheat by searching for flags online
  - Wait until competition ends to publish writeups
* Making wrong choices teaches you what NOT to do
* The flag name "millennium edition" (m1113n1um_3d1710n) references the futuristic 3025 setting

---

## What is Netcat?

**Netcat (nc)** is a networking utility for reading from and writing to network connections:

**Basic usage:**
```bash
# Connect to a server
nc hostname port

# Example from this challenge
nc verbal-sleep.picoctf.net 52341

# Listen on a port (server mode)
nc -l -p 1234
```

**Common uses:**
- Connect to CTF challenge servers
- Port scanning
- File transfers
- Chat between computers
- Debugging network services

---

## Important picoCTF Rules

This challenge teaches these critical rules:

1. **One Account Per Person**
   - Never register multiple accounts
   - This is grounds for disqualification

2. **No Account Sharing**
   - Each person must use their own account
   - Don't share login credentials

3. **No Flag Sharing**
   - Don't share flags with other competitors
   - Don't post flags publicly during the competition

4. **No Cheating**
   - Solve challenges yourself
   - Don't search for flags online during the competition

5. **Writeup Timing**
   - Wait until after winners are announced to publish writeups
   - This gives everyone a fair chance

---

## Terminal Commands
```bash
# Connect to the challenge
nc verbal-sleep.picoctf.net 52341

# Progress through the story
# Press Enter when prompted

# Make choices
# Type 'a', 'b', or 'c' and press Enter

# First choice: c (Register single account)
# Second choice: a (Play the game)
```

---

## Flag

`picoCTF{m1113n1um_3d1710n_1e2b417a}`

---

## Tools Used

* **netcat (nc)** - Connect to network services

---

## Key Takeaways

* **Follow CTF rules** - They exist to keep competitions fair
* **One account per person** - Multiple accounts = disqualification
* **Don't share flags** - This ruins the competition for others
* **Don't cheat** - The point is to learn, not just get points
* **Publish writeups responsibly** - Wait until after the competition
* **Interactive fiction** can be a fun way to learn security concepts
* The challenge is set in 3025, making it the "millennium edition" (hence the flag name)
* "With great power comes great responsibility" - use hacking skills ethically
* Reading carefully and making ethical choices leads to success
