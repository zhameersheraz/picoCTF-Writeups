# Neuron Meet 2D-0 — picoCTF Writeup

**Challenge:** Neuron Meet 2D-0  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{2d_n3ur0n_m3t_892056ff}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Probe a 2D perceptron over the network. Send it pairs of numbers (x, y) and watch whether the neuron fires or not. Use those responses to figure out the decision boundary, then line up the last eight outputs to read the ASCII for p (01110000) and you'll earn the flag. Connect with netcat:
>
> `$ nc aureolin-pixie.cylabacademy.net 64768`
>
> The input bounds are shown on connect to keep the search small. You cannot submit the same (x, y) pair back-to-back.

> Note: the URL host (`cylabacademy.net`) and the flag prefix (`academy{...}`) tell us this is hosted by CyLab CTF Academy, the picoCTF-adjacent platform. The 2D perceptron here is the natural follow-up to the 1D threshold from `perceptron_play_1D_alpha_writeup.md` — same neuron, one extra input.

## Hints

> 1. A 2D perceptron fires when w1*x + w2*y + b >= 0. You need to find the decision boundary in 2D space.
> 2. Test points to determine which side of the boundary produces 0 and which produces 1.
> 3. Remember that repeated inputs are blocked, even if they would produce the same output.

---

## Background Knowledge (Read This First!)

If you have already read `perceptron_play_1D_alpha_writeup.md`, the perceptron math is the same — this challenge just gives you back the neuron as a black-box service instead of a `SET` / `ADJUST` REPL. The Background section here focuses on what is *new* in this challenge: a 2D decision boundary, the network protocol, and the ASCII-to-binary trick.

### What is a 2D perceptron?

A 2D perceptron takes two numbers `(x, y)` and returns a single bit `0` or `1`. The math is:

```
activation = w1·x + w2·y + b
prediction  = 1 if activation >= 0 else 0
```

The three numbers `w1, w2, b` are the perceptron's **parameters** — its "brain". We do not get to see them, but we can learn them indirectly by sending inputs and watching the predictions.

For a 2D perceptron, the **decision boundary** is the set of all `(x, y)` where `w1·x + w2·y + b = 0`. That equation describes a **straight line** in the plane. Everything on one side of the line is classified as `1`, everything on the other side is classified as `0`. The line itself (the boundary) is classified as `1` because the inequality is non-strict (`>= 0`).

### How do we find a line from black-box queries?

Two points are enough to pin down a line in 2D. If I can find two different `(x, y)` points that land exactly on the boundary, I know the line. So the strategy is:

1. **Fix `y = 0`.** The boundary becomes `w1·x + b = 0`, which is a single point on the x-axis. I can find it by **binary search**: probe `x = -10` and `x = +10`; if the predictions differ, the boundary is somewhere in between, and I can halve the interval repeatedly until I have the crossing point.
2. **Fix `y = 1`.** Same idea — binary search on `x` to find the new crossing.
3. I now have two points on the line: `(x0, 0)` and `(x1, 1)`. The line equation falls out from these two points.

In practice I do not even need the line equation. Once I know the boundary roughly, I can pick a point well above the line (guaranteed `1`) and a point well below the line (guaranteed `0`) and reuse them for the rest of the challenge.

### What is binary search?

Binary search is the divide-and-conquer recipe for finding a value inside a sorted range. You check the midpoint, decide which half contains the target, and recurse on that half. After `N` steps, the search interval has shrunk by a factor of `2^N`, so 40 steps can pin down a value to within `10^-12` of the original range (here, `20 / 2^40 ≈ 2e-11`). It is the single most useful algorithm in CTF probe-style challenges.

### How do I turn "fire" and "stays quiet" into a flag?

The challenge wants the **last 8 outputs** to read the binary encoding of the ASCII character `p`. ASCII for `p` is the 8-bit binary string:

```
p = 0x70 = 01110000 (binary)
```

So I need 8 consecutive perceptron outputs in the order `0, 1, 1, 1, 0, 0, 0, 0`:

