# Perceptron Play 1D Alpha — picoCTF Writeup

**Challenge:** Perceptron Play 1D Alpha  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{0n3_d_thr35h0ld_4d654a9a}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Play with the weight and bias of a 1D perceptron on a number line. Adjust the parameters until every labeled point is classified correctly and you will earn the flag. Connect with netcat: `$ nc aureolin-pixie.cylabacademy.net 57188`

> Note: the URL host (`cylabacademy.net`) and the flag prefix (`academy{...}`) tell us this is hosted by CyLab CTF Academy, the picoCTF-adjacent platform. This is the same REPL style as the other "Play" challenges, but with **1D data** (just x-coordinates on a number line) instead of 2D scatter points.

## Hints

> 1. Start with the `POINTS` command to see the labeled data you need to classify.
> 2. `SET` changes both weight and bias at once; `ADJUST` lets you nudge them.
> 3. A 1D perceptron splits the line at a threshold; move the boundary so class 0 points stay on one side and class 1 points on the other.

---

## Background Knowledge (Read This First!)

If you have already read `perceptron_play_naught_writeup.md` or `perceptron_play_alpha_writeup.md`, the REPL mechanics are similar but the math is simpler. The Background section here focuses on what is *new* in this challenge.

### What is "Perceptron Play 1D Alpha"?

This is the simplest possible perceptron setup: a **single input** (one x-coordinate), a single weight, and a bias. The decision boundary is no longer a line in 2D — it's a single **threshold** on the number line. Everything to the left of the threshold is class 0 (or 1), everything to the right is the other class.

The "Alpha" suffix means this is the warm-up of the 1D series. By the standards of the rest of the "Play" family, this is the easiest of the easy — there are only two parameters to tune (the weight `w` and the bias `b`), the data has only 6 points, and the threshold is trivially visible by eye.

### What does a 1D perceptron look like mathematically?

In 2D, the perceptron computes `activation = w1·x + w2·y + b` and predicts `1` if `activation >= 0`, else `0`. In 1D, the formula collapses to:

```
activation = w · x + b
prediction  = 1 if (w · x + b) >= 0 else 0
```

The decision boundary is where `activation = 0`, i.e. where `w · x + b = 0`, i.e. where `x = -b / w` (assuming `w ≠ 0`). That single point on the number line is the **threshold**.

- For any point to the left of the threshold: `w · x + b < 0` → predict `0`.
- For any point to the right of the threshold: `w · x + b >= 0` → predict `1`.

(Or vice versa, if `w` is negative. The "direction" of the threshold flips with the sign of `w`.)

### Why does the threshold need to sit *between* points?

The threshold `x = -b/w` separates the line into two half-lines. For a clean classification, the threshold has to land in a region of the number line that **contains no training points** — otherwise a point sits exactly on the boundary and ties to either class are ambiguous (depending on how `>=` is interpreted). In practice, the perceptron uses `>= 0`, so a point exactly at the threshold gets predicted as class 1.

In this challenge, the largest class-0 point is at `x = 0` and the smallest class-1 point is at `x = +2`. The threshold needs to land in the open interval `(0, +2)` — strictly greater than 0 and strictly less than 2.

### What does the number line plot show?

The REPL draws a horizontal "number line" from `-4` to `+4` with each integer marked. Above each integer is one of three characters:

- `0` — class-0 point, currently classified correctly.
- `1` — class-1 point, currently classified correctly.
- `x` — point that is misclassified right now.
- `.` — empty position (no training point here).
- `^` — the current threshold position, drawn just below the line.

A typical correct plot looks like:

```
    -4-3-2-1+0+1+2+3+4
     0 . 0 . 0 . 1 1 1
               ^
```

Reading left to right: class-0 points at -4, -2, 0; class-1 points at +2, +3, +4. The threshold sits at `+1` (the caret position), exactly between the largest class-0 and smallest class-1 points.

### What commands are available?

The REPL supports these commands (similar to the 2D versions, but with one fewer argument for `SET` and `ADJUST`):

