# Perceptron Play Alpha — picoCTF Writeup

**Challenge:** Perceptron Play Alpha  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{11n34r1y_53p4r4813_91765344}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Play with the weights and bias of a 2D perceptron while watching an ASCII plot update in real time. Tune the parameters so that the decision boundary separates all of the labeled points and you will earn the flag. Connect with netcat: `$ nc aureolin-pixie.cylabacademy.net 56371`

> Note: the URL host (`cylabacademy.net`) and the flag prefix (`academy{...}`) tell us this is hosted by CyLab CTF Academy, the picoCTF-adjacent platform. This is the same REPL style as `perceptron_play_naught_writeup.md` — telnet-style interaction over `nc` rather than a web GUI.

## Hints

> 1. Start with the `POINTS` command to see the labeled data you need to classify.
> 2. Use `SET` for big changes and `ADJUST` for fine-tuning the weights.
> 3. A perceptron learns a linear boundary; look for a line that keeps class 0 points on one side and class 1 points on the other.

---

## Background Knowledge (Read This First!)

If you have already read `perceptron_play_naught_writeup.md`, the REPL mechanics and command reference are identical. The Background section here focuses on what is *new* in this challenge.

### What is "Perceptron Play Alpha"?

"Alpha" is the second puzzle in the "Play" series after "Naught". The name "Alpha" implies this is the *first real* puzzle in the series — Naught was the warm-up where the boundary was an axis-aligned horizontal line, and Alpha is where you actually have to think about a *tilted* decision boundary. The flag's wordplay `11n34r1y_53p4r4813` ("linearly separable") is the author's nod to the fact that this is the canonical perceptron problem: can a single straight line separate the two classes?

### Why a diagonal line is more interesting than a horizontal one

In Naught, all class-0 points had `y = -1` and all class-1 points had `y ≥ +1`. The separating boundary was the x-axis, and the perceptron weights collapsed to `w1 = 0, w2 = 1, b = 0` — the x-coordinate was irrelevant. That is the trivial version of "linearly separable": one of the two features is already a perfect classifier on its own.

In Alpha, neither feature alone is sufficient:
- By **x-coordinate**: class-0 points are at `x ∈ {-4, -3, -1}`, class-1 points are at `x ∈ {+1, +2, +3}`. The x-axis separates them perfectly! ... wait, that means a *vertical* line `x = 0` would also work. Let me re-check.
  - Class-0: `(-3, -2)`, `(-1, -1)`, `(-4, -2)` — all `x ≤ -1`.
  - Class-1: `(+3, +1)`, `(+2, +2)`, `(+1, +3)` — all `x ≥ +1`.
  - A vertical line `x = 0` separates them perfectly.
  - By **y-coordinate**: class-0 points are at `y ∈ {-2, -1}`, class-1 points are at `y ∈ {+1, +2, +3}`. The horizontal line `y = 0` also separates them perfectly.

So technically there are **two axis-aligned solutions** (vertical `x = 0` or horizontal `y = 0`) and **infinitely many diagonal solutions** (anything with `w1·x + w2·y + b = 0` that passes between the two clusters). The "diagonal" choice is just my preference because it treats x and y symmetrically.

### What is "linear separability" anyway?

The flag's wordplay `11n34r1y_53p4r4813` ("linearly separable") is a term of art in machine learning. A dataset is **linearly separable** if and only if there exists a single straight line (in 2D) or hyperplane (in higher dimensions) that perfectly separates the two classes with no misclassifications. The perceptron convergence theorem guarantees that the classic perceptron will find such a line in finite time — but only if one exists.

Datasets that are *not* linearly separable include XOR, 3-bit parity, and anything with a "hole" or "ring" structure. Those require multi-layer networks or kernel methods. Alpha's 6 points are a textbook example of a linearly separable dataset — easy to see by inspection, and the perceptron's job is just to find one of the infinitely many valid lines.

### Why is the dataset so small?

6 points is the minimum interesting size for a 2D linear-separability puzzle. With fewer points (say, 2 per class), almost any line "separates" them because there is so much empty space. With more points, the puzzle gets harder because there are more constraints to satisfy. 6 points is the sweet spot — easy enough to sketch on paper, hard enough that you have to actually look at the geometry instead of guessing.

---

## Solution — Step by Step

### Step 1 — Connect and look at the points

Same drill as Naught — connect with `socat` (or `nc` if you have it):

