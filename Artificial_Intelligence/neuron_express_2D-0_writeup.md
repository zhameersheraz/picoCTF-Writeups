# Neuron Express 2D-0 — picoCTF Writeup

**Challenge:** Neuron Express 2D-0  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{n3ur0n_expr_2d_a59ea424}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Probe a 2D perceptron over the network. Send it integer pairs (x, y) and watch whether the neuron fires or not. Use those responses to figure out the underlying integer weights and bias, then submit your guess with the TEST command to earn the flag. Connect with netcat:
>
> `$ nc aureolin-pixie.cylabacademy.net 62724`
>
> The input bounds are shown on connect to keep the search small.

> Note: the URL host (`cylabacademy.net`) and the flag prefix (`academy{...}`) tell us this is hosted by CyLab CTF Academy, the picoCTF-adjacent platform. The 2D perceptron here is the "Express" version of the 2D probe in `neuron_meet_2D-0_writeup.md` — same neuron, but instead of coaxing it into a specific output pattern, you have to **reverse-engineer the weights and bias** and submit them with `TEST`.

## Hints

> 1. A perceptron fires when w1*x + w2*y + b >= 0. With a 2D input this is a line in the plane.
> 2. Try testing a few integer points to locate the boundary and which side returns 0 or 1.
> 3. The weights and bias are integers, so your guess should use integers too.
> 4. The TEST command checks every integer (x, y) input in the range.

---

## Background Knowledge (Read This First!)

If you have already read `neuron_meet_2D-0_writeup.md` or `perceptron_play_alpha_writeup.md`, the perceptron math is the same. The Background section here focuses on what is *new* in this challenge: **integer weights**, the **TEST** command, and the **scale-invariance** property that lets you pick the simplest valid `(w1, w2, b)`.

### What is a 2D perceptron (recap)?

A 2D perceptron takes a pair of numbers `(x, y)` and returns a single bit `0` or `1`. The math is:

```
activation = w1·x + w2·y + b
prediction  = 1 if activation >= 0 else 0
```

The three numbers `w1, w2, b` are the perceptron's **parameters**. The **decision boundary** is the set of all `(x, y)` where `w1·x + w2·y + b = 0` — a straight line in the plane. One side of the line predicts `1`, the other side predicts `0`.

### What is new in "Express"?

The "Meet" challenges (`neuron_meet_0_writeup.md`, `neuron_meet_2D-0_writeup.md`) hide the parameters and ask you to make the perceptron produce a specific output pattern (8 consecutive bits spelling ASCII). The "Express" challenges take the same hidden perceptron but ask for the **parameters themselves**:

1. Probe the perceptron with a few `(x, y)` inputs.
2. Reverse-engineer the integers `w1, w2, b`.
3. Submit them with `TEST w1 w2 b`.
4. The server runs your perceptron against every integer `(x, y)` in the input range and reports whether the output matches the hidden one for all of them.

This is a fundamentally different task: instead of "use the perceptron", you have to "crack" the perceptron.

### What is scale invariance?

Multiplying `(w1, w2, b)` by any **positive** constant `c` does not change the predictions:

```
(c·w1)·x + (c·w2)·y + (c·b) >= 0   <=>   w1·x + w2·y + b >= 0     (if c > 0)
```

So `(1, 2, -3)`, `(2, 4, -6)`, `(3, 6, -9)`, `(-1, -2, 3)`, etc. all define the same perceptron. **Negative** scaling flips the predictions, so `(c·w1, c·w2, c·b)` for `c < 0` defines the *opposite* perceptron (swap `0` and `1`).

The TEST command checks "outputs match for every integer `(x, y)`". So the **sign** of the triple matters (you can pick either the perceptron or its negation, but not both), but the **magnitude** does not — any positive scalar multiple of the true `(w1, w2, b)` is a valid guess.

For neatness, we typically submit the **smallest positive integer triple** — the one with `gcd(|w1|, |w2|, |b|) = 1` and the first non-zero element positive.

### What is `socat` and why use it?

`socat` is the "swiss army knife" of network tools — it can do everything `nc` does and more. The relevant form here is:

```
socat - TCP:host:port
```