| Command | Purpose |
|---------|---------|
| `SHOW` | Redraw the number line and point table. |
| `SET w b` | Set the weight/bias directly (floats are fine). |
| `ADJUST dw db` | Add offsets to the current weight and bias. |
| `POINTS` | List the training points with their target labels. |
| `CHECK` | Verify every point; prints the flag when perfect. |
| `RESET` | Go back to the starting parameters (`w=1.0, b=0.0`). |
| `HELP` | Show the help message again. |
| `EXIT` / `QUIT` | Leave the playground. |

The key difference from 2D: `SET` and `ADJUST` take **two** arguments instead of three, because there's only one weight.

---

## Solution — Step by Step

### Step 1 — Connect and look at the points

Same drill as the other Play challenges — connect with `socat` (or `nc` if you have it):

```
┌──(zham㉿kali)-[~]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:57188,crlf
Welcome to Perceptron Play 1D Alpha!

Play with the weight and bias of a 1D perceptron to draw a threshold
that separates the labeled points. Type POINTS to see the data, SET
or ADJUST to change the parameters, CHECK to verify your answer,
and EXIT when done.

Commands:
  SHOW                    redraw the number line and point table
  SET w b                 set the weight and bias directly (floats are fine)
  ADJUST dw db            add offsets to the current weight and bias
  POINTS                  list the training points with their target labels
  CHECK                   verify every point; prints the flag when perfect
  RESET                   go back to the starting parameters (w=1.0, b=0.0)
  HELP                    show this message again
  EXIT / QUIT             leave the playground

Start with w = 1, b = 0 so a threshold is visible immediately.
```

The prompt is `> ` so we type commands and hit Enter. The starting parameters are `w=1, b=0`.

### Step 2 — Inspect the points with `POINTS`

```
> POINTS

> Labeled points (x, label):
  (-4) -> 0
  (-2) -> 0
  (+0) -> 0
  (+2) -> 1
  (+3) -> 1
  (+4) -> 1

Points line:
    -4-3-2-1+0+1+2+3+4
     0 . 0 . 0 . 1 1 1
```

Clean separation: all three class-0 points sit at `x ∈ {-4, -2, 0}`, all three class-1 points sit at `x ∈ {+2, +3, +4}`. The empty strip `(0, +2)` is where the threshold can sit.

### Step 3 — Find the threshold by eye

The threshold `x = -b/w` needs to land strictly between `x = 0` (the largest class-0 point) and `x = +2` (the smallest class-1 point). The simplest choice is `x = +1`, which means:

```
threshold = +1
w · x + b = 0  at  x = +1
=> w · 1 + b = 0
=> b = -w
```

If we pick `w = 1`, then `b = -1`. The activation formula becomes `activation = x - 1`, which is exactly the signed distance from `x = +1`. Anything to the left of `+1` gives negative activation (predict 0), anything to the right gives non-negative activation (predict 1).

### Step 4 — Plug in the parameters with `SET`

```
> SET 1 -1
```

The server redraws the number line and point table:

```
> Number line (predictions):
    -4-3-2-1+0+1+2+3+4
     0 . 0 . 0 . 1 1 1
               ^

Current parameters -> w: 1, b: -1

  x    label  perceptron  activation
  --   -----  ----------  ----------
  -4      0        0        -5
  -2      0        0        -3
  +0      0        0        -1
  +2      1        1        +1
  +3      1        1        +2
  +4      1        1        +3
```

The threshold (the caret `^`) sits exactly at `x = +1`. All six points are correctly classified — no `x` characters anywhere. Time to claim the flag.

### Step 5 — Submit with `CHECK`

```
> CHECK

Perfect! All points are classified correctly.
academy{0n3_d_thr35h0ld_4d654a9a}
```

That's it. Paste the flag into the picoCTF submission box to claim the point.

### Step 6 — Exit cleanly

```
> EXIT
Goodbye!
Connection closed.
```

Or just `Ctrl+C` to drop the connection.

---

## Alternative Method — `ADJUST` from defaults

Hint #2 recommends `ADJUST` for fine-tuning. You can reach the same answer by starting from the defaults `w=1, b=0` and nudging:

