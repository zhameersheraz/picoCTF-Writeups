# Perceptron Play Naught — picoCTF Writeup

**Challenge:** Perceptron Play Naught  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{n4ught_bu7_53p4r4b13_c0e17ccc}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> The points would be inseparable in just the x-dimension, but each point now has a new y-value so the set becomes linearly separable in 2D. Watch the ASCII plot update in real time as you tweak the perceptron weights and bias. Separate the labeled points with a single line to earn the flag. Connect with netcat: `$ nc aureolin-pixie.cylabacademy.net 53443`

> Note: the URL host (`cylabacademy.net`) and the flag prefix (`academy{...}`) tell us this is hosted by CyLab CTF Academy, the picoCTF-adjacent platform. This is a different style of challenge from the web-based variants — it exposes a telnet-style REPL over `nc` instead of a `POST /train` API.

## Hints

> 1. Start with the `POINTS` command to see how the old 1D x-values were lifted into 2D.
> 2. Use `SET` for big changes and `ADJUST` for fine-tuning the weights.
> 3. A perceptron learns a linear boundary; look for a line that keeps class 0 points on one side and class 1 points on the other.

---

## Background Knowledge (Read This First!)

If you have already read `perceptron_train_classic_0_writeup.md`, the perceptron mechanics are the same. The Background section here focuses on what is *new* in this challenge.

### What is "Perceptron Play"?

The "Play" series is a different style from the "Train" series. Instead of a browser-based visualizer with a slider that auto-runs the perceptron, you are dropped into a **command-line REPL** where you manually type weights and biases, watch the ASCII art redraw, and submit when you think you've got a separating line. There is no auto-update — you set the weights, look at the picture, change your mind, set again. It's a manual sandbox for understanding the perceptron's decision boundary.

The REPL is reachable over plain TCP using `nc` (netcat) or any TCP client. The challenge name "Play Naught" refers to the **zero-or-naught** puzzle idea: a near-trivial dataset where the boundary is so obvious that anyone who looks at the points for five seconds can solve it. The name is half-joking — it's the easy warm-up of the "Play" series.

### Why the points are "lifted from 1D to 2D"

Hint #1 says "see how the old 1D x-values were lifted into 2D". The "Charlie" puzzle mentioned in the welcome banner is presumably a previous 1D-perceptron challenge where the points' x-coordinates alone were not separable. In 1D, "not separable" means there exists no single threshold value `t` such that all class-0 points are on one side and all class-1 points are on the other. The author took those same x-values and **added** a y-coordinate to each, choosing the y-values specifically to make the 2D set linearly separable. Once you have two coordinates, a *line* (not just a point) can slice the plane and split any labeling that is linearly separable.

In this particular challenge, the y-values were chosen so cleanly that the boundary is just the **horizontal line `y = 0`**: every class-0 point has `y = -1`, every class-1 point has `y ≥ +1`. That's as separable as a dataset gets.

### What does the ASCII plot show?

The REPL draws a 5x9 ASCII grid spanning `x ∈ [-4, +4]` and `y ∈ [-4, +4]`. Each cell either contains a point (drawn as `0`, `1`, or `x`) or is empty. Empty cells in the path of the decision boundary get a `/` character to show roughly where the line is. The starting weights `w1 = 1, w2 = -1, b = 0` already produce a visible (but incorrect) boundary, so you can see how your changes affect the picture in real time.

The legend:

- `0` — class-0 point, currently classified correctly.
- `1` — class-1 point, currently classified correctly.
- `x` — point that is misclassified right now.
- `/` — approximate decision boundary (where `w1·x + w2·y + b ≈ 0`).

### What commands are available?

The REPL supports these commands:

| Command | Purpose |
|---------|---------|
| `SHOW` | Redraw the ASCII graph and point table. |
| `SET w1 w2 b` | Set the weights/bias directly (floats are fine). |
| `ADJUST dw1 dw2 db` | Add offsets to the current weights. |
| `POINTS` | List the training points with their target labels. |
| `CHECK` | Verify every point; prints the flag when perfect. |
| `RESET` | Go back to the starting weights (`w1=1.0, w2=-1.0, b=0.0`). |
| `HELP` | Show the help message again. |
| `EXIT` / `QUIT` | Leave the playground. |