| Output position | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| Required bit    | 0 | 1 | 1 | 1 | 0 | 0 | 0 | 0 |

Each position is a separate `(x, y)` query. Three of them (positions 2, 3, 4) must produce a `1`, and five of them (positions 1, 5, 6, 7, 8) must produce a `0`. I just need to find one good "fire" point and one good "quiet" point for each required bit, and send them in the right order.

### What does "no back-to-back repeats" mean?

The service refuses to process a query that uses the exact same `(x, y)` pair as the previous query — even if I want to keep getting the same output. So if I need 4 consecutive `0`s, I cannot just send `(-9, -9)` four times. I have to use **4 different** "quiet" points (e.g. `(-9, -9)`, `(-8, -9)`, `(-7, -9)`, `(-6, -9)`) and a 5th "quiet" point for position 1, plus 3 different "fire" points for positions 2, 3, 4. That's 7 distinct points total for the final 8 queries.

### What is `socat` and why use it?

`socat` is the "swiss army knife" of network tools — it can do everything `nc` does and more. The relevant form here is:

```
socat - TCP:host:port
```

The `-` reads from stdin and writes to stdout, so the connection behaves exactly like an interactive terminal. Many picoCTF writeups show `nc` but on some systems (like the Debian/Ubuntu default) you have to use `ncat` (the `nmap` version) or `socat` instead. They are all interchangeable for these challenges.

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
└─$ mkdir -p ~/ctf/neuron-meet-2d-0 && cd ~/ctf/neuron-meet-2d-0
```

### Step 2 — Connect and look at the banner

```
┌──(zham㉿kali)-[~/ctf/neuron-meet-2d-0]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:64768,crlf
Welcome to Neuron Meet 2D-0!
Probe the 2D perceptron to coax out the ASCII for 'p'.
Send two numbers (x, y) to see if the perceptron fires (1) or stays quiet (0).
- Bounds: [-10.0, 10.0] for both x and y
- Output rule: w1*x + w2*y + b >= 0 -> 1, else 0.
- No back-to-back repeats of the same (x, y) pair.
- Goal: make the last 8 outputs read 01110000 (ASCII 'p').
- Command: RESET to clear the firing history.
- Format: x,y or x y (comma or space separated)
Type HELP for a reminder or EXIT to quit.

[1/128] (x,y)>
```

Key things to note from the banner:
- The **input bounds are `[-10, 10]`** for both `x` and `y`. The whole decision boundary has to live somewhere in that `20 × 20` square.
- We get **128 queries** total. The 8 most recent are what gets checked.
- The boundary is a line: `w1·x + w2·y + b = 0`.

### Step 3 — Find the boundary by binary search

I will use a Python script because manually sending 100+ queries is painful. Open `nano`:

```
┌──(zham㉿kali)-[~/ctf/neuron-meet-2d-0]
└─$ nano solve.py
```

Paste the following inside nano:

```python
#!/usr/bin/env python3
"""Probe the Neuron Meet 2D-0 perceptron, find the boundary line, and
land the last 8 outputs on 01110000 (ASCII 'p') to earn the flag."""
import re
import socket
import sys

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 64768


def query(sock, x, y):
    """Send (x, y), read the response, return (output, raw_text)."""
    sock.sendall(f"{x:.6f} {y:.6f}\n".encode())
    buf = b""
    while b"(x,y)>" not in buf:
        chunk = sock.recv(4096)
        if not chunk:
            break
        buf += chunk
    text = buf.decode(errors="replace")
    if "fires" in text:
        out = 1
    elif "stays quiet" in text:
        out = 0
    else:
        out = None
    return out, text


def find_boundary_x(sock, y_fixed, lo=-10.0, hi=10.0, tol=1e-4, max_iter=40):
    """Binary search for the x value where the perceptron flips, at a
    fixed y. Returns the x coordinate of the crossing."""
    f_lo, _ = query(sock, lo, y_fixed)
    f_hi, _ = query(sock, hi, y_fixed)
    if f_lo == f_hi:
        return None  # boundary outside the search range for this y
    for _ in range(max_iter):
        if hi - lo < tol:
            break
        mid = (lo + hi) / 2
        f_mid, _ = query(sock, mid, y_fixed)
        if f_mid == f_lo:
            lo = mid
        else:
            hi = mid
    return (lo + hi) / 2