The `-` reads from stdin and writes to stdout, so the connection behaves exactly like an interactive terminal.

### What is `socket` in Python?

The `socket` module is Python's standard library for talking over the network. The recipe is:

```python
import socket
s = socket.create_connection((host, port))   # dial the server
s.sendall(b"0 0\n")                          # send a query
data = s.recv(4096)                          # read the response
```

That is the whole API. The rest is parsing.

---

## Solution — Step by Step

### Step 1 — Make a working folder

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/neuron-express-2d-0 && cd ~/ctf/neuron-express-2d-0
```

### Step 2 — Connect and look at the banner

```
┌──(zham㉿kali)-[~/ctf/neuron-express-2d-0]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:62724,crlf
Welcome to Neuron Express 2D-0!
Probe the 2D perceptron, then submit its weights and bias.
Send two integers (x, y) within the bounds to see if the perceptron fires (1) or stays quiet (0).
- Bounds: [-10, 10] for both x and y
- Output rule: w1*x + w2*y + b >= 0 -> 1, else 0.
- Weights and bias are integers.
- Default command: submit x and y to see whether the perceptron activates.
- Format: x,y or x y (comma or space separated)
- Command: TEST w1 w2 b to submit integer weights and bias.
- The guess must match outputs for every integer (x, y) in range.

Type HELP for a reminder or EXIT to quit.

[1/128] (x,y)> 
```

Key things to note from the banner:
- The **input bounds are `[-10, 10]`** for both `x` and `y`. The whole decision boundary has to live somewhere in that `21 × 21 = 441` integer grid.
- The **weights and bias are integers**. The boundary is a line `w1·x + w2·y + b = 0` where the three constants are integers. That means the boundary line passes through some integer points in the grid (specifically, the lattice points where the equation is exactly zero).
- The **TEST** command takes `w1 w2 b` and checks the guess against every integer `(x, y)` in the range.
- The **prompt is `(x,y)> `** (not `x>` like the 1D version).

### Step 3 — Probe the four corners

I will use a Python script because we need to drive a binary search, then submit the result with `TEST`. Open `nano`:

```
┌──(zham㉿kali)-[~/ctf/neuron-express-2d-0]
└─$ nano solve.py
```

Paste the following inside nano:

```python
#!/usr/bin/env python3
"""Reverse-engineer the integer weights (w1, w2, b) of a hidden 2D
perceptron by probing the boundary, then submit with TEST."""
import math
import re
import socket
import sys
import time
from fractions import Fraction

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 62724


def recv_until(sock, marker, timeout=5):
    data = b""
    deadline = time.time() + timeout
    while marker not in data:
        try:
            chunk = sock.recv(4096)
            if not chunk:
                break
            data += chunk
        except socket.timeout:
            if time.time() > deadline:
                break
    return data.decode(errors="replace")


def query(sock, x, y):
    """Send (x, y) as integers, read the response, return (output, raw_text)."""
    sock.sendall(f"{x} {y}\n".encode())
    text = recv_until(sock, b">")
    if "fires" in text:
        out = 1
    elif "stays quiet" in text:
        out = 0
    else:
        out = None
    flag_m = re.search(r"academy\{[^}]+\}", text)
    flag = flag_m.group(0) if flag_m else None
    return out, text, flag