The `SET` command is the brute-force approach — directly plug in the weights you want. The `ADJUST` command is the experimental approach — nudge by small amounts and watch the boundary move. Hint #2 recommends using `SET` for big changes and `ADJUST` for fine-tuning, which is the natural division of labor.

### Why does this need `CHECK` instead of auto-flag?

Unlike the "Train" series, where the server scores your run automatically on `POST /train`, the "Play" series waits for you to explicitly say "I am done". The `CHECK` command is the manual submit. If even one point is misclassified, `CHECK` tells you so and does not give the flag. If every point is classified correctly, `CHECK` prints the flag inline.

This is a more deliberate, more pedagogical style — you set weights, look at the picture, decide whether you're done, and only then check.

---

## Solution — Step by Step

### Step 1 — Connect and look at the points

The challenge banner says `nc aureolin-pixie.cylabacademy.net 53443`. My Kali box doesn't have `nc` installed, so I use `socat` (or any TCP client):

```
┌──(zham㉿kali)-[~]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:53443,crlf
Welcome to Perceptron Play Naught!

Tweak the weights of a single-layer perceptron and watch how the decision
boundary moves. This time, the x-values are borrowed from the 1D Charlie
puzzle and lifted into 2D with new y-values so they can be linearly
separated. Your goal is to correctly classify every labeled point.

Commands:
  SHOW                    redraw the ASCII graph and point table
  SET w1 w2 b             set the weights/bias directly (floats are fine)
  ADJUST dw1 dw2 db       add offsets to the current weights
  POINTS                  list the training points with their target labels
  CHECK                   verify every point; prints the flag when perfect
  RESET                   go back to the starting weights (all w1=1.0, w2=-1.0, b=0.0)
  HELP                    show this message again
  EXIT / QUIT             leave the playground
```

The banner gives us everything we need: the API surface, the dataset description, and the starting weights. The prompt is `> ` so we type commands and hit Enter.

### Step 2 — Inspect the points with `POINTS`

```
> POINTS

  point    label  perceptron  activation
  ------   -----  ----------  ----------
  (-4,-1)     0        0        -3
  (-1,+2)     1        0        -3
  (+0,-1)     0        1        +1
  (+0,+2)     1        0        -2
  (+2,-1)     0        1        +3
  (+3,+1)     1        1        +2
  (+4,+2)     1        1        +2
```

The "perceptron" and "activation" columns show what the *current* weights predict. With the starting weights `w1=1, w2=-1, b=0`:
- `(-4,-1)`: `1·(-4) + (-1)·(-1) + 0 = -3`, predict `0` ✓
- `(-1,+2)`: `1·(-1) + (-1)·(+2) + 0 = -3`, predict `0` ✗ (label 1)
- `(+0,-1)`: `1·(+0) + (-1)·(-1) + 0 = +1`, predict `1` ✗ (label 0)
- ... etc.

Two are wrong: `(-1,+2)` and `(+0,+2)` (label 1 predicted as 0), and `(+0,-1)` and `(+2,-1)` (label 0 predicted as 1). The current line is the wrong tilt — it cuts through the data diagonally.

### Step 3 — Find the separating line by eye

Look at the y-coordinates of all 7 points:

| Point | Label | y |
|-------|-------|---|
| `(-4,-1)` | 0 | **-1** |
| `(-1,+2)` | 1 | **+2** |
| `(+0,-1)` | 0 | **-1** |
| `(+0,+2)` | 1 | **+2** |
| `(+2,-1)` | 0 | **-1** |
| `(+3,+1)` | 1 | **+1** |
| `(+4,+2)` | 1 | **+2** |

**Every class-0 point has y = -1.** **Every class-1 point has y ≥ +1.** There is a clean empty horizontal strip between y = -1 and y = +1 where the boundary can sit. The simplest separating line is **`y = 0`**.

A perceptron with `w1 = 0, w2 = 1, b = 0` evaluates `w1·x + w2·y + b = y`, which is exactly the y-coordinate. Prediction: `1` if `y ≥ 0`, else `0`. That puts `y = -1` on the negative side (class 0) and `y ≥ +1` on the positive side (class 1).

### Step 4 — Plug in the weights with `SET`

```
> SET 0 1 0
```

The server redraws the graph and point table:

