# Shop — picoCTF Writeup

**Challenge:** Shop  
**Category:** Reverse Engineering  
**Difficulty:** Medium  
**Points:** 50  
**Flag:** `picoCTF{b4d_brogrammer_0fdb35803}`  
**Platform:** picoCTF 2021  
**Writeup by:** zham  

---

## Description

> Best Stuff - Cheap Stuff, Buy Buy Buy..
>
> Store Instance: source. The shop is open for business at `nc wily-courier.picoctf.net 54593`.

The challenge gives us a small text-based shop. We start with 40 coins, the most expensive item is a "Fruitful Flag" priced at 100 coins, and we need to be rich enough to buy it.

## Hints

> 1. Always check edge cases when programming

That single hint is the whole challenge. The author is telling us that the buy/sell logic does *not* validate the edges of its inputs (negative quantities, zero, values right at the boundary of what is allowed). We just have to find which edge case makes the wallet grow.

---

## Background Knowledge

A few small ideas make the rest of the solve click. None of them are advanced — they are the everyday reverse-engineering vocabulary for a beginner.

**1. What "edge cases" means.**
When a programmer writes `if user_input > 0: do_thing()`, they have to also think about what happens for `0`, for negative numbers, and for the largest number the type can hold. Those are *edge cases* — values right at the boundary of what the program was designed to handle. The most common bug in CTF challenges is that the author wrote the happy-path check (`qty > 0`) but forgot one of the edges (e.g. `qty < 0`, `qty == 0`, or `qty > MAX_INT`). When you see "check edge cases" in a hint, immediately try `0`, `-1`, a very large number, and `INT_MAX`/`INT_MIN`.

**2. Signed vs unsigned integers.**
In C (and in Go on 32-bit builds), a regular `int` can be negative. If the code does `count = count - quantity` and `quantity` happens to be `-100`, then the code is really doing `count = count + 100` — the stock *grows*. The same trick applies to the wallet: `coins = coins - price * quantity` with `quantity = -1` becomes `coins = coins + price`, i.e. we are paid to buy.

**3. Reading Go binaries.**
This binary is a Go executable (we will see `go1.10.4` printed in a panic trace). Go binaries are a little intimidating in `objdump` because every call goes through a stack-allocated argument list — there are no register-passed arguments like in C. The trick is not to read every instruction. Instead, look at the function names (`nm` and `strings`), find `main.get_flag`, and read only the few instructions around where it is called. The two functions we actually care about are `main.menu` (the buy/sell logic) and `main.get_flag` (the reward).

**4. Why the program prints "Flag is: [112 105 99 111 ...]".**
`main.get_flag` calls `ioutil.ReadFile("flag.txt")` and prints the bytes. Go's `fmt.Println` on a `[]byte` slice does *not* print the string — it prints the slice as a list of decimal byte values. That is why every Shop-style picoCTF challenge prints a list of numbers and not the literal `picoCTF{...}`. We have to convert the bytes back into ASCII at the end. A one-liner Python script does it.

**5. Why the binary "panics" when run locally.**
The binary reads `/app/source.go` from the panic trace, but more importantly it tries to open `flag.txt` in the current directory. The remote server has the flag file on disk; we do not, so a local run prints `panic: open flag.txt: no such file or directory`. That is fine — we still get to see the buy/sell logic work, and the exploit is identical against the remote.

---

## Step-by-step Solution

I worked this out in a Kali VM on VirtualBox. The challenge gives us an attachment called `source` and a remote instance at `nc wily-courier.picoctf.net 54593`. Both give the same binary; I solved it against the remote because the local binary would need a `flag.txt` next to it to print anything.

### 1. Save the attachment and check what it is

```
┌──(zham㉿kali)-[~/shop]
└─$ mkdir -p ~/shop && cd ~/shop

┌──(zham㉿kali)-[~/shop]
└─$ cp ~/Downloads/source shop

┌──(zham㉿kali)-[~/shop]
└─$ chmod +x shop

┌──(zham㉿kali)-[~/shop]
└─$ file shop
shop: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, Go BuildID=4B889tc1TRgpS9czhvct/FgQ1iPCpaksezCsvmIRb/JEuIrSM0u7bsenZgEPQP/pVFuNtB-YiGf_M2pZFLj, with debug_info, not stripped
```

Two things to note: it is **32-bit Intel**, and it is a **Go binary with debug info and not stripped**. That last bit is gold — `nm` will print every function name and `objdump` will use those names as labels.

### 2. Skim the symbols to find the interesting functions

```
┌──(zham㉿kali)-[~/shop]
└─$ nm shop | grep -E "main\." | head -20
080d4550 T main.check
080d4440 T main.get_flag
080d3960 T main.menu
080d3350 T main.openShop
080d3f90 T main.sell
080d3530 T main.stockUp
```