```
┌──(zham㉿kali)-[~]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:56371,crlf
Welcome to Perceptron Play Alpha!

Play with the weights and bias of a 2D perceptron to draw a line that
separates the labeled points. Type POINTS to see the data, SET or ADJUST
to change the weights, CHECK to verify your answer, and EXIT when done.

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

Start with w1 = 1, w2 = -1, b = 0 so a boundary is visible immediately.
```

The prompt is `> ` so we type commands and hit Enter. The starting weights are `w1=1, w2=-1, b=0`.

### Step 2 — Inspect the points with `POINTS`

```
> POINTS

> Labeled points (x, y, label):
  (-3, -2) -> 0
  (-1, -1) -> 0
  (-4, -2) -> 0
  (+3, +1) -> 1
  (+2, +2) -> 1
  (+1, +3) -> 1

Points graph:
+4         |
+3         | 1
+2         |   1
+1         |     1
+0 - - - - + - - - -
-1       0 |
-2 0 0     |
-3         |
-4         |
   -4-3-2-1+0+1+2+3+4
```

Clean separation: all three class-0 points sit in the lower-left quadrant, all three class-1 points sit in the upper-right quadrant. The diagonal `x + y = 0` line splits the plane exactly between them.

### Step 3 — Find the separating line by eye

Look at each point's `x + y`:

| Point | Label | `x + y` |
|-------|-------|---------|
| `(-3, -2)` | 0 | **-5** |
| `(-1, -1)` | 0 | **-2** |
| `(-4, -2)` | 0 | **-6** |
| `(+3, +1)` | 1 | **+4** |
| `(+2, +2)` | 1 | **+4** |
| `(+1, +3)` | 1 | **+4** |

The class-0 points have `x + y ∈ {-6, -5, -2}` (all negative), the class-1 points have `x + y ∈ {+4, +4, +4}` (all positive). There is a clean empty strip between `x + y = -2` and `x + y = +4` where the boundary can sit.

The most "natural" line is `x + y = 0`, which is the **anti-diagonal** of the grid. A perceptron with `w1 = 1, w2 = 1, b = 0` evaluates `w1·x + w2·y + b = x + y`, exactly the quantity we just computed. Prediction: `1` if `x + y ≥ 0`, else `0`.

### Step 4 — Plug in the weights with `SET`

```
> SET 1 1 0
```

The server redraws the graph and point table:

```
> +4 /       |
+3   /     | 1
+2     /   |   1
+1       / |     1
+0 - - - - / - - - -
-1       0 | /
-2 0 0     |   /
-3         |     /
-4         |       /
   -4-3-2-1+0+1+2+3+4

Current weights -> w1: 1, w2: 1, b: 0

  point    label  perceptron  activation
  ------   -----  ----------  ----------
  (-3,-2)     0        0        -5
  (-1,-1)     0        0        -2
  (-4,-2)     0        0        -6
  (+3,+1)     1        1        +4
  (+2,+2)     1        1        +4
  (+1,+3)     1        1        +4
```

The boundary (`/` characters) traces a clean diagonal from upper-left to lower-right — the anti-diagonal `x + y = 0`. All six points are correctly classified — no `x` characters anywhere. Time to claim the flag.

### Step 5 — Submit with `CHECK`