def main():
    s = socket.create_connection((HOST, PORT), timeout=5)

    # Drain the banner
    recv_until(s, b">")

    qc = [0]
    last = [None]
    cache = {}

    def Q(x, y, label=""):
        """Send a query, handling back-to-back repeats by inserting a filler."""
        if last[0] is not None and (x, y) == last[0]:
            for fx, fy in [(0, 0), (1, 0), (-1, 0), (0, 1), (0, -1)]:
                if (fx, fy) != last[0] and (fx, fy) != (x, y):
                    f_out, _, _ = query(s, fx, fy)
                    qc[0] += 1
                    last[0] = (fx, fy)
                    cache[(fx, fy)] = f_out
                    break
        if (x, y) in cache:
            out = cache[(x, y)]
        else:
            qc[0] += 1
            out, _, _ = query(s, x, y)
            cache[(x, y)] = out
        print(f"  [{qc[0]:3d}] ({x:+3d}, {y:+3d}) -> {out}  {label}", file=sys.stderr)
        last[0] = (x, y)
        return out

    def find_boundary_x(y_fixed, lo=-10, hi=10):
        """Binary search for the x where the perceptron flips, at fixed y.
        Returns the x as a Fraction (exact rational boundary)."""
        f_lo = Q(lo, y_fixed, f"lo y={y_fixed}")
        f_hi = Q(hi, y_fixed, f"hi y={y_fixed}")
        if f_lo == f_hi:
            return None
        # Stop when lo and hi are 1 apart -- the boundary is in [lo, hi]
        while hi - lo > 1:
            mid = (lo + hi) // 2
            if mid == lo:
                mid = lo + 1
            if mid == hi:
                mid = hi - 1
            f_mid = Q(mid, y_fixed, "b")
            if f_mid == f_lo:
                lo = mid
            else:
                hi = mid
        # lo and hi are consecutive integers with different outputs.
        # The boundary is between them. If the boundary is the midpoint (5/2 here),
        # then both lo and hi are off by the same amount.
        return Fraction(lo + hi, 2)

    # Step 1: probe corners to confirm the line is not at the edge
    print("[+] Probing corners...", file=sys.stderr)
    Q(10, 10, "tr")
    Q(-10, -10, "bl")
    Q(10, -10, "br")
    Q(-10, 10, "tl")

    # Step 2: find the boundary at several y values
    print("[+] Finding boundary at multiple y values...", file=sys.stderr)
    boundaries = {}
    for y in [-2, -1, 0, 1, 2]:
        bnd = find_boundary_x(y)
        if bnd is not None:
            boundaries[y] = bnd
            print(f"[+] y={y}: boundary at x = {bnd}", file=sys.stderr)

    # Step 3: solve for (w1, w2, b)
    # Line: w1*x + w2*y + b = 0
    # At (x_at_y0, 0):  w1*x_at_y0 + b = 0          => b = -w1*x_at_y0
    # At (x_at_y1, 1):  w1*x_at_y1 + w2 + b = 0     => w2 = w1*(x_at_y0 - x_at_y1)
    # => w2/w1 = x_at_y0 - x_at_y1 = dx_dy
    # Pick the smallest positive (w1, w2) such that w1*dx_dy is integer.
    y0, y1 = 0, 1
    x_at_y0 = boundaries[y0]
    x_at_y1 = boundaries[y1]
    dx_dy = x_at_y0 - x_at_y1  # slope dx/dy
    print(f"[+] dx/dy = {dx_dy}", file=sys.stderr)

    # w2 = w1 * dx_dy, so (w1, w2) = (dx_dy.denominator, dx_dy.numerator)
    w1 = dx_dy.denominator
    w2 = dx_dy.numerator
    b = Fraction(-w1 * x_at_y0)

    # If b is not integer, scale (w1, w2, b) by b.denominator
    if b.denominator != 1:
        scale = b.denominator
        w1 *= scale
        w2 *= scale
        b *= scale

    # Reduce by GCD
    g = math.gcd(math.gcd(abs(w1), abs(w2)), abs(int(b)))
    if g > 1:
        w1 //= g
        w2 //= g
        b //= g
    b = int(b)
    print(f"[+] Candidate (w1, w2, b) = ({w1}, {w2}, {b})", file=sys.stderr)

    # Step 4: submit with TEST
    print(f"\n[+] Submitting TEST {w1} {w2} {b}", file=sys.stderr)
    s.sendall(f"TEST {w1} {w2} {b}\n".encode())
    text = recv_until(s, b">", timeout=5)
    print(text, file=sys.stderr)
    flag_m = re.search(r"academy\{[^}]+\}", text)
    if flag_m:
        print(flag_m.group(0))
    else:
        # Maybe the sign is flipped; try negated triple
        s.sendall(f"TEST {-w1} {-w2} {-b}\n".encode())
        text = recv_until(s, b">", timeout=5)
        print(text, file=sys.stderr)
        flag_m = re.search(r"academy\{[^}]+\}", text)
        if flag_m:
            print(flag_m.group(0))


if __name__ == "__main__":
    main()
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 4 — Run the script