def main():
    s = socket.create_connection((HOST, PORT), timeout=15)

    # Drain the banner
    s.recv(8192)

    # Step 1: find the boundary at y = 0
    x_at_y0 = find_boundary_x(s, 0.0)
    print(f"[+] Boundary at y=0 is x = {x_at_y0:.4f}", file=sys.stderr)

    # Step 2: find the boundary at y = 1
    x_at_y1 = find_boundary_x(s, 1.0)
    print(f"[+] Boundary at y=1 is x = {x_at_y1:.4f}", file=sys.stderr)

    # The line passes through (x_at_y0, 0) and (x_at_y1, 1).
    # Slope: dx/dy = x_at_y1 - x_at_y0
    # So a point (x, y) is on the "fires" side iff x > x_at_y0 + (x_at_y1 - x_at_y0) * y
    # We don't actually need the line equation -- we just need to verify a few
    # candidate points on each side.

    # We know (0, 0) -> 0 (the perceptron was quiet for the origin in our
    # binary search), so points near (-, -) are quiet, points near (+, +) fire.

    # Pattern we need: 0, 1, 1, 1, 0, 0, 0, 0
    # Need 3 distinct fire points and 4 distinct quiet points.
    final_sequence = [
        (-9.0, -9.0),   # 0
        ( 9.0,  9.0),   # 1
        ( 9.0,  8.0),   # 1
        ( 9.0,  7.0),   # 1
        (-9.0, -8.0),   # 0
        (-9.0, -7.0),   # 0
        (-9.0, -6.0),   # 0
        (-9.0, -5.0),   # 0
    ]

    # Avoid a back-to-back repeat with the last binary-search query
    last = (x_at_y1, 1.0)
    for i, (x, y) in enumerate(final_sequence):
        if (x, y) == last:
            # Insert a safe filler that we know is a quiet point
            query(s, 0.0, 0.0)
        out, text = query(s, x, y)
        print(f"  [{i+1}] ({x:+.1f}, {y:+.1f}) -> {out}", file=sys.stderr)
        last = (x, y)

    # Read the final response (the flag)
    s.settimeout(5)
    final = b""
    try:
        while True:
            chunk = s.recv(4096)
            if not chunk:
                break
            final += chunk
    except socket.timeout:
        pass
    print(final.decode(errors="replace"))


if __name__ == "__main__":
    main()
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`.

### Step 4 — Run the script

```
┌──(zham㉿kali)-[~/ctf/neuron-meet-2d-0]
└─$ python3 solve.py
[+] Boundary at y=0 is x = 4.0000
[+] Boundary at y=1 is x = 2.6667
  [1] (-9.0, -9.0) -> 0
  [2] ( 9.0,  9.0) -> 1
  [3] ( 9.0,  8.0) -> 1
  [4] ( 9.0,  7.0) -> 1
  [5] (-9.0, -8.0) -> 0
  [6] (-9.0, -7.0) -> 0
  [7] (-9.0, -6.0) -> 0
  [8] (-9.0, -5.0) -> 0