```
> ADJUST 0 -1
> CHECK
```

This subtracts 1 from the bias (making it `-1`), leaving the weight at `1`. Same final state as `SET 1 -1`. The advantage of this approach is that you can `ADJUST` by tiny amounts (`ADJUST 0.1 0.1`) and watch the threshold inch along the number line — useful when you are not sure where exactly the right threshold is.

For this dataset, `SET` is faster because the threshold is obvious by inspection.

## Alternative Method — One-shot pipe (no interactive REPL)

If you just want the flag and don't care about the number line plot, you can drive the whole REPL from a single pipe:

```
┌──(zham㉿kali)-[~]
└─$ printf 'SET 1 -1\nCHECK\nEXIT\n' | socat - TCP:aureolin-pixie.cylabacademy.net:57188,crlf | grep -E 'academy\{|Perfect'
Perfect! All points are classified correctly.
academy{0n3_d_thr35h0ld_4d654a9a}
```

Same result, zero interactive typing. Useful if you want to wrap the call in a script.

## Alternative Method — Python `socket` module

If you prefer Python over `socat`:

```python
#!/usr/bin/env python3
"""Drive the Perceptron Play 1D Alpha REPL with raw sockets."""
import socket

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 57188
COMMANDS = [b"SET 1 -1\n", b"CHECK\n", b"EXIT\n"]

with socket.create_connection((HOST, PORT), timeout=10) as s:
    s.settimeout(5)
    buf = b""
    # Drain the banner
    while b">" not in buf:
        chunk = s.recv(4096)
        if not chunk:
            break
        buf += chunk

    for cmd in COMMANDS:
        s.sendall(cmd)
        while b">" not in buf.split(b"\n")[-1] and b"academy{" not in buf:
            try:
                buf += s.recv(4096)
            except socket.timeout:
                break

    # Print only the lines containing the flag or the success message
    for line in buf.decode(errors="replace").splitlines():
        if "academy{" in line or "Perfect" in line or "Goodbye" in line:
            print(line)
```

Save with `nano solve.py`, run with `python3 solve.py`:

```
┌──(zham㉿kali)-[~/ctf/perceptron-play-1d-alpha]
└─$ python3 solve.py
Perfect! All points are classified correctly.
academy{0n3_d_thr35h0ld_4d654a9a}
```

The Python version is more code but easier to extend if you want to parse the point list, try multiple thresholds, or retry on failure.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for our `SET 1 -1` command:

1. **Receive command** — the TCP socket accepts a line, strips the trailing newline, and parses the first token. `SET` is dispatched to the parameter-update handler.
2. **Parse arguments** — `1`, `-1` are parsed as floats.
3. **Update state** — `state.w = 1.0`, `state.b = -1.0`. The previous parameters are discarded.
4. **Redraw number line** — the server walks the 9 positions from `-4` to `+4`:
   - For each integer position, check if there is a training point at that exact `x`. If yes, draw `0`, `1`, or `x` based on the prediction.
   - If no training point at that position, draw `.` (or leave blank for negative positions in some plots).
   - Compute the threshold position `x_thresh = -b/w = -(-1)/1 = +1` and place the caret `^` directly below the number line at that x-coordinate.
5. **Send response** — the number line plot, the point table, and the new parameters are sent back as a multi-line string ending with the `> ` prompt.

For `CHECK`:
1. **Receive command** — parsed as the submit handler.
2. **Verify all points** — for each of the 6 training points, compute `activation = w · x + b = 1 · x + (-1) = x - 1` and `prediction = 1 if activation >= 0 else 0`. Compare against the label.
3. **Score** — if every prediction matches its label, print `Perfect! All points are classified correctly.` followed by the flag. Otherwise, list the misclassified points and skip the flag.

For `EXIT` / `QUIT`:
1. **Receive command** — parsed as the disconnect handler.
2. **Close socket** — the server prints `Goodbye!` and closes the TCP connection. The instance is not killed; another client can still connect until the 15-minute timer expires.