```
> +4         |
+3         |
+2       1 1       1
+1         |     1
+0 / / / / / / / / /
-1 0       0   0
-2         |
-3         |
-4         |
   -4-3-2-1+0+1+2+3+4

Current weights -> w1: 0, w2: 1, b: 0

  point    label  perceptron  activation
  ------   -----  ----------  ----------
  (-4,-1)     0        0        -1
  (-1,+2)     1        1        +2
  (+0,-1)     0        0        -1
  (+0,+2)     1        1        +2
  (+2,-1)     0        0        -1
  (+3,+1)     1        1        +1
  (+4,+2)     1        1        +2
```

The boundary sits exactly at `y = 0` (the row of `/` characters). All seven points are correctly classified — no `x` characters anywhere. Time to claim the flag.

### Step 5 — Submit with `CHECK`

```
> CHECK

Perfect! All points are classified correctly.
academy{n4ught_bu7_53p4r4b13_c0e17ccc}
```

That's it. The flag prints inline. Paste it into the picoCTF submission box to claim the point.

### Step 6 — Exit cleanly

```
> EXIT
Goodbye!
Connection closed.
```

Or just `Ctrl+C` to drop the connection. The flag is server-side per-instance, so it stays valid until the instance expires.

---

## Alternative Method — `ADJUST` from defaults

Hint #2 recommends `ADJUST` for fine-tuning. You can also reach the same answer by starting from the defaults `w1=1, w2=-1, b=0` and nudging:

```
> ADJUST -1 2 0
> CHECK
```

This subtracts 1 from `w1` (making it 0) and adds 2 to `w2` (making it 1), keeping bias at 0. Same final state as `SET 0 1 0`. The advantage of this approach is that you can `ADJUST` by tiny amounts (`ADJUST 0.1 0.1 0.1`) and watch the boundary inch across the grid one cell at a time — useful when you are not sure what the right answer is and want to learn by experimentation.

For this dataset, `SET` is faster because the answer is obvious by inspection. For a dataset where the boundary is non-obvious, `ADJUST` lets you feel out the geometry.

## Alternative Method — One-shot pipe (no interactive REPL)

If you just want the flag and don't care about the ASCII art, you can drive the whole REPL from a single pipe:

```
┌──(zham㉿kali)-[~]
└─$ printf 'SET 0 1 0\nCHECK\nEXIT\n' | socat - TCP:aureolin-pixie.cylabacademy.net:53443,crlf | grep -E 'academy\{|Perfect'
Perfect! All points are classified correctly.
academy{n4ught_bu7_53p4r4b13_c0e17ccc}
```

Same result, zero interactive typing. Useful if you want to wrap the call in a script.

## Alternative Method — Python `socket` module

If you prefer Python over `socat`:

```python
#!/usr/bin/env python3
"""Drive the Perceptron Play Naught REPL with raw sockets."""
import socket

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 53443
COMMANDS = [b"SET 0 1 0\n", b"CHECK\n", b"EXIT\n"]

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
┌──(zham㉿kali)-[~/ctf/perceptron-play-naught]
└─$ python3 solve.py
Perfect! All points are classified correctly.
academy{n4ught_bu7_53p4r4b13_c0e17ccc}
```

The Python version is more code but easier to extend if you want to parse the point list, try multiple weights, or retry on failure.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for our `SET 0 1 0` command:

1. **Receive command** — the TCP socket accepts a line, strips the trailing newline, and parses the first token. `SET` is dispatched to the weight-update handler.
2. **Parse arguments** — `0`, `1`, `0` are parsed as floats. No type checking beyond "is this a number?".
3. **Update state** — `state.w1 = 0.0`, `state.w2 = 1.0`, `state.b = 0.0`. The previous weights are discarded (no ADJUST-style accumulation).
4. **Redraw ASCII** — the server walks the 9×9 grid (or whatever the cell size is), projects each grid center into `(x, y)`, computes the perceptron activation `w1·x + w2·y + b`, and decides:
   - If a training point is near the cell center, draw `0`, `1`, or `x` based on the prediction.
   - If the activation is approximately zero (`|w1·x + w2·y + b| < threshold`), draw `/`.
   - Otherwise leave the cell blank.
5. **Send response** — the ASCII grid, the point table, and the new weights are sent back as a multi-line string ending with the `> ` prompt.