`main.get_flag` is the obvious target — it almost certainly reads and prints the flag. `main.sell` and `main.menu` are where the wallet logic lives. Let me confirm what `main.get_flag` is doing:

```
┌──(zham㉿kali)-[~/shop]
└─$ objdump -d -M intel shop --disassemble=main.get_flag
080d4440 <main.get_flag>:
 80d4440:	...
 80d4459:	lea    eax, ds:0x80fd683        ; "flag.txt"
 80d445f:	mov    [esp], eax
 80d4462:	mov    [esp+0x4], 0x8           ; len("flag.txt") = 8
 80d446a:	call   80d3040 <io/ioutil.ReadFile>
 ...
 80d44c3:	... fmt.Println ...
 80d4529:	mov    [esp], 0x0
 80d4530:	call   80acc50 <os.Exit>
```

Yep — it opens `flag.txt` and prints the bytes. To call this we need to make the buy code path actually reach the `call main.get_flag` instruction in `main.menu`.

### 3. Read the buy code in main.menu and find the bad check

I want to see exactly what the buy code does to my wallet. Let me dump `main.menu`:

```
┌──(zham㉿kali)-[~/shop]
└─$ objdump -d -M intel shop --disassemble=main.menu > /tmp/menu.asm
```

The relevant block, lightly cleaned up, is:

```
 80d3e4c:	mov    eax, [esi+ebx*1+0x8]     ; eax = inventory[item].price   (e.g. 10)
 80d3e50:	mov    ebp, [esp+0xbc]          ; ebp = my current coins       (e.g. 40)
 80d3e57:	cmp    ebp, eax                 ; cmp coins, price              (40 vs 10)
 80d3e59:	jl     80d3ee3                  ; if coins < price → "Not enough money."
 ...
 80d3e5f:	sub    edi, ecx                 ; count -= quantity
 80d3e61:	mov    [esi+ebx*1+0xc], edi     ; save new count
 ...
 80d3e7b:	mov    edi, [esi+eax*1+0x8]     ; edi = price
 80d3e83:	mov    edx, [edx]               ; edx = quantity
 80d3e85:	imul   edi, edx                 ; edi = price * quantity
 80d3e88:	sub    ebp, edi                 ; coins -= price * quantity
```

Read that slowly. The check on line `80d3e57` is comparing my `coins` to the **price of one item**, not to `price * quantity`. The actual deduction (`coins -= price * quantity`) happens *after* the check. So if the price is 10 and I try to buy 100 of them, the code does:

- check `coins (40) >= price (10)` → TRUE, so we do not hit "Not enough money."
- count -= 100 → count becomes `12 - 100 = -88` (signed, so it underflows)
- coins -= `10 * 100` → coins becomes `40 - 1000 = -960`

But what if `quantity` is **negative**? `quantity = -100` makes the *check* still pass (`-100 <= count`, so `cmp ecx, edi; jle` succeeds). Then:

- count -= -100 → count = `12 - (-100) = 112` (shop stock *increases*)
- coins -= `10 * -100` → coins = `40 - (-1000) = 1040` (we get paid 1000 coins)

That is the bug. Negative quantities flip both the count update and the wallet update, so buying `-N` is equivalent to *selling* `-N` items to the shop — and the shop pays us full price.

### 4. Connect to the remote instance and try buying -100 of quiches

```
┌──(zham㉿kali)-[~/shop]
└─$ nc wily-courier.picoctf.net 54593
Welcome to the market!
=====================
You have 40 coins
	Item		Price	Count
(0) Quiet Quiches	10	12
(1) Average Apple	15	8
(2) Fruitful Flag	100	1
(3) Sell an Item
(4) Exit
Choose an option: 0
How many do you want to buy? -100
You have 1040 coins
	Item		Price	Count
(0) Quiet Quiches	10	112
(1) Average Apple	15	8
(2) Fruitful Flag	100	1
(3) Sell an Item
(4) Exit
Choose an option: 
```

Wallet jumped from 40 to 1040. Shop stock for quiches jumped from 12 to 112. Both make sense if you read the bug above.

### 5. Buy the Fruitful Flag and capture the flag

```
Choose an option: 2
How many do you want to buy? 1
Flag is:  [112 105 99 111 67 84 70 123 98 52 100 95 98 114 111 103 114 97 109 109 101 114 95 48 102 100 98 51 53 56 48 51 125 10]
```

The flag is printed as a list of decimal byte values. Convert them to ASCII:

```
┌──(zham㉿kali)-[~/shop]
└─$ python3 -c "
vals = [112, 105, 99, 111, 67, 84, 70, 123, 98, 52, 100, 95, 98, 114,
        111, 103, 114, 97, 109, 109, 101, 114, 95, 48, 102, 100, 98,
        51, 53, 56, 48, 51, 125]
print(''.join(chr(v) for v in vals))
"
picoCTF{b4d_brogrammer_0fdb35803}
```