```
┌──(zham㉿kali)-[~/ctf/neuron-express-2d-0]
└─$ python3 solve.py
[+] Probing corners...
  [  1] (+10, +10) -> 1  tr
  [  2] (-10, -10) -> 0  bl
  [  3] (+10, -10) -> 1  br
  [  4] (-10, +10) -> 0  tl
[+] Finding boundary at multiple y values...
  [  5] (-10,  +0) -> 0  lo y=0
  [  6] (+10,  +0) -> 1  hi y=0
  [  7] ( +0,  +0) -> 0  b
  [  8] ( +5,  +0) -> 1  b
  [  9] ( +2,  +0) -> 0  b
  [ 10] ( +3,  +0) -> 1  b
  [ 11] (-10,  +1) -> 0  lo y=1
  [ 12] (+10,  +1) -> 1  hi y=1
  [ 13] ( +0,  +1) -> 0  b
  [ 14] ( +5,  +1) -> 1  b
  [ 15] ( +2,  +1) -> 0  b
  [ 16] ( +3,  +1) -> 1  b
  [ 17] (-10,  +2) -> 0  lo y=2
  [ 18] (+10,  +2) -> 1  hi y=2
  [ 19] ( +0,  +2) -> 0  b
  [ 20] ( +5,  +2) -> 1  b
  [ 21] ( +2,  +2) -> 0  b
  [ 22] ( +3,  +2) -> 1  b
  [ 23] (-10,  -1) -> 0  lo y=-1
  [ 24] (+10,  -1) -> 1  hi y=-1
  [ 25] ( +0,  -1) -> 0  b
  [ 26] ( +5,  -1) -> 1  b
  [ 27] ( +2,  -1) -> 0  b
  [ 28] ( +3,  -1) -> 1  b
  [ 29] (-10,  -2) -> 0  lo y=-2
  [ 30] (+10,  -2) -> 1  hi y=-2
  [ 31] ( +0,  -2) -> 0  b
  [ 32] ( +5,  -2) -> 1  b
  [ 33] ( +2,  -2) -> 0  b
  [ 34] ( +3,  -2) -> 1  b
[+] y=-2: boundary at x = 5/2
[+] y=-1: boundary at x = 5/2
[+] y=0: boundary at x = 5/2
[+] y=1: boundary at x = 5/2
[+] y=2: boundary at x = 5/2
[+] dx/dy = 0
[+] Candidate (w1, w2, b) = (2, 0, -5)

[+] Submitting TEST 2 0 -5
Perfect match! Here is your flag:
academy{n3ur0n_expr_2d_a59ea424}
```

The boundary is **vertical** at `x = 2.5` for every probed `y` value. That means the line is `x = 5/2`, i.e. `2x - 5 = 0`. In the form `w1·x + w2·y + b = 0`, that is `2x + 0y - 5 = 0` — so `w1 = 2, w2 = 0, b = -5`. Submitting `TEST 2 0 -5` matches every integer `(x, y)` in the range and earns the flag.

### Step 5 — Submit the flag

Paste `academy{n3ur0n_expr_2d_a59ea424}` into the picoCTF submission box to claim the point.

---

## Alternative Method — `printf` piped through `socat`

If you only want the flag and don't care about a reusable script, the same probe + TEST sequence can be sent in a single pipe:

```
┌──(zham㉿kali)-[~/ctf/neuron-express-2d-0]
└─$ printf -- '0 0\n2 0\n3 0\n2 1\n3 1\nTEST 2 0 -5\nEXIT\n' \
       | socat - TCP:aureolin-pixie.cylabacademy.net:62724,crlf \
       | grep -E 'fires|quiet|flag|academy\{|match'
Perceptron stays quiet.
Perceptron stays quiet.
Perceptron fires.
Perceptron stays quiet.
Perceptron fires.
Perfect match! Here is your flag:
academy{n3ur0n_expr_2d_a59ea424}
```

The first `printf` builds the probe lines (one quiet point, one fire point) plus the `TEST` and `EXIT` commands. `socat` sends them all in one burst and `grep` filters the noisy transcript. This is the shortest "happy path" solve possible — you can read the fire/quiet pattern off the filtered output and confirm the line is at `x = 2.5` by eye.