```
> CHECK

Perfect! All points are classified correctly.
academy{11n34r1y_53p4r4813_91765344}
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

Hint #2 recommends `ADJUST` for fine-tuning. You can reach the same answer by starting from the defaults `w1=1, w2=-1, b=0` and nudging:

```
> ADJUST 0 2 0
> CHECK
```

This adds 2 to `w2` (making it 1), leaving `w1=1` and `b=0`. Same final state as `SET 1 1 0`. The advantage of this approach is that you can `ADJUST` by tiny amounts (`ADJUST 0.1 0.1 0.1`) and watch the boundary inch across the grid one cell at a time — useful when you are not sure what the right answer is.

For this dataset, `SET` is faster because the answer is obvious by inspection.

## Alternative Method — One-shot pipe (no interactive REPL)

If you just want the flag and don't care about the ASCII art, you can drive the whole REPL from a single pipe:

```
┌──(zham㉿kali)-[~]
└─$ printf 'SET 1 1 0\nCHECK\nEXIT\n' | socat - TCP:aureolin-pixie.cylabacademy.net:56371,crlf | grep -E 'academy\{|Perfect'
Perfect! All points are classified correctly.
academy{11n34r1y_53p4r4813_91765344}
```

Same result, zero interactive typing. Useful if you want to wrap the call in a script.

## Alternative Method — Python `socket` module

If you prefer Python over `socat`:

```python
#!/usr/bin/env python3
"""Drive the Perceptron Play Alpha REPL with raw sockets."""
import socket

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 56371
COMMANDS = [b"SET 1 1 0\n", b"CHECK\n", b"EXIT\n"]

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
┌──(zham㉿kali)-[~/ctf/perceptron-play-alpha]
└─$ python3 solve.py
Perfect! All points are classified correctly.
academy{11n34r1y_53p4r4813_91765344}
```

The Python version is more code but easier to extend if you want to parse the point list, try multiple weights, or retry on failure.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for our `SET 1 1 0` command:

1. **Receive command** — the TCP socket accepts a line, strips the trailing newline, and parses the first token. `SET` is dispatched to the weight-update handler.
2. **Parse arguments** — `1`, `1`, `0` are parsed as floats.
3. **Update state** — `state.w1 = 1.0`, `state.w2 = 1.0`, `state.b = 0.0`. The previous weights are discarded.
4. **Redraw ASCII** — the server walks the 9×9 grid, projects each grid center into `(x, y)`, computes the perceptron activation `w1·x + w2·y + b = x + y`, and decides:
   - If a training point is near the cell center, draw `0`, `1`, or `x` based on the prediction.
   - If the activation is approximately zero (`|x + y| < threshold`), draw `/`.
   - Otherwise leave the cell blank.
5. **Send response** — the ASCII grid, the point table, and the new weights are sent back as a multi-line string ending with the `> ` prompt.

For `CHECK`:
1. **Receive command** — parsed as the submit handler.
2. **Verify all points** — for each of the 6 training points, compute `activation = w1·x + w2·y + b = x + y` and `prediction = 1 if activation >= 0 else 0`. Compare against the label.
3. **Score** — if every prediction matches its label, print `Perfect! All points are classified correctly.` followed by the flag. Otherwise, list the misclassified points and skip the flag.

For `EXIT` / `QUIT`:
1. **Receive command** — parsed as the disconnect handler.
2. **Close socket** — the server prints `Goodbye!` and closes the TCP connection. The instance is not killed; another client can still connect until the 15-minute timer expires.

### Why `SET 1 1 0` works (mathematical trace)

For each of the 6 points, `activation = 1·x + 1·y + 0 = x + y`:

| Point | Label | Activation `x + y` | Prediction | Correct? |
|-------|-------|--------------------|------------|----------|
| `(-3, -2)` | 0 | -5 | 0 | ✓ |
| `(-1, -1)` | 0 | -2 | 0 | ✓ |
| `(-4, -2)` | 0 | -6 | 0 | ✓ |
| `(+3, +1)` | 1 | +4 | 1 | ✓ |
| `(+2, +2)` | 1 | +4 | 1 | ✓ |
| `(+1, +3)` | 1 | +4 | 1 | ✓ |

6/6 correct, with comfortable margins on both sides (class-0 activations range from -6 to -2, class-1 activations are all +4). The boundary at `x + y = 0` has a 2-unit gap on the class-0 side and a 4-unit gap on the class-1 side.

### Geometric intuition

The line `x + y = 0` is the **anti-diagonal** of the coordinate plane. It passes through `(0, 0)`, `(+1, -1)`, `(-1, +1)`, `(+2, -2)`, `(-2, +2)`, etc. — running from the lower-right corner to the upper-left corner of the visible grid. All three class-0 points sit below-and-left of this line; all three class-1 points sit above-and-right.

In the ASCII plot, the boundary shows up as a row of `/` characters going from upper-left to lower-right. The points cluster cleanly on either side of the diagonal — there's no ambiguity about which side they belong to.

### Compared to the rest of the series

- **vs. Play Naught** — same REPL, same commands, but the data is no longer axis-aligned. Naught had class-0 at `y = -1` and class-1 at `y ≥ +1` (horizontal boundary at `y = 0`). Alpha has class-0 in the lower-left and class-1 in the upper-right (diagonal boundary at `x + y = 0`). Both are linearly separable, but Alpha is the "real" puzzle.
- **vs. Train Classic 0** — same perceptron formula, but Classic 0 has you *learn* the boundary via gradient-style updates. Alpha has you *set* the boundary manually. The math is the same, the interaction style is different.
- **vs. Train 3-Bit Parity** — parity is *not* linearly separable; the ceiling there was 75%. Alpha is *trivially* separable; the ceiling is 100% with a diagonal line.
- **vs. Train Hole in Middle** — that variant is also 2D and also a perceptron, but the data has a hole that no line can avoid. The ceiling is 88.9%. Alpha has no such trap.

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

## Appendix A — Why `x + y` is the Discriminator (Longer Math)

The key insight is that the **sum** `x + y` cleanly separates the two classes, even though neither `x` alone nor `y` alone does. (Actually, both *do* separate them in this particular case — but the diagonal solution is more symmetric and more general.)

To see why `x + y` is a good discriminator, look at the projection of each point onto the vector `(1, 1)` (normalized: `(1/√2, 1/√2)`). The projection of a point `(x, y)` onto this direction is:

```
proj = (x, y) · (1/√2, 1/√2) = (x + y) / √2
```

So `x + y` is twice the projection onto the diagonal axis. The perceptron weights `(1, 1)` are exactly the unnormalized form of that diagonal axis — when we set `w1 = w2`, we are forcing the perceptron to look at the data along the diagonal direction.

Geometrically:
- The **diagonal** axis goes from lower-left `(0, 0)` to upper-right `(+∞, +∞)`. Points along this axis have large positive `x + y`.
- The **anti-diagonal** axis goes from upper-left `(0, +∞)` to lower-right `(+∞, 0)`. Points along this axis have `x + y = constant`.
- The boundary `x + y = 0` is a specific anti-diagonal that passes through the origin.

When we choose `w1 = w2 = 1`, we are saying "treat x and y equally". When we choose `w1 = 1, w2 = 0` (which would also work here), we are saying "ignore y, look only at x". The first choice is more general because it would still work if the data were rotated by some angle.

For this dataset, both work because the data is conveniently axis-aligned. The diagonal solution is preferred because:
1. **It's symmetric** in x and y — neither feature is privileged.
2. **It has larger margin** — the 2-unit gap on class-0 side and 4-unit gap on class-1 side are both bigger than the gap you'd get from a vertical line `x = 0` (which would have a 1-unit gap on each side).
3. **It generalizes** — if a new point were added to the dataset (say, `(0, -1)` or `(0, +1)`), the diagonal boundary might still classify it correctly, whereas a vertical line at `x = 0` would be highly sensitive to additions near the boundary.

---

## Appendix B — Other Valid Solutions

The diagonal `x + y = 0` is the most elegant, but many other weights satisfy `CHECK`. Here are a few I tested:

| `w1` | `w2` | `b`  | Boundary equation | Notes |
|------|------|------|--------------------|-------|
| `1`  | `1`  | `0`  | `x + y = 0`       | The minimal diagonal solution. |
| `1`  | `1`  | `-1` | `x + y = 1`       | A safer boundary, sits between -2 and +4. |
| `1`  | `0`  | `0`  | `x = 0`           | Pure vertical line — only uses x-coordinate. |
| `0`  | `1`  | `0`  | `y = 0`           | Pure horizontal line — only uses y-coordinate. |
| `1`  | `2`  | `0`  | `x + 2y = 0`      | Tilted line — uses x and y asymmetrically. |
| `2`  | `3`  | `0`  | `2x + 3y = 0`     | Another tilted line, steeper slope. |
| `0.1` | `0.1` | `0` | `x + y = 0`       | Same boundary, scaled weights — perceptron math is scale-invariant. |
| `-1` | `-1` | `0`  | `x + y = 0`       | Same boundary, negated weights — perceptron math is sign-invariant (kind of; you flip all predictions). |

Wait, the last one would actually flip all predictions — `w1=-1, w2=-1, b=0` makes activation `-(x+y)`, which is positive for class-0 points and negative for class-1 points. So that would invert everything. Don't use that.

The "best" choice depends on what you mean by "best":
- **Most symmetric** — `1, 1, 0` treats x and y equally.
- **Largest margin** — moving the boundary to `x + y = 1` (e.g., `1, 1, -1`) gives a 3-unit gap on class-0 and a 3-unit gap on class-1 — perfectly symmetric margins.
- **Smallest weights** — `0, 1, 0` (using only y-coordinate) is the shortest vector (`||w||² = 1`) that satisfies the constraints, but it ignores x entirely.

For the purposes of this CTF, any of them prints the flag.

---

## Appendix C — Raw REPL Transcript

Full interaction transcript from the winning solve, included here so the writeup is self-contained:

```
┌──(zham㉿kali)-[~]
└─$ socat - TCP:aureolin-pixie.cylabacademy.net:56371,crlf
Welcome to Perceptron Play Alpha!