The flag is `picoCTF{b4d_brogrammer_0fdb35803}`.

### 6. Sanity-check: try the obvious attack first and confirm it does NOT work

```
Choose an option: 2
How many do you want to buy? 1
Not enough money.
You have 40 coins
```

Buying 1 flag with 40 coins fails as expected — the price is 100, so `coins < price` triggers the "Not enough money" branch. We had to push the wallet above 100 first.

### 7. Sanity-check: try positive sell-more-than-you-have and confirm it does NOT work either

A natural second guess is "what if I sell 100 of an item I have 0 of?" — that *also* sounds like an edge case:

```
Choose an option: 3
Your inventory
(0) Quiet Quiches	10	0
(1) Average Apple	15	0
(2) Fruitful Flag	100	0
What do you want to sell? 0
How many? 100
Hey you don't have that many on your cart! What kind of scam is this?
You have 40 coins
```

Wallet did *not* change. The scam message is real here — the sell function prints the message and falls through, but on its way out it writes back the *original* coins (the buy function we exploited earlier does not even fall through; it never enters the guard at all). The only working exploit is the negative-quantity buy.

---

## Alternative Solves

**A. Use `printf` + netcat instead of typing interactively.**
If you are scripting the exploit (or if your terminal does not like sending raw newlines to a Go binary), pipe everything in one go. `printf` lets you avoid the trailing newline that `echo` adds:

```
┌──(zham㉿kali)-[~/shop]
└─$ printf "0\n-100\n2\n1\n4\n" | nc wily-courier.picoctf.net 54593
Welcome to the market!
=====================
You have 40 coins
	Item		Price	Count
(0) Quiet Quiches	10	12
(1) Average Apple	15	8
(2) Fruitful Flag	100	1
(3) Sell an Item
(4) Exit
Choose an option: 
How many do you want to buy?
You have 1040 coins
...
Choose an option: 
How many do you want to buy?
Flag is:  [112 105 99 111 67 84 70 123 98 52 100 95 98 114 111 103 114 97 109 109 101 114 95 48 102 100 98 51 53 56 48 51 125 10]
```

The trailing `4` exits the shop after we collect the flag so `nc` returns cleanly.

**B. Use pwntools (Python) to script the entire exploit.**
The same five lines become five `sendline` calls when you want a reproducible script:

```
┌──(zham㉿kali)-[~/shop]
└─$ nano shop_solve.py
```

Paste:

```python
from pwn import remote

sh = remote("wily-courier.picoctf.net", 54593)
sh.recvuntil(b"option:")
sh.sendline(b"0")          # option 0 = buy
sh.sendline(b"-100")       # quantity = -100, makes coins = 1040
sh.recvuntil(b"option:")
sh.sendline(b"2")          # option 2 = Fruitful Flag
sh.sendline(b"1")          # quantity = 1
sh.recvuntil(b"is:")
line = sh.recvline().decode().strip()
nums = [int(x) for x in line.strip("[]").split()]
flag = "".join(chr(n) for n in nums)
print(flag)                # picoCTF{b4d_brogrammer_0fdb35803}
sh.close()
```

Save and exit with Ctrl+O, Enter, Ctrl+X.

```
┌──(zham㉿kali)-[~/shop]
└─$ python3 shop_solve.py
[+] Opening connection to wily-courier.picoctf.net on port 54593
picoCTF{b4d_brogrammer_0fdb35803}
```

This is the form you want for any future Shop-style challenge: hard-code the four inputs, wait for the `option:` prompts, decode the byte list at the end. It will survive any instance the picoCTF platform spins up.

**C. Try the same exploit on the local binary by stubbing `flag.txt`.**
If the remote instance is down and you only have the binary, you can still watch the exploit run end-to-end. The binary just calls `os.Open("flag.txt")` — drop a file with that name in the same directory and the same `printf` pipe prints your fake flag:

```
┌──(zham㉿kali)-[~/shop]
└─$ echo "picoCTF{local_test}" > flag.txt

┌──(zham㉿kali)-[~/shop]
└─$ printf "0\n-100\n2\n1\n4\n" | ./shop
Welcome to the market!
=====================
You have 40 coins
...
How many do you want to buy?
Flag is:  [112 105 99 111 67 84 70 123 108 111 99 97 108 95 116 101 115 116 125 10]
```

The decimal bytes still decode to the string we wrote. Useful for proving the exploit works without needing the remote server to be live.

---

## What Happened Internally

