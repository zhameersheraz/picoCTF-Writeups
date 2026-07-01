# Perceptron Train Classic 0 — picoCTF Writeup

**Challenge:** Perceptron Train Classic 0  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{perceptron_classic_mode_4ef55864}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Watch a perceptron learn in real time using the classic update rule: only misclassified points trigger updates, with no weight decay. This dataset is intentionally gentle, so many learning rates still converge quickly. Dial in a rate, trigger a run, and the interface will animate each step until every point is classified correctly. Visit the service in your browser: `http://aureolin-pixie.cylabacademy.net:55388/`

> Note: the URL host (`cylabacademy.net`) and the flag prefix (`academy{...}`) tell us this is hosted by CyLab CTF Academy, the picoCTF-adjacent platform. The series (Classic 0, 1, 2 Alpha) is from that platform.

## Hints

> 1. You get 16 updates per run, and correctly classified points do not change the model in the classic rule.
> 2. Overshooting is harder in this dataset because the clusters are well separated, but you can still watch the line swing too far if the rate is very high.
> 3. Use the slider to sweep the range and watch how the decision boundary moves between misclassification updates; there are many sweet spots that separate all points.
> 4. Binary search is a great strategy for finding a good learning rate. Try "this problem" to learn how to do a binary search.

---

## Background Knowledge (Read This First!)

If you have already read my `perceptron_train_classic_1_writeup.md`, you can skip to the Solution section — this challenge is the same perceptron, same API, simpler unlock. The Background section is here for newcomers who want a self-contained read.

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
4. If the prediction was **right**, do nothing — hint #1 calls this out explicitly.

The `learning_rate` is just a knob that controls how big each nudge is. Tiny values make tiny careful corrections; huge values make huge swings that can flip the boundary across the plane.

### What is the "perceptron convergence theorem"?

For any dataset that is **linearly separable** (i.e. there exists at least one straight line that splits the classes perfectly), the perceptron is *guaranteed* to find such a line in a finite number of steps. This was proven by Rosenblatt in 1960. The catch is that the theorem does not say *how many* steps it will take, and it does not say *how big* the final weights will be — only that they will eventually classify every training point correctly.

That guarantee is exactly why this challenge works. The 8 points in this dataset sit in two clean diagonal clusters with a huge gap between them, so the perceptron always finds a separating line — fast.

### What is a "well separated" cluster?

Imagine two blobs of points on the plane, one in the lower-left and one in the upper-right, with a wide empty diagonal strip between them. "Well separated" means the gap is large compared to the spread of each cluster — a small wiggle of the decision line is enough to slot it into the gap. Compare this to the "Classic 2 Alpha" challenge, where the 12 points were tightly packed and the gap was razor-thin; in that one, a wrong `lr` could make the line bounce around the narrow gap and never settle. Here the gap is wide, so almost any `lr` works — exactly what hint #2 is telling us.

### Why is overshooting "harder" on a well-separated dataset?

Overshooting happens when a big learning rate pushes the boundary past the separating line, so it lands on the wrong side of the cluster. On a thin gap, a small overshoot is enough to land inside the cluster. On a wide gap, the boundary has to overshoot a *lot* before it hits a cluster, and with only 16 update steps that rarely happens. Hint #2 is the author confirming this: even at `lr=2.0` (the maximum slider value here), the perceptron still converges in 16 steps on this dataset.

### Why does this challenge unlock in a single run?

Unlike Classic 1 (which demands 5 separate successful runs) and Classic 2 Alpha (which demands several runs on a finicky dataset), Classic 0 is the warm-up: one `POST /train` request that returns `accuracy === 1` is enough to get the flag in the response. There is no persistent counter on the server for this one. The flag is included in the JSON payload on the very first successful call.

### What is binary search and why is hint #4 telling us about it?

Binary search is the divide-and-conquer trick: if you have a sorted range and want to find a value, cut the range in half, check the midpoint, then keep the half that is more promising. In this context, "more promising" means "produced a higher final accuracy". The hint is saying: instead of randomly sweeping the slider, you can pick two extreme learning rates, find the midpoint, test it, and iteratively zoom in on a sweet spot.