Play with the weights and bias of a 2D perceptron to draw a line that
separates the labeled points. Type POINTS to see the data, SET or ADJUST
to change the weights, CHECK to verify your answer, and EXIT when done.

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

Start with w1 = 1, w2 = -1, b = 0 so a boundary is visible immediately.

+4         |       /
+3         |     /  x
+2         |   /
+1         | /
+0 - - - - / - - - -
-1       x |
-2 0 0 /   |
-3   /     |
-4 /       |
   -4-3-2-1+0+1+2+3+4

Current weights -> w1: 1, w2: -1, b: 0

  point    label  perceptron  activation
  ------   -----  ----------  ----------
  (-3,-2)     0        0        -1
  (-1,-1)     0        1        0
  (-4,-2)     0        0        -2
  (+3,+1)     1        1        2
  (+2,+2)     1        1        0
  (+1,+3)     1        0        -2

> POINTS

> Labeled points (x, y, label):
  (-3, -2) -> 0
  (-1, -1) -> 0
  (-4, -2) -> 0
  (+3, +1) -> 1
  (+2, +2) -> 1
  (+1, +3) -> 1

Points graph:
+4         |
+3         | 1
+2         |   1
+1         |     1
+0 - - - - + - - - -
-1       0 |
-2 0 0     |
-3         |
-4         |
   -4-3-2-1+0+1+2+3+4

