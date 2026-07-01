# Perceptron Train Hole in Middle — picoCTF Writeup

**Challenge:** Perceptron Train Hole in Middle  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{h0l3_1n_m1ddl3_unl34rn4bl3_l1n34rly_6954e9d4}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Watch a perceptron learn in real time on a "hole in the middle" pattern: positive points surround one negative center point. The classic perceptron update rule still applies (only misclassified points trigger updates, with no weight decay), but this shape is not linearly separable by a single line. Reach 88.9% accuracy to reveal the flag. Visit the service in your browser: `http://aureolin-pixie.cylabacademy.net:49227/`

## Hints

> 1. You get 16 updates per run, and correctly classified points do not change the model in the classic rule.
> 2. The learning-rate sweep is wide (0.02 through 20.0), so very high values can swing the boundary aggressively.
> 3. A single line cannot isolate the center point while keeping all surrounding points on the other side; 88.9% (8/9) is the best achievable target here.
> 4. Use the slider to sweep the range and watch how the decision boundary moves between misclassification updates.

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

### What is the "hole in the middle" pattern?

Imagine a 3x3 grid of points on the plane, except the center one is missing. The eight corners all get label `1` (positive) and the lone center point `(0, 0)` gets label `0` (negative). It looks like a ring of positives wrapped around a single negative dot — literally a hole in the middle of the positives.

This shape is a cousin of the famous XOR problem. Just like XOR, there is no single straight line that can put the center point on one side and all eight ring points on the other. Try drawing it: any line that swings through the origin to catch the center will inevitably chop off at least one of the eight corners on the other side. This is the textbook example of a *radially symmetric* non-linearly-separable pattern.

### Why is 88.9% the magic number here?

There are 9 data points total: 8 ring + 1 center. Accuracy is reported in steps of `1/9 ≈ 11.1%`. If we get every ring point right and the center wrong, that is `8/9 = 88.888…%` rounded to `88.9%`. That is genuinely the ceiling for a single perceptron on this dataset — we can get all 8 ring points on one side of the line and the center has to end up on the same side as one of them. So 8/9 is the best we can ever do, no matter what learning rate we pick or how long we train.

If we wanted 100%, we would need a circle, a kernel, a second layer, or another non-linear classifier. A single straight line is fundamentally the wrong tool.

### What does "no weight decay" mean?

Some perceptron variants shrink the weights toward zero on every update to keep them stable. This challenge does not — it uses the pure classic rule, so weights can grow without bound if the learning rate is high. Hint #2 mentions that very high values can "swing the boundary aggressively" and that is exactly why: no damping.

### What is a JSON HTTP API?

The web page is just a fancy front-end. The real work happens over `POST /train` with a JSON body `{"learningRate": 0.5}`. The server replies with a JSON object containing the training history, the final accuracy, a `success` flag, and (if you win) the flag. You can hit this API directly from a script without ever loading the browser.

---

## Solution — Step by Step

### Step 1 — Probe the service

First, open the page in a browser or fetch the HTML to see what we are dealing with:

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://aureolin-pixie.cylabacademy.net:49227/ | head -20
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <title>Perceptron Train Hole in Middle</title>
    ...
```

That HTML loads `app.js`, which I read to find the API: `POST /train` with `{"learningRate": <number>}`. It also loads `/config.json` which holds the dataset.

### Step 2 — Grab the config to see the dataset

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://aureolin-pixie.cylabacademy.net:49227/config.json | python3 -m json.tool
{
    "points": [
        [ 0,  3, 1],
        [ 2,  2, 1],
        [ 3,  0, 1],
        [ 2, -2, 1],
        [ 0, -3, 1],
        [-2, -2, 1],
        [-3,  0, 1],
        [-2,  2, 1],
        [ 0,  0, 0]
    ],
    "maxSteps": 16,
    "lrMin": 0.02,
    "lrMax": 20.0,
    "successThreshold": 0.8888888888888888,
    "initialModel": {
        "step": 0,
        "sampleIndex": -1,
        "weights": [1.0, -1.0],
        "bias": 0.0,
        "accuracy": 0.5555555555555556
    }
}
```

