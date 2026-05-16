# Lets Warm Up — picoCTF Writeup

**Challenge:** Lets Warm Up  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{p}`  
**Platform:** picoCTF 2019  
**Writeup by:** zham  

---

## Description

> If I told you a word started with 0x70 in hexadecimal, what would it start with in ASCII?

**Hint 1:** `Submit your answer in our flag format. For example, if your answer was 'hello', you would submit 'picoCTF{hello}' as the flag.`

---

## Background Knowledge (Read This First!)

### Hex to ASCII

`0x70` is a hexadecimal number. Converting to decimal: `7 × 16 + 0 = 112`. Looking up ASCII code 112 gives the character `p`.

| Hex | Decimal | ASCII |
|-----|---------|-------|
| 0x70 | 112 | `p` |

---

## Solution

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "print(chr(0x70))"
p
```

Wrap in picoCTF format → **`picoCTF{p}`** ✅ Got the flag! 🎯

---

## Connection to "Lets Warm Up"

`p` is the first letter of **picoCTF** — which starts with `0x70` in hex. The challenge is warming you up to recognise hex-to-ASCII conversions, a skill used constantly in CTFs.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Python `chr(0x70)` | Convert hex ASCII code to character | ⭐ Easy |

---

## Key Takeaways

- **`chr(0x70)`** in Python converts any hex ASCII value to its character
- `0x70 = 112 = 'p'` — the start of picoCTF itself!
- Hex-to-ASCII conversion is one of the most fundamental skills in CTF — every flag starts with `picoCTF` which is `0x70 0x69 0x63 0x6f 0x43 0x54 0x46` in hex
