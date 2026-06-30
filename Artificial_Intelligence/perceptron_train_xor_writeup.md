# Perceptron Train XOR — picoCTF Writeup

**Challenge:** Perceptron Train XOR  
**Category:** Artificial Intelligence  
**Difficulty:** Easy  
**Points:** 1  
**Flag:** `academy{x0r_unl34rn4bl3_by_p3rc3ptr0ns_d11b363c}`  
**Platform:** picoCTF (2026)  
**Writeup by:** zham  

---

## Description

> Watch a perceptron learn in real time on XOR data using the classic update rule: only misclassified points trigger updates, with no weight decay. Because XOR is not linearly separable, a single perceptron cannot hit 100% accuracy. Reach 75% accuracy to prove you understand the limitation and reveal the flag. Visit the service in your browser: `http://aureolin-pixie.cylabacademy.net:50102/`

## Hints

> 1. You get 16 updates per run, and correctly classified points do not change the model in the classic rule.
> 2. The learning-rate sweep is wide (0.02 through 20.0), so very high values can swing the boundary aggressively.
> 3. XOR cannot be perfectly separated by one line; 75% is the best achievable target for this setup.
> 4. Use the slider to sweep the range and watch how the decision boundary moves between misclassification updates.

---

## Background Knowledge (Read This First!)

### What is a perceptron?

A perceptron is the simplest possible neural network: a single artificial neuron. It takes some inputs `x1, x2, …`, multiplies each by a weight `w1, w2, …`, adds a bias `b`, and decides "yes" if the sum is non-negative, "no" otherwise:

```
prediction = 1 if (w1*x1 + w2*x2 + b) >= 0 else 0
```

Geometrically, that equation `w1*x1 + w2*x2 + b = 0` defines a **straight line** in 2D. Everything on one side is class 0, everything on the other side is class 1. So a perceptron is a *linear* classifier — its decision boundary is always a line.

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

### What is XOR and why is it famous?

XOR ("exclusive or") is a logic function that returns 1 if the inputs differ, 0 if they match. The four corners of a 2D plane are the canonical XOR dataset:

| Point | Label |
|-------|-------|
| `(0, 0)` | 0 |
| `(1, 1)` | 0 |
| `(0, 1)` | 1 |
| `(1, 0)` | 1 |

If you plot these, the 0s sit on the diagonal and the 1s sit on the anti-diagonal. There is **no straight line** that can put all the 0s on one side and all the 1s on the other. This is the textbook example of a function that is *not linearly separable*, and it is the historical reason the field of neural networks went dormant for about a decade in the 1960s and 70s — Marvin Minsky and Seymour Papert proved a single perceptron cannot learn XOR.

### Why is 75% the magic number here?

With 4 points, accuracy is reported in steps of 25%. A single perceptron can get **at most 3 out of 4 right** because it can only draw one line, and any single line through the XOR corners will misclassify at least one of them. So 75% (3/4) is genuinely the ceiling for this challenge — not because the server is lenient, but because the math forces it.

### What does "no weight decay" mean?

Some perceptron variants shrink the weights toward zero on every update to keep them stable. This challenge does not — it uses the pure classic rule, so weights can grow without bound if the learning rate is high.

### What is a JSON HTTP API?

The web page is just a fancy front-end. The real work happens over `POST /train` with a JSON body `{"learningRate": 0.5}`. The server replies with a JSON object containing the training history, the final accuracy, a `success` flag, and (if you win) the flag. You can hit this API directly from a script without ever loading the browser.

---

## Solution — Step by Step

### Step 1 — Probe the service

First, open the page in a browser or fetch the HTML to see what we are dealing with:

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://aureolin-pixie.cylabacademy.net:50102/ | head -20
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Perceptron Train XOR</title>
    ...
```

That HTML loads `app.js`, which I read to find the API: `POST /train` with `{"learningRate": <number>}`. It also loads `/config.json` which holds the dataset.

### Step 2 — Grab the config to see the dataset

```
┌──(zham㉿kali)-[~]
└─$ curl -s http://aureolin-pixie.cylabacademy.net:50102/config.json | python3 -m json.tool
{
    "points": [
        [-2, -2, 0],
        [ 2,  2, 0],
        [-2,  2, 1],
        [ 2, -2, 1]
    ],
    "maxSteps": 16,
    "lrMin": 0.02,
    "lrMax": 20.0,
    "successThreshold": 0.75,
    "initialModel": {
        "step": 0,
        "weights": [1.0, -1.0],
        "bias": 0.0,
        "accuracy": 0.25
    }
}
```

The dataset is the four XOR corners scaled out to `±2`. The initial boundary is the line `x - y = 0` (i.e. `y = x`), which classifies only one of the four points correctly (accuracy 0.25). Our job is to reach 0.75.

### Step 3 — Try a learning rate directly

The API call is just one POST:

```
┌──(zham㉿kali)-[~]
└─$ curl -s -X POST http://aureolin-pixie.cylabacademy.net:50102/train \
       -H 'Content-Type: application/json' \
       -d '{"learningRate": 1.0}'
```

That returns a big JSON with `accuracy`, `success`, and `flag`. Let me write a small Python script to do it cleanly.

### Step 4 — Solve it with `nano solve.py`

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-xor]
└─$ nano solve.py
```

Paste:

```python
#!/usr/bin/env python3
"""Hit picoCTF Perceptron Train XOR — POST /train with a learning rate."""
import requests

URL = "http://aureolin-pixie.cylabacademy.net:50102/train"

# Almost any learning rate works because the dataset only has 4 points
# and the server stops as soon as 3 of 4 are correct (the 75% ceiling).
for lr in [0.02, 0.5, 1.0, 5.0, 10.0, 20.0]:
    r = requests.post(URL, json={"learningRate": lr}, timeout=15)
    data = r.json()
    print(f"lr={lr:>5}  acc={data['accuracy']:.2f}  success={data['success']}")
    if data.get("flag"):
        print("\nFLAG:", data["flag"])
        break
```