Perceptron stays quiet.
Recent outputs (8/8): 01110000
Pattern matched! ASCII 'p' unlocked. Here is your flag:
academy{2d_n3ur0n_m3t_892056ff}
```

The boundary crosses the x-axis at `x = 4.0` and crosses `y = 1` at `x = 2.667`, so the line equation is `x = 4 - (4/3)·y` (slope `-4/3`, y-intercept `+3`). The final 8 outputs land cleanly on `01110000`, and the server hands over the flag.

### Step 5 — Submit the flag

Paste `academy{2d_n3ur0n_m3t_892056ff}` into the picoCTF submission box to claim the point.

---

## Alternative Method — `printf` piped through `socat`

If you only want the flag and don't care about a reusable script, the same 8-query "fire" / "quiet" pattern can be sent in a single pipe. This works **only if you already know the boundary** (or are willing to spend the rest of the 128-query budget as the boundary-finding probes; a manual approach gets very tedious).

For this challenge, the boundary lives at `x = 4` for `y = 0` and `x = 8/3` for `y = 1`, so a points well inside the "fire" region are anywhere with `x + (4/3)·y > 4` and a points well inside the "quiet" region are anywhere with `x + (4/3)·y < 4`. The script above uses `(±9, ±9)` etc. which are safely on the correct side.

Once you know the boundary, the 8-query payload is just:

```
┌──(zham㉿kali)-[~/ctf/neuron-meet-2d-0]
└─$ printf -- '-9 -9\n9 9\n9 8\n9 7\n-9 -8\n-9 -7\n-9 -6\n-9 -5\n' \
       | socat - TCP:aureolin-pixie.cylabacademy.net:64768,crlf \
       | grep -E 'Recent|flag|academy\{|unlocked'