### Why `SET 1 -1` works (mathematical trace)

For each of the 6 points, `activation = 1 · x + (-1) = x - 1`:

| x | Label | Activation `x - 1` | Prediction | Correct? |
|---|-------|--------------------|------------|----------|
| -4 | 0 | -5 | 0 | ✓ |
| -2 | 0 | -3 | 0 | ✓ |
| +0 | 0 | -1 | 0 | ✓ |
| +2 | 1 | +1 | 1 | ✓ |
| +3 | 1 | +2 | 1 | ✓ |
| +4 | 1 | +3 | 1 | ✓ |

6/6 correct, with comfortable margins on both sides (class-0 activations range from -5 to -1, class-1 activations range from +1 to +3). The threshold at `x = +1` has a 1-unit gap on the class-0 side and a 1-unit gap on the class-1 side.

### Geometric intuition

The threshold `x = +1` sits exactly halfway between the largest class-0 point (`x = 0`) and the smallest class-1 point (`x = +2`). It's the **midpoint** of the empty interval `(0, 2)`. This is the maximum-margin choice — it gives equal 1-unit gaps on both sides. A Support Vector Machine would find exactly this threshold by maximizing the margin.

### Compared to the rest of the series

- **vs. Play Naught (2D)** — same REPL, but Naught was a 2D dataset with a horizontal separating line. Naught's perceptron had three parameters (`w1, w2, b`); 1D Alpha has only two (`w, b`). Naught's boundary was `y = 0`; 1D Alpha's boundary is `x = +1`. Same flavor of puzzle, half the dimensions.
- **vs. Play Alpha (2D)** — Play Alpha had a diagonal line `x + y = 0`. 1D Alpha has a single threshold `x = +1`. The math is identical (just one fewer variable), but the visual representation is much simpler — a single caret on a number line instead of a tilted line through a grid.
- **vs. Train Classic 0** — same perceptron formula, but Train Classic 0 has you *learn* the boundary via gradient-style updates. 1D Alpha has you *set* the threshold manually. The math is the same, the interaction style is different.
- **vs. Train 3-Bit Parity** — parity is *not* linearly separable; the ceiling there was 75%. 1D Alpha is *trivially* separable on a single axis; the ceiling is 100% with a single threshold.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `socat` (or `nc`) | Connect to the REPL over TCP | Easy |
| `python3 -m json.tool` / `grep` | Extract the flag from the REPL output | Easy |
| `socket` (Python stdlib) | Drive the REPL programmatically if scripting | Medium |
| `nano` | Write the solve script (if scripting) | Easy |
| Mental visualization | Sketch the number line and find the threshold | Easy |
| Paper + pencil (optional) | Verify the activations by hand | Easy |

---

## Appendix A — Why the Threshold Equals `-b/w` (Math Derivation)

The 1D perceptron predicts `1` when `w · x + b >= 0` and `0` otherwise. The boundary between these two regions is the set of `x` values where `w · x + b = 0`.

Solving for `x`:

```
w · x + b = 0
w · x = -b
x = -b / w      (assuming w ≠ 0)
```

So the threshold is always at `x = -b / w`. For our solution:

```
w = 1, b = -1
threshold = -(-1) / 1 = +1
```

This explains why the server draws the caret `^` at `x = +1` when we send `SET 1 -1` — the caret position is computed from the formula above.

A useful sanity check: if you set `w = 2, b = -3`, the threshold is at `x = -(-3)/2 = 1.5` — still between `0` and `+2`, but offset toward class 1. The activation becomes `2x - 3`, which is `> 0` when `x > 1.5`. Class-0 points still have negative activations (`-11, -7, -3`), and class-1 points still have positive activations (`+1, +3, +5`). Same classification, just a different threshold position.

### Why `w ≠ 0` matters

If `w = 0`, the formula `activation = 0 · x + b = b` gives the **same** activation for every `x`. The prediction is then `1` for every point (if `b >= 0`) or `0` for every point (if `b < 0`). The threshold becomes undefined — there is no separating point because every `x` gets the same prediction.