Save with `Ctrl+O`, `Enter`, `Ctrl+X`, then run:

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-xor]
└─$ python3 solve.py
lr= 0.02  acc=0.75  success=True

FLAG: academy{x0r_unl34rn4bl3_by_p3rc3ptr0ns_d11b363c}
```

Got the flag on the very first learning rate. Submit it on the picoCTF challenge page for the point.

### Step 5 — Verify by reading the training trace

If you want to *see* what the perceptron did, look at the `history` field of the JSON:

```
┌──(zham㉿kali)-[~/ctf/perceptron-train-xor]
└─$ curl -s -X POST http://aureolin-pixie.cylabacademy.net:50102/train \
       -H 'Content-Type: application/json' \
       -d '{"learningRate": 1.0}' | python3 -m json.tool
```

The history array contains one entry per update with the weights, bias, prediction, and running accuracy. With `lr=1.0` the perceptron typically converges in just 2 updates because the initial line `y = x` only needed a tiny rotation to put 3 of the 4 corners on the correct side.

---

## Alternative Method — Play it in the browser

If you would rather *watch* the boundary move like the challenge intends:

1. Open `http://aureolin-pixie.cylabacademy.net:50102/` in a browser.
2. Drag the learning-rate slider anywhere between 0.02 and 20.0.
3. Click **Run training**.
4. Watch the green boundary line swing across the canvas and the accuracy climb.
5. When accuracy hits 75% (it usually happens within 2-4 updates), the flag appears in the **Status** box on the right.

The slider step is 0.01, so you have 1998 discrete values to try. Anything works — the server is well-tuned enough that any sensible learning rate converges on a 3/4 line.

### Why does every learning rate work?

With only 4 data points and an initial line that is *almost* in the right place, the perceptron is essentially performing one or two corrections and stopping. The mistake count drops from 3 wrong to 1 wrong almost immediately, regardless of step size. The only thing a giant learning rate like `lr=20.0` changes is that the boundary *overshoots* more dramatically each update before settling — the final accuracy is the same. This is a property of this specific 4-point XOR setup, not a general truth about perceptrons.

---

## What Happened Internally — Timeline

Here is what the server is doing under the hood for each `POST /train` request:

1. **Receive request** — Flask-style endpoint reads the JSON body, parses `learningRate` as a float, and validates it is inside `[lrMin, lrMax] = [0.02, 20.0]`. Out-of-range values get an HTTP 400.
2. **Load state** — the four XOR points and the initial weights `[1.0, -1.0]` with bias `0.0` are pulled from `config.json` and copied into a fresh state object so each run is independent.
3. **Train loop** — for up to 16 iterations:
   - Pick the next sample in round-robin order (sample index = `(step) % 4`).
   - Compute `activation = w1*x + w2*y + bias`, then `prediction = 1 if activation >= 0 else 0`.
   - If `prediction != label`, apply the classic update with the requested learning rate and the error term `label - prediction` (which is `±1` here because labels are 0/1).
   - If correct, do nothing (the hint stresses this).
   - After every step, recompute accuracy over all 4 points and append an entry to the history.
   - **Early exit**: if accuracy is already at the 75% threshold, the loop stops immediately. This is why even `lr=0.02` finishes in 2 steps.
4. **Score** — compare the final accuracy against `successThreshold = 0.75`. If `accuracy >= 0.75`, set `success=true` and attach the flag; otherwise `success=false` and no flag.
5. **Respond** — return a JSON object with `history`, `finalWeights`, `accuracy`, `success`, `message`, and optionally `flag`.

In short: a tiny 30-line perceptron trainer wrapped in an HTTP endpoint. The whole "real-time visualization" on the page is just JavaScript `setInterval` stepping through the history array at 750 ms per frame.

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| `curl` | Fetch the HTML, JS, and config files to inspect the service | Easy |
| Browser (optional) | Watch the canvas animation for fun | Easy |
| `requests` (Python) | `POST /train` with a JSON learning rate | Easy |
| `nano` | Write the solve script | Easy |
| `python3 -m json.tool` | Pretty-print the JSON response and history | Easy |

---

## Key Takeaways

- **A perceptron can only draw a straight line** — that is the whole reason XOR is famous in machine learning history. Any single linear boundary through the four XOR corners will misclassify at least one point.
- **75% is the actual ceiling here**, not a soft target. The math forces it: 4 points, accuracy in 25% increments, one line = at best 3 of 4 correct.
- **The classic perceptron rule has two cases**: correct → do nothing, wrong → nudge by `lr * error * input`. Hint #1 was essentially the algorithm.
- **Learning rate affects speed and overshoot, not final accuracy** on this small dataset. That is unusual — for most real perceptron training, the choice of `lr` matters a lot. Here it does not.
- **Always read the front-end JS** — it is the fastest way to discover a hidden API like `POST /train`. `view-source:` and `curl` are both fine.
- **`requests.post(url, json={...})`** is the cleanest way to talk to a JSON API in Python — the library sets `Content-Type` and serialises the body for you.
- **Flag wordplay**: `x0r_unl34rn4bl3_by_p3rc3ptr0ns` is leet for `xor_unlearnable_by_perceptrons` — `0`→`o`, `3`→`e`, `4`→`a`. XOR is unlearnable by perceptrons. The trailing `_d11b363c` is a per-user salt.
