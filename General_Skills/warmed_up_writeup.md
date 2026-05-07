# Warmed Up — picoCTF Writeup

**Challenge:** Warmed Up  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{61}`  

---

## Description

> What is 0x3D (base 16) in decimal (base 10)?

**Hint 1:** `Submit your answer in our flag format. For example, if your answer was '22', you would submit 'picoCTF{22}' as the flag.`

---

## Background Knowledge (Read This First!)

### What is Hexadecimal (base 16)?

**Hexadecimal** uses 16 symbols: digits `0-9` and letters `a-f`. The `0x` prefix tells you a number is in hex. Each hex digit represents 4 bits, making it a compact way to write binary data.

### Converting hex to decimal

`0x3D` means:
```
3 × 16¹ + D × 16⁰
= 3 × 16 + 13 × 1
= 48 + 13
= 61
```

(`D` in hex = 13 in decimal)

---

## Solution

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "print(0x3D)"
61
```

Wrap in picoCTF format → **`picoCTF{61}`** ✅ Got the flag! 🎯

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Python `print(0x3D)` | Convert hex to decimal instantly | ⭐ Easy |

---

## Key Takeaways

- **`0x` prefix** means hexadecimal — Python evaluates it as a number directly
- `D` in hex = 13 in decimal — the hex digits A-F map to 10-15
- The flag `61` is also the ASCII code for lowercase `a` — a nice connection to later challenges