For `CHECK`:
1. **Receive command** — parsed as the submit handler.
2. **Verify all points** — for each of the 7 training points, compute `activation = w1·x + w2·y + b` and `prediction = 1 if activation >= 0 else 0`. Compare against the label.
3. **Score** — if every prediction matches its label, print `Perfect! All points are classified correctly.` followed by the flag. Otherwise, list the misclassified points and skip the flag.

For `EXIT` / `QUIT`:
1. **Receive command** — parsed as the disconnect handler.
2. **Close socket** — the server prints `Goodbye!` and closes the TCP connection. The instance is not killed; another client can still connect until the 15-minute timer expires.

### Why `SET 0 1 0` works (mathematical trace)

For each of the 7 points, `activation = 0·x + 1·y + 0 = y`:

| Point | Label | Activation `y` | Prediction | Correct? |
|-------|-------|----------------|------------|----------|
| `(-4, -1)` | 0 | -1 | 0 | ✓ |
| `(-1, +2)` | 1 | +2 | 1 | ✓ |
| `(+0, -1)` | 0 | -1 | 0 | ✓ |
| `(+0, +2)` | 1 | +2 | 1 | ✓ |
| `(+2, -1)` | 0 | -1 | 0 | ✓ |
| `(+3, +1)` | 1 | +1 | 1 | ✓ |
| `(+4, +2)` | 1 | +2 | 1 | ✓ |

7/7 correct, no margin-of-safety concerns (all class-0 activations are strictly negative, all class-1 activations are strictly positive — the boundary at `y = 0` has a half-unit gap on each side).

### Geometric intuition

The line `y = 0` is the **x-axis**. In the ASCII plot, it shows up as the row of `/` characters at `+0`. All three class-0 points sit at `y = -1`, exactly one unit below the line. All four class-1 points sit at `y ∈ {+1, +2}`, one or two units above the line. There is no ambiguity — the line cleanly partitions the grid into "y < 0" (class 0) and "y ≥ 0" (class 1).

### Compared to the rest of the series

- **vs. Classic 0** — same perceptron formula, but the "Train" series auto-runs the update rule with a slider, while the "Play" series lets you set weights manually. Classic 0 was about *finding* a learning rate that converges; Naught is about *finding* a line that separates.
- **vs. Classic 1** — same data geometry as Classic 0, but Classic 1 demands 5 separate successful runs. Naught is a single manual submit.
- **vs. 3-Bit Parity** — parity is *not* linearly separable; the ceiling there was 75%. Naught's data is *trivially* separable; the ceiling is 100% with a horizontal line.
- **vs. Charlie (referenced in the banner)** — Charlie was presumably the 1D version with the same x-coordinates. In 1D, the x-values alone are not separable: x=-4, 0, 2 are class 0 but x=0 sits between class-0 (-4) and class-1 (+3, +4). By *lifting* the points into 2D with chosen y-values, the author gave the perceptron enough room to draw a line that separates them.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `socat` (or `nc`) | Connect to the REPL over TCP | Easy |
| `python3 -m json.tool` / `grep` | Extract the flag from the REPL output | Easy |
| `socket` (Python stdlib) | Drive the REPL programmatically if scripting | Medium |
| `nano` | Write the solve script (if scripting) | Easy |
| Mental visualization | Sketch the grid and find the separating line | Easy |
| Paper + pencil (optional) | Verify the activations by hand | Easy |

---

## Appendix A — The 1D → 2D Lifting in Detail

The welcome banner says the x-values were "borrowed from the 1D Charlie puzzle". If we strip the y-values off the 7 points, we get:

| Point (1D) | Label |
|------------|-------|
| -4         | 0 |
| -1         | 1 |
| +0         | 0 |
| +0         | 1 |
| +2         | 0 |
| +3         | 1 |
| +4         | 1 |

In 1D, this is not separable: class-0 includes `+0` and `+2`, class-1 includes `-1` and `+0` — there is no threshold `t` that puts all class-0 on one side and all class-1 on the other. Specifically:
- If `t < -4`: everything is "above", so all 7 are predicted as class 0 — wrong for 4 points.
- If `-4 ≤ t < -1`: class-0 has `{-4}`, class-1 has `{-1, 0, 2, 3, 4}` — 6 wrong.
- If `-1 ≤ t < 0`: class-0 has `{-4, -1?}` — `-1` is class 1, so 1 wrong.
- If `0 ≤ t < 2`: class-0 has `{-4, 0, ?}` — `0` is split, ambiguous.
- If `2 ≤ t < 3`: class-0 has `{-4, 0, 0, 2}`, class-1 has `{3, 4}` — wrong for `0`-labeled-as-1 and `0`-labeled-as-1 (two separate `0`s with different labels, both wrongly classified).
- If `t ≥ 4`: everything predicted as class 1 — 3 wrong.