On a *well-separated* dataset like this one, binary search is overkill — almost any value works, so a coarse sweep converges in the very first try. On a *finicky* dataset like Classic 2 Alpha, binary search actually pays off because there is a thin band of working rates. The hint applies more strongly to that one, but the author included it here so the technique generalizes across the series.

---

## Solution — Step by Step

### Step 1 — Probe the service

First, open the page in a browser or fetch the HTML to see what we are dealing with:

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://aureolin-pixie.cylabacademy.net:55388/ | head -10
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <title>Perceptron Train Classic</title>
    ...
```

That HTML loads `app.js`, which I read to find the API: `POST /train` with `{"learningRate": <number>}`. It also loads `/config.json` which holds the dataset, the initial model, and the slider's `[min, max]` range.

### Step 2 — Grab the config to see the dataset

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://aureolin-pixie.cylabacademy.net:55388/config.json | python3 -m json.tool
{
    "points": [
        [-4, -2, 0],
        [-3, -4, 0],
        [-2, -3, 0],
        [-3, -1, 0],
        [ 3,  4, 1],
        [ 4,  2, 1],
        [ 2,  3, 1],
        [ 3,  1, 1]
    ],
    "maxSteps": 16,
    "lrMin": 0.02,
    "lrMax": 2.0,
    "initialModel": {
        "step": 0,
        "sampleIndex": -1,
        "weights": [1.0, -1.0],
        "bias": 0.0,
        "accuracy": 0.5
    }
}
```

The dataset has **8 points** in two clean diagonal clusters: 4 negatives in the lower-left (`(-4,-2)`, `(-3,-4)`, `(-2,-3)`, `(-3,-1)`) and 4 positives in the upper-right (`(3,4)`, `(4,2)`, `(2,3)`, `(3,1)`). The gap between the clusters is huge — a wide diagonal strip with no points at all. The initial line `w1*x + w2*y + b = 0`, i.e. `x − y = 0` (the line `y = x`), separates them poorly (only 4/8 = 50% accuracy). We just need to find *one* learning rate that pushes the model to 100%.

### Step 3 — Simulate locally to scout a good LR

Before hammering the server, I wrote a quick local simulator in Python to find which learning rates work. The trick is to reproduce the server's update rule exactly: iterate through the 8 points in order, log a step, and update weights only on misclassification.

```
┌──(zham㉿kali)-[~]
└─$ nano /tmp/sim.py
```

Paste:

```python
#!/usr/bin/env python3
"""Simulate the server-side perceptron locally to scout good learning rates."""
POINTS = [
    (-4, -2, 0), (-3, -4, 0), (-2, -3, 0), (-3, -1, 0),
    ( 3,  4, 1), ( 4,  2, 1), ( 2,  3, 1), ( 3,  1, 1),
]
INIT_W = (1.0, -1.0)
INIT_B = 0.0
MAX_STEPS = 16


def predict(w, b, x, y):
    return 1 if (w[0] * x + w[1] * y + b) >= 0 else 0


def run(lr):
    w = list(INIT_W)
    b = INIT_B
    for _ in range(MAX_STEPS):
        for x, y, label in POINTS:
            pred = predict(w, b, x, y)
            if pred != label:
                err = label - pred          # +1 or -1
                w[0] += lr * err * x
                w[1] += lr * err * y
                b   += lr * err
        acc = sum(1 for x, y, l in POINTS if predict(w, b, x, y) == l) / len(POINTS)
        if acc == 1.0:
            return True, w, b
    return False, w, b


good = []
lr = 0.02
while lr <= 2.0 + 1e-9:
    ok, w, b = run(round(lr, 2))
    if ok:
        good.append(round(lr, 2))
    lr += 0.01

print(f"Found {len(good)} learning rates that reach 100% in <=16 steps")
print("First 5:", good[:5])
print("Last 5: ", good[-5:])
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`, then run:

```
┌──(zham㉿kali)-[~]
└─$ python3 /tmp/sim.py
Found 199 learning rates that reach 100% in <=16 steps
First 5: [0.02, 0.03, 0.04, 0.05, 0.06]
Last 5:  [1.96, 1.97, 1.98, 1.99, 2.0]
```