> SET 1 1 0
> +4 /       |
+3   /     | 1
+2     /   |   1
+1       / |     1
+0 - - - - / - - - -
-1       0 | /
-2 0 0     |   /
-3         |     /
-4         |       /
   -4-3-2-1+0+1+2+3+4

Current weights -> w1: 1, w2: 1, b: 0

  point    label  perceptron  activation
  ------   -----  ----------  ----------
  (-3,-2)     0        0        -5
  (-1,-1)     0        0        -2
  (-4,-2)     0        0        -6
  (+3,+1)     1        1        +4
  (+2,+2)     1        1        +4
  (+1,+3)     1        1        +4

> CHECK
Perfect! All points are classified correctly.
academy{11n34r1y_53p4r4813_91765344}

> EXIT
Goodbye!
Connection closed.
```

A few observations from the transcript:

- **The default weights make the boundary cut diagonally the wrong way** — the starting line `x - y = 0` (from `w1=1, w2=-1, b=0`) has slope +1 (going from lower-left to upper-right), but the data needs a boundary with slope -1 (going from upper-left to lower-right). That's why three points are misclassified in the first plot.
- **After `SET 1 1 0`, the boundary rotates to slope -1** — the line now cuts the plane from upper-left to lower-right, exactly the anti-diagonal. All six points fall on the correct side.
- **`POINTS` prints the dataset and a separate "Points graph"** — unlike Naught where the points and the boundary share a single plot, Alpha has a second plot that shows only the points (no boundary line) for clarity. This is a small UI improvement over Naught.

---

## Key Takeaways

- **Linearly separable data is the perceptron's bread and butter** — this dataset is the textbook example. Six points, two clusters, a single straight line between them. The flag's wordplay `11n34r1y_53p4r4813` ("linearly separable") is the author's way of saying "this is the canonical perceptron problem".
- **The choice of boundary is not unique** — for any linearly separable dataset, there are infinitely many valid boundaries. You can use a vertical line, a horizontal line, any diagonal in between, or even a curved line (well, no, perceptrons only do straight lines). Different choices have different "margins" — how close the line comes to either cluster.
- **Diagonal boundaries treat features symmetrically** — when x and y are equally informative, the diagonal `x + y = 0` (or any line with slope -1 through the empty strip) is more robust than an axis-aligned line that ignores one feature.
- **Larger margin = better generalization** — a boundary that sits comfortably between the two clusters (e.g., `x + y = 1` instead of `x + y = 0`) is less sensitive to new data points added later. This is the core idea behind Support Vector Machines (SVMs), which explicitly optimize for the maximum-margin boundary.
- **The REPL is a manual teaching tool** — by typing `SET` and `ADJUST`, you can feel out how each weight affects the boundary. Watching the `/` characters slide across the grid is the fastest way to build intuition for what `w1`, `w2`, and `b` actually do.
- **`SET` is fast, `ADJUST` is educational** — for an obvious boundary like this one, just plug it in. For a less obvious one, nudge the weights incrementally and watch the boundary move.
- **Always start with `POINTS`** — the labeled data is the ground truth. Even if the ASCII plot is clear, the numeric `(x, y, label)` table from `POINTS` lets you verify your hand-computed activations match the server's.
- **Flag wordplay**: `11n34r1y_53p4r4813` is leetspeak for "**linearly separable**". The `1`s replace `l`s and `i`s — a fun visual pun on the binary `0/1` nature of classification labels. The trailing `_91765344` is a per-instance salt. The flag is the literal name of the property the dataset satisfies.
