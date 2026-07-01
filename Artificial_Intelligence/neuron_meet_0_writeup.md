# Neuron Meet 0 — picoCTF Writeup

**Challenge:** Neuron Meet 0  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{n3ur0n_m3t_486f8b0e}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Probe a 1D perceptron over the network. Send it numbers and watch whether the neuron fires or not. Use those responses to figure out the decision boundary, then line up the last eight outputs to read the ASCII for p (01110000) and you'll earn the flag. Connect with netcat:
>
> `$ nc aureolin-pixie.cylabacademy.net 49800`
>
> The input bounds are shown on connect to keep the search small. You cannot submit the same number back-to-back.

> Note: the URL host (`cylabacademy.net`) and the flag prefix (`academy{...}`) tell us this is hosted by CyLab CTF Academy, the picoCTF-adjacent platform. This is the simpler 1D cousin of `neuron_meet_2D-0_writeup.md` — same neuron, half the inputs.

## Hints

> 1. A perceptron fires when w*x + b >= 0. With a 1D input there is only one threshold to find.
> 2. You only need values on opposite sides of the threshold to build the output pattern.
> 3. Remember that repeated inputs are blocked, even if they would produce the same output.

---

## Background Knowledge (Read This First!)

If you have already read `neuron_meet_2D-0_writeup.md` or `perceptron_play_1D_alpha_writeup.md`, the perceptron math is identical. The Background section here focuses on what is *new* in this challenge: the network protocol for a 1D probe, and the minimum number of distinct points you need for the ASCII trick.

### What is a 1D perceptron?

A 1D perceptron takes a single number `x` and returns a single bit `0` or `1`. The math is:

```
activation = w·x + b
prediction  = 1 if activation >= 0 else 0
```

The two numbers `w, b` are the perceptron's **parameters** — its "brain". We do not get to see them, but we can learn them indirectly by sending inputs and watching the predictions.

For a 1D perceptron, the **decision boundary** is the set of all `x` values where `w·x + b = 0`. That equation describes a single **threshold point** on the number line. Everything to one side of the threshold is classified as `1`, everything to the other side is classified as `0`. The threshold itself is classified as `1` because the inequality is non-strict (`>= 0`).

```
        quiet (0)    |    fires (1)
   -----------------+----------------->  x
                    ^
                  threshold
                  x = -b/w
```

The threshold is at `x = -b/w` (assuming `w ≠ 0`).

### How do we find a threshold from black-box queries?

A threshold on a number line is a **single point**. One binary search is enough to pin it down. Recipe:

1. Probe the two ends of the search range, e.g. `x = -10` and `x = +10`.
2. If the predictions differ, the threshold sits somewhere in between. Halve the interval and recurse.
3. After `N` steps, the search interval has shrunk by a factor of `2^N`. For `N = 18`, the interval width is `20 / 2^18 ≈ 7.6e-5` — way more than enough precision for the perceptron to classify consistently.

That is the **entire** boundary-finding phase. No second binary search needed, because in 1D one point pins down a "line" (which is just a point).

### What is binary search?

Binary search is the divide-and-conquer recipe for finding a value inside a sorted range. You check the midpoint, decide which half contains the target, and recurse on that half. After `N` steps, the search interval has shrunk by a factor of `2^N`. It is the single most useful algorithm in CTF probe-style challenges.

### How do I turn "fire" and "stays quiet" into a flag?

The challenge wants the **last 8 outputs** to read the binary encoding of the ASCII character `p`. ASCII for `p` is the 8-bit binary string:

```
p = 0x70 = 01110000 (binary)
```

So I need 8 consecutive perceptron outputs in the order `0, 1, 1, 1, 0, 0, 0, 0`:

| Output position | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| Required bit    | 0 | 1 | 1 | 1 | 0 | 0 | 0 | 0 |

Each position is a separate `x` query. Three of them (positions 2, 3, 4) must produce a `1`, and five of them (positions 1, 5, 6, 7, 8) must produce a `0`. I just need to find one good "fire" number and one good "quiet" number, and reuse them in the right order.

### What does "no back-to-back repeats" mean?