**199 out of 199 candidate rates reach 100% accuracy.** This is what hint #2 meant by "overshooting is harder" — the gap is so wide that even `lr=2.0` (the maximum on the slider) still converges in 16 steps. Compare to the "Classic 2 Alpha" challenge where only 320 of 2000 worked.

### Step 4 — Pick a clean LR and send one request

We just need one winning call. I am picking `lr=0.06` because it converges in a single update on this dataset — the smallest and most "elegant" win possible:

```
┌──(zham㉿kali)-[~]
└─$ curl -s -X POST http://aureolin-pixie.cylabacademy.net:55388/train \
       -H 'Content-Type: application/json' \
       -d '{"learningRate": 0.06}' \
       | python3 -c "import json,sys; d=json.load(sys.stdin); \
           print('accuracy:', d['accuracy']); \
           print('success: ', d.get('success')); \
           print('flag:    ', d.get('flag'))"
accuracy: 1.0
success:  True
flag:     academy{perceptron_classic_mode_4ef55864}
```

One curl, one flag. Submit it on the picoCTF challenge page for the point.

### Step 5 — (Optional) Watch the canvas animate it

If you want to actually see the perceptron learn, paste this URL into a browser:

```
http://aureolin-pixie.cylabacademy.net:55388/
```

Set the slider to `0.06`, click **Run training**, and watch the green line snap from `y = x` to a separating line that sits in the wide diagonal gap after a single update. The training log will show one row with `error = -1` (the only misclassified point was `(-3, -4, 0)`) and `accuracy = 1.0` from step 2 onwards. The flag is rendered in the orange box at the bottom-right of the page.

---

## Alternative Method — One-liner with `printf` + `nc`

If you don't want to spin up a Python script, the same call works as a one-liner using `nc`:

```
┌──(zham㉿kali)-[~]
└─$ printf 'POST /train HTTP/1.1\r\nHost: aureolin-pixie.cylabacademy.net:55388\r\nContent-Type: application/json\r\nContent-Length: 24\r\n\r\n{"learningRate": 0.06}' \
       | nc aureolin-pixie.cylabacademy.net 55388 | tail -n 1
```

The last line of the response body is the JSON containing the flag. (If you prefer cleaner output, drop the `| tail` and grep for `flag`.)

## Alternative Method — Bash + `jq`

If you want pretty output and have `jq` installed, this is the most readable one-shot:

```
┌──(zham㉿kali)-[~]
└─$ curl -s -X POST http://aureolin-pixie.cylabacademy.net:55388/train \
       -H 'Content-Type: application/json' \
       -d '{"learningRate": 0.06}' | jq '{accuracy, success, flag}'
{
  "accuracy": 1,
  "success": true,
  "flag": "academy{perceptron_classic_mode_4ef55864}"
}
```

Both alternatives produce the same flag; pick whichever fits your tooling preferences.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for our `POST /train` request:

