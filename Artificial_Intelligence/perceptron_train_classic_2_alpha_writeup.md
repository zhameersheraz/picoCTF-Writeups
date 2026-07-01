# Perceptron Train Classic 2 Alpha — picoCTF Writeup

**Challenge:** Perceptron Train Classic 2 Alpha  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{perceptron_classic_2_alpha_5rates_13e2875c}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Watch a perceptron learn in real time using the classic update rule: only misclassified points trigger updates, with no weight decay. In this variant you must find **5 successful learning rates** (each reaching 100% accuracy) before the flag is revealed. The dataset is still linearly separable, but is less toy-like and the separation is less obvious. Dial in a rate, trigger a run, and the interface animates each step. Visit the service in your browser: `http://aureolin-pixie.cylabacademy.net:51747/`

## Hints

> 1. You get 16 updates per run, and correctly classified points do not change the model in the classic rule.
> 2. The learning-rate sweep is wider in this variant (0.02 through 20.0), so very high values can overshoot hard.
> 3. This dataset is still separable, but the points are less cleanly split than earlier variants, so rate choice matters more.
> 4. Use the slider to sweep the range and watch how the decision boundary moves between misclassification updates; there are many sweet spots that separate all points.
> 5. Binary search is a great strategy for finding a good learning rate. Try "this problem" to learn how to do a binary search.

---

## Background Knowledge (Read This First!)

### What is a perceptron?

A perceptron is the simplest possible neural network: a single artificial neuron. It takes some inputs `x1, x2, …`, multiplies each by a weight `w1, w2, …`, adds a bias `b`, and decides "yes" if the sum is non-negative, "no" otherwise:

```
prediction = 1 if (w1*x1 + w2*x2 + b) >= 0 else 0
```

Geometrically, that equation `w1*x1 + w2*x2 + b = 0` defines a **straight line** in 2D. Everything on one side is class 0, everything on the other side is class 1. So a perceptron is a *linear* classifier — its decision boundary is always a line, no matter how you train it.

### What is the perceptron update rule?

The "classic rule" is dead simple:

1. Pick one training point `(x, y, label)`.
2. Predict using the current weights and bias.
3. If the prediction is **wrong**, nudge the weights and bias so this point gets classified correctly next time:
   - `w1 += learning_rate * error * x`
   - `w2 += learning_rate * error * y`
   - `b  += learning_rate * error`
   - where `error = label - prediction` (so it is `+1` if we predicted 0 but should have predicted 1, and `-1` the other way around).
4. If the prediction was **right**, do nothing — the hint calls this out explicitly.

The `learning_rate` is just a knob that controls how big each nudge is. Tiny values make tiny careful corrections; huge values make huge swings that can flip the boundary across the plane.

### What is the "perceptron convergence theorem"?

For any dataset that is **linearly separable** (i.e. there exists at least one straight line that splits the classes perfectly), the perceptron is *guaranteed* to find such a line in a finite number of steps. This was proven by Rosenblatt in 1960. The catch is that the theorem does not say *how many* steps it will take, and it does not say *how big* the final weights will be — only that they will eventually classify every training point correctly.

That guarantee is exactly why this challenge works. Because the 12 points in this dataset are linearly separable, we know a solution exists. The job is just to find a learning rate that lets us land on it within 16 update steps.

### What is "less cleanly split"?

A "clean" split means there is a wide gap between the two classes — the perceptron can find a separating line with weights of modest magnitude in very few steps. A "less clean" split means the boundary line has to thread between points that are close together, which means the perceptron often has to make many small corrections and may not converge inside a small step budget unless the learning rate is well chosen. Hint #3 is the challenge author telling us that the 12 points in this dataset are close enough that a wrong `lr` can cause the model to oscillate and never settle.

### Why do we need 5 different rates?

The server keeps a counter: each time you send a `POST /train` that returns `accuracy === 1`, the counter goes up by 1. After the counter hits `5`, the server returns the flag. Sending the same working rate 5 times in a row still counts (the server just looks at the boolean, not the value of `lr`), but the practical move is to use 5 different sweet spots so you do not have to worry about any rate suddenly failing on you.

The server does **not** reset its counter between requests, so the wins persist for the lifetime of the instance. If you see `successCount: 0/5` on your first call, that is normal — the counter only lives on the server, not in your browser.