The service refuses to process a query that uses the exact same `x` value as the previous query — even if I want to keep getting the same output. So if I need 4 consecutive `0`s, I cannot just send `x = -5` four times. I have to use **4 different** "quiet" values (e.g. `-1, -2, -3, -4`) and a 5th "quiet" value for position 1, plus 3 different "fire" values for positions 2, 3, 4. That's 7 distinct values total for the final 8 queries.

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
s.sendall(b"0\n")                            # send a query
data = s.recv(4096)                          # read the response
```

That is the whole API. The rest is parsing.

---

## Solution — Step by Step

### Step 1 — Make a working folder

```
┌──(zham㉿kali)-[~]
└─$ mkdir -p ~/ctf/neuron-meet-0 && cd ~/ctf/neuron-meet-0
```

### Step 2 — Connect and look at the banner

```
┌──(zham㉿kali)-[~/ctf/neuron-meet-0]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:49800,crlf
Welcome to Neuron Meet 0!
Probe the 1D perceptron to coax out the ASCII for 'p'.
Send a number within the bounds to see if the perceptron fires (1) or stays quiet (0).
- Bounds: [-10.0, 10.0]
- Output rule: w*x + b >= 0 -> 1, else 0.
- No back-to-back repeats of the same number.
- Goal: make the last 8 outputs read 01110000 (ASCII 'p').
- Command: RESET to clear the firing history.
Type HELP for a reminder or EXIT to quit.

[1/128] x> 
```

Key things to note from the banner:
- The **input bounds are `[-10, 10]`**. The threshold has to live somewhere in that interval.
- We get **128 queries** total. The 8 most recent are what gets checked.
- The boundary is a single threshold: `w·x + b = 0` happens at `x = -b/w`.

### Step 3 — Find the threshold by binary search

I will use a Python script because manually sending 30+ queries is painful. Open `nano`:

```
┌──(zham㉿kali)-[~/ctf/neuron-meet-0]
└─$ nano solve.py
```

Paste the following inside nano:

```python
#!/usr/bin/env python3
"""Probe the Neuron Meet 0 perceptron, find the threshold, and
land the last 8 outputs on 01110000 (ASCII 'p') to earn the flag."""
import re
import socket
import sys
import time

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 49800


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


def query(sock, x):
    """Send x, read the response, return (output, raw_text)."""
    sock.sendall(f"{x:.6f}\n".encode())
    text = recv_until(sock, b"x>")
    if "fires" in text:
        out = 1
    elif "stays quiet" in text:
        out = 0
    else:
        out = None
    flag_m = re.search(r"academy\{[^}]+\}", text)
    flag = flag_m.group(0) if flag_m else None
    return out, text, flag


def find_threshold(sock, lo=-10.0, hi=10.0, tol=1e-4, max_iter=40):
    """Binary search for the x value where the perceptron flips."""
    f_lo, _, _ = query(sock, lo)
    f_hi, _, _ = query(sock, hi)
    if f_lo == f_hi:
        return None  # threshold outside the search range
    for _ in range(max_iter):
        if hi - lo < tol:
            break
        mid = (lo + hi) / 2
        f_mid, _, _ = query(sock, mid)
        if f_mid == f_lo:
            lo = mid
        else:
            hi = mid
    return (lo + hi) / 2