1. `_rt0_go` (Go's `_start`) set up the goroutine, called `runtime.main`, which called `main.main` at `0x80d3XXX`.
2. `main.main` created the player wallet (`coins = 40`), the shop inventory slice `[Quiet Quiches $10 x12, Average Apple $15 x8, Fruitful Flag $100 x1]`, the player inventory slice of zeros, and called `main.openShop()`.
3. `main.openShop` printed the welcome banner via `fmt.Println` and the item table via `fmt.Printf`, then entered the loop by calling `main.menu()`.
4. `main.menu` ran the menu loop. On the first iteration it called `fmt.Scanf` with format string `"%d"` and read our `0` into the option variable. It `cmp`'d the option against `3` (the index of "Sell an Item") — not equal, not less than 3, so it fell through to the buy branch.
5. The buy branch read our `-100` into the quantity variable with another `fmt.Scanf("%d", &qty)`. It bounds-checked the item index (0 < len(inventory) → OK), bounds-checked the quantity (`cmp qty, count`; `jle` → `-100 <= 12` is true, so we skip the "No more X" message).
6. The buy math ran: `eax = inventory[0].price = 10`, `ebp = coins = 40`. The check `cmp ebp, eax; jl not_enough_money` was `40 >= 10` → true, no rejection. Then `count -= qty` → `12 - (-100) = 112` (stock grew). Then `coins -= price * qty` → `40 - (10 * -100) = 1040` (we got paid).
7. The menu loop printed the new wallet and went back to the top. We typed `2` (Fruitful Flag index), `1` (quantity). The buy path ran again: `coins (1040) >= price (100)` → true, `count (1) -= 1 = 0`, `coins -= 100 * 1 = 940`. The success branch then jumped to `main.get_flag`.
8. `main.get_flag` opened `flag.txt`, read the bytes, called `main.check` to verify the error was nil, and printed `Flag is: ` followed by the byte slice. It then called `os.Exit(0)`, so the connection closed cleanly.

---

## Tools Used

| Tool             | Why I used it                                                  |
|------------------|----------------------------------------------------------------|
| Kali Linux (VM)  | My standard CTF environment.                                   |
| `file`           | Confirmed the artifact is a 32-bit ELF, Go, not stripped.      |
| `chmod +x`       | Made the binary executable for the local test.                 |
| `nm`             | Listed every Go symbol so I could find `main.get_flag` fast.  |
| `objdump -d -M intel` | Disassembled `main.get_flag`, `main.menu`, `main.sell` and traced the buy/sell math. |
| `nc wily-courier.picoctf.net 54593` | Connected to the remote instance and ran the exploit. |
| `printf "0\n-100\n2\n1\n4\n" \| nc` | The actual solve — pipe the four inputs and one exit command. |
| `python3 -c "..."` | Decoded the `Flag is: [...]` decimal byte list back to ASCII. |
| `nano shop_solve.py` | Wrapped the whole exploit in a reusable pwntools script (Alternative Solve B). |
| `pwntools` (`from pwn import remote`) | Scripted the four-step exploit for reproducibility. |

---

## Key Takeaways

- **The hint is the solve.** "Always check edge cases when programming" means *the author knows the bug is at the boundary of an input*. Try `0`, `-1`, the largest valid number, and the smallest negative number before you do anything else. In this challenge the bug is that the buy code only checks `coins >= price` (one item) instead of `coins >= price * quantity`. With a positive quantity that is fine because every integer in range passes — but with a negative quantity it becomes a wallet-inflator.
- **Negative numbers are a feature, not a bug, in signed math.** `a - (-b)` is `a + b`. The shop's `count -= quantity` and `coins -= price * quantity` lines both happily flip signs. Any time you see a CTF program that adds/subtracts using user-controlled quantities, immediately try `-1` and see if the wallet moves the "wrong" way.
- **Go binaries are not scary — `nm` and `strings` do most of the work.** Because the binary is `not stripped`, every Go function (`main.menu`, `main.sell`, `main.get_flag`, ...) is a named symbol. Find the symbol you care about (`main.get_flag`), look at where it is called from (`grep -n call main.get_flag` in the disassembly), and read the 20 instructions around that call site. You do not need to understand the whole program.
- **`fmt.Println([]byte)` prints decimal bytes, not the string.** Every picoCTF challenge that prints the flag this way is forcing you to do the ASCII conversion yourself. Memorise `python3 -c "print(''.join(chr(n) for n in [...]))"` — you will use it again.
- **The flag's leet:** `b4d_brogrammer` reads as **bad brogrammer** (`b→b`, `4→a`, `d→d`, `_→_`, `b→b`, `r→r`, `o→o`, `g→g`, `r→r`, `a→a`, `m→m`, `m→m`, `e→e`, `r→r`). The author is poking fun at themselves — a brogrammer is a stereotype of a coder who skips input validation, ships the happy path, and ships the bug. The challenge is literally named *Shop* and the lesson is "validate your inputs, bro." The trailing hex (`0fdb35803` here) is unique per challenge instance, so your exact suffix will differ from mine — the wordplay up to the underscore is what stays the same.