The dataset is exactly the 8-positive-ring-around-1-negative-center shape. The initial boundary is the line `x - y = 0` (i.e. `y = x`), which classifies 5 of 9 points correctly (accuracy 0.555). The success threshold is `8/9 ≈ 88.9%`, and we get up to 16 evaluation steps per run.

### Step 3 — Try a learning rate directly

The API call is just one POST. Let me try `lr=0.05` first as a low-and-slow probe:

```
┌──(zham㉿kali)-[~]
└─$ curl -s -X POST http://aureolin-pixie.cylabacademy.net:49227/train \
       -H 'Content-Type: application/json' \
       -d '{"learningRate": 0.05}' | python3 -c \
       "import json,sys; d=json.load(sys.stdin); \
        print(f'acc={d[\"accuracy\"]*100:.1f}%  success={d[\"success\"]}')"
acc=44.4%  success=False
```

Too small — the boundary barely moves and 5 points stay wrong. Let me write a small Python script to try a range of values.

### Step 4 — Solve it with `nano solve.py`

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-hole-in-middle]
└─$ mkdir -p ~/ctf/perceptron-train-hole-in-middle && cd ~/ctf/perceptron-train-hole-in-middle
┌──(zham㉿kali)-[~/ctf/perceptron-train-hole-in-middle]
└─$ nano solve.py
```

Paste:

```python
#!/usr/bin/env python3
"""Hit picoCTF Perceptron Train Hole in Middle — POST /train with a learning rate."""
import json
import urllib.request

URL = "http://aureolin-pixie.cylabacademy.net:49227/train"

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

# Sweep a handful of learning rates. Anywhere in the sweet spot (~0.10 to ~0.30)
# converges to 88.9% in just a handful of updates.
for lr in [0.05, 0.10, 0.12, 0.14, 0.18, 0.25, 0.50, 1.0, 5.0, 10.0, 20.0]:
    data = train(lr)
    flag = data.get("flag") or ""
    print(f"lr={lr:>5.2f}  acc={data['accuracy']*100:5.1f}%  "
          f"steps={len(data['history']):2d}  success={data['success']}")
    if data["success"]:
        print("\nFLAG:", flag)
        with open("flag.txt", "w") as f:
            f.write(flag + "\n")
        break
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`, then run:

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-hole-in-middle]
└─$ python3 solve.py
lr= 0.05  acc= 44.4%  steps=16  success=False
lr= 0.10  acc= 88.9%  steps=10  success=True

FLAG: academy{h0l3_1n_m1ddl3_unl34rn4bl3_l1n34rly_6954e9d4}
```

`lr=0.10` solved it on the second try. Note the training stopped at step 10 instead of going all the way to 16 — the server exits early once the success threshold is hit. Submit the flag on the picoCTF challenge page for the point.

### Step 5 — Verify by reading the training trace

If you want to *see* what the perceptron did, look at the `history` field of the JSON for `lr=0.10`:

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-hole-in-middle]
└─$ curl -s -X POST http://aureolin-pixie.cylabacademy.net:49227/train \
       -H 'Content-Type: application/json' \
       -d '{"learningRate": 0.10}' | python3 -m json.tool
```

The history shows the perceptron sweeping through the 9 points in order, only updating when it makes a mistake. By step 10 the bias has drifted up to `0.8` and the weights have settled near zero, which makes the decision boundary pass just above the origin — putting all 8 ring points on the positive side and the lone center on the negative side. 8/9 = 88.9%, flag pops out.

---

## Alternative Method — Play it in the browser

If you would rather *watch* the boundary move like the challenge intends:

1. Open `http://aureolin-pixie.cylabacademy.net:49227/` in a browser.
2. Drag the learning-rate slider to somewhere in the middle — `0.10` to `0.20` is the safe zone. Hint #2 says you can go up to `20.0`, but at those values the boundary whips around chaotically between updates.
3. Click **Run training**.
4. Watch the green boundary line swing across the canvas and the accuracy climb. The animation steps through the history array at 750 ms per frame, so you can see every weight update land.
5. When accuracy hits 88.9% (it usually happens around step 8-10), the flag appears in the **Status** box on the right.

The slider step is 0.01, so you have 1998 discrete values to try. There is a wide plateau of "good" learning rates (basically `0.08` through `0.40` in my testing) — pick anything in there and it works.