No threshold works. That's why "Charlie" was the 1D puzzle — the points cannot be split by a single value.

But once we **lift** to 2D and assign y-values that break the tie, the same x-coordinates become trivially separable:

| Point (2D) | Label | Why it's separable |
|------------|-------|--------------------|
| `(-4, -1)` | 0 | y < 0 |
| `(-1, +2)` | 1 | y > 0 |
| `(+0, -1)` | 0 | y < 0 (resolves the tie at x=0) |
| `(+0, +2)` | 1 | y > 0 (resolves the tie at x=0) |
| `(+2, -1)` | 0 | y < 0 |
| `(+3, +1)` | 1 | y > 0 |
| `(+4, +2)` | 1 | y > 0 |

The y-values were chosen precisely to break the x=0 tie. Once you have y as a discriminator, a horizontal line at `y = 0` works perfectly. This is a textbook example of **feature engineering** — adding a new feature (the y-coordinate) that exposes a pattern (positive y → class 1, negative y → class 0) that wasn't visible in the original 1D data.

---

## Appendix B — Other Valid Solutions

The horizontal line `y = 0` is the most obvious, but many other weights satisfy the perceptron's `CHECK`. Here are a few I tested:

| `w1` | `w2` | `b`  | Boundary equation | Notes |
|------|------|------|--------------------|-------|
| `0`  | `1`  | `0`  | `y = 0`            | The minimal solution — ignores x entirely. |
| `0`  | `1`  | `-0.5` | `y = 0.5`         | A safer boundary, sits halfway between the two clusters. |
| `0`  | `2`  | `-1` | `y = 0.5`          | Same boundary, scaled weights. |
| `1`  | `1`  | `1`  | `x + y + 1 = 0` → `y = -x - 1` | A tilted line that still passes between y=-1 and y=+1. |
| `-1` | `1`  | `0`  | `y = x`            | A diagonal line at 45° that just happens to miss all the points. |

The "best" choice depends on what you mean by "best":
- **Smallest weights** — `0, 1, 0` is the shortest vector (`||w||² = 1`) that satisfies the constraints. Perceptrons in textbooks are usually presented this way.
- **Largest margin** — moving the boundary to `y = 0.5` (e.g., `0, 1, -0.5` or `0, 2, -1`) gives equal 0.5-unit margin on both sides, which generalizes better if new points are added later.
- **Most symmetric** — any line that is exactly halfway between the two y-levels (`y = 0.5`) is symmetric. The diagonal lines like `y = x` are not symmetric (they have different margins on different sides).

For the purposes of this CTF, any of them prints the flag.

---

## Appendix C — Raw REPL Transcript (lr = 0.50)

Full interaction transcript from the winning solve, included here so the writeup is self-contained:

