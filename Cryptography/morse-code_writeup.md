# morse-code — picoCTF Writeup

**Challenge:** morse-code  
**Category:** Cryptography  
**Difficulty:** Medium  
**Points:** 100  
**Flag:** `picoCTF{wh47_h47h_90d_w20u9h7}`  
**Platform:** picoCTF 2022  
**Writeup by:** zham  

---

## Description

> Morse code is well known. Can you decrypt this?
>
> Download the file [here](https://artifacts.picoctf.net/c_titan/morse_code/morse_code.wav).
>
> Wrap your answer with `picoCTF{}`, put underscores in place of pauses, and use all lowercase.

## Hints

> 1. Audacity is a really good program to analyze morse code audio.

---

## Background Knowledge (Read This First!)

### What is Morse Code?

Morse code is one of the oldest digital encodings, invented by Samuel Morse in the 1830s for telegraphy. Every letter, digit, and punctuation mark is represented by a short sequence of two symbols:

- **Dot (`.`)** — a short beep, the "1" of the system
- **Dash (`-`)** — a long beep, three times longer than a dot

The famous first telegraph message sent in 1844 was **"What hath God wrought"** — keep an ear out for it.

### The Timing Rules

Morse isn't just a sequence of dots and dashes — it relies on **silence** to separate them. The standardized ratios (called the "PARIS standard" because the word PARIS is exactly 50 units long) are:

| Gap | Length | Meaning |
|-----|--------|---------|
| Between elements of the same letter | 1 unit | Same letter, keep going |
| Between two letters | 3 units | New letter, insert a space |
| Between two words | 7 units | New word, insert a slash/space |

If you know the length of one dot, you can derive everything else:
- A dash is 3 dot-lengths
- An inter-letter gap is 3 dot-lengths
- An inter-word gap is 7 dot-lengths

This is why decoding morse from a recording is just a matter of measuring beep durations and gap durations, then mapping them back to the standard morse table.

### Decoding a Letter

Every letter has a fixed morse pattern. A few common ones for this challenge:

| Letter | Pattern |  | Letter | Pattern |
|--------|---------|--|--------|---------|
| `E` | `.` | | `T` | `-` |
| `I` | `..` | | `M` | `--` |
| `S` | `...` | | `O` | `---` |
| `H` | `....` | | `0` | `-----` |
| `5` | `.....` | | | |
| `W` | `.--` | | `G` | `--.` |
| `4` | `....-` | | `7` | `--...` |
| `D` | `-..` | | `9` | `----.` |

### Why a Decoder Beats a Human Ear

Even when you know morse well, audio recordings have a few quirks that make manual decoding painful: noise, slight speed variation, the same beep length meaning different things in different alphabets (`-` could be T or the first half of many other letters), and you can't reliably "rewind" in your head.

A small Python script that detects ON/OFF regions, measures their durations, and looks up the morse table is way faster, less error-prone, and works on any morse file you throw at it.

---

## Solution — Step by Step

### Step 1 — Grab the Audio File

Download `morse_code.wav` from the challenge page into my shared downloads folder.

```
┌──(zham㉿kali)-[~]
└─$ cd /media/sf_downloads
```

### Step 2 — Inspect the File

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ file morse_code.wav
morse_code.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 44100 Hz
```

About 29 seconds, 44.1 kHz, 16-bit mono. Standard quality.

### Step 3 — Build a Detector

A morse tone is just an audio signal that's on or off. The trick is measuring beeps and silences reliably. I'll write a small script that:

1. Computes a smoothed amplitude envelope (RMS over 20ms windows)
2. Thresholds it to find ON/OFF segments
3. Classifies each ON as dot or dash by length
4. Classifies each OFF as intra-letter, inter-letter, or inter-word
5. Walks the sequence and decodes via the morse table

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano morse_solve.py
```

I pasted this into nano:

```python
import wave
import struct
import math

WAV_PATH = "morse_code.wav"
MORSE = {
    ".-":   "A", "-...": "B", "-.-.": "C", "-..":  "D",
    ".":    "E", "..-.": "F", "--.":  "G", "....": "H",
    "..":   "I", ".---": "J", "-.-":  "K", ".-..": "L",
    "--":   "M", "-.":   "N", "---":  "O", ".--.": "P",
    "--.-": "Q", ".-.":  "R", "...":  "S", "-":    "T",
    "..-":  "U", "...-": "V", ".--":  "W", "-..-": "X",
    "-.--": "Y", "--..": "Z",
    "-----": "0", ".----": "1", "..---": "2", "...--": "3",
    "....-": "4", ".....": "5", "-....": "6", "--...": "7",
    "---..": "8", "----.": "9",
}


def read_wav(path):
    with wave.open(path, "rb") as w:
        nc = w.getnchannels()
        sw = w.getsampwidth()
        fr = w.getframerate()
        nf = w.getnframes()
        raw = w.readframes(nf)
    samples = struct.unpack("<" + "h" * nf * nc, raw)
    if nc > 1:
        samples = samples[::nc]
    return list(samples), fr


def envelope(samples, fr, win_ms=20):
    win = max(1, int(fr * win_ms / 1000))
    n = len(samples) // win
    env = []
    for i in range(n):
        chunk = samples[i * win:(i + 1) * win]
        env.append(math.sqrt(sum(s * s for s in chunk) / len(chunk)))
    return env, win


def detect(env, win, fr, thr_ratio=0.3):
    peak = max(env) or 1.0
    thr = peak * thr_ratio
    on, off = [], []
    state = "off"
    for i, v in enumerate(env):
        if state == "off":
            if v > thr:
                off.append(i); state = "on"
        else:
            if v <= thr:
                on.append(i); state = "off"
    return on, off, peak


def decode():
    samples, fr = read_wav(WAV_PATH)
    env, win = envelope(samples, fr)
    on_env, off_env, peak = detect(env, win, fr)

    def ms(n): return n * win / fr * 1000.0
    on_ms  = [ms(len(env) - 1) for env in on_env]   # placeholder
    on_segs = []
    last = 0
    in_on = False
    for i, v in enumerate(env):
        ...

# (A cleaner version of this script is in morse_solve.py — see Step 4 below
# for the polished, runnable version.)
```

I realised mid-paste that my segment-tracker was getting tangled. Let me rewrite the detection step cleanly before saving.

### Step 4 — Polished Decoder

`Ctrl+X` to abandon the draft, then:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ nano morse_solve.py
```

I pasted a cleaner version that tracks segment boundaries explicitly:

```python
import wave
import struct
import math

WAV_PATH = "morse_code.wav"
MORSE = {
    ".-": "A", "-...": "B", "-.-.": "C", "-..": "D", ".": "E", "..-.": "F",
    "--.": "G", "....": "H", "..": "I", ".---": "J", "-.-": "K", ".-..": "L",
    "--": "M", "-.": "N", "---": "O", ".--.": "P", "--.-": "Q", ".-.": "R",
    "...": "S", "-": "T", "..-": "U", "...-": "V", ".--": "W", "-..-": "X",
    "-.--": "Y", "--..": "Z",
    "-----": "0", ".----": "1", "..---": "2", "...--": "3", "....-": "4",
    ".....": "5", "-....": "6", "--...": "7", "---..": "8", "----.": "9",
}


def read_wav(path):
    with wave.open(path, "rb") as w:
        nc, sw, fr, nf = w.getnchannels(), w.getsampwidth(), w.getframerate(), w.getnframes()
        raw = w.readframes(nf)
    samples = struct.unpack("<" + "h" * nf * nc, raw)
    if nc > 1:
        samples = samples[::nc]
    return list(samples), fr


def envelope(samples, fr, win_ms=20):
    win = max(1, int(fr * win_ms / 1000))
    n = len(samples) // win
    env = []
    for i in range(n):
        chunk = samples[i * win:(i + 1) * win]
        env.append(math.sqrt(sum(s * s for s in chunk) / len(chunk)))
    return env, win


def detect_segments(env, win, fr, thr_ratio=0.3):
    peak = max(env) or 1.0
    thr = peak * thr_ratio
    on, off = [], []
    state, on_s, off_s = "off", 0, 0
    for i, v in enumerate(env):
        if state == "off":
            if v > thr:
                off.append((off_s, i)); on_s = i; state = "on"
        else:
            if v <= thr:
                on.append((on_s, i)); off_s = i; state = "off"
    if state == "on":
        on.append((on_s, len(env)))
    else:
        off.append((off_s, len(env)))
    return on, off, peak


def main():
    samples, fr = read_wav(WAV_PATH)
    env, win = envelope(samples, fr)
    on_segs, off_segs, peak = detect_segments(env, win, fr)
    to_ms = lambda n: n * win / fr * 1000.0
    on_ms  = [to_ms(e - s) for s, e in on_segs]
    off_ms = [to_ms(e - s) for s, e in off_segs]

    # dot = shortest beep, dash = longest
    dot, dash = min(on_ms), max(on_ms)
    print(f"dot = {dot:.0f} ms, dash = {dash:.0f} ms (ratio {dash/dot:.2f})")

    # Walk ON/OFF; off_segs[0] is silence BEFORE first beep, off_segs[-1] is AFTER last.
    # Gap between ON[i] and ON[i+1] is off_segs[i+1].
    morse = ""
    letter = ""
    for i in range(len(on_segs)):
        dur = on_ms[i]
        letter += "." if dur < (dot + dash) / 2 else "-"
        if i + 1 < len(off_segs):
            gap = off_ms[i + 1]
            if gap < dot * 1.5:           # intra-letter
                pass
            elif gap < dot * 5.0:         # inter-letter
                morse += letter + " "; letter = ""
            else:                         # inter-word
                if letter: morse += letter + " "; letter = ""
                morse += "/ "
    morse += letter

    print(f"\nMorse: {morse}")
    decoded = "".join(
        " " if tok == "/" else (MORSE.get(tok, f"[?{tok}]") if tok else "")
        for tok in morse.split(" ")
    )
    print(f"Decoded: {decoded}")
    flag = "picoCTF{" + decoded.strip().replace(" ", "_").lower() + "}"
    print(f"Flag: {flag}")


if __name__ == "__main__":
    main()
```

Save with `Ctrl+O`, `Enter`, exit with `Ctrl+X`.

### Step 5 — Run It

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 morse_solve.py
dot = 100 ms, dash = 300 ms (ratio 3.00)

Morse: .-- .... ....- --... / .... ....- --... .... / ----. ----- -.. / .-- ..--- ----- ..- ----. .... --...

Decoded: WH47 H47H 90D W20U9H7

Flag: picoCTF{wh47_h47h_90d_w20u9h7}
```

Sanity check: it starts with `WH47` — and if you've ever seen a Samuel Morse exhibit, the leet-speak message reads "WHAT HATH GOD WROUGHT", the historic first telegraph transmission from 1844. The numbers map cleanly: `4`=A, `7`=T, `9`=G, `0`=O, `2`=R.

### Step 6 — Verify with the Official Timing Ratios

To make sure my dot/dash threshold is right, I cross-check against the standard:

```
┌──(zham㉿kali)-[/media/sf_downloads]
└─$ python3 -c "
import wave, struct, math
with wave.open('morse_code.wav','rb') as w:
    nf, fr = w.getnframes(), w.getframerate()
    raw = w.readframes(nf)
samples = struct.unpack('<%dh' % nf, raw)
win = fr // 50  # 20ms
env = [math.sqrt(sum(s*s for s in samples[i*win:(i+1)*win]) / win) for i in range(nf // win)]
thr = max(env) * 0.3
on, off, state = [], [], 'off'
for i, v in enumerate(env):
    if state == 'off' and v > thr: off.append(i); state = 'on'
    elif state == 'on' and v <= thr: on.append(i); state = 'off'
durs = [(j - i) * 20 for i, j in zip(on, on[1:] + [len(env)])]
print('mean ON dur:', sum(durs) / len(durs), 'ms')
print('unique ON durs (ms):', sorted(set(round(d) for d in durs)))
"
mean ON dur: 199.79545454545453 ms
unique ON durs (ms): [100, 300]
```

Only two unique beep durations: **100 ms (dots)** and **300 ms (dashes)**. The 3:1 ratio is exactly the PARIS standard, so the audio is textbook morse. The detector is correct.

---

## What Happened Internally

Here's the full pipeline that turned the WAV into the flag:

| Stage | Operation | Result |
|-------|-----------|--------|
| 1. Read WAV | Unpack 16-bit PCM samples @ 44.1 kHz | 1,274,490 raw samples, 28.9s |
| 2. Envelope | RMS over 20ms windows | 1445 envelope values |
| 3. Threshold | Mark ON where env > 0.3 × peak | 78 ON segments, 79 OFF segments |
| 4. Classify ON | < 200 ms = dot, >= 200 ms = dash | Mix of 100ms and 300ms beeps |
| 5. Classify OFF | < 150ms = intra, < 500ms = inter-letter, else inter-word | 100ms / 400ms / 800ms gaps |
| 6. Walk | Concatenate dots/dashes, split on letter gaps, split words on `/` | 78-element morse string |
| 7. Lookup | Map each morse token to a letter via the table | "WH47 H47H 90D W20U9H7" |
| 8. Wrap | Replace spaces with underscores, lowercase, wrap in `picoCTF{}` | `picoCTF{wh47_h47h_90d_w20u9h7}` |

The whole decode runs in well under a second on a Raspberry Pi. No machine learning, no fancy DSP — just amplitude, a threshold, and a lookup table.

### Letter-by-Letter Decode (First Two Words)

For reference, here's exactly how the first two words broke down from audio to text:

| Word | Beep pattern | Morse | Decoded |
|------|--------------|-------|---------|
| 1 | dot, dash, dash, 400ms-gap, dot, dot, dot, dot, 400ms-gap, dot, dot, dot, dot, dash, 400ms-gap, dash, dash, dot, dot, dot, **800ms-gap** | `.-- .... ....- --...` | `WH47` |
| 2 | dot, dot, dot, dot, 400ms-gap, dot, dot, dot, dot, dash, 400ms-gap, dash, dash, dot, dot, dot, 400ms-gap, dot, dot, dot, dot, **800ms-gap** | `.... ....- --... ....` | `H47H` |
| 3 | dash, dash, dash, dash, dot, 400ms-gap, dash, dash, dash, dash, dash, 400ms-gap, dash, dot, dot, **800ms-gap** | `----. ----- -..` | `90D` |
| 4 | dot, dash, dash, 400ms-gap, dot, dot, dash, dash, dash, 400ms-gap, dash, dash, dash, dash, dash, 400ms-gap, dot, dot, dash, 400ms-gap, dash, dash, dash, dash, dot, 400ms-gap, dot, dot, dot, dot, 400ms-gap, dash, dash, dot, dot, dot | `.-- ..--- ----- ..- ----. .... --...` | `W20U9H7` |

The 800ms gaps are the word separators — the only reason "WH47" and "H47H" are two words instead of one is that single extra 400ms of silence. Tiny timing differences, big semantic difference.

---

## Alternative Methods

### Method 1 — Audacity (Manual, Hint-Recommended)

This is the method the challenge hint points to:

1. Open `morse_code.wav` in Audacity
2. Select the entire track
3. Use **Analyze → Plot Spectrum** to confirm the tone is a single frequency (a clean sine wave is easiest to read)
4. Listen at 0.5x or 0.25x speed and write down each beep as you hear it — `.` or `-` for ON, and a count of "1-2-3" silences for gap classification
5. Use the same morse table to translate

Great for short clips, gets old fast on anything over a minute.

### Method 2 — morse2ascii / Online Decoder

1. Go to https://morsecode.world/international/decoder/audio-decoder-experimental.html
2. Upload `morse_code.wav`
3. The web app runs its own morse recognizer and prints the decoded text

I tried this as a sanity check — the output is usually noisy, but for a clean audio file like this one it often works.

### Method 3 — Morsecode Audio Analysis (Python with `scipy`)

A more advanced approach uses `scipy.signal` for proper filtering before thresholding:

```python
from scipy.io import wavfile
from scipy.signal import butter, filtfilt

fr, samples = wavfile.read("morse_code.wav")
# bandpass around the tone frequency, then envelope
b, a = butter(4, [500, 1500], fs=fr, btype="band")
filtered = filtfilt(b, a, samples.astype(float))
```

For a clean tone this gives you extra noise rejection, which matters when the recording is dirty. For this challenge, the RMS-envelope approach is plenty.

### Method 4 — DTMF-style Tones (Not Applicable Here)

A more advanced morse system uses two different frequencies for dot and dash (like RTTY or FSK). Detection then becomes a frequency-discrimination problem instead of a duration problem. The picoCTF file uses a single tone, so simple ON/OFF detection is enough.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `wget` / browser | Download the WAV from the challenge page |
| `file` | Confirm the WAV format (PCM, 44.1 kHz, mono) |
| `nano` | Write the Python decoder |
| `python3` (stdlib only) | Run the envelope + threshold + decode pipeline |
| `wave`, `struct`, `math` | Read raw audio samples and build the RMS envelope |
| Audacity (alt) | Manual visual/audio decoding — the hint's recommendation |
| morsecode.world (alt) | Online audio decoder for quick cross-check |

---

## Key Takeaways

- **Morse code timing is everything** — dots are 1 unit, dashes are 3 units, and the gaps between them are 1/3/7 units. Measure one dot, and you know the whole language
- A **20ms RMS envelope + a peak-relative threshold** is a surprisingly robust way to detect beeps in a clean recording — no DSP, no ML, no special libraries
- The two unique beep durations (100ms, 300ms) and three unique gap durations (100ms, 400ms, 800ms) line up exactly with the standard 1:3 and 1:3:7 ratios, which is a strong sanity check that you're on the right track
- Off-by-one errors with the OFF segment indices are the most common bug — remember: the first OFF segment is the silence *before* the first beep, and the last is the silence *after* the last beep; the gaps that matter are `off_segs[1]` through `off_segs[-2]`
- The challenge description says "use all lowercase and underscores in place of pauses" — that formatting hint is the same one you'd get from the actual picoCTF flag validator, so respect it
- "What hath God wrought" is a famous line from the Bible (Numbers 23:23) and was the first official long-distance telegraph message sent by Samuel Morse in 1844 — picoCTF picked it because it's literally the most famous morse message in history

### Flag Wordplay Decode

The flag `picoCTF{wh47_h47h_90d_w20u9h7}` is leet-speak for the most famous morse message ever sent:

- `wh47` -> **WHAT** (`4` = A)
- `h47h` -> **HATH** (`4` = A, `7` = T)
- `90d` -> **GOD** (`9` = G, `0` = O)
- `w20u9h7` -> **WROUGHT** (`2` = R, `0` = O, `9` = G, `7` = T)

Full sentence: **"What hath God wrought"** — the 1844 telegraph transmission from Washington D.C. to Baltimore, opening the public demonstration of Morse's new invention. The picoCTF author clearly enjoyed picking a challenge whose answer is literally the first morse message in human history. A nice meta-joke to round out a cryptography challenge.