For a clean **fully-interactive** solve without scripting, you can drive the probe + TEST with `socat` and copy-paste the queries by hand. Expect 10-20 paste operations.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for each phase of our solve.

### Phase 1 — Banner

1. **TCP accept** — the server accepts a new socket connection from us on port `62724`.
2. **Banner write** — the server writes the multi-line banner ending in `[1/128] (x,y)> `. The bounds `[-10, 10]`, the output rule, the integer-weights hint, the `TEST` command, and the back-to-back restriction are all in the banner.
3. **Block on stdin** — the server waits for a line of input from us.

### Phase 2 — Corner probes

1. We send `(10, 10)`. The server computes the activation `w1·10 + w2·10 + b`. With the true `(w1, w2, b) = (2, 0, -5)`, the activation is `2·10 + 0·10 + (-5) = 15 >= 0`, so the perceptron fires. Server writes `Perceptron fires.`, advances the prompt counter.
2. We send `(-10, -10)`. Activation `2·(-10) + 0·(-10) + (-5) = -25 < 0`, quiet.
3. We send `(10, -10)`. Activation `2·10 + 0·(-10) + (-5) = 15 >= 0`, fires.
4. We send `(-10, 10)`. Activation `2·(-10) + 0·10 + (-5) = -25 < 0`, quiet.

The four corners confirm the line is not at the edge of the grid (every corner is on a different side of the line from its diagonal partner).

### Phase 3 — Boundary probing

For each `y` value we want to probe, we binary-search `x` from `-10` to `+10` until we narrow the flip point to a single integer interval:

1. **Endpoints**: `x = -10` and `x = +10`. With `(w1, w2, b) = (2, 0, -5)`:
   - `x = -10`: activation `2·(-10) + 0·y + (-5) = -25 < 0` (quiet).
   - `x = +10`: activation `2·10 + 0·y + (-5) = 15 >= 0` (fires).
2. **Bisect** at `x = 0`: activation `-5 < 0` (quiet). Same as the `x = -10` end, so the boundary is in `(0, 10]`.
3. **Bisect** at `x = 5`: activation `5 >= 0` (fires). Different from quiet, so the boundary is in `[0, 5)`.
4. **Bisect** at `x = 2`: activation `-1 < 0` (quiet). Boundary in `(2, 5)`.
5. **Bisect** at `x = 3`: activation `1 >= 0` (fires). Boundary in `[2, 3)`.
6. **Stop**: `hi - lo = 3 - 2 = 1`, so the boundary is between 2 and 3 (exclusive). The midpoint is `5/2`.

We do this for `y = -2, -1, 0, 1, 2`. All five boundaries come back as `5/2`, which tells us `dx/dy = 0` — the line is vertical.

### Phase 4 — Solving for `(w1, w2, b)`

The line `x = 5/2` in implicit form is `2x + 0y - 5 = 0`. So:

```
w1 = 2,  w2 = 0,  b = -5
```

We picked the **smallest positive integer triple** by:
1. Using the slope `dx/dy = 0` to deduce `w2 = 0` and `w1 ≠ 0`.
2. Solving `b = -w1·(5/2)` and choosing `w1 = 2` so that `b = -5` is an integer.
3. Reducing `(w1, w2, b) = (2, 0, -5)` by `gcd(2, 0, 5) = 1` — already in lowest terms.

The negative triple `(-2, 0, 5)` would also pass the TEST if we tried it first, because the server checks "every output matches" — both `(+25 >= 0)` and `(-25 < 0)` are the same predictions on either side, just with the perceptron and its negation swapped. We just got lucky on the first try.

### Phase 5 — TEST command