Perceptron stays quiet.
Recent outputs (8/8): 01110000
Pattern matched! ASCII 'p' unlocked. Here is your flag:
academy{2d_n3ur0n_m3t_892056ff}
```

The first `printf` builds the 8 query lines, `socat` sends them all in one burst, and `grep` filters the noisy transcript down to the lines we care about. This is the shortest "happy path" solve possible.

For a clean **fully-interactive** solve without scripting, you can drive the boundary-finding + final pattern with `socat` and copy-paste the queries by hand. Expect 100+ paste operations — fine if you enjoy the detective work, painful if you do not.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for each phase of our solve.

### Phase 1 — Banner

1. **TCP accept** — the server accepts a new socket connection from us on port `64768`.
2. **Banner write** — the server writes the multi-line banner ending in `[1/128] (x,y)> `. The bounds `[-10, 10]`, the output rule, the back-to-back restriction, the goal, and the `RESET` command are all in the banner.
3. **Block on stdin** — the server waits for a line of input from us.

### Phase 2 — Binary search at `y = 0`

1. We send `-10 0`. The server parses `x = -10, y = 0`, looks up the perceptron output. The line is `x + (4/3)·y = 4`, so at `(x, y) = (-10, 0)` the left-hand side is `-10 < 4`, so the perceptron is quiet. The server writes `Perceptron stays quiet.`, updates the recent-outputs buffer, and prints the next prompt.
2. We send `10 0`. The left-hand side is `10 > 4`, the perceptron fires. Server writes `Perceptron fires.`, updates the buffer, prints the prompt.
3. The boundary sits between, so the server is flipping back and forth as our `x` queries cross the line. We halve the interval each time. After 18 iterations, our interval width is `20 / 2^18 ≈ 7.6e-5` and the boundary is pinned to `x ≈ 4.0000`.

### Phase 3 — Binary search at `y = 1`

1. We send `-10 1`. Left-hand side `-10 + 4/3 = -8.67 < 4`, perceptron quiet.
2. We send `10 1`. Left-hand side `10 + 4/3 = 11.33 > 4`, perceptron fires.
3. Binary search converges to the boundary at `x ≈ 2.6667` (the exact value is `8/3`).

### Phase 4 — Final 8 queries

For each of the 8 final queries, the server:

1. **Parse** the `(x, y)` line.
2. **Check the cache** — if this `(x, y)` matches the previous query, refuse with a back-to-back error and do not count it. Our script inserts a `(0, 0)` filler in this case (it never actually fires here, because the script's filler position never collides with the last binary-search query).
3. **Compute the activation** `w1·x + w2·y + b`. For this challenge, the server's true parameters are roughly `w1 = 3, w2 = 4, b = -16`, so the line `3x + 4y = 16` (or equivalently `x + (4/3)·y = 16/3 = 5.33`... wait, let me re-check).

Actually let me redo this. The boundary crosses `y = 0` at `x = 4` and `y = 1` at `x = 8/3`. The line in `w1·x + w2·y + b = 0` form is:

```
(4, 0) on line  ->  4·w1 + b = 0          ->  b = -4·w1
(8/3, 1) on line ->  (8/3)·w1 + w2 + b = 0 ->  w2 = -4·w1/3 (using b = -4·w1, that's (8/3)w1 + w2 - 4w1 = 0 -> w2 = (4 - 8/3)w1 = (4/3)w1)
```

If we pick `w1 = 3, w2 = 4, b = -16`, we get the line `3x + 4y - 16 = 0`, i.e. `3x + 4y = 16`, which is the same as `x + (4/3)y = 16/3 ≈ 5.33`. At `y = 0`, `x = 16/3 ≈ 5.33`. Hmm, that doesn't match.

Let me redo. The boundary crosses `y = 0` at `x = 4`, not `5.33`. So the line equation should give `x = 4` when `y = 0` and `x = 8/3` when `y = 1`.

```
Line:  x = 4 - (4/3)·y        (in slope-intercept form)
     =>  x + (4/3)·y = 4
     =>  3x + 4y = 12
     =>  3x + 4y - 12 = 0
```

So `w1 = 3, w2 = 4, b = -12`. At `y = 0`: `w1·x + b = 3x - 12 = 0` -> `x = 4`. ✓ At `y = 1`: `3x + 4 - 12 = 0` -> `x = 8/3`. ✓ 

Activation: `3x + 4y - 12`. So the perceptron fires when `3x + 4y >= 12`.

For our final 8 points:

| Position | `(x, y)`     | `3x + 4y`  | Activation >= 12? | Output |
|----------|--------------|------------|-------------------|--------|
| 1        | `(-9, -9)`   | `-27 - 36 = -63` | No            | 0      |
| 2        | `(9, 9)`     | `27 + 36 = 63`   | Yes           | 1      |
| 3        | `(9, 8)`     | `27 + 32 = 59`   | Yes           | 1      |
| 4        | `(9, 7)`     | `27 + 28 = 55`   | Yes           | 1      |
| 5        | `(-9, -8)`   | `-27 - 32 = -59` | No            | 0      |
| 6        | `(-9, -7)`   | `-27 - 28 = -55` | No            | 0      |
| 7        | `(-9, -6)`   | `-27 - 24 = -51` | No            | 0      |
| 8        | `(-9, -5)`   | `-27 - 20 = -47` | No            | 0      |

The 8 outputs are `0, 1, 1, 1, 0, 0, 0, 0` — exactly the binary for `p` (ASCII `0x70`).

### Phase 5 — Flag check

1. After receiving the 8th final query, the server reads the last 8 outputs from its ring buffer: `[0, 1, 1, 1, 0, 0, 0, 0]`.
2. It joins them into a string: `"01110000"`.
3. It converts to an integer: `0b01110000 = 112` (or `0x70`).
4. It converts to a character: `chr(112) = 'p'`.
5. It compares with the target character `'p'`. Match!
6. It writes the success line and the flag, then closes the connection (or waits for `EXIT` — depends on the implementation).

### Geometric intuition

The 2D perceptron's "brain" is a single line. Everything on one side is class 1, everything on the other side is class 0. The line in this challenge is `3x + 4y = 12`, which is a downward-sloping line through `(4, 0)` and `(0, 3)`. The "fires" region is the upper-right half-plane (large `x + y`), the "quiet" region is the lower-left half-plane (small `x + y`). Our 8 final points are scattered in the corners of the `[-10, 10]²` square, so each one is **deep** inside its target region — the closest one to the boundary is `(9, 7)` with a margin of `55 - 12 = 43`, the next is `(-9, -5)` with a margin of `12 - (-47) = 59`. No risk of crossing the line by accident.

### Compared to the rest of the series

- **vs. Play 1D Alpha** — 1D Alpha was a `SET w b` REPL with one weight. 2D-0 hides the parameters and gives you back a black-box service; the math is `w1·x + w2·y + b` instead of `w·x + b`. The boundary is a line in 2D (a single threshold in 1D).
- **vs. Play Naught / Play Alpha (2D REPLs)** — same formula, but the REPLs let you *set* the parameters and watch the boundary move. 2D-0 inverts the relationship: the parameters are fixed and you have to *discover* the boundary by probing. Same 2D math, opposite interaction model.
- **vs. Train Classic 0 / 1 / 2** — the Train challenges have the perceptron *learn* the parameters from labeled data. 2D-0 skips the learning step and goes straight to the *inference* step: given a fixed perceptron, classify unknown points.
- **vs. Trust But Verify** — that one is a pure interactive-fiction challenge with no math. 2D-0 is the "real" AI challenge in the AI category on CyLab Academy: same difficulty tier (1 point) but actually requires understanding the perceptron.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `socat` (or `nc` / `ncat`) | Connect to the REPL over TCP | Easy |
| `python3` (stdlib only) | Drive the binary search + 8-query pattern | Medium |
| `socket` (Python stdlib) | Open the TCP connection, send lines, read responses | Easy |
| `re` (Python stdlib) | Optional: parse the flag out of the response | Easy |
| `nano` | Write the `solve.py` script | Easy |
| `printf` | Build the 8-query payload in one line (alternative method) | Easy |
| `grep` | Filter the REPL transcript for the flag (alternative method) | Easy |
| Mental math (linear equations) | Solve for the boundary `x0, x1` from the binary search | Medium |
| Pen + paper (optional) | Verify the 8 final activations by hand | Easy |

---

## Appendix A — Binary Search Math

For a fixed `y`, the activation `w1·x + w2·y + b` is **linear in `x`** (because `w1` and `b` are constants and `y` is fixed). A linear function is either always increasing or always decreasing in `x`, never both. That is exactly the property binary search needs: the sign of the activation changes at most **once** as `x` sweeps from `-10` to `+10`, and the change happens at the boundary `x = -(w2·y + b) / w1`.

The binary search invariant is:

> At every step, the true boundary lies in the closed interval `[lo, hi]`.

We initialize `lo = -10, hi = 10` (the input bounds), probe the endpoints to learn the sign on each side, and then on each step:

1. Compute `mid = (lo + hi) / 2`.
2. Probe `mid` and read its sign.
3. If the sign at `mid` matches the sign at `lo`, the boundary is in `(mid, hi]`, so set `lo = mid`.
4. Otherwise, the boundary is in `[lo, mid)`, so set `hi = mid`.

After `N` steps the interval width is `20 / 2^N`. For `N = 18` the width is about `7.6e-5`; for `N = 40` it is about `1.8e-11`. We stop when the interval is below our chosen tolerance (`1e-4` in the script, which is way more than enough for the perceptron to classify consistently).

### Why we need TWO binary searches

One binary search gives us **one point on the boundary** (at the chosen `y`). A single point does not pin down a line — infinitely many lines pass through a single point. A second binary search at a different `y` gives us a second point, and two distinct points pin down a line. (In `2D` the "line" is general; in `3D` you would need three points, etc.)

### Why the boundary cannot be outside `[-10, 10]²`

If the line `w1·x + w2·y + b = 0` does not intersect the `[-10, 10]²` square, then for every point in the square the activation `w1·x + w2·y + b` has the **same sign** (because the line is the only place the sign can flip). In that case the perceptron is constant over the entire input region, and we could never produce the `0, 1, 1, 1, 0, 0, 0, 0` pattern (which requires both `0`s and `1`s). So the puzzle implicitly guarantees the line passes through the square, and our binary search will always find a crossing.

---

## Appendix B — Recovering the Parameters `w1, w2, b`

For the curious: from the two boundary points `(4, 0)` and `(8/3, 1)` we can recover the parameters up to a positive scale factor. The line equation in implicit form is:

```
det | x   y   1 |
    | 4   0   1 | = 0
    | 8/3 1   1 |

=>  x · (0·1 - 1·1) - y · (4·1 - (8/3)·1) + 1 · (4·1 - (8/3)·0) = 0
=> -x - y · (4 - 8/3) + 4 = 0
=> -x - (4/3)·y + 4 = 0
=> x + (4/3)·y = 4
=> 3x + 4y = 12
=> 3x + 4y - 12 = 0
```

So one valid choice of parameters is `w1 = 3, w2 = 4, b = -12`. The perceptron's output is:

```
fires if 3x + 4y - 12 >= 0   <=>   3x + 4y >= 12
```

You can verify the 8 final outputs in the "What Happened Internally" table above.

Any **positive** scalar multiple of `(3, 4, -12)` is also a valid parameter set — for example `(6, 8, -24)`, `(0.3, 0.4, -1.2)`. They all draw the same line. **Negative** multiples flip the line's "fire" and "quiet" sides, which would invert every output we computed.

---

## Appendix C — Variations and Edge Cases

A few things that can go wrong with this approach and how to recover:

1. **Boundary outside `[-10, 10]` for a given `y`.** If both endpoints `(-10, y)` and `(10, y)` give the same output, the line does not cross the x-axis at that `y` within the search range. Try a different `y` (e.g. `y = 1`, `y = 5`, `y = -5`) until you find a `y` where the endpoints disagree. The puzzle guarantees at least one such `y` exists, otherwise the perceptron would be constant over the whole square.

2. **All four corners give the same output.** Unusual, but possible if the line cuts the square along a very shallow angle and all four corners happen to fall on the same side. Solution: probe `(0, 10)` and `(0, -10)` to find a `y` where the perceptron does flip.

3. **Your chosen "fire" or "quiet" point is actually on the wrong side of the line.** Always **verify** every candidate point by sending it once and reading the actual output, before you commit to using it in the final 8. The script above does this implicitly by including the verification queries in the binary-search phase (the 18 + 18 iterations test a lot of `x` values for `y = 0` and `y = 1`, and our final points are all far from the line so they are obviously on the right side).

4. **You exceed 128 queries.** The script uses `2 + 18 + 18 + 1 (filler) + 8 = 47` queries, well under the 128 limit. If you use a more wasteful strategy (e.g. linear scan instead of binary search) you can blow through the budget. Stick to binary search.

5. **The server disconnects after exactly 128 queries.** It does not, based on the challenge spec — the limit is just a counter, not a hard cut-off. But if you see a disconnect, you have probably tripped the count.

---

## Key Takeaways

- A 2D perceptron's decision boundary is a **straight line** `w1·x + w2·y + b = 0`. To pin it down, you need **two points** on the line, which you can find with **two binary searches** at different `y` values.
- Binary search is the workhorse of every black-box CTF probe challenge. It cuts the search interval in half on each step, so `log2(range / tolerance)` queries pin a value to within the tolerance. Here, `log2(20 / 1e-4) ≈ 18` queries per binary search.
- The "last 8 outputs spell ASCII" pattern is a classic CTF encoding trick. The character `p` has ASCII code `112 = 0b01110000`, so we need 3 `1`s and 5 `0`s in the right order. The same trick generalises: 8 outputs = 1 byte = 1 ASCII character.
- The "no back-to-back repeats" rule means a pattern of `N` consecutive identical bits requires `N` distinct `(x, y)` points. For our pattern `01110000`, positions 5-8 are four consecutive `0`s, so we need four distinct "quiet" points; positions 2-4 are three consecutive `1`s, so we need three distinct "fire" points. Total: 7 distinct points for the final 8 queries.
- Automating network challenges with Python's `socket` module is way faster than copy-pasting into `socat`. The 47-query solve above would be brutal by hand and takes about 2 seconds to run from a script.

Flag wordplay decode: `academy{2d_n3ur0n_m3t_892056ff}` decodes as **"2D neuron met"** in leet speak (`2d` = 2D, `n3ur0n` = neuron, `m3t` = met) followed by the random hex suffix `892056ff`. A perfect description of the challenge: we met a 2D neuron (perceptron) by probing it through the network.