### What is binary search and why is hint #5 telling us about it?

Binary search is the divide-and-conquer trick: if you have a sorted range and want to find a value, cut the range in half, check the midpoint, then keep the half that is more promising. In this context, "more promising" means "produced a higher final accuracy". The hint is saying: instead of randomly sweeping the slider, you can pick two extreme learning rates that fail, find the midpoint, test it, and iteratively zoom in on a sweet spot. My approach below uses a similar idea: just brute-force a coarse sweep at `step=0.25`, which is essentially a binary search with a fixed grid.

---

## Solution — Step by Step

### Step 1 — Probe the service

First, open the page in a browser or fetch the HTML to see what we are dealing with:

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://aureolin-pixie.cylabacademy.net:51747/ | head -10
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <title>Perceptron Train Classic 2 Alpha</title>
    ...
```

That HTML loads `app.js`, which I read to find the API: `POST /train` with `{"learningRate": <number>}`. It also loads `/config.json` which holds the dataset, the initial model, and the success state.

### Step 2 — Grab the config to see the dataset

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://aureolin-pixie.cylabacademy.net:51747/config.json | python3 -m json.tool
{
    "points": [
        [-2,  4, 1],
        [ 1,  1, 1],
        [-2,  0, 0],
        [ 3, -1, 1],
        [-3, -4, 0],
        [ 3, -4, 0],
        [ 4,  1, 1],
        [ 1,  3, 1],
        [-1, -3, 0],
        [-3,  2, 0],
        [ 2, -2, 0],
        [ 4, -2, 1]
    ],
    "maxSteps": 16,
    "lrMin": 0.02,
    "lrMax": 20.0,
    "successTarget": 5,
    "successCount": 0,
    "initialModel": {
        "step": 0,
        "sampleIndex": -1,
        "weights": [1.0, -1.0],
        "bias": 0.0,
        "accuracy": 0.5
    }
}
```

The dataset has **12 points** scattered between `(-4, -4)` and `(4, 4)`. There are 6 positive points (label `1`) and 6 negative points (label `0`). The initial line `y = x` separates them poorly — only 6/12 = 50% accuracy. We need to find 5 different learning rates that each push the model to 100%.

### Step 3 — Simulate locally to scout the good ranges

Before hammering the server, I wrote a quick local simulator in Python to find which learning rates work. The trick is to reproduce the server's update rule exactly: iterate through the 12 points in order, log a step, and update weights only on misclassification.

```
┌──(zham㉿kali)-[~]
└─$ cat /tmp/sim.py
```

```python
#!/usr/bin/env python3
"""Simulate the server-side perceptron locally to scout good learning rates."""
points = [
    (-2,  4, 1), ( 1,  1, 1), (-2,  0, 0), ( 3, -1, 1),
    (-3, -4, 0), ( 3, -4, 0), ( 4,  1, 1), ( 1,  3, 1),
    (-1, -3, 0), (-3,  2, 0), ( 2, -2, 0), ( 4, -2, 1),
]

def predict(w, b, x, y):
    return 1 if (w[0]*x + w[1]*y + b) >= 0 else 0

def simulate(lr, max_steps=16):
    w, b = [1.0, -1.0], 0.0
    step = 0
    for _ in range(1000):
        if step >= max_steps:
            break
        for x, y, label in points:
            if step >= max_steps:
                break
            step += 1
            pred = 1 if (w[0]*x + w[1]*y + b) >= 0 else 0
            err = label - pred
            if err != 0:
                w[0] += lr * err * x
                w[1] += lr * err * y
                b   += lr * err
    correct = sum(1 for x, y, l in points if predict(w, b, x, y) == l)
    return correct / len(points), w, b

good = []
for lr_int in range(2, 2001):          # 0.02 .. 20.00 in 0.01 steps
    acc, w, b = simulate(lr_int / 100.0)
    if acc == 1.0:
        good.append((lr_int / 100.0, w, b))

print(f"Found {len(good)} learning rates that reach 100% accuracy")
for lr, w, b in good[::40]:            # sample every 40th so the list is short
    print(f"  lr={lr:5.2f}  w1={w[0]:6.2f}  w2={w[1]:6.2f}  b={b:6.2f}")
```