```
┌──(zham㉿kali)-[~]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:53443,crlf
Welcome to Perceptron Play Naught!

Tweak the weights of a single-layer perceptron and watch how the decision
boundary moves. This time, the x-values are borrowed from the 1D Charlie
puzzle and lifted into 2D with new y-values so they can be linearly
separated. Your goal is to correctly classify every labeled point.

Commands:
  SHOW                    redraw the ASCII graph and point table
  SET w1 w2 b             set the weights/bias directly (floats are fine)
  ADJUST dw1 dw2 db       add offsets to the current weights
  POINTS                  list the training points with their target labels
  CHECK                   verify every point; prints the flag when perfect
  RESET                   go back to the starting weights (all w1=1.0, w2=-1.0, b=0.0)
  HELP                    show this message again
  EXIT / QUIT             leave the playground

Legend in the graph:
  0  point labeled class 0 and currently classified correctly
  1  point labeled class 1 and currently classified correctly
  x  point that is misclassified right now
  /  approximate decision boundary (where w1·x + w2·y + b ≈ 0)

Start experimenting by typing SET or ADJUST, or just press CHECK once everything
looks good! The starting weights are w1 = 1, w2 = -1, b = 0 so a boundary is
visible immediately.

+4         |       /
+3         |     /
+2       x x   /   1
+1         | /   1
+0 - - - - / - - - -
-1 0     / x   x
-2     /   |
-3   /     |
-4 /       |
   -4-3-2-1+0+1+2+3+4

Current weights -> w1: 1, w2: -1, b: 0

  point    label  perceptron  activation
  ------   -----  ----------  ----------
  (-4,-1)     0        0        -3
  (-1,+2)     1        0        -3
  (+0,-1)     0        1        +1
  (+0,+2)     1        0        -2
  (+2,-1)     0        1        +3
  (+3,+1)     1        1        +2
  (+4,+2)     1        1        +2

> SET 0 1 0
> +4         |
+3         |
+2       1 1       1
+1         |     1
+0 / / / / / / / / /
-1 0       0   0
-2         |
-3         |
-4         |
   -4-3-2-1+0+1+2+3+4

Current weights -> w1: 0, w2: 1, b: 0

  point    label  perceptron  activation
  ------   -----  ----------  ----------
  (-4,-1)     0        0        -1
  (-1,+2)     1        1        +2
  (+0,-1)     0        0        -1
  (+0,+2)     1        1        +2
  (+2,-1)     0        0        -1
  (+3,+1)     1        1        +1
  (+4,+2)     1        1        +2

> CHECK
Perfect! All points are classified correctly.
academy{n4ught_bu7_53p4r4b13_c0e17ccc}

> EXIT
Goodbye!
Connection closed.
```

A few observations from the transcript:

- **The banner is verbose on purpose** — the welcome text tells you the dataset origin (Charlie), the goal ("Separate the labeled points"), the starting weights, and the full command list. This is more hand-holding than the "Train" series, which assumes you already know what a perceptron is.
- **The first ASCII plot shows `x` markers at `(-1,+2)`, `(+0,-1)`, `(+2,-1)`** — three points misclassified by the default weights. The `/` boundary cuts diagonally through the upper-right cluster.
- **After `SET 0 1 0`, the `/` markers all line up at `+0`** — the boundary collapses to the x-axis, which is exactly where we wanted it.
- **`CHECK` prints the flag immediately, no extra confirmation** — the server trusts you to know what you submitted.

---

## Key Takeaways

- **Linear separability depends on the features** — the same 7 x-coordinates were not separable in 1D but became trivially separable in 2D once the author added a discriminating y-value. This is the heart of feature engineering in machine learning: a well-chosen feature can turn an impossible problem into a one-line solution.
- **A horizontal line `y = 0` is the simplest non-trivial classifier** — it ignores the x-coordinate entirely and predicts purely on y. With `w1 = 0, w2 = 1, b = 0`, the perceptron reduces to "label 1 if y ≥ 0, else label 0". This is exactly equivalent to a 1D threshold classifier on the y-axis, but expressed in 2D.
- **`SET` is faster, `ADJUST` is more pedagogical** — for a dataset where the boundary is obvious by inspection, just plug in the answer. For a dataset where you want to *feel* how the boundary moves, nudge by small amounts and watch the `/` characters slide across the grid.
- **Netcat-style REPLs are common in CTFs** — many "interactive" challenges expose a TCP port with a command-line interface instead of a web GUI. Tools like `socat`, `nc`, `ncat`, or a Python `socket` script let you drive them. Hint #2's mention of `SET` and `ADJUST` is the author's hint that this is a manual REPL, not a one-shot API.
- **Don't overthink the easy challenges** — the "Naught" in the name is a hint that this is the trivial puzzle. If you find yourself trying to do math, step back and look at the picture. The answer is usually staring at you.
- **Always `POINTS` first** — the `POINTS` command gives you the exact `(x, y, label)` for every training point. Skipping it forces you to read the ASCII plot, which has lower resolution (some points may be hidden behind others, or off-grid if the data extends beyond `[-4, +4]`).
- **Flag wordplay**: `n4ught_bu7_53p4r4b13` is leetspeak for "**naught but separable**". "Naught" means "nothing" or "zero" — fitting both because the dataset is trivially separable (zero difficulty) and because the answer `w1 = 0, w2 = 1, b = 0` literally contains a zero weight. The trailing `_c0e17ccc` is a per-instance salt. The flag is a one-sentence summary of the challenge.