1. **Receive request** — Flask-style endpoint reads the JSON body, parses `learningRate` as a float, and validates it is inside `[lrMin, lrMax] = [0.02, 2.0]`. Out-of-range values get an HTTP 400.
2. **Load state** — the 8 points and the initial weights `[1.0, -1.0]` with bias `0.0` are pulled from `config.json`. There is **no persistent counter** for Classic 0 (unlike Classic 1, which counts successful runs toward a target of 5).
3. **Train loop** — for up to 16 iterations:
   - Pick the next sample in round-robin order (sample index = `(step) % 8`).
   - Compute `activation = w1*x + w2*y + bias`, then `prediction = 1 if activation >= 0 else 0`.
   - If `prediction != label`, apply the classic update with the requested learning rate and the error term `label - prediction` (which is `±1` here because labels are 0/1).
   - If correct, do nothing (hint #1 stresses this).
   - After every step, recompute accuracy over all 8 points and append an entry to the history.
4. **Score** — if the final `accuracy === 1`, mark the run as successful. For Classic 0, success on the first call is enough — the flag is included in the same response.
5. **Respond** — return a JSON object with `history`, `accuracy`, `success`, `message`, and (if unlocked) `flag`.

### Trace of our winning run at `lr=0.06`

Step-by-step from the server's `history` array (trimmed):

| Step | Sample | Predict | Error | w₁ | w₂ | b | Accuracy |
|------|--------|---------|-------|------|------|------|----------|
| 1 | (-4, -2, 0) | 0 | 0 | 1.00 | -1.00 | 0.00 | 50.0% |
| 2 | (-3, -4, 0) | 1 | **-1** | **1.18** | **-0.76** | **-0.06** | **100.0%** |
| 3 | (-2, -3, 0) | 0 | 0 | 1.18 | -0.76 | -0.06 | 100.0% |
| 4 | (-3, -1, 0) | 0 | 0 | 1.18 | -0.76 | -0.06 | 100.0% |
| 5 | (3, 4, 1)   | 1 | 0 | 1.18 | -0.76 | -0.06 | 100.0% |
| ... | ...        | ... | ... | ... | ... | ... | 100.0% |

Exactly **one update** — on point `(-3, -4, 0)` — was enough. After that single nudge, every one of the 8 points sits on the correct side of the boundary `1.18*x − 0.76*y − 0.06 = 0`.

### Compared to the rest of the series

- **vs. Classic 1** — that variant requires 5 successful runs against a wider slider range (`lrMax = 20.0`). Same dataset, same perceptron, but a persistent counter on the server turns one win into five wins.
- **vs. Classic 2 Alpha** — that variant uses a 12-point finicky dataset where only ~320 of 2000 candidate rates converge. The same single-call API, but the math is much less forgiving.
- **vs. Hole in Middle** — that variant is mathematically capped at 88.9% accuracy (the perceptron cannot draw a straight line through a hole in the middle), so it uses a different scoring rule.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Fetch the HTML, JS, and config files to inspect the service | Easy |
| `nc` | (Alternative) Send the raw HTTP request as a one-liner | Easy |
| `jq` | (Alternative) Pretty-print the JSON response | Easy |
| Browser (optional) | Watch the canvas animation for fun | Easy |
| `nano` | Write the local simulator | Easy |
| Local Python simulator | Scout all 199 candidate learning rates in milliseconds | Medium |
| `python3 -m json.tool` | Pretty-print the JSON response and history | Easy |

---

## Key Takeaways

- **A linearly separable dataset guarantees convergence** — the perceptron convergence theorem says we will *eventually* find a separating line. The question is just how many steps, and how big the final weights get.
- **Local simulation first, network calls second** — running a 199-iteration sweep over the network would be slow and noisy. Mirroring the server's algorithm in a local script lets you scout the entire space in a fraction of a second.
- **Dataset geometry dictates learning-rate sensitivity** — on the "Hole in Middle" dataset the math caps accuracy at 88.9%, on "Classic 2 Alpha" only 320 of 2000 rates converge, but on *this* well-separated dataset 199 of 199 rates converge. Same perceptron, same 16-step budget, wildly different difficulty.
- **Some challenges unlock in a single call** — Classic 0 has no persistent counter. Hit the right LR once, get the flag immediately. Classic 1's 5-rates target and Classic 2 Alpha's similar mechanic are *more* demanding.
- **Read the front-end JS** — `view-source:` and `curl` both reveal the hidden `/train` API in seconds. Don't blindly click in the GUI when you can script it.
- **`printf | nc` works for trivial JSON POSTs** — but only for tiny, clean payloads. The moment you need headers, retries, or streaming, switch to `curl` or Python.
- **Binary search is the right algorithm when you have a sorted continuous space and a binary success criterion** — exactly what hint #4 describes. On a finicky dataset this matters; on an easy one like this it is overkill, but the technique generalizes.
- **Flag wordplay**: `perceptron_classic_mode` is plain English again — it is literally the challenge name (`Perceptron Train Classic 0`) plus `_mode` for the "classic update mode" mentioned in the description. The trailing `_4ef55864` is a per-instance salt. No leet in this one.