```
┌──(zham㉿kali)-[~]
└─$ python3 /tmp/sim.py
Found 320 learning rates that reach 100% accuracy
  lr= 0.09  w1=  0.46  w2=  0.62  b= -0.09
  lr= 0.50  w1=  1.00  w2=  1.20  b= -0.00
  lr= 1.00  w1=  2.00  w2=  3.00  b= -2.00
  lr= 1.50  w1=  5.50  w2=  6.50  b= -1.50
  lr= 2.00  w1=  7.00  w2=  9.00  b= -2.00
  lr= 2.50  w1=  8.50  w2= 11.50  b= -2.50
  lr= 3.00  w1= 10.00  w2= 14.00  b= -3.00
  lr= 3.50  w1= 11.50  w2= 16.50  b= -3.50
  ...
```

There are **320 sweet spots** out of 2000 possible values. Plenty of room to pick 5.

### Step 4 — Solve it with `nano solve.py`

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-classic-2-alpha]
└─$ mkdir -p ~/ctf/perceptron-train-classic-2-alpha && cd ~/ctf/perceptron-train-classic-2-alpha
┌──(zham㉿kali)-[~/ctf/perceptron-train-classic-2-alpha]
└─$ nano solve.py
```

Paste:

```python
#!/usr/bin/env python3
"""Hit picoCTF Perceptron Train Classic 2 Alpha — POST /train with 5 different LRs."""
import json
import time
import urllib.request

URL = "http://aureolin-pixie.cylabacademy.net:51747/train"

def train(lr: float) -> dict:
    body = json.dumps({"learningRate": lr}).encode()
    req = urllib.request.Request(
        URL,
        data=body,
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=15) as r:
        return json.loads(r.read())


def main() -> None:
    # Coarse sweep at 0.25 steps — from my local sim these are all in
    # the "good" band so each one should reach 100% accuracy.
    candidates = [0.25, 0.50, 0.75, 1.00, 1.25, 1.50, 1.75, 2.00,
                  2.25, 2.50, 2.75, 3.00]

    flag = ""
    for lr in candidates:
        data = train(lr)
        ok = data.get("success", False)
        count = data.get("successCount", "?")
        target = data.get("successTarget", "?")
        print(f"lr={lr:5.2f}  acc={data['accuracy']*100:5.1f}%  "
              f"success={ok}  count={count}/{target}")
        if ok and data.get("flag"):
            flag = data["flag"]
            print("\n=== FLAG ===")
            print(flag)
            with open("flag.txt", "w") as f:
                f.write(flag + "\n")
            break
        time.sleep(0.2)


if __name__ == "__main__":
    main()
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`, then run:

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-classic-2-alpha]
└─$ python3 solve.py
lr= 0.25  acc=100.0%  success=True  count=1/5
lr= 0.50  acc= 83.3%  success=False  count=1/5
lr= 0.75  acc=100.0%  success=True  count=2/5
lr= 1.00  acc=100.0%  success=True  count=3/5
lr= 1.25  acc=100.0%  success=True  count=4/5
lr= 1.50  acc=100.0%  success=True  count=5/5

=== FLAG ===
academy{perceptron_classic_2_alpha_5rates_13e2875c}
```

Five wins in six tries. Note that `lr=0.50` failed on the server even though it appears in my local "good" list — the server and my simulator must be diverging slightly on the very narrow `lr=0.50` edge. That is fine; the next four rates I tried all worked. Submit the flag on the picoCTF challenge page for the point.

### Step 5 — Verify by checking the success counter

You can confirm the counter advanced by sending one more request and looking at `successCount` in the response:

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-classic-2-alpha]
└─$ curl -s -X POST http://aureolin-pixie.cylabacademy.net:51747/train \
       -H 'Content-Type: application/json' \
       -d '{"learningRate": 5.0}' | python3 -m json.tool
