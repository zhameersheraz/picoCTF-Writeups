# 2warm — picoCTF 2019 Writeup

**Challenge:** 2warm  
**Category:** General Skills  
**Difficulty:** Easy  
**Flag:** `picoCTF{101010}`  

---

## Description

> Can you convert the number 42 (base 10) to binary (base 2)?

**Hint shown in challenge:** `Submit your answer in our competition's flag format. For example, if your answer was '11111', you would submit 'picoCTF{11111}' as the flag.`

---

## Background Knowledge (Read This First!)

### What is Binary?

Binary (base 2) is a number system that uses only two digits: **0** and **1**.
Computers use binary because electronic circuits have two states — off (0) and on (1).

Our normal number system is base 10 (decimal) — it uses digits 0 through 9.

```
Base 10:  0  1  2  3  4  5  6  7  8  9  10  11 ...
Base 2:   0  1  10 11 100 101 110 111 1000 ...
```

### How to Convert Decimal to Binary

Repeatedly divide the number by 2 and collect the remainders.
Read the remainders **bottom to top** to get the binary result.

---

## Solution — Step by Step

### Step 1 — Divide 42 by 2 Repeatedly

```
42 ÷ 2 = 21  remainder 0
21 ÷ 2 = 10  remainder 1
10 ÷ 2 = 5   remainder 0
 5 ÷ 2 = 2   remainder 1
 2 ÷ 2 = 1   remainder 0
 1 ÷ 2 = 0   remainder 1
```

### Step 2 — Read Remainders Bottom to Top

```
Remainders (bottom to top): 1 0 1 0 1 0
Binary result: 101010
```

✅ 42 in binary = **101010**

### Step 3 — Verify with Python (optional)

```
┌──(zham㉿kali)-[~]
└─$ python3 -c "print(bin(42))"
0b101010
```

The `0b` prefix just means "this is binary" in Python — ignore it.
The answer is `101010`. ✅

### Step 4 — Submit the Flag

Wrap the binary answer in the picoCTF flag format:

```
picoCTF{101010}
```

Got the flag! 🎯

---

## Alternative Methods

**Method 1 — Python one-liner**
```bash
python3 -c "print(bin(42)[2:])"
```
`[2:]` strips the `0b` prefix so you get just `101010`.

**Method 2 — bc command (Linux calculator)**
```bash
echo "obase=2; 42" | bc
```
`obase=2` tells `bc` to output in base 2.

**Method 3 — Online Tool**
Use RapidTables binary converter:
https://www.rapidtables.com/convert/number/decimal-to-binary.html

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Manual math | Divide by 2 method to convert decimal to binary |
| `python3` | Verify the binary conversion instantly |

---

## Key Takeaways

- Binary (base 2) uses only 0s and 1s — the language of computers
- To convert decimal → binary: divide by 2 repeatedly, read remainders bottom to top
- 42 in binary is `101010` — also known as the "Answer to Everything" in binary 😄
- `python3 -c "print(bin(42))"` is the fastest way to check any decimal → binary conversion
- The flag format wraps the raw binary digits: `picoCTF{101010}`