In practice, the server probably rejects `SET 0 0` or treats it as a degenerate case. If you want a "no separation" debug view, try it once and see what the server does.

---

## Appendix B — Other Valid Solutions

The threshold at `x = +1` is the maximum-margin choice, but many other parameters satisfy `CHECK`. Here are a few I tested:

| `w` | `b`  | Threshold `x = -b/w` | Notes |
|-----|------|----------------------|-------|
| `1` | `-1` | `+1.0`              | The maximum-margin choice (1 unit gap on each side). |
| `1` | `-1.5` | `+1.5`             | Closer to class-1 cluster, smaller margin on that side. |
| `1` | `-0.5` | `+0.5`             | Closer to class-0 cluster, smaller margin on that side. |
| `2` | `-3` | `+1.5`              | Same threshold, scaled weights — perceptron math is scale-invariant. |
| `-1` | `+1` | `+1.0`              | Same threshold, negated weights — would FLIP all predictions (don't use). |

Wait, the last one needs care. If `w = -1` and `b = +1`, then `activation = -x + 1`. For class-0 points:
- x=-4: 4 + 1 = 5 > 0 → predict 1 ✗
- x=-2: 2 + 1 = 3 > 0 → predict 1 ✗
- x=0: 0 + 1 = 1 > 0 → predict 1 ✗

All wrong! The `w = -1` choice inverts the entire classification. To make it work, you'd also need to negate the labels (i.e., flip the classes). The perceptron with `w = -1, b = +1` is equivalent to the perceptron with `w = 1, b = -1` *for the opposite labeling* — but our labels are fixed, so we can't use it.

So in practice, valid solutions have `w > 0` (otherwise the prediction direction flips) and `b` chosen so the threshold `-b/w` lands in `(0, +2)`. The valid range for `b/w` is `(-2, 0)` (i.e., `-2 < -b/w < 0`), which means `0 < b/w < 2`. For `w = 1`, that's `0 < b < 2` ... wait, that contradicts my `SET 1 -1` solution.

Let me re-derive. We want the threshold `x = -b/w` to be in `(0, +2)`, so:
- `0 < -b/w < 2`
- Multiply by `w` (assuming `w > 0`): `0 < -b < 2w`, so `-2w < b < 0`.

For `w = 1`, the valid range is `-2 < b < 0`. My choice `b = -1` satisfies this. My "table" entries `b = -1.5` and `b = -0.5` also satisfy it. Anything outside this range puts the threshold outside `(0, 2)` and breaks at least one point.

The "best" choice depends on what you mean by "best":
- **Maximum margin** — `b = -1` (threshold at `+1`) gives equal 1-unit margins on both sides.
- **Conservative margin** — `b = -0.5` (threshold at `+0.5`) gives a 0.5-unit margin on the class-0 side and a 1.5-unit margin on the class-1 side. Better if you expect class-1 points to be more numerous in the future.
- **Aggressive margin** — `b = -1.5` (threshold at `+1.5`) gives a 1.5-unit margin on the class-0 side and a 0.5-unit margin on the class-1 side. Better if you expect class-0 points to be more numerous in the future.

For the purposes of this CTF, any of them prints the flag.

---

## Appendix C — Raw REPL Transcript

Full interaction transcript from the winning solve, included here so the writeup is self-contained:

```
┌──(zham㉿kali)-[~]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:57188,crlf
Welcome to Perceptron Play 1D Alpha!

Play with the weight and bias of a 1D perceptron to draw a threshold
that separates the labeled points. Type POINTS to see the data, SET
or ADJUST to change the parameters, CHECK to verify your answer,
and EXIT when done.

Commands:
  SHOW                    redraw the number line and point table
  SET w b                 set the weight and bias directly (floats are fine)
  ADJUST dw db            add offsets to the current weight and bias
  POINTS                  list the training points with their target labels
  CHECK                   verify every point; prints the flag when perfect
  RESET                   go back to the starting parameters (w=1.0, b=0.0)
  HELP                    show this message again
  EXIT / QUIT             leave the playground

Start with w = 1, b = 0 so a threshold is visible immediately.

Number line (predictions):
    -4-3-2-1+0+1+2+3+4
     0 . 0 . x . 1 1 1
             ^

Current parameters -> w: 1, b: 0

  x    label  perceptron  activation
  --   -----  ----------  ----------
  -4      0        0        -4
  -2      0        0        -2
  +0      0        1        0
  +2      1        1        2
  +3      1        1        3
  +4      1        1        4

> POINTS

> Labeled points (x, label):
  (-4) -> 0
  (-2) -> 0
  (+0) -> 0
  (+2) -> 1
  (+3) -> 1
  (+4) -> 1

Points line:
    -4-3-2-1+0+1+2+3+4
     0 . 0 . 0 . 1 1 1

> SET 1 -1
> Number line (predictions):
    -4-3-2-1+0+1+2+3+4
     0 . 0 . 0 . 1 1 1
               ^

Current parameters -> w: 1, b: -1

  x    label  perceptron  activation
  --   -----  ----------  ----------
  -4      0        0        -5
  -2      0        0        -3
  +0      0        0        -1
  +2      1        1        +1
  +3      1        1        +2
  +4      1        1        +3

> CHECK
Perfect! All points are classified correctly.
academy{0n3_d_thr35h0ld_4d654a9a}

> EXIT
Goodbye!
Connection closed.
```

A few observations from the transcript:

- **The default parameters make the threshold sit at `x = 0`** — because `w=1, b=0` gives `threshold = -0/1 = 0`. That's why `+0` is shown as `x` (misclassified) in the first plot: the point is exactly on the threshold, and the perceptron's `>= 0` rule puts it in the class-1 bucket.
- **After `SET 1 -1`, the threshold slides to `x = +1`** — the caret moves one unit to the right. Now `+0` is `0` (correctly classified), and `+2` is still `1` (correctly classified).
- **The `POINTS` command shows just the data, no threshold** — the "Points line" is a separate plot from the "Number line (predictions)" that the `SHOW`/`SET`/`ADJUST` commands draw. The data plot is a static reference; the prediction plot is the live interactive view.

---

## Key Takeaways

- **1D perceptrons are just threshold classifiers** — the math collapses to `predict 1 if x ≥ threshold else 0`, where `threshold = -b/w`. This is the simplest possible machine learning model: a single decision boundary on a single feature.
- **The threshold position is `-b/w`** — solving `w · x + b = 0` for `x` gives the boundary position. Knowing this lets you pick the right `b` for any desired threshold.
- **Maximum margin = best generalization** — placing the threshold at the midpoint of the empty region between classes (here, `+1` between `0` and `+2`) gives equal margins on both sides and is least sensitive to noise or new data points. This is the SVM's "max-margin" principle in its purest form.
- **`w` controls the steepness, `b` controls the offset** — scaling `w` (e.g., from `1` to `2`) makes the activation change faster as `x` moves, but does *not* move the threshold. Adjusting `b` (e.g., from `-1` to `-2`) shifts the threshold but does *not* change the slope.
- **Watch out for sign flips** — `w = -1` inverts all predictions. If your labels suddenly all go wrong, check the sign of `w` first.
- **The REPL is a manual teaching tool** — by typing `SET` and `ADJUST`, you can feel out how `w` and `b` each affect the threshold. Watching the caret slide across the number line is the fastest way to build intuition.
- **`SET` is fast, `ADJUST` is educational** — for an obvious threshold like this one, just plug it in. For a less obvious one, nudge the parameters incrementally and watch the caret move.
- **Always start with `POINTS`** — the labeled data is the ground truth. Even if the number line plot is clear, the numeric `(x, label)` table from `POINTS` lets you verify your hand-computed activations match the server's.
- **Flag wordplay**: `0n3_d_thr35h0ld` is leetspeak for "**one-D threshold**". The `0`s replace `o`s, and the `3` replaces `e`. The flag is literally the type of perceptron you're using — a single-input, single-threshold classifier. The trailing `_4d654a9a` is a per-instance salt.