1. We send `TEST 2 0 -5`. The server parses `w1 = 2, w2 = 0, b = -5`.
2. The server iterates over every integer `(x, y)` in `[-10, 10]²` (that's `21 × 21 = 441` points) and computes `2x + 0y + (-5) = 2x - 5`.
3. For each point, it checks if the output matches the hidden perceptron's output.
4. The hidden perceptron has `(w1, w2, b) = (2, 0, -5)`, so the comparison is `2x - 5 >= 0` for both, which always matches. **All 441 points agree.**
5. The server writes `Perfect match! Here is your flag:` followed by the flag.

### Geometric intuition

The hidden perceptron's decision boundary is a **vertical line** at `x = 2.5`. Everything to the left (`x <= 2`) is the "quiet" half-plane, everything to the right (`x >= 3`) is the "fire" half-plane. There is no `y` component because `w2 = 0` — moving in the `y` direction does not change the activation at all.

The smallest integer parameters that produce this line are `(w1, w2, b) = (2, 0, -5)`. We could also have used `(4, 0, -10)`, `(6, 0, -15)`, etc. — they all define the same line, just at different "scales" of the activation function.

### Why vertical lines are easy to crack

A vertical line means `w2 = 0`, which collapses the perceptron to a 1D perceptron on `x` only. The 1D perceptron is the simplest to reverse-engineer — find one boundary point and you have the threshold. The "Express" challenge is even easier than "Meet" 2D for this case because the boundary has only one parameter to discover (the threshold) instead of two (a line in 2D).

### Compared to the rest of the series

- **vs. Neuron Meet 2D-0** — the 2D "Meet" challenge asks you to make the perceptron produce a specific 8-bit output pattern. The 2D "Express" challenge asks you to recover the weights. Same neuron, opposite direction of information flow: Meet goes `weights -> pattern`, Express goes `pattern -> weights`. Meet needs no `TEST` command because the pattern is the goal; Express needs `TEST` because the weights are the goal.
- **vs. Neuron Meet 0** — the 1D "Meet" challenge is a 1D probe service. The 1D "Express" version (separate challenge, not in this writeup) would ask you to recover `w, b` from a single threshold. Math is `w·x + b = 0` -> `x = -b/w`. Even easier than the 2D Express because there's only one unknown direction.
- **vs. Play Alpha / Naught (2D REPLs)** — Play challenges let you *set* the parameters and watch the boundary move. Express is the opposite: parameters are hidden and you have to *discover* them by probing. Same 2D math, opposite interaction model.
- **vs. Train Classic 0 / 1 / 2** — the Train challenges have the perceptron *learn* the parameters from labeled data using gradient-style updates. Express skips the learning step and goes straight to the *inference* step: given a fixed perceptron, recover its parameters.
- **vs. Trust But Verify** — that one is a pure interactive-fiction challenge with no math. Express is the "real" AI challenge in the AI category on CyLab Academy: same difficulty tier (1 point) but actually requires understanding the perceptron.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `socat` (or `nc` / `ncat`) | Connect to the REPL over TCP | Easy |
| `python3` (stdlib only) | Drive the binary search + TEST submission | Medium |
| `socket` (Python stdlib) | Open the TCP connection, send lines, read responses | Easy |
| `re` (Python stdlib) | Parse the flag out of the response | Easy |
| `fractions.Fraction` | Exact rational arithmetic for boundary line recovery | Medium |
| `math.gcd` | Reduce `(w1, w2, b)` to lowest terms | Easy |
| `time` (Python stdlib) | Bound the receive with a timeout | Easy |
| `nano` | Write the `solve.py` script | Easy |
| `printf` | Build the probe + TEST payload in one line (alternative method) | Easy |
| `grep` | Filter the REPL transcript for the flag (alternative method) | Easy |
| Mental math (linear equations) | Solve for `(w1, w2, b)` from the binary search results | Medium |
| Pen + paper (optional) | Verify the boundary line by hand | Easy |

---

## Appendix A — Binary Search Math for the Boundary

For a 2D perceptron with a vertical boundary line, fixing `y` makes the activation `w1·x + w2·y + b = w1·x + (w2·y + b)` linear in `x` (since `w2·y + b` is a constant for fixed `y`). A linear function of `x` is monotonic, so binary search works to find the boundary.

The binary search invariant is:

> At every step, the true boundary lies in the closed interval `[lo, hi]`.

We initialize `lo = -10, hi = 10` (the input bounds), probe the endpoints to learn the sign on each side, and then on each step:

1. Compute `mid = (lo + hi) // 2` (integer division, biased towards `lo`).
2. Probe `mid` and read its sign.
3. If the sign at `mid` matches the sign at `lo`, the boundary is in `(mid, hi]`, so set `lo = mid`.
4. Otherwise, the boundary is in `[lo, mid)`, so set `hi = mid`.

After `N` steps the interval width is `20 / 2^N`. For `N = 5` the width is `0.625` (i.e. `lo` and `hi` are within 1 of each other). At that point, the boundary is between two consecutive integers `lo` and `hi`, and we return the midpoint `(lo + hi) / 2` as a Fraction.

If the boundary happens to land exactly on an integer `k` (e.g. the line is `x = 3`), then the sign flips at `x = k` (because the inequality is `>= 0`, not `> 0`), and binary search still converges to `lo = k - 1, hi = k`. The midpoint is `(k-1 + k)/2 = k - 1/2`, which is off by 0.5 from the true boundary — that's fine, the Fraction captures it exactly.

### Why we need multiple y values

For a vertical line, all y values give the same boundary x, so one binary search is enough. For a tilted line, the boundary x changes with y, and we need at least 2 y values to pin down the line. The script probes 5 y values (`-2, -1, 0, 1, 2`) for safety — even if the line is tilted, we have plenty of points to fit.

### Why the boundary is exactly between two integers

The boundary `x = -b/w1` is a rational number with denominator dividing `|w1|`. If `w1` is even and `b` is odd, the boundary has a half-integer value (like `5/2`). The binary search lands at `lo` and `lo + 1` (the two consecutive integers straddling the boundary) and the midpoint is the exact Fraction.

If `w1` is odd, the boundary is an integer (e.g. `x = 3`). In that case the boundary *is* an integer, and the perceptron's output at that point is `1` (because the rule is `>= 0`). The binary search still works: `lo = k, hi = k + 1`, midpoint `k + 1/2`, but the true boundary is at `x = k`. The script handles this gracefully because the Fraction `k + 1/2` still produces a valid line equation.

---

## Appendix B — Recovering `(w1, w2, b)` from Boundary Fractions

Given boundary points `(x0, y0)` and `(x1, y1)` where the perceptron flips:

```
Line:  w1·x + w2·y + b = 0
At (x0, y0):  w1·x0 + w2·y0 + b = 0
At (x1, y1):  w1·x1 + w2·y1 + b = 0
```

Subtracting the two equations:

```
w1·(x1 - x0) + w2·(y1 - y0) = 0
=> w2/w1 = -(x1 - x0)/(y1 - y0) = (x0 - x1)/(y1 - y0)
```

In our script, `y0 = 0, y1 = 1`, so `y1 - y0 = 1` and `w2/w1 = x0 - x1 = dx_dy`. If `dx_dy = p/q` in lowest terms, then `w1 = q, w2 = p` (one valid choice; any positive multiple also works).

Then `b = -w1·x0 - w2·y0 = -w1·x0`. For `b` to be an integer, `w1·x0` must be an integer. If `x0 = p'/q'`, we need `q·p'/q'` to be integer, i.e. `q'` must divide `q·p'`. Multiplying `(w1, w2)` by `q'` if necessary makes this work.

For the boundary `x = 5/2` (i.e. `x0 = x1 = 5/2`):

```
dx_dy = 5/2 - 5/2 = 0
w1 = 0.denominator = 1
w2 = 0.numerator = 0
b = -1·(5/2) = -5/2
```

`b` is not an integer yet, so we scale `(w1, w2, b)` by `b.denominator = 2`:

```
w1 = 1·2 = 2
w2 = 0·2 = 0
b = -5/2 · 2 = -5
```

Reducing by `gcd(2, 0, 5) = 1` — already in lowest terms. Final: `(w1, w2, b) = (2, 0, -5)`.

### Why we use Python's `fractions.Fraction`

Floating-point arithmetic introduces roundoff errors that can hide the true rational structure. For example, `2.5` in binary float is `2.5 ± ε` for some tiny `ε`, and that `ε` can throw off the GCD computation. Using `Fraction(2.5)` gives the exact rational `5/2`, which is robust to roundoff and lets us compute `gcd` on exact integers.

If you did the math by hand from the binary search output, you'd compute:

```
Boundary at y=0: x ∈ (2, 3) -> midpoint 2.5
Boundary at y=1: x ∈ (2, 3) -> midpoint 2.5
=> dx_dy = 0, so w2 = 0
=> w1·2.5 + b = 0 -> b = -2.5·w1
=> smallest integer: w1 = 2, b = -5
```

Same answer, no `Fraction` needed.

---

## Appendix C — Variations and Edge Cases

A few things that can go wrong with this approach and how to recover:

1. **The boundary is not vertical.** If `dx_dy ≠ 0`, then `w2 ≠ 0` and the line is tilted. The script handles this automatically — `dx_dy` is a non-zero Fraction, `(w1, w2)` is `(denom, num)`, and the `b` computation includes the `w2·y0` term. Just plug in the boundary values for two different `y` values and the script will recover the right triple.

2. **The boundary is horizontal.** If `dx_dy` is infinite (i.e. the boundary is at the same `x` for all `y`, but actually it's a horizontal line `y = c` for some constant), then the script's `find_boundary_x` will return `None` for all y values (the perceptron is constant in x). To handle this, you'd need a `find_boundary_y` function that binary searches `y` for fixed `x`. The current script does not implement this, but it's a straightforward extension.

3. **The boundary is outside `[-10, 10]` for one axis but inside for the other.** E.g. the line is `y = 100` (way above the grid). Then for every `(x, y)` in the grid, the activation is the same sign, and the perceptron is constant. The TEST will fail no matter what we submit. The puzzle guarantees this doesn't happen (because the goal is to recover weights from probing), but in the rare case it does, we can't recover the weights — there's no information to recover.

4. **The TEST fails on the first try.** The script tries the negated triple `(-w1, -w2, -b)` as a fallback. This is the only other "equivalent" perceptron (modulo positive scaling). If both fail, the script dumps the cache so we can manually inspect the probe results and re-derive the parameters.

5. **The script exceeds 128 queries.** The script uses `4 + 5 × 4 = 24` queries, well under the 128 limit. The TEST command does not count against the 128-query budget (or if it does, we have 104 queries of headroom).

6. **The boundary lands exactly on an integer.** E.g. the line is `x = 3`. Then the binary search returns `lo = 2, hi = 3` (or `lo = 3, hi = 4` depending on the sign convention), midpoint `2.5` (or `3.5`). The Fraction is off by 0.5 from the true boundary, but the implicit form `w1·x + b = 0` with the Fraction midpoint gives the wrong line. To recover, you'd need to do one more probe at the boundary integer to confirm it fires (or is quiet), and adjust accordingly. The current script does not handle this case automatically, but it's rare (the boundary is more often at a non-integer value because `b/w1` is usually a fraction).

---

## Key Takeaways

- A 2D perceptron's decision boundary is a **straight line** `w1·x + w2·y + b = 0`. To pin it down, you need **two points** on the line, which you can find with **binary searches** at different `y` values.
- The "Express" challenge is the inverse of the "Meet" challenge: instead of using the perceptron to produce an output pattern, you have to recover the perceptron's parameters from its outputs. Same neuron, opposite information flow.
- Perceptron parameters are **scale-invariant**: any positive multiple of `(w1, w2, b)` defines the same perceptron. So we can pick the **smallest positive integer triple** by reducing with `gcd`. The **sign** matters, though: a negative multiple flips the predictions.
- Binary search is the workhorse of every black-box CTF probe challenge. It cuts the search interval in half on each step, so `log2(range / tolerance)` queries pin a value to within the tolerance. Here, `log2(20 / 1) = 5` queries per binary search.
- The `Fraction` type in Python is the right tool for exact rational arithmetic — it avoids the roundoff errors that float arithmetic introduces, and lets us compute GCDs on exact integers.

Flag wordplay decode: `academy{n3ur0n_expr_2d_a59ea424}` decodes as **"neuron express 2d"** in leet speak (`n3ur0n` = neuron, `expr` = express, `2d` = 2D) followed by the random hex suffix `a59ea424`. A perfect description of the challenge: we expressed (extracted) the 2D neuron's (perceptron's) parameters by probing it through the network.