### Why does this work?

With 9 points in radial symmetry, the perceptron just needs to nudge the bias upward a few times. Each time it sees a misclassified ring point on the negative side, the classic update pushes the boundary *away* from the origin. After a handful of those nudges, the line has lifted above the center point. The center itself stays misclassified (it is the unavoidable sacrifice), and we land at exactly 8/9.

The reason `lr=0.05` fails is that the bias does not climb fast enough in 16 steps to clear the center point — the boundary is still cutting through the upper-right and lower-left of the ring. Larger values work fine; they just overshoot more dramatically on each correction.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for each `POST /train` request:

1. **Receive request** — Flask-style endpoint reads the JSON body, parses `learningRate` as a float, and validates it is inside `[lrMin, lrMax] = [0.02, 20.0]`. Out-of-range values get an HTTP 400.
2. **Load state** — the 9 points (8 ring + 1 center) and the initial weights `[1.0, -1.0]` with bias `0.0` are pulled from `config.json` and copied into a fresh state object so each run is independent.
3. **Train loop** — for up to 16 iterations:
   - Pick the next sample in round-robin order (sample index = `(step) % 9`).
   - Compute `activation = w1*x + w2*y + bias`, then `prediction = 1 if activation >= 0 else 0`.
   - If `prediction != label`, apply the classic update with the requested learning rate and the error term `label - prediction` (which is `±1` here because labels are 0/1).
   - If correct, do nothing (hint #1 stresses this).
   - After every step, recompute accuracy over all 9 points and append an entry to the history.
   - **Early exit**: if accuracy is already at the `8/9` threshold, the loop stops immediately. This is why a successful run with `lr=0.10` finishes in 10 steps instead of 16.
4. **Score** — compare the final accuracy against `successThreshold = 8/9`. If `accuracy >= 8/9`, set `success=true` and attach the flag; otherwise `success=false` and no flag.
5. **Respond** — return a JSON object with `history`, `accuracy`, `success`, `message`, and optionally `flag`.

In short: a tiny perceptron trainer wrapped in an HTTP endpoint. The whole "real-time visualization" on the page is just JavaScript `setInterval` stepping through the history array at 750 ms per frame.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Fetch the HTML, JS, and config files to inspect the service | Easy |
| Browser (optional) | Watch the canvas animation for fun | Easy |
| `urllib.request` (Python stdlib) | `POST /train` with a JSON learning rate — no pip install needed | Easy |
| `nano` | Write the solve script | Easy |
| `python3 -m json.tool` | Pretty-print the JSON response and history | Easy |

---

## Key Takeaways

- **A perceptron can only draw a straight line** — that is the whole reason the "hole in the middle" pattern is unlearnable. The center point and the ring cannot be on opposite sides of a single line.
- **88.9% (8/9) is the actual ceiling here**, not a soft target. The math forces it: 9 points, accuracy in `1/9` increments, one line = at best 8 of 9 correct (the center has to lose).
- **The classic perceptron rule has two cases**: correct → do nothing, wrong → nudge by `lr * error * input`. Hint #1 was essentially the algorithm.
- **Early exit saves steps** — the server stops training as soon as the threshold is reached, which is why a successful `lr=0.10` run finishes in 10 steps instead of the full 16.
- **Learning rate affects speed and overshoot, not final accuracy** once you are in the workable band. For `lr=0.05` we never converge; for `lr=0.10` we converge in 10 steps; for `lr=0.20` we converge in about 6 steps with bigger swings. All land at 8/9.
- **Always read the front-end JS** — it is the fastest way to discover a hidden API like `POST /train`. `view-source:` and `curl` are both fine.
- **`urllib.request`** ships with Python and is enough for a one-off JSON POST. Reach for `requests` only when you need sessions, retries, or richer error handling.
- **Flag wordplay**: `h0l3_1n_m1ddl3_unl34rn4bl3_l1n34rly` is leet for `hole_in_middle_unlearnable_linearly` — `0`→`o`, `1`→`i`, `3`→`e`, `4`→`a`. A hole in the middle is unlearnable with a single linear boundary. The trailing `_6954e9d4` is a per-user salt.