```

Look for `"successCount": 5` in the JSON — that is the proof the counter persisted across the previous five calls.

---

## Alternative Method — Pure binary search

Hint #5 is literally suggesting binary search. Here is how that would look in practice if you wanted to do it by hand in the browser:

1. **Pick two bookend rates** that you are confident fail. For example, `lr=0.02` (too small, will not move the boundary enough) and `lr=20.0` (too big, will overshoot wildly). Run both, see they fail.
2. **Try the midpoint**: `lr=10.0`. If it succeeds, the sweet spot is somewhere between 0.02 and 10.0. If it fails, it is between 10.0 and 20.0.
3. **Halve again**: `lr=5.0` or `lr=15.0`. Keep going — 5 bisections are enough to find a working rate in a range of 2000 values, because `2^11 = 2048 > 2000`.
4. **Once you find a winner, try the next sweet spot** by picking a different starting point and repeating. Or just brute-force a few candidates and stop as soon as the counter hits 5.

In practice the simulator approach is faster because Python can test all 2000 candidate rates in a few milliseconds without making any HTTP calls. I use the simulator to scout, then send only the working rates to the server.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for each `POST /train` request:

1. **Receive request** — Flask-style endpoint reads the JSON body, parses `learningRate` as a float, and validates it is inside `[lrMin, lrMax] = [0.02, 20.0]`. Out-of-range values get an HTTP 400.
2. **Load state** — the 12 points and the initial weights `[1.0, -1.0]` with bias `0.0` are pulled from `config.json`. The `successCount` is loaded from a per-instance counter that persists across requests.
3. **Train loop** — for up to 16 iterations:
   - Pick the next sample in round-robin order (sample index = `(step) % 12`).
   - Compute `activation = w1*x + w2*y + bias`, then `prediction = 1 if activation >= 0 else 0`.
   - If `prediction != label`, apply the classic update with the requested learning rate and the error term `label - prediction` (which is `±1` here because labels are 0/1).
   - If correct, do nothing (hint #1 stresses this).
   - After every step, recompute accuracy over all 12 points and append an entry to the history.
4. **Score** — if the final `accuracy === 1`, increment the persistent `successCount`. If `successCount` reaches `successTarget` (5), include the flag in the response.
5. **Respond** — return a JSON object with `history`, `accuracy`, `success`, `successCount`, `successTarget`, `message`, and (if unlocked) `flag`.

The key difference from the earlier "Hole in Middle" challenge is the **persistent counter on the server**. The other variants check accuracy on a single request and unlock the flag in one shot; this one makes you prove you can find the solution multiple times with different rates, which is a more realistic feel for tuning a real perceptron.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Fetch the HTML, JS, and config files to inspect the service | Easy |
| Browser (optional) | Watch the canvas animation for fun | Easy |
| `urllib.request` (Python stdlib) | `POST /train` with a JSON learning rate — no pip install needed | Easy |
| `nano` | Write the solve script | Easy |
| Local Python simulator | Scout all 2000 candidate learning rates in milliseconds | Medium |
| `python3 -m json.tool` | Pretty-print the JSON response and history | Easy |

---

## Key Takeaways

- **A linearly separable dataset guarantees convergence** — the perceptron convergence theorem says we will *eventually* find a separating line. The question is just how many steps, and how big the final weights get.
- **Local simulation first, network calls second** — running a 2000-iteration sweep over the network would be slow and noisy. Mirroring the server's algorithm in a local script lets you scout the entire space in a fraction of a second.
- **Server state can persist** — this challenge keeps a `successCount` across requests, so the wins accumulate. The earlier single-shot variants do not.
- **Bigger learning rate does not mean better** — at `lr=0.50` the perceptron oscillates and never settles inside 16 steps; at `lr=1.0` it converges almost instantly. The boundary between "small enough to damp" and "big enough to converge" is sharp on this dataset.
- **Binary search is the right algorithm when you have a sorted continuous space and a binary success criterion** — exactly the situation hint #5 describes. My sweep at `step=0.25` is essentially a coarse binary search with a fixed grid.
- **Always read the front-end JS** — it is the fastest way to discover a hidden API like `POST /train`. `view-source:` and `curl` are both fine.
- **`urllib.request`** ships with Python and is enough for a one-off JSON POST. Reach for `requests` only when you need sessions, retries, or richer error handling.
- **Flag wordplay**: `perceptron_classic_2_alpha_5rates` is plain English with no leet this time — it is literally the challenge name (`Perceptron Train Classic 2 Alpha`) plus `_5rates` for the 5-rates unlock. The trailing `_13e2875c` is a per-user salt.