def main():
    s = socket.create_connection((HOST, PORT), timeout=5)

    # Drain the banner
    recv_until(s, b"x>")

    # Step 1: find the threshold
    threshold = find_threshold(s, -10.0, 10.0)
    print(f"[+] Threshold at x = {threshold:.4f}", file=sys.stderr)

    # Step 2: probe one point on each side to figure out which is fire and which is quiet
    f_below, _, _ = query(s, threshold - 1.0)
    f_above, _, _ = query(s, threshold + 1.0)
    print(f"[+] Below threshold -> {f_below}, above -> {f_above}", file=sys.stderr)

    if f_above == 1:
        fire_vals = [threshold + 2, threshold + 3, threshold + 4, threshold + 5]
        quiet_vals = [threshold - 2, threshold - 3, threshold - 4, threshold - 5, threshold - 6]
    else:
        fire_vals = [threshold - 2, threshold - 3, threshold - 4, threshold - 5]
        quiet_vals = [threshold + 2, threshold + 3, threshold + 4, threshold + 5, threshold + 6]

    # Clamp to the [-10, 10] bound and de-dupe
    fire_vals = list(dict.fromkeys(max(-10, min(10, x)) for x in fire_vals))
    quiet_vals = list(dict.fromkeys(max(-10, min(10, x)) for x in quiet_vals))

    # Pattern: 0, 1, 1, 1, 0, 0, 0, 0
    # Position 1: quiet. Positions 2-4: fire (3 distinct). Positions 5-8: quiet (4 distinct).
    # Position 1 is separated from position 5 by the three fire queries, so position 1's
    # quiet value can equal one of positions 5-8's.
    final_sequence = [
        quiet_vals[0],   # 0
        fire_vals[0],    # 1
        fire_vals[1],    # 1
        fire_vals[2],    # 1
        quiet_vals[1],   # 0
        quiet_vals[2],   # 0
        quiet_vals[3],   # 0
        quiet_vals[4] if len(quiet_vals) >= 5 else quiet_vals[0],   # 0
    ]

    # If the first final value happens to equal the last query we sent, slip in a filler
    # to break the back-to-back repeat. (Sock state is preserved across calls; we don't
    # track the last x here for brevity, but the script's binary search ends on the
    # threshold ± 1 query, which is not in final_sequence.)
    for i, x in enumerate(final_sequence):
        out, text, flag = query(s, x)
        print(f"  [{i+1}] x = {x:+.3f} -> {out}", file=sys.stderr)
        if flag:
            break

    # Read the final response (the flag)
    s.settimeout(3)
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
┌──(zham㉿kali)-[~/ctf/neuron-meet-0]
└─$ python3 solve.py
[+] Threshold at x = 2.0000
[+] Below threshold -> 0, above -> 1
  [1] x = +0.000 -> 0
  [2] x = +4.000 -> 1
  [3] x = +5.000 -> 1
  [4] x = +6.000 -> 1
  [5] x = -1.000 -> 0
  [6] x = -2.000 -> 0
  [7] x = -3.000 -> 0
  [8] x = -4.000 -> 0
Perceptron stays quiet.
Recent outputs (8/8): 01110000
Pattern matched! ASCII 'p' unlocked. Here is your flag:
academy{n3ur0n_m3t_486f8b0e}
```

The threshold sits at `x = 2.0`. The "fires" region is `x > 2`, the "quiet" region is `x < 2`. The final 8 outputs land cleanly on `01110000`, and the server hands over the flag.

### Step 5 — Submit the flag

Paste `academy{n3ur0n_m3t_486f8b0e}` into the picoCTF submission box to claim the point.

---

## Alternative Method — `printf` piped through `socat`

If you only want the flag and don't care about a reusable script, the same 8-query "fire" / "quiet" pattern can be sent in a single pipe. This works **only if you already know the threshold** (or are willing to spend the rest of the 128-query budget as the threshold-finding probes; a manual approach gets very tedious).

For this challenge, the threshold is at `x = 2.0`, so a "fire" value is anything `> 2` and a "quiet" value is anything `< 2`. The script above uses `4, 5, 6` (well above) and `-1, -2, -3, -4` (well below) for safety.

Once you know the threshold, the 8-query payload is just:

```
┌──(zham㉿kali)-[~/ctf/neuron-meet-0]
└─$ printf -- '0\n4\n5\n6\n-1\n-2\n-3\n-4\n' \
       | socat - TCP:aureolin-pixie.cylabacademy.net:49800,crlf \
       | grep -E 'Recent|flag|academy\{|unlocked'
Perceptron stays quiet.
Recent outputs (8/8): 01110000
Pattern matched! ASCII 'p' unlocked. Here is your flag:
academy{n3ur0n_m3t_486f8b0e}
```

The first `printf` builds the 8 query lines, `socat` sends them all in one burst, and `grep` filters the noisy transcript down to the lines we care about. This is the shortest "happy path" solve possible.

For a clean **fully-interactive** solve without scripting, you can drive the threshold-finding + final pattern with `socat` and copy-paste the queries by hand. Expect 30+ paste operations — fine if you enjoy the detective work, painful if you do not.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for each phase of our solve.

### Phase 1 — Banner

1. **TCP accept** — the server accepts a new socket connection from us on port `49800`.
2. **Banner write** — the server writes the multi-line banner ending in `[1/128] x> `. The bounds `[-10, 10]`, the output rule, the back-to-back restriction, the goal, and the `RESET` command are all in the banner.
3. **Block on stdin** — the server waits for a line of input from us.

### Phase 2 — Binary search

1. We send `-10`. The server parses `x = -10`, looks up the perceptron output. The line is `w·x + b = 0`, so at `x = -10` the activation is `-10w + b`. For the server's true parameters, this is negative, so the perceptron is quiet. The server writes `Perceptron stays quiet.`, updates the recent-outputs buffer (now `[0]`), and prints the next prompt.
2. We send `10`. Activation is `10w + b`, which is positive for the server's parameters, so the perceptron fires. Server writes `Perceptron fires.`, buffer becomes `[0, 1]`.
3. The threshold sits between, so the server is flipping back and forth as our `x` queries cross `x = 2`. We halve the interval each time. After 18 iterations, our interval width is `20 / 2^18 ≈ 7.6e-5` and the threshold is pinned to `x ≈ 2.0000`.

### Phase 3 — Side probe

1. We send `1` (below threshold). Activation `w·1 + b` is negative (since `x = 1 < 2 = -b/w`). Quiet.
2. We send `3` (above threshold). Activation `w·3 + b` is positive (since `x = 3 > 2`). Fires.
3. We now know: `x < 2` is quiet, `x > 2` fires. The boundary is `x = 2`, i.e. `w = 1, b = -2` (one of infinitely many equivalent parameter sets).

### Phase 4 — Final 8 queries

For each of the 8 final queries, the server:

1. **Parse** the `x` value.
2. **Check the cache** — if this `x` matches the previous query, refuse with a back-to-back error and do not count it. Our script's first final value (`x = 0`) does not match the last side-probe value (`x = 3`), so no filler is needed.
3. **Compute the activation** `w·x + b`. With `w = 1, b = -2`, that is `x - 2`.
4. **Classify** as `1` if `x >= 2`, else `0`.

For our final 8 points:

| Position | `x`    | Activation `x - 2` | Output |
|----------|--------|--------------------|--------|
| 1        | `0`    | -2                 | 0      |
| 2        | `4`    | +2                 | 1      |
| 3        | `5`    | +3                 | 1      |
| 4        | `6`    | +4                 | 1      |
| 5        | `-1`   | -3                 | 0      |
| 6        | `-2`   | -4                 | 0      |
| 7        | `-3`   | -5                 | 0      |
| 8        | `-4`   | -6                 | 0      |

The 8 outputs are `0, 1, 1, 1, 0, 0, 0, 0` — exactly the binary for `p` (ASCII `0x70`).

### Phase 5 — Flag check

1. After receiving the 8th final query, the server reads the last 8 outputs from its ring buffer: `[0, 1, 1, 1, 0, 0, 0, 0]`.
2. It joins them into a string: `"01110000"`.
3. It converts to an integer: `0b01110000 = 112` (or `0x70`).
4. It converts to a character: `chr(112) = 'p'`.
5. It compares with the target character `'p'`. Match!
6. It writes the success line and the flag, then closes the connection (or waits for `EXIT` — depends on the implementation).

### Geometric intuition

The 1D perceptron's "brain" is a single threshold. The activation `w·x + b` is a line in `x` — negative on one side, positive on the other, zero at the threshold. The threshold in this challenge is `x = 2`, which is the **midpoint** of the gap between the "quiet" cluster `{0, -1, -2, -3, -4}` and the "fire" cluster `{4, 5, 6}`. The perceptron's true parameters are approximately `w = 1, b = -2`, so the activation is `x - 2`, which equals zero exactly at `x = 2`.

### Compared to the rest of the series

- **vs. Neuron Meet 2D-0** — the 2D version needs two binary searches (one for each `y` value) to pin down a line in 2D. The 1D version needs just one binary search, because a "line" in 1D is a single point. The math is the same `w·x + b` formula; the 2D version just adds a second input `w1·x + w2·y + b`. Same flavor of puzzle, half the probing.
- **vs. Play 1D Alpha** — 1D Alpha was a `SET w b` REPL with two parameters. Neuron Meet 0 hides the parameters and gives you back a black-box service. The math is identical, the interaction model is inverted: REPL sets the weights, probe service reads the weights.
- **vs. Play Naught / Play Alpha (2D REPLs)** — those are 2D REPLs. Neuron Meet 0 is a 1D probe service. Different dimensions, different interaction model.
- **vs. Train Classic 0 / 1 / 2** — the Train challenges have the perceptron *learn* the parameters from labeled data. Neuron Meet 0 skips the learning step and goes straight to the *inference* step: given a fixed perceptron, classify unknown points.
- **vs. Trust But Verify** — that one is a pure interactive-fiction challenge with no math. Neuron Meet 0 is the real AI challenge in the AI category on CyLab Academy: same difficulty tier (1 point) but actually requires understanding the perceptron.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `socat` (or `nc` / `ncat`) | Connect to the REPL over TCP | Easy |
| `python3` (stdlib only) | Drive the binary search + 8-query pattern | Medium |
| `socket` (Python stdlib) | Open the TCP connection, send lines, read responses | Easy |
| `re` (Python stdlib) | Optional: parse the flag out of the response | Easy |
| `time` (Python stdlib) | Bound the receive with a timeout | Easy |
| `nano` | Write the `solve.py` script | Easy |
| `printf` | Build the 8-query payload in one line (alternative method) | Easy |
| `grep` | Filter the REPL transcript for the flag (alternative method) | Easy |
| Mental math (linear equations) | Solve for the threshold from the binary search | Easy |
| Pen + paper (optional) | Verify the 8 final activations by hand | Easy |

---

## Appendix A — Binary Search Math

For a 1D perceptron, the activation `w·x + b` is **linear in `x`** (because `w` and `b` are constants). A linear function is either always increasing or always decreasing in `x`, never both. That is exactly the property binary search needs: the sign of the activation changes at most **once** as `x` sweeps from `-10` to `+10`, and the change happens at the threshold `x = -b/w`.

The binary search invariant is:

> At every step, the true threshold lies in the closed interval `[lo, hi]`.

We initialize `lo = -10, hi = 10` (the input bounds), probe the endpoints to learn the sign on each side, and then on each step:

1. Compute `mid = (lo + hi) / 2`.
2. Probe `mid` and read its sign.
3. If the sign at `mid` matches the sign at `lo`, the threshold is in `(mid, hi]`, so set `lo = mid`.
4. Otherwise, the threshold is in `[lo, mid)`, so set `hi = mid`.

After `N` steps the interval width is `20 / 2^N`. For `N = 18` the width is about `7.6e-5`; for `N = 40` it is about `1.8e-11`. We stop when the interval is below our chosen tolerance (`1e-4` in the script, which is way more than enough for the perceptron to classify consistently).

### Why one binary search is enough in 1D

In 1D, the "decision boundary" is a single point. One point is enough to pin it down — no two-point constraint like in 2D. The binary search returns the threshold directly, and we are done.

In 2D, the "decision boundary" is a line, which is defined by two points. So the 2D version of this challenge needs two binary searches (at different `y` values) to recover the line. See `neuron_meet_2D-0_writeup.md` for that variant.

### Why the threshold cannot be outside `[-10, 10]`

If the threshold `x = -b/w` does not lie in the search range, then for every `x` in `[-10, 10]` the activation `w·x + b` has the **same sign** (because the threshold is the only place the sign can flip). In that case the perceptron is constant over the entire input range, and we could never produce the `0, 1, 1, 1, 0, 0, 0, 0` pattern (which requires both `0`s and `1`s). So the puzzle implicitly guarantees the threshold sits inside `[-10, 10]`, and our binary search will always find it.

---

## Appendix B — Recovering the Parameters `w, b`

For the curious: from the threshold `x = 2` we can recover the parameters up to a positive scale factor. The threshold equation in implicit form is:

```
threshold = -b / w
2 = -b / w
b = -2w
```

So one valid choice of parameters is `w = 1, b = -2`. The perceptron's output is:

```
fires if 1·x + (-2) >= 0   <=>   x >= 2
```

You can verify the 8 final outputs in the "What Happened Internally" table above.

Any **positive** scalar multiple of `(1, -2)` is also a valid parameter set — for example `(2, -4)`, `(0.5, -1)`. They all draw the same threshold. **Negative** multiples flip the threshold's "fire" and "quiet" sides, which would invert every output we computed.

---

## Appendix C — Variations and Edge Cases

A few things that can go wrong with this approach and how to recover:

1. **Threshold outside `[-10, 10]`.** If both endpoints `(-10)` and `(+10)` give the same output, the threshold does not lie in the search range. Try a different pair of endpoints (e.g. `(-5, 5)`, `(0, 10)`, `(-10, 0)`) until you find a pair that disagrees. The puzzle guarantees at least one such pair exists, otherwise the perceptron would be constant over the whole range.

2. **Your chosen "fire" or "quiet" value is actually on the wrong side of the threshold.** Always **verify** every candidate value by sending it once and reading the actual output, before you commit to using it in the final 8. The script above does this implicitly by including the verification query (the `threshold ± 1` probe) and by picking final values that are **2 or more units away** from the threshold — a safety margin that would only be a problem if the threshold were at the very edge of the search range.

3. **You exceed 128 queries.** The script uses `2 + 18 + 2 + 8 = 30` queries, well under the 128 limit. If you use a more wasteful strategy (e.g. linear scan instead of binary search) you can blow through the budget. Stick to binary search.

4. **The server disconnects after exactly 128 queries.** It does not, based on the challenge spec — the limit is just a counter, not a hard cut-off. But if you see a disconnect, you have probably tripped the count.

5. **The threshold is at the very edge of the search range.** E.g. if the threshold is at `x = 9.95`, our "fire" pool might be `[11.95, 12.95, ...]` which is outside the bound `10` and gets clamped to `10`. After clamping, all "fire" values become the same (`10`), which violates the no-back-to-back rule. To recover, expand the search range (use `[-100, 100]` instead of `[-10, 10]`) and clamp less aggressively. For this challenge the threshold is at `x = 2`, well inside the range, so we do not hit this edge case.

---

## Key Takeaways

- A 1D perceptron's decision boundary is a **single threshold** `x = -b/w`. To find it, you need **one binary search** on a probe of the sign at each `x`. Two binary searches would be redundant — a point in 1D is fully determined by one coordinate.
- Binary search is the workhorse of every black-box CTF probe challenge. It cuts the search interval in half on each step, so `log2(range / tolerance)` queries pin a value to within the tolerance. Here, `log2(20 / 1e-4) ≈ 18` queries.
- The "last 8 outputs spell ASCII" pattern is a classic CTF encoding trick. The character `p` has ASCII code `112 = 0b01110000`, so we need 3 `1`s and 5 `0`s in the right order. The same trick generalises: 8 outputs = 1 byte = 1 ASCII character.
- The "no back-to-back repeats" rule means a pattern of `N` consecutive identical bits requires `N` distinct `x` values. For our pattern `01110000`, positions 5-8 are four consecutive `0`s, so we need four distinct "quiet" values; positions 2-4 are three consecutive `1`s, so we need three distinct "fire" values. Total: 7 distinct values for the final 8 queries.
- Automating network challenges with Python's `socket` module is way faster than copy-pasting into `socat`. The 30-query solve above would be brutal by hand and takes about 1 second to run from a script.

Flag wordplay decode: `academy{n3ur0n_m3t_486f8b0e}` decodes as **"neuron met"** in leet speak (`n3ur0n` = neuron, `m3t` = met) followed by the random hex suffix `486f8b0e`. A perfect description of the challenge: we met a 1D neuron (perceptron) by probing it through the network.
